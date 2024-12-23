---
title: vsomeip源码分析
date: 2024-04-17 14:45 +0800
last_modified_at: 2024-04-18 11:17 +0800
author: FeetingTimes
categories: ["开源库", "vsomeip"]
tags: ["c++", "someip", "autosar"]
pin: true
math: true
mermaid: true
img_path: /assets/img/vsomeip-code-analysis/
---

## 什么是SOME/IP？

**Scalable service-Oriented MiddlewarE over IP**，是指基于 IP 的可扩展的**面向服务**的中间件。

SOME/IP 协议采用 C/S（Client/Server）的通信架构，，我们把**请求服务**的 ECU 看成是 **Client**，而**提供服务**的 ECU 就是 **Server**。根据服务接口类型，使用远程服务调用（Remote Procedure Call）机制，通过数据序列化和反序列化（Serialization/Deserialization）使得数据得以在网络中传输。通过可用服务发现 SD（Service Discovery）机制来实现服务的动态配置。

由上可见，SOME/IP 主要可以提供以下功能：

1. **数据序列化和反序列化（Serialization/Deserialization）**：服务通信数据与二进制数据流之间的双向转换；
2. **可用服务发现 SD（Service Discovery）**：管理服务状态，发现和提供服务，动态配置 SOME/IP 报文发送；
3. **服务发布与订阅（Publish/Subscribe）**：管理服务的发布与订阅关系；
4. **远程服务调用（Remote Procedure Call）**：实现控制器（Client）使用网络内其他控制器（Server）提供的服务。

## SOME/IP-SD （服务发现协议）

### 什么是SOME/IP-SD？

SOME/IP-SD 协议是 SOME/IP 协议的一种，其中 SD 是服务发现 **Service Discovery** 的简称。基于服务的通信需要由 Server 和 Client 共同完成，因此在服务创建并且可用之后，Server 和 Client 需要通过 SOME/IP-SD **动态创建**两者之间的连接。Client 可以远程调用Server 提供的服务，或者订阅 Server 发布的内容，**但是**在 Client 调用服务或者订阅内容之前，**需要**知道 Server 提供哪些服务，这个过程就是通过服务发现来实现的。

SOME/IP-SD 是服务的**信息清单**及**管理机制**，主要实现**服务寻址**及**事件订阅**两种功能。对服务进行寻址时，服务提供者（Server 端）通过服务发现（SD）通知其他 ECU（Client 端）有哪些服务可用，并间接地通知该服务的**地址信息**（Server 端 **IP 地址、端口号、协议**），服务消费者（Client 端）了解到某服务的状态后，能够调用该服务的相关内容。

### SOME/IP-SD 的主要功能

SOME/IP-SD 有**两个**主要功能：**一是**应用程序之间传达自己的服务或获取对方的服务是否可用。**二是**向其他应用程序订阅服务，也就是 **Client** 端通过 **SOME/IP-SD** 对服务进行订阅，然后 **Server** 端用 **SOME/IP** 里的 Notification 类型消息发布订阅内容。

SOME/IP-SD 报文主要有以下几类：

1. **OfferService**：Server 的服务 Ready 并满足发布条件后，主动发出 OfferService报文，告知组播内其他节点该服务已经启动，可以创建连接。
2. **FindService**：当 Client 在网络中未收到相关的 **OfferService** 报文，或者暂时未收到，而 Client 又需要访问该服务，那么 Client 就可以发送 **FindService** 报文主动寻找服务，如果 Server 服务 Ready，会回复 **OfferService** 报文。
3. **StopOffserService**：当 Server 发现服务不可用，不满足服务发布条件时，会主动发送 **StopOffserService** 报文，告知组播内其他节点，该服务已不可用，停止服务支持。
4. **Subscribe**：事件组的交互采用 “订阅 - 发布” 机制，当 Client 收到OfferService 报文之后，通过发送 Subscribe 报文主动跟 Server 订阅相关事件组。
5. **SubscribeACK/SubscribeNACK**：当 Server 收到 Client 的订阅报文之后，**需要**先行判断该 Client 是否符合可订阅的条件，**如果**该 Client 满足事件组订阅条件，则发送**SubscribeACK**，告知 Client 订阅成功。此后当事件组内的事件准备就绪之后，Server 会以约定好的形式发送相关事件给成功订阅的 Client。而如果该 Client 不符合事件组订阅条件，Server 会直接回复 SubscribeNACK，告知订阅不成功。
6. **StopSubscribe**：当 Client 订阅某个事件组之后，发现后续并**不再需要**该事件组的数据了，可发送 **StopSubscribe** 报文向 Server 取消订阅相应事件。

