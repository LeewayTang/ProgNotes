
> 主要关注`zorro_android_sdk/src/third_party/webrtc`

# Zorro SDK 如何调用 WebRTC

主要关注`zorro_android_sdk/src/third_party/webrtc/zorro/`

## zorro::CallManager

全局单例，管理几乎所有的通话信息。整个通话过程中的控制流都在CallManager手上，包括媒体流、房间信息等。

CallManager的多线程用的是WebRTC的多线程机制。可以将CallManager在会话中的角色看作PeerConnection

> 参见 `webrtc/src/pc/peer_connection.cc`

```cpp
// CallManager 持有Thread对象
std::unique_ptr<rtc::Thread> signaling_thread_;
std::unique_ptr<rtc::Thread> worker_thread_;
std::unique_ptr<rtc::Thread> network_thread_;
```

在`call_manager.cc`中定义了很多种`rtc::MessageData`，用于CallManager的控制流管理。所以实际上就是用WebRTC的消息队列，例如

```cpp
int CallManager::SetRtcEngineConfig(RtcEngineObserverInterface* observer,
                                    const std::string& app_id,
                                    const std::string& country_code) {
  ...

  SetRtcEngineConfigMsg* msg = new SetRtcEngineConfigMsg(app_id, country_code);
  SignalingThread()->Post(RTC_FROM_HERE, this, MSG_SET_RTC_ENGINE_CONFIG, msg);
  ...
}
```

## zorro::MediaEngine

CallManager通过持有的MediaEngine来管理媒体流；

可以直接理解为WebRTC中的media engine，即持有
- Capture Device
- Player Device
- MediaStream

MediaStream持有所有的媒体track（track就是一个单向的媒体传输）
```cpp
const std::string id_;
AudioTrackVector audio_tracks_;
VideoTrackVector video_tracks_;
```

# Zorro SDK 在接收端的修改

## webrtc::voe::ChannelReceive

`audio/channel_receive.cc`

接收音频数据包，主要改动是添加了2个函数

```cpp
absl::optional<int32_t> ChannelReceive::GetEstimatedE2EDelay() {
  uint32_t rtp_timestamp;
  int64_t time_ms;
  if (GetPlayoutRtpTimestamp(&rtp_timestamp, &time_ms)) {
    int64_t playout_ntp_ms;
    absl::optional<ZorroNtpClockOffset> sender_clock_offset;
    {
      MutexLock lock(&ts_stats_lock_);
      if (rtp_to_ntp_.Estimate(rtp_timestamp, &playout_ntp_ms)) {
        sender_clock_offset = sender_clock_offset_;
      }
    }
    auto e2e_clock_offset =
        ZorroNtpClockOffsetReceiver::GetSenderClockOffset(sender_clock_offset);
    if (e2e_clock_offset) {
      constexpr int kMaxNotUpdateTimeMs = 2000;
      // GetPlayoutRtpTimestamp获取的time_ms是网络收到rtp音频的时间点,考虑到neteq缓存,kMaxNotUpdateTimeMs不能太小
      // 需要加上与time_ms的时间偏差才是当前的播放位置,超过kMaxNotUpdateTimeMs后就认为没有数据可播放了,此时返回无效数据-1
      auto time_us = rtc::TimeMicros();
      uint32_t time_since_last_playout_ms = (time_us / 1000 - time_ms);
      if (time_since_last_playout_ms > kMaxNotUpdateTimeMs) {
        return -1;
      }
      auto sender_ntp_ms =
          webrtc::TimeMicrosToNtp(time_us).ToMs() + *e2e_clock_offset;
      playout_ntp_ms += time_since_last_playout_ms;
      return sender_ntp_ms > playout_ntp_ms ? sender_ntp_ms - playout_ntp_ms
                                            : -1;
    }
  }
  return absl::nullopt;
}

absl::optional<int64_t> ChannelReceive::GetE2EClockOffset() {
  absl::optional<ZorroNtpClockOffset> sender_clock_offset;
  {
    MutexLock lock(&ts_stats_lock_);
    sender_clock_offset = sender_clock_offset_;
  }
  return ZorroNtpClockOffsetReceiver::GetSenderClockOffset(sender_clock_offset);
}
```

## webrtc::DelayManager
`modules/audio_coding/neteq/delay_manager.cc`

功能是在NetEq中**维护音频数据接收和播放之间的延迟**

`DelayManager::Update()` 在 `DecisionLogic::PacketArrived` 之中被调用，用于在新的数据包到来时更新延迟计算。这个函数在zorro SDK（左）中有较大改动：

![DelayManager](images/46f1bb28a936c8ffd43b30ac0e8a4fd6.png)


```cpp
// original
const int iat_ms = packet_iat_stopwatch_->ElapsedMs();

// modified
int iat_ms = packet_iat_stopwatch_->ElapsedMs();
int64_t current_time_ms = rtc::TimeMillis();
if (last_update_time_ms_ != 0) {
  iat_ms = current_time_ms - last_update_time_ms_;
}
last_update_time_ms_ = current_time_ms;

/*
“iat”就是 inter-arrival time，也就是两个连续包之间的间隔。
显然这里的改动是用时钟计时代替了 packet_iat_stopwatch_
*/
```


