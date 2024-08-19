---
title: cyclonedds库的简单介绍
date: 2024-08-19 15:21 +0800
last_modified_at: 2024-08-19 15:21 +0800
author: AClumsyDog
categories: ["开源库", "cyclonedds"]
tags: ["c++", "cyclonedds"]
pin: true
math: true
mermaid: true
---

DDS（Data Distribution Service for Real-Time Systems）是一种用于分布式系统的中间件协议和API标准，由对象管理组织（Object Management Group, OMG）定义。DDS的主要目标是支持高效、可靠和实时的数据分发，特别适用于对低延迟和高吞吐量有严格要求的应用场景，如工业控制、军事系统、航空电子系统、智能交通系统以及物联网（IoT）应用。

### DDS的主要特点

1. **数据中心的模型**：DDS采用了“发布-订阅”模式，这意味着数据发布者和订阅者不直接通信，而是通过数据主题进行交互。发布者将数据发布到特定主题上，任何对该主题感兴趣的订阅者都会自动接收数据。这种方式使得系统更易于扩展和管理。

2. **质量服务（QoS）控制**：DDS支持多种QoS策略，允许开发人员定义数据传输的行为和约束条件。例如，可以设置数据的可靠性、优先级、延迟、持久性等。这使得DDS能够满足不同应用场景的需求，从严格的实时系统到较为宽松的非实时系统。

3. **分布式架构**：DDS设计为一个完全分布式的架构，没有集中式的服务器或单点故障。每个参与者（发布者或订阅者）可以独立地加入或离开网络，这使得系统具有高容错性和扩展性。

4. **自动发现**：DDS提供了内建的自动发现机制，使得新加入的节点能够自动发现并与其他节点建立通信，而无需手动配置。这极大地方便了系统的部署和管理。

5. **可扩展性**：DDS支持大规模系统，能够处理成千上万的节点和数据流。它可以跨越多个网络进行数据传输，适用于广域网和局域网。

6. **跨平台支持**：DDS是一个跨平台的标准，支持多种操作系统和编程语言，如C、C++、Java等。

### DDS的典型应用场景

- **工业自动化**：在复杂的工厂自动化系统中，DDS可以用于实时传输传感器数据和控制指令。
- **无人机控制**：在无人机群组中，DDS可以用来实时分发飞行控制数据，确保无人机之间的协同工作。
- **智能交通系统**：在智能交通系统中，DDS可以用于车辆和基础设施之间的数据交换，以实现交通流量优化和自动驾驶。
- **军事和国防**：在军事系统中，DDS可以用于分发实时战场数据，如目标跟踪和通信。

### DDS实现

DDS标准有多种实现，其中一些是开源的，一些是商业产品。著名的实现包括：
- **RTI Connext**：由Real-Time Innovations公司开发，是DDS领域最流行的商业实现之一。
- **OpenDDS**：一个开源的DDS实现，由Object Computing, Inc.开发。
- **Eclipse Cyclone DDS**：一个由Eclipse基金会支持的开源DDS实现，专注于物联网和嵌入式系统。

### 构建 Eclipse Cyclone DDS

