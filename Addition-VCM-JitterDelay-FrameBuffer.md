# VCM Jitter Delay 的计算机制

在`modules/video_coding/jitter_estimator.cc`中计算

> 首先在VCM中的 jitter delay 是用来预测“下一帧”的时间的。通过已知的视频帧时间推测下一帧可能在什么时间到达。

在WebRTC的接收端，jitter delay的计算调用栈如下：

1. `FrameBuffer::GetNextFrame()`
   此函数从frame buffer中获取下一个要解码的帧。在此过程中，会根据帧间的时间间隔（inter-frame delay）更新JitterEstimator。

2. `VCMJitterEstimator::UpdateEstimate()`
   此函数根据新的帧间时间间隔（inter-frame delay）更新JitterEstimator。在这个函数中，它会调用`EstimateRandomJitter()`来实际更新jitter delay估计。

3. `VCMJitterEstimator::EstimateRandomJitter()`
   此函数根据帧间时间间隔计算jitter delay，并更新内部状态（如绝对延迟的指数移动平均`abs_dev_`和延迟的指数移动平均`avg_delay_`）。

4. `VCMJitterEstimator::GetJitterEstimate()`
   此函数基于内部状态计算并返回jitter delay。这个估计值主要基于绝对延迟的指数移动平均`abs_dev_`和其他一些参数，如解码器的解码时间和渲染延迟。

总结一下，接收端的jitter delay计算调用栈如下：

```
FrameBuffer::GetNextFrame()
  -> VCMJitterEstimator::UpdateEstimate()
    -> VCMJitterEstimator::EstimateRandomJitter()
      -> VCMJitterEstimator::GetJitterEstimate()
```

这个调用栈说明了如何从FrameBuffer的GetNextFrame()方法开始，经过一系列的函数调用，最终得到jitter delay的计算结果。


在WebRTC中，VCMJitterEstimator类用于估计jitter delay。这个类主要依赖于`EstimateRandomJitter()`方法来更新jitter估计。以下是该方法的实现：

```cpp
void VCMJitterEstimator::EstimateRandomJitter(int64_t frame_delay_ms) {
  double abs_delta = fabs(static_cast<double>(frame_delay_ms) - avg_delay_);
  if (abs_delta > 0.0) {
    // Calculate the new exponential moving average of the absolute delay.
    abs_dev_ = kAlpha * abs_dev_ + (1.0 - kAlpha) * abs_delta;
  }

  // Calculate the new exponential moving average of the delay.
  avg_delay_ = kAlpha * avg_delay_ + (1.0 - kAlpha) * frame_delay_ms;
}
```

以下是计算jitter delay的主要步骤：

1. 计算连续帧之间的到达时间差（inter-arrival time），即`frame_delay_ms`。

2. 计算帧之间的绝对时间差，即`abs_delta`。

3. 更新绝对延迟的指数移动平均（Exponential Moving Average, EMA），这里使用了一个加权平均方法，将当前的`abs_delta`与之前的`abs_dev_`结合起来，以得到一个平滑的绝对延迟估计。加权因子`kAlpha`决定了对新数据和旧数据的权重分配。

   `abs_dev_ = kAlpha * abs_dev_ + (1.0 - kAlpha) * abs_delta`

4. 更新延迟的指数移动平均（Exponential Moving Average, EMA）。同样地，这里使用了一个加权平均方法，将当前的`frame_delay_ms`与之前的`avg_delay_`结合起来，以得到一个平滑的延迟估计。

   `avg_delay_ = kAlpha * avg_delay_ + (1.0 - kAlpha) * frame_delay_ms`

5. 最后，在`GetJitterEstimate()`方法中，计算并返回jitter delay。这个估计值主要基于绝对延迟的指数移动平均`abs_dev_`和其他一些参数，如解码器的解码时间和渲染延迟。

在这个过程中，关键的数学公式是更新指数移动平均（EMA）的公式：

- `abs_dev_ = kAlpha * abs_dev_ + (1.0 - kAlpha) * abs_delta`
- `avg_delay_ = kAlpha * avg_delay_ + (1.0 - kAlpha) * frame_delay_ms`