```cpp
// modification
reorder_prob_ = reorder_prob_ * 0.999f + (reordered ? 0.001 : 0);
RTC_LOG_D_C << reorder_prob_;
if (reordered) {
  if (relative_delay - min_disordered_delay_ > target_noret_ms_ / 2) {
    disordered_delay_ = 0.99f * disordered_delay_+ 0.01f * pow(relative_delay, 2);
    more_delay_for_audience_ms_ =
        int(fmax(0, sqrt(disordered_delay_) - target_noret_ms_) *
            (1 - reorder_prob_) + min_disordered_delay_);
    RTC_LOG_D_C << "rtt from reorder is: " << more_delay_for_audience_ms_;
  } else {
    min_disordered_delay_ =
        fmin(min_disordered_delay_,
             0.8 * min_disordered_delay_ + 0.2f * relative_delay);
    RTC_LOG_D_C << "min delay from reorder is: " << min_disordered_delay_;
  }
} else {
  min_disordered_delay_ = fmax(min_disordered_delay_,
      0.8 * min_disordered_delay_ + 0.2f * relative_delay);
  RTC_LOG_D_C << "not reorder delay from reorder: " << relative_delay;
}

/*
EMA(Exponential Moving Average) 是一种遍历数据点计算加权平均的方法，其比较看重当前数据点，可以通过平滑因子α进行调节：
t = α*t + (1-α)*T
上面过程的reorder_prob和disordered_delay_, min_dis_ordered_delay_都是用EMA计算的。

如果发生乱序的同时，到达时间间隔还足够大，那么此次乱序的时间间隔需要纳入统计。
more_delay_for_audience_ms_计算需要给观众引入更多的延迟。
*/
```


```cpp
// original
target_level_ms_ = (1 + bucket_index) * kBucketSizeMs;
target_level_ms_ = std::max(target_level_ms_, effective_minimum_delay_ms_);
if (maximum_delay_ms_ > 0) {
  target_level_ms_ = std::min(target_level_ms_, maximum_delay_ms_);
}
if (packet_len_ms_ > 0) {
  // Target level should be at least one packet.
  target_level_ms_ = std::max(target_level_ms_, packet_len_ms_);
  // Limit to 75% of maximum buffer size.
  target_level_ms_ = std::min(
      target_level_ms_, 3 * max_packets_in_buffer_ * packet_len_ms_ / 4);
}

// modification
int safe_bucket_index = histogram_->Quantile((1 << 30) * 0.97);
target_noret_ms_ = (1 + 3 * bucket_index) * kBucketSizeMs;
int more_delay_for_audience =
    fmin((safe_bucket_index + 1) * kBucketSizeMs - target_noret_ms_,
         more_delay_for_audience_ms_);
more_delay_for_audience = fmax(more_delay_for_audience, 0);
if (zorro::JoinChanExtraInfo::IsAudioFecEnabled(remote_info_)) {
  switch(local_scenario_) {
    case NetEq::Scenario::kAudioSendRecv: {
      target_level_ms_ = target_noret_ms_ + 1.5 * more_delay_for_audience;
      break;
    }
    case NetEq::Scenario::kAudioRecvOnly: {
      target_level_ms_ = target_noret_ms_ + 2.5 * more_delay_for_audience;
      break;
    }
    case NetEq::Scenario::kAudioSendRecvLowLatency: {
      target_level_ms_ = target_noret_ms_ / 2 + more_delay_for_audience;
      break;
    }
    case NetEq::Scenario::kInvalid:
    case NetEq::Scenario::kVideo: {
      target_level_ms_ = target_noret_ms_ + 4 * more_delay_for_audience;
      break;
    }   
  }
} else {
  target_level_ms_ = (safe_bucket_index + 1) * kBucketSizeMs;
}

/*
Histogram用于维护直方图，Quantile是根据给定的量取分位点的函数。

在开启了FEC的情况下，target_level_ms的计算分场景。计算过程涉及 target_noret_ms 和 more_delay_for_audience

不开启FEC的情况下，target_level_ms_的计算直接根据分位点得出。
*/
```

## webrtc::RtpSeqNumOnlyRefFinder

`modules/video_coding/rtp_seq_num_only_ref_finder.cc`


![RtpSeqNumOnlyRefFinder1](images/26f2d84579aee1b33235c8321e005218.png)

清理历史帧的区间加长，也就是留下更多历史帧


![RtpSeqNumOnlyRefFinder2](images/c5884faef7eba18236e43e438994752d.png)

对关键帧的handoff策略作了修改：如果stash里有更早的非关键帧，那么就先留下这个可以handoff的关键帧。直到这些更早的帧因为超时被清除，关键帧才会被handoff。

## webrtc::VCMTiming

`modules/video_coding/timing.cc`

![VCMTiming1](images/aec721382ae8c35b410ae093901e5209.png)

