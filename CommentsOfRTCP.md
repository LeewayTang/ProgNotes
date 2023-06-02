> 主要记录WebRTC中统计RTCP报文相关信息的部分，以及收发和处理的过程。

# 报文结构

## SR

  A Sender Report message consists of the header, the sender information block, a variable number of receiver report blocks, and potentially a profile-specific extensions.

**The header:**
```
 0               1               2               3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|V=2|P|    RC   |   PT=SR=200   |             length L          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         SSRC of sender                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```
V, P:

As described for RTP data packets, see above.

RC:
5 bits
The number of reception report blocks contained in this packet.

PT:
8 bits
The packet type constant 200 designates an RTCP SR packet.

L:
16 bits
The length of this RTCP packet in 32-bit words minus one, including the header and any padding.

SSRC:
32 bits
The synchronisation source identifier for the sender of this SR packet.

**The sender information block:**
```
 0               1               2               3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            NTP timestamp, most significant word NTS           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|             NTP timestamp, least significant word             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       RTP timestamp RTS                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                   sender's packet count SPC                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    sender's octet count SOC                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```
NTS:
64 bits
The NTP timestampgif indicates the point of time measured in wall clock time when this report was sent. In combination with timestamps returned in reception reports from the respective receivers, it can be used to estimate the round-trip propagation time to and from the receivers.

RTS:
32 bits
The RTP timestamp resembles the same time as the NTP timestamp (above), but is measured in the same units and with the same random offset as the RTP timestamps in data packets. This correspondence may be used for intra- and inter-media synchronisation for sources whose NTP timestamps are synchronised, and may be used by media-independent receivers to estimate the nominal RTP clock frequency.

SPC:
32 bits
The sender's packet count totals up the number of RTP data packets transmitted by the sender since joining the RTP session. This field can be used to estimate the average data packet rate.

SOC:
32 bits
The total number of payload octets (i.e., not including the header or any padding) transmitted in RTP data packets by the sender since starting up transmission. This field can be used to estimate the average payload data rate.

**The receiver report blocks:**

和RR的Receiver Report Block是一样的，如果发送端并没有接收媒体流的话，SR报文就不会有下面的这些RR block。也就是说，SR报文的RR block是对RR报文的功能覆盖。
```
 0               1               2               3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 SSRC_1 (SSRC of first source)                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|fraction lost F|      cumulative number of packets lost C      |
-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         extended highest sequence number received  EHSN       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    inter-arrival jitter J                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          last SR LSR                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    delay since last SR DLSR                   |
+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
|                 SSRC_2 (SSRC of second source)                |
:                               ...                             :
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```
SSRC_n:

32 bits
The SSRC identifier of the sender whose reception is reported in this block.

F:
8 bits
The sender of the receiver report estimates the fraction of the RTP data packets from source SSRC_n that it assumes to be lost since it sent the previous SR or RR packet.

C:
24 bits
The sender of a receiver report blocks also tries to estimate the total number of RTP data packets from source SSRC_n that have been lost since the beginning of reception. Packets that arrive late are not counted as lost, and the loss may be negative if there are duplicates.

EHSN:
32 bits
The low 16 bits of the extended highest sequence number contain the highest sequence number received in an RTP data packet from source SSRC_n, and the most significant 16 bits extend that sequence number with the corresponding count of sequence number cycles.

J:
32 bits
An estimate of the statistical variance of the RTP data packet inter-arrival time, measured in timestamp units and expressed as an unsigned integer.

**The extensions:**
```
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                  profile-specific extensions                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

## RR

SR报文的功能是覆盖RR报文的，RR报文的结构如下。其中的Receiver Report Block就是SR报文中相同的结构。

```
 0               1               2               3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|V=2|P|    RC   |   PT=RR=201   |            length L           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     SSRC of packet sender                     |
+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
|                 SSRC_1 (SSRC of first source)                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| fraction lost |       cumulative number of packets lost       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           extended highest sequence number received           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      inter-arrival jitter                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         last SR (LSR)                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                   delay since last SR (DLSR)                  |
+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
|                 SSRC_2 (SSRC of second source)                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
:                               ...                             :
+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
|                  profile-specific extensions                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

# ReportBlock 的生成

实现在 modules/rtp_rtcp/source/rtcp_packet/report_block.h

是SR和RR中的report block的类实现。持有的字段和上面的结构是对应的：

```cpp
 private:
  uint32_t source_ssrc_;     // 32 bits
  uint8_t fraction_lost_;    // 8 bits representing a fixed point value 0..1
  int32_t cumulative_lost_;  // Signed 24-bit value
  uint32_t extended_high_seq_num_;  // 32 bits
  uint32_t jitter_;                 // 32 bits
  uint32_t last_sr_;                // 32 bits
  uint32_t delay_since_last_sr_;    // 32 bits, units of 1/65536 seconds
```