要构建 Cyclone DDS，你需要一台运行 Linux、Mac 或 Windows 10 的机器（或者在某些情况下，可以使用 *BSD、QNX、OpenIndiana 或 Solaris 2.6），并在主机上安装以下工具：

  * C 编译器（最常见的是 Linux 上的 GCC，Windows 上的 Visual Studio，macOS 上的 Xcode）；
  * 可选：GIT 版本控制系统；
  * [CMake](https://cmake.org/download/)，版本 3.16 或更高；
  * 可选：[OpenSSL](https://www.openssl.org/)，我们推荐使用已完全修补和支持的版本，但 1.1.1 版本仍然可用；
  * 可选：[Eclipse Iceoryx](https://iceoryx.io) 2.0 版本，用于共享内存和零拷贝支持；
  * 可选：[Bison](https://www.gnu.org/software/bison/) 解析器生成器。缓存的源代码已检查到存储库中。

如果你想玩转解析器，你需要安装 bison 解析器生成器。在 Ubuntu 上，`apt install bison` 应该就能安装它。在 Windows 上，安装 Chocolatey 并运行 `choco install winflexbison3` 也能满足需求。在 macOS 上，`brew install bison` 是最简单的方式。

要获取 Eclipse Cyclone DDS，执行以下命令：

```bash
$ git clone https://github.com/eclipse-cyclonedds/cyclonedds.git
$ cd cyclonedds
$ mkdir build
```

根据你是想使用 Cyclone DDS 开发应用程序还是为其做贡献，你可以按照不同的步骤进行操作：

#### 构建配置

除了标准选项如 `CMAKE_BUILD_TYPE` 之外，还有一些通过 CMake 定义的配置选项：

* `-DBUILD_EXAMPLES=ON`：构建包含的示例
* `-DBUILD_TESTING=ON`：构建测试套件（强制从库中导出所有符号）
* `-DBUILD_IDLC=NO`：禁用构建 IDL 编译器（影响构建示例、测试和 `ddsperf`）
* `-DBUILD_DDSPERF=NO`：禁用构建 [`ddsperf`](https://github.com/eclipse-cyclonedds/cyclonedds/tree/master/src/tools/ddsperf) 性能测试工具
* `-DENABLE_SSL=NO`：不查找 OpenSSL，移除 TLS/TCP 支持并避免构建实现认证和加密的插件（默认是 `AUTO`，如果找到 OpenSSL 会启用它们）
* `-DENABLE_ICEORYX=NO`：不查找 Iceoryx，禁用构建 PSMX Iceoryx 插件（默认是 `AUTO`，如果找到 Iceoryx 会启用它）
* `-DENABLE_SECURITY=NO`：不构建核心代码中的安全接口和挂钩，也不构建插件（在没有 OpenSSL 的情况下可以启用安全性，只需在其他地方找到插件即可）
* `-DENABLE_LIFESPAN=NO`：排除有限生命周期 QoS 的支持
* `-DENABLE_DEADLINE_MISSED=NO`：排除有限截止日期 QoS 设置的支持
* `-DENABLE_TYPELIB=NO`：排除类型库的支持，也需要禁用类型和主题发现，使用 `-DENABLE_TYPE_DISCOVERY=NO` 和 `-DENABLE_TOPIC_DISCOVERY=NO`
* `-DENABLE_TYPE_DISCOVERY=NO`：排除类型发现和类型兼容性检查的支持（实际上是大部分 XTypes），也需要禁用主题发现，使用 `-DENABLE_TOPIC_DISCOVERY=NO`
* `-DENABLE_TOPIC_DISCOVERY=NO`：排除主题发现的支持
* `-DENABLE_SOURCE_SPECIFIC_MULTICAST=NO`：禁用特定源多播的支持（在 QNX 构建中可能需要禁用此项和 `-DENABLE_IPV6=NO`）
* `-DENABLE_IPV6=NO`：禁用 IPv6 支持（在 QNX 构建中可能需要禁用此项和 `-DENABLE_SOURCE_SPECIFIC_MULTICAST=NO`）
* `-DBUILD_IDLC_XTESTS=NO`：包括一组 IDL 编译器的测试，这些测试使用 C 后端在（测试）运行时编译一个 idl 文件，并使用 C 编译器构建一个测试应用程序来执行实际测试（不支持 Windows）
* `-DENABLE_QOS_PROVIDER=NO`：禁用 QoS 提供程序的支持

#### 针对应用程序开发者

要构建并安装使用 Cyclone DDS 开发你自己的应用程序所需的库，只需执行以下几步。在 Linux 和 macOS 与 Windows 之间有一些小差异。
对于 Linux 或 macOS：

```bash
$ cd build
$ cmake -DCMAKE_INSTALL_PREFIX=<install-location> ..
$ cmake --build .
```

对于 Windows：

```bash
$ cd build
$ cmake -G "<generator-name>" -DCMAKE_INSTALL_PREFIX=<install-location> ..
$ cmake --build .
```

其中，`<install-location>` 表示你希望安装 Cyclone DDS 的目录，`<generator-name>` 表示 CMake [生成器](https://cmake.org/cmake/help/latest/manual/cmake-generators.7.html) 提供的生成构建文件的方式之一。例如，"Visual Studio 15 2017 Win64" 将针对使用 Visual Studio 2017 的 64 位构建。

成功构建后，执行以下命令进行安装：

```bash
$ cmake --build . --target install
```

这将会将文件复制到以下位置：
  * `<install-location>/lib`
  * `<install-location>/bin`
  * `<install-location>/include/ddsc`
  * `<install-location>/share/CycloneDDS`

根据安装位置的不同，你可能需要管理员权限。

此时，你就可以在自己的项目中使用 Eclipse Cyclone DDS 了。

请注意，默认的构建类型是包含调试信息的发布版本（RelWithDebInfo），这通常是应用程序最方便使用的构建类型，因为它在性能和调试能力之间提供了良好的平衡。如果你更喜欢调试（Debug）或纯发布（Release）版本，请相应地设置 `CMAKE_BUILD_TYPE`。

#### 为 Eclipse Cyclone DDS 做贡献

我们非常欢迎对项目的所有贡献，无论是问题、示例、错误修复、增强功能、文档改进，还是其他任何方面的贡献。
如果考虑贡献代码，了解一下仓库中存在的 Azure pipelines 构建配置可能会有所帮助，并且有一个使用简单测试框架和 CTest 构建的测试套件，如果需要，可以在本地构建。
要构建它，请在配置时将 cmake 变量 `BUILD_TESTING` 设置为 on，例如：

```bash
$ cd build
$ cmake -DCMAKE_BUILD_TYPE=Debug -DBUILD_TESTING=ON ..
$ cmake --build .
$ ctest
```

### 构建 Eclipse Cyclone DDS C++ 绑定

要构建 Cyclone DDS 的 C++ 绑定，你需要一台运行 Linux、Mac 或 Windows 10 的机器（或者在某些情况下，可以使用 *BSD、OpenIndiana），并在主机上安装以下工具：

 * C 和 C++ 编译器（最常见的是 Linux 上的 GCC，Windows 上的 Visual Studio，macOS 上的 Xcode）；
 * [Git](https://git-scm.com/) 版本控制系统；
 * [CMake](https://cmake.org/download/)，版本 3.16 或更高；
 * [Eclipse Cyclone DDS](https://github.com/eclipse-cyclonedds/cyclonedds/)

*Eclipse Cyclone DDS* 有一些自己的依赖项，最重要的是 Bison。要构建和安装它，请参考构建说明。通过指定 `CMAKE_INSTALL_PREFIX` 确保将项目安装到方便的位置。

要获取 Cyclone DDS 的 C++ 绑定，执行以下命令：

```bash
$ git clone https://github.com/eclipse-cyclonedds/cyclonedds-cxx.git
$ cd cyclonedds-cxx
$ mkdir build
```

根据你是想使用 Cyclone DDS 的 C++ 绑定开发应用程序还是为其做贡献，你可以遵循不同的操作步骤。

#### 构建配置

除了标准选项如 `CMAKE_BUILD_TYPE` 之外，还有一些通过 CMake 定义的配置选项：

* `-DBUILD_DDSLIB=OFF`：禁用 DDS 库构建，在只需要生成器并使用其他构建的 DDS 库的交叉编译场景中有用。
* `-DBUILD_IDLLIB=OFF`：禁用 IDL 预处理库构建
* `-DBUILD_DOCS=ON`：构建文档
* `-DBUILD_TESTING=ON`：构建测试树
* `-DBUILD_EXAMPLES=ON`：构建示例
* `-DENABLE_LEGACY=YES`：启用旧版 C++11 模式，添加 Boost 作为依赖项（否则使用 C++17）
* `-DENABLE_ICEORYX=YES`：启用 Iceoryx 测试
* `-DENABLE_TYPELIB=YES`：启用类型库支持
* `-DENABLE_TOPIC_DISCOVERY=YES`：启用主题发现支持
* `-DENABLE_COVERAGE=YES`：启用覆盖构建
* `-DENABLE_QOS_PROVIDER=YES`：启用 QoS 提供程序支持

#### 针对应用程序开发者

要构建并安装使用 Cyclone DDS 的 C++ 绑定开发你自己的应用程序所需的库，只需执行以下几步。在 Linux 和 macOS 与 Windows 之间有一些小差异。

对于 Linux 或 macOS：

```bash
$ cd build
$ cmake -DCMAKE_INSTALL_PREFIX=<install-location> \
        -DCMAKE_PREFIX_PATH="<cyclonedds-install-location>" \
        ..
$ cmake --build .
```

对于 Windows：

```bash
$ cd build
$ cmake -G "<generator-name>" \
        -DCMAKE_INSTALL_PREFIX=<install-location> \
        -DCMAKE_PREFIX_PATH="<cyclonedds-install-location>" \
        ..
$ cmake --build .
```

其中，`<install-location>` 表示你希望安装 Cyclone DDS C++ 绑定的目录，`<generator-name>` 表示 CMake [生成器](https://cmake.org/cmake/help/latest/manual/cmake-generators.7.html) 提供的生成构建文件的方式之一。例如，"Visual Studio 15 2017 Win64" 将针对使用 Visual Studio 2017 的 64 位构建。

成功构建后，执行以下命令进行安装：

```bash
$ cmake --build . --target install
```

这将会将文件复制到以下位置：

 * `<install-location>/lib`
 * `<install-location>/bin`
 * `<install-location>/include/ddsc`
 * `<install-location>/share/CycloneDDS-CXX`

根据安装位置的不同，你可能需要管理员权限。

此时，你就可以在自己的项目中使用 Eclipse Cyclone DDS 了。

请注意，默认的构建类型是包含调试信息的发布版本（RelWithDebInfo），这通常是应用程序最方便使用的构建类型，因为它在性能和调试能力之间提供了良好的平衡。如果你更喜欢调试（Debug）或纯发布（Release）版本，请相应地设置 `CMAKE_BUILD_TYPE`。

#### 为 Eclipse Cyclone DDS 做贡献

我们非常欢迎对项目的所有贡献，无论是问题、示例、错误修复、增强功能、文档改进，还是其他任何方面的贡献。如果考虑贡献代码，了解一下仓库中存在的 Travis CI 和 AppVeyor 的构建配置可能会有所帮助，并且有一个使用 CTest 和 Google Test 构建的测试套件，如果需要，可以在本地构建。要构建它，请在配置时将 CMake 变量 `BUILD_TESTING` 设置为 on，例如：

```bash
$ cd build
$ cmake -DCMAKE_BUILD_TYPE=Debug -DBUILD_TESTING=ON ..
$ cmake --build .
$ ctest
```

这样的构建需要安装 [Google Test](https://github.com/google/googletest)。你需要自己安装它。