这两个公式都使用了一个加权平均方法，将当前的观测值与之前的估计值结合起来，以得到一个平滑的估计。这个方法可以减小短暂的网络波动对jitter估计的影响，从而提高估计的准确性。

# FrameBuffer取帧策略

首先需要认识FrameBuffer中的几个数据结构：
```cpp
// FrameMap是Frame ID和FrameInfo一一对应的哈希表
using FrameMap = std::map<int64_t, FrameInfo>;

FrameMap frames_ RTC_GUARDED_BY(mutex_);

std::vector<FrameMap::iterator> frames_to_decode_ RTC_GUARDED_BY(mutex_);

std::vector<FrameMap::iterator> current_superframe;

```

组帧完成后，待解码的视频帧被插入`frames_`里，按frame ID顺序排好。在一个过程之中，将符合条件的帧送入frames_to_decode_里。

下面来看这个过程：

## FrameBuffer::StartWaitForNextFrameOnQueue

`FrameBuffer::StartWaitForNextFrameOnQueue()`函数主要负责启动一个等待任务，该任务在等待一定时间后检查是否有新的帧可供解码。如果在等待期间收到新帧，任务将被取消。否则，任务将继续进行帧传递。以下是对这个函数的代码分析：

1. 首先，确保回调队列和回调任务都存在且未运行。
```cpp
RTC_DCHECK(callback_queue_);
RTC_DCHECK(!callback_task_.Running());
```

2. 调用`FindNextFrame()`函数，计算等待下一个帧的时间。
```cpp
int64_t wait_ms = FindNextFrame(clock_->TimeInMilliseconds());
```

3. 使用`RepeatingTaskHandle::DelayedStart()`启动一个延迟任务，任务在等待`wait_ms`时间后开始执行。
```cpp
callback_task_ = RepeatingTaskHandle::DelayedStart(
    callback_queue_->Get(), TimeDelta::Millis(wait_ms), [this] {
```

4. 在任务执行时，检查`frames_to_decode_`队列是否为空。如果不为空，说明有帧可供解码，那么调用`GetNextFrame()`获取下一个帧。
```cpp
if (!frames_to_decode_.empty()) {
  // We have frames, deliver!
  frame = absl::WrapUnique(GetNextFrame());
}
```

5. 如果`frames_to_decode_`队列为空，但还有剩余等待时间，则说明`FrameBuffer`在任务创建和执行之间被清空。继续等待剩余时间。
```cpp
else if (clock_->TimeInMilliseconds() < latest_return_time_ms_) {
  // If there's no frames to decode and there is still time left, it
  // means that the frame buffer was cleared between creation and
  // execution of this task. Continue waiting for the remaining time.
  int64_t wait_ms = FindNextFrame(clock_->TimeInMilliseconds());
  return TimeDelta::Millis(wait_ms);
}
```

6. 将`frame_handler_`移动到局部变量`frame_handler`中并取消回调。
```cpp
frame_handler = std::move(frame_handler_);
CancelCallback();
```

7. 如果有帧可供解码，则将原因设置为`kFrameFound`，否则设置为`kTimeout`。然后调用`frame_handler()`处理帧或超时情况。
```cpp
ReturnReason reason = frame ? kFrameFound : kTimeout;
frame_handler(std::move(frame), reason);
```

通过这个函数，`FrameBuffer`能够在适当的时间获取下一个帧并处理，避免了过早或过晚地尝试解码帧。

## FrameBuffer::FindNextFrame

在WebRTC的`FrameBuffer`中，`frames`和`frames_to_decode`之间的关系以及帧如何从`frames`中被送入`frames_to_decode`等待解码的过程可以通过分析`FindNextFrame()`函数来理解。这个函数负责查找下一个可供解码的帧并将其放入`frames_to_decode`队列。以下是关键部分的代码分析：

1. 首先，函数将`frames_to_decode_`清空。
```cpp
frames_to_decode_.clear();
```

2. 接着，遍历`frames_`中的所有帧，查找连续且满足解码条件的帧。
```cpp
for (auto frame_it = frames_.begin();
     frame_it != frames_.end() && frame_it->first <= last_continuous_frame_;
     ++frame_it) {
```

