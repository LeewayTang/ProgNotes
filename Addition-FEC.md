# FEC in WebRTC

FEC forward error corrrection 即前向纠错技术，在WebRTC中常见以下几类：
## in-band FEC
```
+-----------------+-----------------+-------------------+
| RTP header      | primary payload | redundant payload |
+-----------------+-----------------+-------------------+
| V=2|P|X| CC |M| | PT1 | data1     | PT2 | data2       |
| PT | seq num    |                 |                   |
| timestamp       |                 |                   |
| SSRC            |                 |                   |
| CSRC (optional) |                 |                   |
+-----------------+-----------------+-------------------+
```
带内fec是opus的一种特性，它可以在发送端增加一些额外的数据，用于在接收端恢复丢失或损坏的数据。带内fec的原理是，在每个opus包中，除了包含当前帧的编码数据外，还包含上一帧的冗余编码数据（LBRR），这些冗余数据是通过低比特率编码（LBR）来生成的。

如果接收端收到了一个完整的opus包，那么它就可以直接解码当前帧的数据，并丢弃冗余数据；如果接收端发现丢失了一个opus包，那么它就可以从下一个opus包中提取冗余数据，并用它来恢复丢失帧的数据。这样就可以减少丢包造成的音质损失和听感不连贯。


为了使用带内fec，需要满足以下条件：

- opus编码器必须通过OPUS_SET_INBAND_FEC选项来启用带内fec功能。
- opus编码器必须通过OPUS_SET_PACKET_LOSS_PERC选项来告知预期的丢包率。
- opus编码器必须使用足够高的比特率来为冗余数据留出空间。
- opus编码器必须使用线性预测或混合模式来进行编码。

## RED
Redundant Encoding

RED本身不是一种FEC，而是一种FEC的载体。它可以携带不同类型的FEC数据，如Generic FEC 或ULP FEC。RED的作用是在一个RTP包中包含多个不同编码器或不同时间戳的音视频数据，以提高容错能力。

### Generic FEC

```
+-----------------+-----------------+
| RTP header      | FEC payload     |
+-----------------+-----------------+
| V=2|P|X| CC |M| | L | D | type    |
| PT = FEC        | mask            |
| seq num         | data            |
| timestamp       |                 |
| SSRC            |                 |
| CSRC (optional) |                 |
+-----------------+-----------------+
```

Generic FEC是一种基于包的FEC，它对每个RTP数据包单独处理，生成一个FEC包，用于恢复一个丢失的RTP包。Generic FEC使用XOR运算来生成和恢复FEC数据。

- L: 1 bit，长度恢复标志。如果L=1，则FEC数据包的长度与被保护的RTP数据包的长度相同；如果L=0，则FEC数据包的长度可能不同于被保护的RTP数据包的长度。
- D: 1 bit，时间戳恢复标志。如果D=1，则FEC数据包的时间戳与被保护的RTP数据包的时间戳相同；如果D=0，则FEC数据包的时间戳可能不同于被保护的RTP数据包的时间戳。
type: 6 bits，FEC类型。用于区分不同的FEC算法或参数。目前只定义了一种类型，即XOR运算。
- mask: 24 bits，保护位掩码。用于指示哪些RTP数据包被保护。每个比特对应一个RTP数据包，从最低有效位开始，依次为seq num, seq num - 1, seq num - 2, ..., seq num - 23。如果某个比特为1，则对应的RTP数据包被保护；如果为0，则不被保护。
- data: variable length，FEC数据。用于恢复丢失或损坏的RTP数据包。它是通过对被保护的RTP数据包的有效载荷进行XOR运算得到的。


### ULP FEC


```
+-----------------+-----------------+
| RTP header      | ULP FEC payload |
+-----------------+-----------------+
| V=2|P|X| CC |M| | E  | L  | P     |
| PT = ULP FEC    | SN base         |
| seq num         | TS recovery     |
| timestamp       | length recovery |
| SSRC            | mask            |
| CSRC (optional) | data            |
+-----------------+-----------------+
```

ULP FEC是一种基于块的FEC，它将一组RTP数据包作为一个编码块来处理，生成一个或多个FEC包，用于恢复多个丢失的RTP包。ULP FEC使用LDPC编码来生成和恢复FEC数据。

- E: 1 bit，扩展标志。如果E=0，则表示这是一个基本FEC包；如果E=1，则表示这是一个扩展FEC包。
- L: 1 bit，长度恢复标志。如果L=1，则表示length recovery字段存在；如果L=0，则表示length recovery字段不存在。
- P: 1 bit，填充标志。如果P=1，则表示在FEC数据之后有填充字节；如果P=0，则表示没有填充字节。
- SN base: 16 bits，序列号基准。用于指示编码块中第一个RTP数据包的序列号。
- TS recovery: 32 bits，时间戳恢复。用于恢复编码块中所有RTP数据包的时间戳。
length recovery: 16 bits，长度恢复。用于恢复编码块中所有RTP数据包的长度。只有当L=1时才存在。
- mask: variable length，保护位掩码。用于指示哪些RTP数据包被保护。每个比特对应一个RTP数据包，从最低有效位开始，依次为SN base, SN base + 1, SN base + 2, ..., SN base + N - 1。如果某个比特为1，则对应的RTP数据包被保护；如果为0，则不被保护。N是编码块的大小，由E和P决定。
- data: variable length，FEC数据。用于恢复丢失或损坏的RTP数据包。它是通过对被保护的RTP数据包的有效载荷进行LDPC编码得到的。

## RS-FEC

```
+-----------------+-----------------+
| RTP header      | RS FEC payload  |
+-----------------+-----------------+
| V=2|P|X| CC |M| | data            |
| PT = RS FEC     |                 |
| seq num         |                 |
| timestamp       |                 |
| SSRC            |                 |
| CSRC (optional) |                 |
+-----------------+-----------------+
```


RS-FEC（Reed Solomon Forward Error Correction）是一种FEC的技术，它使用Reed Solomon编码来生成冗余数据，用于在接收端恢复丢失或损坏的数据。RS-FEC有多种不同的参数和配置，如RS (528,514)，RS (544,514)，RS (272,258)等，它们表示生成编码长度，原始信息数据长度和生成多项式。