SOME/IP 可用在 TCP/UDP 协议上，但是 SOME/IP-SD **只能用在 UDP 上**，因为SOME/IP-SD 只是为了发现服务，并不需要 TCP 的可靠性连接等特性。

### SOME/IP-SD 协议报文格式

SOME/IP-SD 协议的报文格式如下：

![SOMEIP-SD-Header-Format](SOMEIP-SD-Header-Format.png)

从上表可知，你会发现字段 Message ID、Protocol Version、Interface Version、Message Type、Return Code 的值都是固定的，SOME/IP-SD 的 Service ID 是 **0xFFFF**，Method ID 是 **0x8100** 。另外，SOME/IP-SD 的 Client ID 必须是 **0x0000**，因为每个设备上只有一个 SD Instance；Session ID 则是从 1 开始，逐次累加，直到翻转后再次从 1 开始（Session ID 不能设为 0）。单播和组播，**每种通信关系单独维护一个 Session ID，每组 发送-接收 关系组也单独维护一个 Session ID。**

#### Flags 的各个 Field 描述

其中这个 **8-bit Flags** 的各个位段格式如下：

|                     | Bit 7 （Reboot Flag）        | Bit 6（Unicast Flag） | Bit 5（Explicit initialdata control flag） | Bits[4:0] |
| ------------------- | ---------------------------- | --------------------- | ------------------------------------------ | --------- |
| Value & Description | 0 -> Session ID wraps around | 0 -> No Support       | 1（Shall always set to 1？？？）           | 0         |
|                     | 1 -> Reboot                  | 1 -> Support          |                                            |           |

#### Entry Type and Format

存在 2 种类型的 Entry：**Service** 和 **Eventgroup**，这两种 Entry 的长度都是 **16byte**。

两种类型 Entry 的**格式**如下：

- **Service**
  ![SOMEIP-SD-Service-Entry-Type](SOMEIP-SD-Service-Entry-Type.png)
- **Eventgroup**
  ![SOMEIP-SD-Eventgroup-Entry-Type](SOMEIP-SD-Eventgroup-Entry-Type.png)

上表中的 **X** 是一个 1-bit 的标志 **Initial data Requested Flag**，如果需要 Server 发送初始数据，就需要将其置 1 。

两种类型 Entry **各个位段的赋值**如下：

|               | Service                          | Eventgroup                                 |
| ------------- | -------------------------------- | ------------------------------------------ |
| **Type**      | **0x00**：`FindService`          | **0x06**：`（Stop）SubscribeEventgroup`    |
|               | **0x01**：`（Stop）OfferService` | **0x07**：`SubscribeEventgroupAck（NACK）` |
| Service ID    | X                                | X*                                         |
| Instance ID   | X                                | X*                                         |
| Major Version | X                                | X*                                         |
| TTL           | X**                              | X**                                        |
| Minor Version | X                                | -                                          |
| Counter       | -                                | X                                          |
| Eventgroup ID | -                                | X*                                         |

- *：All "F" are forbiden.
- **：TTL = 0 for Stop and NACK.
- Different Major Version are not compatible
- Different Minor Version are compatible
- Counter：Different Subscriptions

#### Options （选项的分类及其格式）

存在以下**选项**（Option）：

- **Configuration Option （0x01）**

  用来传送配置字符串（configuration strings）。

- **Load Balancing Option （0x02）**

  配合 **Offer Service Entry** 使用，用来给同一个服务（Service） 的多个实例（Instances）分配优先级（Prioriy）和权重（Weight）。优先级的值越小，代表优先级越高；权重值越大，代表在机会越大。

- **IPv4/IPv6 Endpoint Option （0x04/0x06）**

  由 **SOME/IP-SD Instance** 用来配合 **Offer Service Entry** 使用，指明服务提供者（Server）的本地 **Unicast** IPv4/IPv6 Address、本 Service 使用的传输协议和端口号。

- **IPv4/IPv6 Multicast Option （0x14/0x16）**

  由 **SOME/IP-SD Instance** 用来配合 **Subscribe Eventgroup Ack Entry** 使用，指明本服务（Service）使用的 **Multicast** IPv4/IPv6 Address、传输协议（**只能是** UDP）和端口号。

- **IPv4/IPv6 SD Endpoint Option （0x24/0x26）**

  当 SOME/IP-SD Instance 使用 **Unicast** IP Address 的时候，才会使用该 Option 来告诉对方本 SD Instance 的 Unicast IP Address，传输协议（**只能是** UDP）和端口号（**只能是** 30490）。该 Option **需要**放置在 Options Array 的第一个位置，**不能被任何 SD Entry 引用**（就是说能够在消息中附带该 Option，但不引用？），而且在所有的 SD Message 交互中，只能在某一条 SD Message 中附带该 Option 来发送一次（只能出现 1 次）。