3. 对于每个帧，检查其是否连续且没有缺失的可解码帧。
```cpp
if (!frame_it->second.continuous ||
    frame_it->second.num_missing_decodable > 0) {
  continue;
}
```

4. 如果当前帧需要关键帧，但帧不是关键帧，则跳过此帧。
```cpp
if (keyframe_required_ && !frame->is_keyframe())
  continue;
```

5. 检查帧是否比最后解码的帧更新。
```cpp
if (last_decoded_frame_timestamp &&
    AheadOf(*last_decoded_frame_timestamp, frame->Timestamp())) {
  continue;
}
```

6. 将当前帧与同一superframe中的其他帧一起放入`frames_to_decode`中。
```cpp
current_superframe.push_back(frame_it);
```

7. 检查当前superframe是否完整。如果不完整，则继续查找下一个帧。
```cpp
if (!last_layer_completed) {
  continue;
}
```

8. 将完整的superframe移动到`frames_to_decode_`中。
```cpp
frames_to_decode_ = std::move(current_superframe);
```

通过这个过程，`FrameBuffer`会将满足解码条件的帧从`frames`中取出并放入`frames_to_decode`队列中。然后，在`GetNextFrame()`方法中，这些帧将被提取并送到解码器进行解码。

## FrameBuffer::GetNextFrame

它负责从`frames_to_decode_`队列中获取下一个要解码的帧。以下是代码的关键部分分析：

1. 为了确保在回调上下文中运行，使用`RTC_DCHECK_RUN_ON`宏进行检查。
```cpp
RTC_DCHECK_RUN_ON(&callback_checker_);
```

2. 定义一些局部变量并检查`frames_to_decode_`不为空。
```cpp
std::vector<EncodedFrame*> frames_out;
RTC_DCHECK(!frames_to_decode_.empty());
```

3. 初始化与第一帧相关的变量，并处理任何可能的渲染时间问题。
```cpp
EncodedFrame* first_frame = frames_to_decode_[0]->second.frame.get();
int64_t render_time_ms = first_frame->RenderTime();
int64_t receive_time_ms = first_frame->ReceivedTime();
if (HasBadRenderTiming(*first_frame, now_ms)) {
  jitter_estimator_.Reset();
  timing_->Reset();
  render_time_ms = timing_->RenderTimeMs(first_frame->Timestamp(), now_ms);
}
```

4. 遍历`frames_to_decode_`，设置帧的渲染时间、更新解码帧历史记录，并从`frames_`映射中删除已解码的帧。
```cpp
for (FrameMap::iterator& frame_it : frames_to_decode_) {
  // ...
  frame->SetRenderTime(render_time_ms);
  // ...
  decoded_frames_history_.InsertDecoded(frame_it->first, frame->Timestamp());
  // ...
  frames_.erase(frames_.begin(), ++frame_it);
  // ...
}
```

5. 根据帧是否因重传而延迟，更新抖动估计和播放延迟。
```cpp
if (!superframe_delayed_by_retransmission) {
  // ...
  jitter_estimator_.UpdateEstimate(frame_delay, superframe_size);
  // ...
  timing_->SetJitterDelay(
      jitter_estimator_.GetJitterEstimate(rtt_mult, rtt_mult_add_cap_ms));
  timing_->UpdateCurrentDelay(render_time_ms, now_ms);
} else {
  if (RttMultExperiment::RttMultEnabled() || add_rtt_to_playout_delay_)
    jitter_estimator_.FrameNacked();
}
```

6. 更新抖动延迟和定时帧信息。
```cpp
UpdateJitterDelay();
UpdateTimingFrameInfo();
```

7. 如果`frames_out`只包含一个帧，则返回该帧。否则，将多个帧合并为一个帧并返回。
```cpp
if (frames_out.size() == 1) {
  return frames_out[0];
} else {
  return CombineAndDeleteFrames(frames_out);
}
```

通过这个函数，`FrameBuffer`从`frames_to_decode_`队列中提取下一个待解码的帧，更新解码帧历史记录，并在必要时调整渲染时间和抖动延迟。