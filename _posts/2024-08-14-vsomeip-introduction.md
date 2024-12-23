---
title: vsomeip库的简单介绍
date: 2024-08-14 15:21 +0800
last_modified_at: 2024-08-14 15:21 +0800
author: FeetingTimes
categories: ["开源库", "vsomeip"]
tags: ["c++", "vsomeip"]
pin: true
math: true
mermaid: true
---

## VSOMEIP 介绍

VSOMEIP 是一个 C++ 实现的开源框架，旨在支持汽车行业中的服务导向中间件（SOME/IP）通信协议。它允许在分布式系统中开发和部署基于服务的应用程序，特别是在汽车的电子控制单元（ECUs）之间进行通信。

## 主要特点
- **服务发现：** 动态服务发现机制，使得ECUs能够发现网络中提供的服务，并建立通信。
- **服务通信：** 提供了请求/响应（Request/Response）模式以及事件通知（Event Notification）模式，适合不同类型的应用场景。
- **多平台支持：** VSOMEIP 可在多个操作系统上运行，包括 Linux 和 QNX。
- **可配置性：** 提供灵活的配置选项，通过配置文件定义通信的具体参数，如端口、服务ID等。
- **高效的网络资源使用：** 通过优化的协议设计，最大限度地减少网络流量和延迟。

## 典型应用场景
- **汽车内部通信：** 在汽车内部的不同ECUs之间进行服务导向的通信，例如自动驾驶系统、车身控制系统等。
- **跨厂商协作：** 由于VSOMEIP是一个开源项目，汽车制造商、供应商和开发者都可以使用和扩展它，确保不同厂商的设备能够互操作。

## 优势
- **开源性：** 作为开源项目，VSOMEIP 可以被自由使用和修改，促进了在汽车行业中的广泛采用和协作开发。
- **符合标准：** 完全符合 AUTOSAR SOME/IP 规范，确保与其他基于 SOME/IP 的系统兼容。
- **社区支持：** 拥有活跃的开发者社区，持续改进和更新。

VSOMEIP 的开发使得汽车行业中的不同厂商能够基于统一的通信协议进行开发，降低了集成和兼容性问题，是现代智能汽车系统开发中的重要工具之一。

## 如何编译VSOMEIP

编译 VSOMEIP 需要一些基本的准备工作，包括设置依赖项和配置编译环境。以下是编译 VSOMEIP 的步骤：

### 1. 环境准备

#### 安装依赖项
VSOMEIP 依赖一些库和工具，需要提前安装。这些依赖项包括：
- **Boost**（通常需要 Boost 1.55 或更高版本）
- **CMake**（建议版本 3.0 或更高）
- **gcc**（需要支持c++14，需要gcc >= 6.1）

可以使用以下命令安装这些依赖项：

```bash
sudo apt-get update
sudo apt-get install build-essential cmake gcc
```

boost库的编译安装可参考[boost库的简单介绍](/posts/boost-introduction)，vsomeip只需要system,thread,filesystem这三个模块。
如果需要运行 VSOMEIP 的测试用例，则需要下载gtest（需要1.7.0版本）

### 2. 获取 VSOMEIP 源代码

从 GitHub 仓库克隆 VSOMEIP 源代码：

```bash
git clone https://github.com/COVESA/vsomeip.git
cd vsomeip
```

### 3. 配置编译

在进行编译之前，需要使用 `CMake` 配置编译环境。首先，在项目目录中创建一个 `build` 目录：

```bash
mkdir build
cd build
```

然后，运行 `CMake` 命令以生成 Makefile：

```bash
cmake .. -DCMAKE_INSTALL_PREFIX=~/vsomeip_output \
         -CMAKE_PREFIX_PATH=/home/test/boost-output \
         -DVSOMEIP_INSTALL_ROUTINGMANAGERD=ON \
         -DGTEST_ROOT=/home/test/workspace/googletest \
         -DENABLE_SIGNAL_HANDLING=ON \
         -DENABLE_SESSION_HANDLING_CONFIG=ON \
         -DTEST_IP_MASTER=172.17.0.6 \
         -DTEST_IP_SLAVE=172.17.0.2 \
         -DTEST_IP_SLAVE_SECOND=172.17.0.3 \
         -DTEST_UID=1000 \
         -DTEST_GID=1000
```
此命令将检查所有依赖项，并生成编译所需的配置文件。下面对这些命令行参数进行说明：
- **CMAKE_INSTALL_PREFIX**：vsomeip的安装路径。
- **VSOMEIP_INSTALL_ROUTINGMANAGERD**：否安装routingmanagerd。
- **NABLE_SIGNAL_HANDLING**：是否启用信号处理。
- **ENABLE_SESSION_HANDLING_CONFIG**：是否启用会话配置，如果需要需要测试e2e的p04，必须得开启这个选项。
- **TEST_IP_MASTER**：测试的ip地址，用于client侧。
- **TEST_IP_SLAVE**：测试的ip地址，用于server侧。
- **TEST_UID和TEST_GID**：测试的用户的uid和gid。
- **CMAKE_PREFIX_PATH**：cmake优先搜索路径，用于指定boost库的安装路径。
- **GTEST_ROOT**：gtest源码所存放的路径。

### 4. 编译 VSOMEIP

在配置成功之后，使用 `make` 命令进行编译：

```bash
make -j4
```

编译过程可能需要一些时间，具体取决于系统性能和配置。如果一切顺利，编译完成后将生成 VSOMEIP 库和可执行文件。

### 5. 安装 VSOMEIP（可选）

如果你想将 VSOMEIP 安装到指定目录中，可以使用以下命令：

```bash
make install
```

### 6. 常见问题解决

- **Boost 版本不兼容：** 确保安装的 Boost 版本与 VSOMEIP 需求一致，对于v3.1.20.3版本的 Vsomeip 不支持高于v1.74.0的 Bosst 库。