各个类型的 SOME/IP-SD 协议报文都**只能携带特定的选项**（Option），如下表所示：

| Entry type              | Endpoint Options（IPv4 and IPv6） | Multicast Options（IPv4 and IPv6） | Configuration Option |
| ----------------------- | --------------------------------- | ---------------------------------- | -------------------- |
| FindService             |                                   |                                    | Allowed              |
| OfferService            | Allowed                           |                                    | Allowed              |
| StopOffserService       | Allowed                           |                                    | Allowed              |
| SubscribeEventgroup     | Allowed                           |                                    | Allowed              |
| StopSubscribeEventgroup | Allowed                           |                                    | Allowed              |
| SubscribeEventgroupAck  |                                   | Allowed                            | Allowed              |
| SubscribeEventgroupNACK |                                   |                                    | Allowed              |

**注意**：以下 Entries 只能通过单播（Unicast）发送：

- Subscribe Eventgroup
- Stop Subscribe Eventgroup
- Subscribe Eventgroup Ack
- Subscribe Eventgroup Nack

各种类型选项的格式如下：

1. **Configuration Option**
  ![SOMEIP-SD-Configuration-Option](SOMEIP-SD-Configuration-Option.png)
   -  Length = 16（**0x0010**）（其值是指本 option 除了 Length 和 type 字段以外，剩下的部分的长度） 
   - Type = **0x01**
   - [len]id=value （例如 "[5]abc=x"，其中 len = 5，代表后面表达式 "abc=5" 的字符长度）
   - Append a **0x00** uint8 at the end（在最后附上一个 0x00）

   示例如下：（其中的 [] 括号实际是不存在的，在此加上强调这个字节是个数字）
  ![SOMEIP-SD-Configuration-Option-Example](SOMEIP-SD-Configuration-Option-Example.png)

2. **Endpoint Option（IPv4）**
  ![SOMEIP-SD-IPv4-Endpoint-Option](SOMEIP-SD-IPv4-Endpoint-Option.png)
   - Length = 9（**0x0009**） （其值是指本 option 除了 Length 和 type 字段以外，剩下的部分的长度）
   - Type = **0x04**
   - L4-Proto = Level 4 Protocol
     - **0x06**：**TCP**
     - **0x11**：**UDP**

### SOME/IP-SD 各个阶段的通信行为

Server 服务端和 Client 客户端的通信行为包含几个阶段，如下图所示：

![SOMEIP-SD-Behavior](SOMEIP-SD-Behavior.png)

#### Server 服务端各个阶段的通信行为

服务端的通信行为如下图所示：

![SOMEIP-SD-Server-Behavior](SOMEIP-SD-Server-Behavior.png)

1. **Down Phase**

   在这个阶段，Service 是不可用的，即服务端无法提供服务。

2. **Initial Wait Phase**

   当服务准备完毕（Available）后，进入此阶段。如果此阶段收到 FindService 报文，服务端忽略此消息，不做任何处理。如果服务不可用了，将返回进入 Down Phase。此阶段需要定义时间参数 **INITIAL_DELAY_Min** 和 **INITIAL_DELAY_Max**，初始化时间取其之间的随机值，当定时器超时后，发送第一帧 **OfferService** 报文，标志着进入下一个阶段。

3. **Repetition Phase**

   为了让客户端快速找到有哪些 Service，此阶段重复发送 **OfferService** 报文，重复次数由 **REPETITIONS_MAX** 决定，而发送时间间隔以 **REPETITIONS_BASE_DELAY** 为基本时间，每发送一次，间隔时间是前一个间隔时间的 **2 倍**。

   如果收到某个客户端的 **FindService** 报文，不影响当前阶段的发送计数和计时，延迟一定时间（**REQUEST_RESPONSE_DELAY**）后，单独发送**单播**的 **OfferService** 给服务请求端。

   如果收到 **SubscribeEventgroup** 订阅事件组的报文，发送**单播 Ack/Nack**，启动此订阅Entry 的 TTL 计时器。如果收到 StopSubcribeEventgroup 停止订阅事件组的报文，停止此订阅 Entry 的 TTL计时器。

   如果收到 **StopSubcribeEventgroup** 停止订阅事件组的报文，停止此订阅 Entry 的 TTL

   计时器。

   如果服务不可用，离开此阶段进入到 **Down Phase**，并**组播**发送 **StopOfferService** 通知所有客户端。