生成ReportBlock的过程在 
```cpp
std::vector<rtcp::ReportBlock> RTCPSender::CreateReportBlocks(const FeedbackState& feedback_state);
```
方法中实现。在这个方法中，生成 report block的核心语句是
```cpp
result = receive_statistics_->RtcpReportBlocks(RTCP_MAX_REPORT_BLOCKS);
```

这里的 `receive_statistics_` 是 `ReceiveStatisticsProvider` 接口，实际上使用的是 `ReceiveStatisticsImpl`的下面方法

```cpp
std::vector<rtcp::ReportBlock> ReceiveStatisticsImpl::RtcpReportBlocks(
    size_t max_blocks) {
  std::vector<rtcp::ReportBlock> result;
  result.reserve(std::min(max_blocks, all_ssrcs_.size()));

  size_t ssrc_idx = 0;
  for (size_t i = 0; i < all_ssrcs_.size() && result.size() < max_blocks; ++i) {
    ssrc_idx = (last_returned_ssrc_idx_ + i + 1) % all_ssrcs_.size();
    const uint32_t media_ssrc = all_ssrcs_[ssrc_idx];
    auto statistician_it = statisticians_.find(media_ssrc);
    RTC_DCHECK(statistician_it != statisticians_.end());
    statistician_it->second->MaybeAppendReportBlockAndReset(result);
  }
  last_returned_ssrc_idx_ = ssrc_idx;
  return result;
}
```

这个方法中实际上只是将已有的信息填充进 report block，核心数据结构是

```cpp
std::unordered_map<uint32_t /*ssrc*/,std::unique_ptr<StreamStatisticianImplInterface> statisticians_;
```

实际负责收集数据的类是 `StreamStatisticianImpl` 
```cpp
void StreamStatisticianImpl::MaybeAppendReportBlockAndReset(
    std::vector<rtcp::ReportBlock>& report_blocks) {
  int64_t now_ms = clock_->TimeInMilliseconds();
  if (now_ms - last_receive_time_ms_ >= kStatisticsTimeoutMs) {
    // Not active.
    return;
  }
  if (!ReceivedRtpPacket()) {
    return;
  }

  report_blocks.emplace_back();
  rtcp::ReportBlock& stats = report_blocks.back();
  stats.SetMediaSsrc(ssrc_);
  // Calculate fraction lost.
  int64_t exp_since_last = received_seq_max_ - last_report_seq_max_;
  RTC_DCHECK_GE(exp_since_last, 0);

  int32_t lost_since_last = cumulative_loss_ - last_report_cumulative_loss_;
  if (exp_since_last > 0 && lost_since_last > 0) {
    // Scale 0 to 255, where 255 is 100% loss.
    stats.SetFractionLost(255 * lost_since_last / exp_since_last);
  }

  int packets_lost = cumulative_loss_ + cumulative_loss_rtcp_offset_;
  if (packets_lost < 0) {
    // Clamp to zero. Work around to accomodate for senders that misbehave with
    // negative cumulative loss.
    packets_lost = 0;
    cumulative_loss_rtcp_offset_ = -cumulative_loss_;
  }
  if (packets_lost > 0x7fffff) {
    // Packets lost is a 24 bit signed field, and thus should be clamped, as
    // described in https://datatracker.ietf.org/doc/html/rfc3550#appendix-A.3
    if (!cumulative_loss_is_capped_) {
      cumulative_loss_is_capped_ = true;
      RTC_LOG(LS_WARNING) << "Cumulative loss reached maximum value for ssrc "
                          << ssrc_;
    }
    packets_lost = 0x7fffff;
  }
  stats.SetCumulativeLost(packets_lost);
  stats.SetExtHighestSeqNum(received_seq_max_);
  // Note: internal jitter value is in Q4 and needs to be scaled by 1/16.
  stats.SetJitter(jitter_q4_ >> 4);

  // Only for report blocks in RTCP SR and RR.
  last_report_cumulative_loss_ = cumulative_loss_;
  last_report_seq_max_ = received_seq_max_;
  BWE_TEST_LOGGING_PLOT_WITH_SSRC(1, "cumulative_loss_pkts", now_ms,
                                  cumulative_loss_, ssrc_);
  BWE_TEST_LOGGING_PLOT_WITH_SSRC(1, "received_seq_max_pkts", now_ms,
                                  (received_seq_max_ - received_seq_first_),
                                  ssrc_);
}
```

上面这段代码展示了向 report block中填充信息的全部过程。
