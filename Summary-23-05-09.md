> 在调试完视频部分的WebRTC源代码后，对此部分的主要问题及方案做一个总结。

# 背景


<span style="font-weight: bold; color: red;">视频数据加密导致无法解析获得必要信息</span>

视频编码信息
- H264编码
- 视频包全部为FU-A分片类型
- 帧率15或9，GOP设置为1s
- 一个关键帧的分包超过20个（经验）
- 采样90000，也就是说两个关键帧之间的采样时间差是1s左右
- 一个视频帧的尾包的MARKER位是1

# 模糊判断视频数据包的信息

在`video/rtp_video_stream_receiver2.cc`中实现了`bool RtpVideoStreamReceiver2::ParsedPayloadModifier::Modify`  

在`RtpVideoStreamReceiver2::OnReceivedPayloadData`之前，正常情况下通过解析视频包的payload已经获得了必要的信息。但是因为视频数据被加密，所以在`RtpVideoStreamReceiver2::OnReceivedPayloadData`的开头添加下面的修改过程：
```cpp
auto packet =
      std::make_unique<video_coding::PacketBuffer::Packet>(rtp_packet, video);

  if (!modifier_.Modify(rtp_packet, packet->video_header))
    return;
```

这个函数保证视频数据包的frame type, is first packet in frame, is last packet in frame的属性的正确性。

# 防止包信息被后续改动

在`PacketBuffer::FindFrames`中，会根据解析得到的参数集对视频包信息做一次修正，将这个过程中对frame type的改动去掉。

# 跳过关键帧需求判断

当视频解码器在解码上下文中判断需要关键帧才能继续解码时，会设置`keyframe_required_`标记，此时将忽略之后的一切非关键帧。

但是我们的测试过程没有涉及到真正的解码过程，所以非关键帧实际上会被一直遗漏。所以需要将`FrameBuffer::FindNextFrame`中的关键帧需求判断去掉。