![VCMTiming2](images/c29735b45308a174767fa48680eee21b.png)
zorro对`VCMTiming::UpdateCurrentDelay(int64_t render_time_ms,int64_t actual_decode_time_ms)`做了大量修改：

```cpp
void VCMTiming::UpdateCurrentDelay(int64_t render_time_ms,
                                   int64_t actual_decode_time_ms) {
  MutexLock lock(&mutex_);
  int64_t target_delay_ms = TargetDelayInternal();
  int64_t decode_delay_ms = RequiredDecodeTimeMs();
  // current delay is caculated based on buffer, decodabale time, and jitter delay.
  // Current delay to become too large than jitter delay meaans jitter delay cannot represent
  // frames jitter because of late decodable time and shallow buffer. In this case we adopt a
  // more agressive delay estimation to reduce dependency for jitter delay.
  if (jitter_delay_ms_ > 0) {
    float extra_need_delay = (float)(current_delay_ms_ - decode_delay_ms) / (float)jitter_delay_ms_;
    if (!enlarged_percentile_ && extra_need_delay > percentile_update_threshold_) {
      RTC_LOG_D_C << "current delay / up";
      cur_delay_filter_.ResetPercentile(0.99f);
      enlarged_percentile_ = true;
    }
    if (enlarged_percentile_ && extra_need_delay < 0.6 * percentile_update_threshold_) {
      RTC_LOG_D_C << "current delay / down";
      cur_delay_filter_.ResetPercentile(0.90f);
      enlarged_percentile_ = false;
    }
  }
  /*
  调整百分比 :当发现current delay已经远超jitter delay时，提高percentile；当发现percentile过低时会回调
  这个percentile用于下面的到current delay的值

  PercentileFilter实际上就是通过接收很多数据点，然后根据设置的百分比返回分位点
  */




  // Now, delayed ms = decodable time - render time - buffering time
  // - (time to decode and render) + jitter delay
  int64_t delayed_ms = actual_decode_time_ms -
                       (render_time_ms - decode_delay_ms - render_delay_ms_) +
                       jitter_delay_ms_;
  // Update current delay filter.
  int64_t current_delay_ms = current_delay_ms_ + delayed_ms;
  cur_delay_filter_.Insert(current_delay_ms);
  history_.emplace(current_delay_ms, actual_decode_time_ms);
  while (!history_.empty() &&
      actual_decode_time_ms - history_.front().sample_time_ms > kDelayDetectWindowMs) {
    cur_delay_filter_.Erase(history_.front().delay_time_ms);
    history_.pop();
  }
/*
  计算并加入delay的增值
  同时维护 cur_delay_filter_，用的是一个queue of Samples: history_，目的是使cur_delay_filter里的数据点都只在一个时间窗口内
*/




  // Unbind current_delay with target_delay to allow more chances for retransmit
  current_delay_ms = cur_delay_filter_.GetPercentileValue();
  // Smooth current_delay_ms changes.
  current_delay_ms_ = current_delay_ms > current_delay_ms_ ? current_delay_ms :
      0.9f * current_delay_ms_ + 0.1f * current_delay_ms;
  // Limit current delay between MaxVideoDelay and Jitter Delay.
  if (current_delay_ms > kMaxVideoDelayMs) {
    current_delay_ms_ = kMaxVideoDelayMs;
  } else if (current_delay_ms < target_delay_ms) {
    current_delay_ms_ = target_delay_ms;
  } else {
    current_delay_ms_ = current_delay_ms;
  }

  /*
  最终确定current delay ms的值
  1. 从数据点中选择一个分位点
  2. 如果分位点delay较大就选分位点，这样有助于使current delay和target delay解绑；否则加权算均值

  还需要限定范围[target_delay_ms, kMaxVideoDelayMs]
  */
...
}
```

![VCMTiming3](images/e0d2b4fb6ef49dbd118457dc76306081.png)

1. 为了和音频同步，目标视频延迟不再返回目标延迟而是当前延迟；
2. 为了弥补上面的改动导致无法获得基准视频延迟，添加了一个方法`BaseTargetVideoDelay`

## webrtc::StreamSychronization

![StreamSynchronization1](images/d4d661a0af87893a6721725fc8baddc4.png)
在变量上做了一些修改，不再以音频delay为准，而是将音频和视频的base_target_delay分开，这样可以更灵活地调整同步策略。

在 `StreamSynchronization::ComputeDelays`和`StreamSynchronization::SetTargetBufferingDelay`中的改动基本围绕这个base target delay的改动进行。

![StreamSynchronization2](images/c661dd0135c9ec4018e2f9625ea5029e.png)

## webrtc::AudioReceiveStream::Config
媒体流设置项也有改动，这里就只举音频的例子。

![Config1](images/6c2dd83ef16f9bd4bd4e5302becaf394.png)
添加了`estimated_e2e_delay`和`fraction_lost`
![Config2](images/caa08a6469abc7bff6abb4e4dc3f7f30.png)
音频流添加了RTX流

# Zorro SDK在发送端的修改
（待补充）