4. **Main Phase**

   此阶段将周期性地发送 **OfferService**，周期时间为 **CYCLIC_OFFER_DELAY**。

   如果收到某个客户端的 **FindService** 报文，不影响发送计数，延迟一定时间（**REQUEST_RESPONSE_DELAY**）后，发送**单播**的 **OfferService** 给服务请求端。

   如果收到 **SubscribeEventgroup** 订阅事件组的报文，发送**单播 Ack/Nack**，启动此订阅Entry 的 TTL 计时器。

   如果收到 **StopSubcribeEventgroup** 停止订阅事件组的报文，停止此订阅 Entry 的 TTL计时器。

   如果服务不可用，离开此阶段进入到 **Down Phase**，并**组播**发送 **StopOfferService** 通知所有客户端。

服务端**状态机转换图**如下所示：

![SOMEIP-SD-Server-Behavior-State](SOMEIP-SD-Server-Behavior-State.png)

#### Client 客户端各个阶段的通信行为

Client 客户端的通信行为如下图所示：

![SOMEIP-SD-Client-Behavior](SOMEIP-SD-Client-Behavior.png)

1. **Down Phase**

   服务未被上层应用请求。

   收到 **OfferService** 报文，存储当前的服务实例状态，启动 TTL 计时器，此时服务若被

   上层应用请求，则直接进入 **Main Phase**。

2. **Initial Wait Phase**

   服务被上层应用请求后，进入此阶段。

   等待 **INITIAL_DELAY** 时间（最大和最小值之间的随机值）。

   如果此时收到 **OfferService** 报文，则取消等待计时器，直接进入 **Main Phase**。

   如果服务请求被上层应用释放，则进入 **Down Phase**。（在进入 Down Phase

   前，应该先关停等待计时器）

   等待计时器超时后，发送第一个 **FindService** 报文，进入下一阶段。

3. **Repetition Phase**

   重复发送 **FindService** 报文，重复次数由 **REPETITIONS_MAX** 决定，而发送时间间隔以**REPETITIONS_BASE_DELAY** 为基本时间，每发送一次，间隔时间是前一个间隔时间的 **2倍**。

   在重复发送 **FindService** 报文的过程中，一旦接收到了 **OfferService** 报文，则停止发送计数和计时器，立即进入 Main Phase，并在延迟一定时间后发送 **SubscribeEventgroup** 订阅事件组报文。

   如果在此阶段上层应用的服务请求被释放，则进入 **Down Phase**。若此时有订阅行为，则先发送 **StopSubcribeEventgroup** 报文后再进入 **Down Phase** 。（应该也需要停止 **FindService** 报文的发送计数和计时器。）

4. **Main Phase**

   不再周期地发送 **FindService** 报文。

   如果收到 **StopOfferService** 报文，则停止所有的计时器。

   如果上层应用的服务请求被释放，则进入 **Down Phase**。若此时有订阅行为，则先发

   送 **StopSubcribeEventgroup** 报文后再进入 **Down Phase** 。

Client 客户端的状态机转化图如下图所示：

![SOMEIP-SD-Client-Behavior-State](SOMEIP-SD-Client-Behavior-State.png)

## vsomeip -- SOME/IP 的开源实现

vsomeip 是 GENIVI 实现的开源 SOME/IP 库，由 C++ 编写，**目前主要实现**了 SOME/IP的**通信**和**服务发现**功能，并在此基础上增加了**少许的安全机制**。

GENIVI 是一个联盟组织，由 BMW 倡导，是**汽车信息娱乐领域系统软件标准**的倡导者，创建基于 linux 系统的 IVI 软件平台和操作系统。GENIVI 倡导了很多开源软件项目，比如：DLT、CommonAPI C++、VSOMEIP。

### vsomeip 概述

![VSOMEIP-Framework](VSOMEIP-Framework.png)

上图所示为 vsomeip 的基本框架。

如图所示，vsomeip **不仅**涵盖了**设备之间的 SOME/IP 通信**（外部通信），**还**涵盖了**内部进程间通信**。两个设备通过 communication endpoints（通信端点）进行通信，**endpoints**确定传输使用的协议（TCP 或 UDP）及端口号或其他参数。所有这些参数都是可以在vsomeip 配置文件中设置的（配置文件是 json 格式）。**内部通信**是通过本地 endpoints 完成的，这些 endpoints 由 unix 域套接字使用 **Boost.Asio 库**实现。由于这种内部通信不是通过中心组件 (例如 D-Bus 守护进程) 路由的，所以它非常快。

中央 vsomeip **路由管理器（routing manager）只接收**必须发送到外部设备的消息，**并分发**来自外部的消息。每个设备**只有一个**路由管理器，如果没有配置，那么第一个运行的 vsomeip 应用程序也会启动路由管理器。

### vsomeip 的源码分析

[sd模块源码分析](/posts/vsomeip-sd-code-analysis)