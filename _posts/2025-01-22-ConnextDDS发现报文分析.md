---
title: ConnextDDS发现报文分析
date: 2025-01-22 17:31 +0800
last_modified_at: 2025-01-22 17:31 +0800
author: FeetingTimes
categories: ["DDS", "rtidds"]
tags: ["c++", "rtidds", "dds"]
pin: true
math: true
mermaid: true
img_path: /assets/img/ConnextDDS发现报文分析/
---

| 类别      | 说明                                                         |
| --------- | ------------------------------------------------------------ |
| DATA(p)   | 域参与者的信息，包含参与者的GUID，QoS等信息。                |
| DATA(w)   | DataWriter的信息，包含topic name，type，QoS等信息。          |
| DATA(r)   | DataReader的信息，包含topic name，type，QoS等信息。          |
| HEARTBEAT | 1.通知Reader在Writer的历史缓存中可用的序列号，这样Reader就可以请求（发送ACKNACK）任何错过的内容。2.它要求Reader发送对已进入Reader历史缓存的CacheChange更改的确认，以便Writer可以了解Reader的状态。 |
| ACKNACK   | 将Reader的状态通知给Writer，用于告知Writer所收到的消息的序列号或所缺少的消息的序列号。 |
| INFO_TS   | 显示数据包的源时间戳，Endianness表示小字节序(1)还是大字节序(0)。 |
| INFO_DST  | 指示处理该条消息中之后的子消息的RTPS reader实体，该实体的GUID为INFO_DST.GuidPrefix。 |
| GAP       | 通知DataReader 特定sample在DataWriter的队列中不可用。        |

![image1](image1.png)

![image2](image2.png)

端点发现阶段：DataWriters

![image3](image3.png)

![image4](image4.png)

\#57 当DATA(p)包被接收到后，参与者发现阶段就结束。此时publication WRITER会发送一条HEARTBEAT，在该条信息中，firstAvailableSeqNumber为1，lastSeqNumber为1，表示这个Publisher application中有一个DataWriter。这条HEARTBEAT消息会被一条ACKNACK消息回复。如#60

![image5](image5.png)

\#60 通过该条ACKNACK，订阅者告诉发布者，它需要接收一个DATA(w) sample。之后，发布者的publications Writer就会发送一条DATA(w)，如#61。

![image6](image6.png)

\#61 在该条消息中，包含了topic name，type，Qos。 发送方式为reliably，因此也会发送一条HEARTBEAT，其中firstSeqNumber(1)，lastSeqNumber(1)表示发布者期望订阅者对接收到这一条DATA(w)消息进行确认。

![image7](image7.png)

![image8](image8.png)

\#69 订阅者发送ACKNACK确认收到了DATA(w)。

![image9](image9.png)

**端点发现阶段：DataReaders**

![image10](image10.png)

![image11](image11.png)

\#50 当DATA(p)包被接收到后，参与者发现阶段就结束。此时subscription WRITER会发送一条HEARTBEAT，在该条信息中，firstAvailableSeqNumber为1，lastSeqNumber为1，表示这个Subscriber application中有一个DataReader。这条HEARTBEAT消息会被一条ACKNACK消息回复。如#51

![image12](image12.png)

\#51 通过该条ACKNACK，发布者告诉订阅者，它需要接收一个DATA(r) sample。之后，订阅者的subscriptions Writer就会发送一条DATA(r)，如#65。

![image13](image13.png)

\#65 在该条消息中，包含了topic name，type，Qos。 发送方式为reliable，因此也会发送一条HEARTBEAT，其中firstSeqNumber(1)，lastSeqNumber(1)表示订阅者期望发布者对接收到这一条DATA(r)消息进行确认。

![image14](image14.png)

![image15](image15.png)

\#67 发布者发送ACKNACK确认收到了DATA(r)。

![image16](image16.png)

**DataWrier和DataReader在相互发现后，开始通信：**

在这个demo中，在DataReader和DataWriter相互发现之前，DataWriter已经发布了一些samples。由于它们的durability被设置为VOLATILE，wrtiters需要让readers知道它们将接收不到第一个samples，通过给每个匹配的DataReaders发送GAP消息来实现。

![image17](image17.png)

![image18](image18.png)

#72 本demo中，仅包含一个DataWriter和一个DataReader，因此可以看到只有一条GAP消息，该条消息中bitmapBase的值为2表示，该DataWriter告知所有相匹配的DataReaders通信从sequence Number为2的sample开始。

![image19](image19.png)

\#74 这是第一条通信开始后发送的sample，可以看到SeqNumber为2，与#72中相对应。

![image20](image20.png)