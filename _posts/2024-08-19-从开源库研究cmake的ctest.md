---
title: 从开源库研究cmake的ctest
date: 2024-08-19 10:55 +0800
last_modified_at: 2024-08-19 10:55 +0800
author: fleetime0
categories: ["cmake"]
tags: ["c++", "cmake"]
pin: true
math: true
mermaid: true
---

`CTest` 是 CMake 的一个组件，用于测试 CMake 项目中的代码。`CTest` 提供了一个简单的框架来运行单元测试、集成测试或其他类型的自动化测试，并报告测试结果。它与 CMake 结合得很好，可以无缝集成到构建过程中。以下是关于 `CTest` 的一些关键点和用法说明：

### 1. 启用 CTest
要使用 `CTest`，你首先需要在 `CMakeLists.txt` 文件中启用它。只需要添加以下一行代码：

```cmake
enable_testing()
```

这将告诉 CMake 这个项目使用 `CTest` 来运行测试。

### 2. 添加测试
在 CMake 中定义测试可以使用 `add_test()` 命令。该命令的基本语法如下：

```cmake
add_test(NAME <test_name> COMMAND <executable> [<arg1> <arg2> ...])
```

- `NAME <test_name>`：为测试指定一个名称。
- `COMMAND <executable>`：指定要运行的可执行文件或命令。
- `[<arg1> <arg2> ...]`：可选的，传递给可执行文件的命令行参数。

例如：

```cmake
add_executable(my_test my_test.cpp)
add_test(NAME MyTest COMMAND my_test)
```

这将编译 `my_test.cpp` 并将生成的可执行文件作为 `MyTest` 测试。

### 3. 运行测试
配置并生成项目后，可以使用 `ctest` 命令运行测试。运行 `ctest` 通常有以下几种方式：

- **在构建目录中运行所有测试**：
  ```bash
  ctest
  ```

- **运行指定的测试**：
  ```bash
  ctest -R MyTest
  ```
  `-R` 选项后面跟的是测试名称的正则表达式匹配。

- **查看详细输出**：
  ```bash
  ctest -V
  ```

### 4. 设置测试属性
`CTest` 允许你为测试设置一些属性。例如，你可以设置一个测试的超时时间：

```cmake
set_tests_properties(MyTest PROPERTIES TIMEOUT 10)
```

这个命令将 `MyTest` 测试的超时时间设置为 10 秒。如果测试在 10 秒内没有完成，它将被标记为失败。

### 5. 处理测试结果
`CTest` 可以与 CI（持续集成）系统集成，并生成不同格式的测试报告，例如 XML 格式，供 CI 系统使用：

```bash
ctest -D ExperimentalTest
```

这将生成 `CTest` 的测试结果，并可以进一步上传到 CI 系统或测试仪表板。

### 6. 与 Google Test 的集成
如果你使用 Google Test，你可以使用 `gtest_add_tests()` 命令来自动发现和添加所有 Google Test 的测试用例，而不需要手动为每个测试用例使用 `add_test()`。

```cmake
gtest_add_tests(TARGET my_test_target)
```

这个命令将自动扫描 Google Test 测试，并将它们添加到 `CTest` 中。

### 7. 自定义 CTest 的行为
你还可以通过 `CTest` 的配置文件 `CTestConfig.cmake` 和 `CTestCustom.cmake` 自定义 `CTest` 的行为。你可以配置如何查找测试，如何报告测试结果，甚至定义一些自定义的测试处理逻辑。

### 总结
`CTest` 是一个功能强大的工具，能够帮助你在 CMake 项目中轻松实现自动化测试。通过 `CTest` 的集成，你可以确保在代码变更后，项目的各项功能都能正常工作，并且可以将测试流程无缝集成到你的构建和发布流程中。

如果你有特定的需求或问题，比如如何处理测试失败或如何与 CI 系统集成，欢迎进一步探讨。

## vsomeip中ctest的使用

在vsomeip的主CMakeLists.txt文件中，首先进行了一些测试相关的设置：

```cmake
enable_testing()
SET(TESTS_BAT "OFF" CACHE BOOL
    "Controls whether only BAT tests should be build or not")
SET(TEST_SYMLINK_CONFIG_FILES "OFF" CACHE BOOL
    "Controls if the json and scripts needed needed to run the tests are copied or symlinked into the build directroy (ignored on Windows)")
SET(TEST_SYMLINK_CONFIG_FILES_RELATIVE "OFF" CACHE BOOL
    "Controls if the json and scripts needed needed to run the tests are symlinked relatively into the build directroy (ignored on Windows)")

SET(TEST_IP_DEFAULT_VALUE "XXX.XXX.XXX.XXX")
SET(TEST_IP_MASTER "${TEST_IP_DEFAULT_VALUE}" CACHE STRING
    "The IP address of the interface which will act as test master")
SET(TEST_IP_SLAVE "${TEST_IP_DEFAULT_VALUE}" CACHE STRING
    "The IP address of the interface which will act as test slave")

if((${TEST_IP_MASTER} STREQUAL ${TEST_IP_DEFAULT_VALUE}) OR
   (${TEST_IP_SLAVE}  STREQUAL ${TEST_IP_DEFAULT_VALUE}))
    message(WARNING "TEST_IP_MASTER and/or TEST_IP_SLAVE isn't set. "
                    "Only local tests will be runnable "
                    "Please specify them via for example "
                    "-DTEST_IP_MASTER=10.0.3.1 -DTEST_IP_SLAVE=10.0.3.125")
endif()

SET(TEST_IP_SLAVE_SECOND "${TEST_IP_DEFAULT_VALUE}" CACHE STRING
    "The second IP address of the interface which will act as test slave")
set(TEST_SECOND_ADDRESS "OFF" CACHE BOOL
    "Controls whether second address tests should run or not")

if(${TEST_IP_SLAVE_SECOND} STREQUAL ${TEST_IP_DEFAULT_VALUE})
    message(WARNING "TEST_IP_SLAVE_SECOND isn't set. "
                    "Test with more than one IP address on same interface is not enabled."
                    "Please specify them via for example "
                    "-TEST_IP_SLAVE_SECOND=10.0.3.126")
else()
    set(TEST_SECOND_ADDRESS "ON")
endif()

set(TEST_E2E_PROFILE_04 "OFF" CACHE BOOL
    "Controls whether E2E Profile 04 tests should run or not")
if (ENABLE_SESSION_HANDLING_CONFIG)
    set(TEST_E2E_PROFILE_04 "ON")
else ()
    message(WARNING "ENABLE_SESSION_HANDLING_CONFIG isn't set. "
                    "Test of E2E Profile 04 is not enabled.")
endif ()


SET(TEST_UID_DEFAULT_VALUE "123456789")
SET(TEST_UID "${TEST_UID_DEFAULT_VALUE}" CACHE STRING
    "The User ID of the user running the test: Needed for security")
SET(TEST_GID_DEFAULT_VALUE "123456789")
SET(TEST_GID "${TEST_GID_DEFAULT_VALUE}" CACHE STRING
    "The Group ID of the user running the test: Needed for security")

SET(TEST_SECURITY "ON" CACHE BOOL
    "Controls whether security tests should run or not")

if((${TEST_UID} STREQUAL ${TEST_UID_DEFAULT_VALUE}) OR
   (${TEST_GID}  STREQUAL ${TEST_GID_DEFAULT_VALUE}))
    message(WARNING "TEST_UID and/or TEST_GID isn't set. "
                    "Security Tests are not runnable "
                    "Please specify them for example "
                    "-DTEST_UID=1000 -DTEST_GID=1000")
    SET(TEST_SECURITY "OFF")
endif()
```

然后vsomeip又进行了一下操作：
```cmake
add_custom_target(build_tests)
add_dependencies(build_tests vsomeip3)
add_dependencies(build_tests vsomeip3-sd)

set(CMAKE_CTEST_COMMAND ctest -V)
add_custom_target(check COMMAND ${CMAKE_CTEST_COMMAND})

add_dependencies(check build_tests)
```

这一段 CMake 代码的主要作用是定义两个自定义目标 `build_tests` 和 `check`，并管理它们之间的依赖关系。下面是逐行解析：

1. `add_custom_target(build_tests)`
```cmake
add_custom_target(build_tests)
```
这行代码定义了一个名为 `build_tests` 的自定义目标。

2. `add_dependencies(build_tests vsomeip3)`
```cmake
add_dependencies(build_tests vsomeip3)
add_dependencies(build_tests vsomeip3-sd)
```
这两行代码为 `build_tests` 目标添加了依赖项 `vsomeip3` 和 `vsomeip3-sd`。这意味着在执行 `build_tests` 目标时，CMake 会确保 `vsomeip3` 和 `vsomeip3-sd` 目标先被构建。

- `vsomeip3` 和 `vsomeip3-sd` 应该是项目中的其他目标，可能是某些库或者可执行文件。

3. `set(CMAKE_CTEST_COMMAND ctest -V)`
```cmake
set(CMAKE_CTEST_COMMAND ctest -V)
```
这行代码设置了一个 CMake 变量 `CMAKE_CTEST_COMMAND`，它包含了运行 `ctest` 的命令。`-V` 选项表示 `ctest` 会在运行时输出详细信息。这一变量稍后会在定义 `check` 目标时使用。

4. `add_custom_target(check COMMAND ${CMAKE_CTEST_COMMAND})`
```cmake
add_custom_target(check COMMAND ${CMAKE_CTEST_COMMAND})
```
这行代码定义了另一个自定义目标 `check`，它的任务是执行 CMake 的测试工具 `ctest`。当你运行 `make check` 或 `cmake --build . --target check` 时，它将运行 `ctest` 命令，并显示详细的测试输出。

5. `add_dependencies(check build_tests)`
```cmake
add_dependencies(check build_tests)
```
这行代码添加了 `check` 目标对 `build_tests` 目标的依赖关系。这意味着在执行 `check` 目标之前，`build_tests` 目标将会先被执行。由于 `build_tests` 依赖 `vsomeip3` 和 `vsomeip3-sd`，所以 `check` 目标最终依赖于 `vsomeip3` 和 `vsomeip3-sd` 的构建。

然后，vsomeip就在cmake中添加了测试代码目录（`add_subdirectory( test EXCLUDE_FROM_ALL )`），这里使用了`EXCLUDE_FROM_ALL`排除在默认的构建中。

在test目录的CMakeLists.txt中，首先是引入了gtest的头文件和gtest库。

```cmake
include_directories(
    .
    ${gtest_SOURCE_DIR}/include
)

set(TEST_LINK_LIBRARIES gtest)
```

随后定义了名为copy_to_builddir的cmake函数，该函数的功能是将测试代码拷贝到构建目录。

```cmake
function(copy_to_builddir SOURCE_PATH DESTINATION_PATH TARGET_TO_DEPEND)
    if(${CMAKE_SYSTEM_NAME} MATCHES "Windows" OR NOT ${TEST_SYMLINK_CONFIG_FILES})
        ADD_CUSTOM_COMMAND(
            OUTPUT "${DESTINATION_PATH}"
            COMMAND ${CMAKE_COMMAND} -E copy "${SOURCE_PATH}" "${DESTINATION_PATH}"
            DEPENDS "${SOURCE_PATH}"
            COMMENT "Copying \"${SOURCE_PATH}\" into build directory"
        )
    else()
        if(${TEST_SYMLINK_CONFIG_FILES_RELATIVE})
            ADD_CUSTOM_COMMAND(
                OUTPUT "${DESTINATION_PATH}"
                # Create a relative link
                COMMAND ln -s -r "${SOURCE_PATH}" "${DESTINATION_PATH}"
                DEPENDS "${SOURCE_PATH}"
                COMMENT "Symlinking \"${SOURCE_PATH}\" into build directory"
            )
        else()
            ADD_CUSTOM_COMMAND(
                OUTPUT "${DESTINATION_PATH}"
                # Create an absolute link
                COMMAND ${CMAKE_COMMAND} -E create_symlink "${SOURCE_PATH}" "${DESTINATION_PATH}"
                DEPENDS "${SOURCE_PATH}"
                COMMENT "Symlinking \"${SOURCE_PATH}\" into build directory"
            )
        endif()
    endif()
    # Add a random number to the end of the string to avoid problems with
    # duplicate filenames
    set(FILENAME "")
    get_filename_component(FILENAME ${SOURCE_PATH} NAME )
    string(RANDOM LENGTH 4 ALPHABET 0123456789 RANDOMNUMBER)
    if (${CMAKE_SYSTEM_NAME} MATCHES "Windows")
        ADD_CUSTOM_TARGET(copy_${FILENAME}_${RANDOMNUMBER}
            DEPENDS "${DESTINATION_PATH}"
        )
        ADD_DEPENDENCIES(${TARGET_TO_DEPEND} copy_${FILENAME}_${RANDOMNUMBER})
    else()
        ADD_CUSTOM_TARGET(symlink_${FILENAME}_${RANDOMNUMBER}
            DEPENDS "${DESTINATION_PATH}"
        )
        ADD_DEPENDENCIES(${TARGET_TO_DEPEND} symlink_${FILENAME}_${RANDOMNUMBER})
    endif()
endfunction()

```

这个 CMake 函数 `copy_to_builddir` 用于在构建过程中将文件从源路径复制或链接到目标构建目录。函数根据操作系统类型和配置选项选择具体的操作方式，避免重复文件名问题，并为目标添加依赖关系。以下是函数的详细分析：

### 函数参数

- **`SOURCE_PATH`**: 要复制或链接的源文件路径。
- **`DESTINATION_PATH`**: 复制或链接到的目标路径。
- **`TARGET_TO_DEPEND`**: 目标文件依赖的构建目标。

### 逻辑分析

#### 1. 判断系统类型和配置项
```cmake
if(${CMAKE_SYSTEM_NAME} MATCHES "Windows" OR NOT ${TEST_SYMLINK_CONFIG_FILES})
```
- 如果系统是 Windows，或者配置项 `TEST_SYMLINK_CONFIG_FILES` 为 `OFF`，则选择复制文件的方式（Windows 系统通常不使用符号链接，除非特别配置）。
- 否则，将使用符号链接。

#### 2. 处理文件复制
```cmake
ADD_CUSTOM_COMMAND(
    OUTPUT "${DESTINATION_PATH}"
    COMMAND ${CMAKE_COMMAND} -E copy "${SOURCE_PATH}" "${DESTINATION_PATH}"
    DEPENDS "${SOURCE_PATH}"
    COMMENT "Copying \"${SOURCE_PATH}\" into build directory"
)
```
- 如果选择了文件复制，使用 `CMake` 命令 `-E copy` 进行文件复制操作。
- 生成的目标文件路径是 `DESTINATION_PATH`，依赖于 `SOURCE_PATH`，并在控制台输出复制的注释信息。

#### 3. 处理符号链接
根据配置项 `TEST_SYMLINK_CONFIG_FILES_RELATIVE` 判断是否创建相对符号链接：
- **相对链接**:
  ```cmake
  COMMAND ln -s -r "${SOURCE_PATH}" "${DESTINATION_PATH}"
  ```
  使用 `ln -s -r` 创建一个相对符号链接。

- **绝对链接**:
  ```cmake
  COMMAND ${CMAKE_COMMAND} -E create_symlink "${SOURCE_PATH}" "${DESTINATION_PATH}"
  ```
  使用 CMake 提供的 `create_symlink` 命令创建一个绝对符号链接。

#### 4. 避免文件名冲突
```cmake
string(RANDOM LENGTH 4 ALPHABET 0123456789 RANDOMNUMBER)
```
- 生成一个 4 位长度的随机数，避免文件名冲突，尤其是多个目标依赖相同源文件的情况下。

#### 5. 为目标添加依赖关系
根据系统类型，创建不同的目标并添加依赖关系：
- **Windows**:
  ```cmake
  ADD_CUSTOM_TARGET(copy_${FILENAME}_${RANDOMNUMBER}
      DEPENDS "${DESTINATION_PATH}"
  )
  ADD_DEPENDENCIES(${TARGET_TO_DEPEND} copy_${FILENAME}_${RANDOMNUMBER})
  ```
  - 定义一个自定义目标 `copy_${FILENAME}_${RANDOMNUMBER}`，它依赖于 `DESTINATION_PATH`。
  - 将该自定义目标添加为 `TARGET_TO_DEPEND` 的依赖项。

- **其他系统**:
  ```cmake
  ADD_CUSTOM_TARGET(symlink_${FILENAME}_${RANDOMNUMBER}
      DEPENDS "${DESTINATION_PATH}"
  )
  ADD_DEPENDENCIES(${TARGET_TO_DEPEND} symlink_${FILENAME}_${RANDOMNUMBER})
  ```
  - 定义一个自定义目标 `symlink_${FILENAME}_${RANDOMNUMBER}`，它依赖于 `DESTINATION_PATH`。
  - 将该自定义目标添加为 `TARGET_TO_DEPEND` 的依赖项。

随后vsomeip针对不同的选项，编译test代码，以及调用上面定义的copy_to_builddir函数将配置文件以及脚本拷贝到指定目录。

```cmake
if(NOT ${TESTS_BAT} AND ${TEST_E2E_PROFILE_04})
    set(TEST_E2E_PROFILE_04_NAME e2e_profile_04_test)
    set(TEST_E2E_PROFILE_04_SERVICE e2e_profile_04_test_service)
    set(TEST_E2E_PROFILE_04_CLIENT e2e_profile_04_test_client)

    add_executable(${TEST_E2E_PROFILE_04_SERVICE} e2e_tests/${TEST_E2E_PROFILE_04_SERVICE}.cpp)
    target_link_libraries(${TEST_E2E_PROFILE_04_SERVICE}
        vsomeip3
        ${Boost_LIBRARIES}
        ${DL_LIBRARY}
        ${TEST_LINK_LIBRARIES}
    )

    add_executable(${TEST_E2E_PROFILE_04_CLIENT}
        e2e_tests/${TEST_E2E_PROFILE_04_CLIENT}.cpp
    )
    target_link_libraries(${TEST_E2E_PROFILE_04_CLIENT}
        vsomeip3
        ${Boost_LIBRARIES}
        ${DL_LIBRARY}
        ${TEST_LINK_LIBRARIES}
    )

    # Copy service config file for external allow tests into $BUILDDIR/test
    set(TEST_E2E_PROFILE_04_SERVICE_CONFIG_FILE_EXTERNAL ${TEST_E2E_PROFILE_04_NAME}_service_external.json)
    configure_file(
        ${PROJECT_SOURCE_DIR}/test/e2e_tests/conf/${TEST_E2E_PROFILE_04_SERVICE_CONFIG_FILE_EXTERNAL}.in
        ${PROJECT_SOURCE_DIR}/test/e2e_tests/${TEST_E2E_PROFILE_04_SERVICE_CONFIG_FILE_EXTERNAL}
        @ONLY)
    copy_to_builddir(
        ${PROJECT_SOURCE_DIR}/test/e2e_tests/${TEST_E2E_PROFILE_04_SERVICE_CONFIG_FILE_EXTERNAL}
        ${PROJECT_BINARY_DIR}/test/${TEST_E2E_PROFILE_04_SERVICE_CONFIG_FILE_EXTERNAL}
        ${TEST_E2E_PROFILE_04_SERVICE}
    )

    # Copy client config file for external allow tests into $BUILDDIR/test
    set(TEST_E2E_PROFILE_04_CLIENT_CONFIG_FILE_EXTERNAL ${TEST_E2E_PROFILE_04_NAME}_client_external.json)
    configure_file(
        ${PROJECT_SOURCE_DIR}/test/e2e_tests/conf/${TEST_E2E_PROFILE_04_CLIENT_CONFIG_FILE_EXTERNAL}.in
        ${PROJECT_SOURCE_DIR}/test/e2e_tests/${TEST_E2E_PROFILE_04_CLIENT_CONFIG_FILE_EXTERNAL}
        @ONLY)
    copy_to_builddir(
        ${PROJECT_SOURCE_DIR}/test/e2e_tests/${TEST_E2E_PROFILE_04_CLIENT_CONFIG_FILE_EXTERNAL}
        ${PROJECT_BINARY_DIR}/test/${TEST_E2E_PROFILE_04_CLIENT_CONFIG_FILE_EXTERNAL}
        ${TEST_E2E_PROFILE_04_SERVICE}
    )

    # Copy bashscript to start external tests (master) into $BUILDDIR/test
    set(TEST_E2E_PROFILE_04_EXTERNAL_MASTER_START_SCRIPT ${TEST_E2E_PROFILE_04_NAME}_external_master_start.sh)
    copy_to_builddir(
        ${PROJECT_SOURCE_DIR}/test/e2e_tests/${TEST_E2E_PROFILE_04_EXTERNAL_MASTER_START_SCRIPT}
        ${PROJECT_BINARY_DIR}/test/${TEST_E2E_PROFILE_04_EXTERNAL_MASTER_START_SCRIPT}
        ${TEST_E2E_PROFILE_04_SERVICE}
    )

    # Copy bashscript to start external tests (slave) into $BUILDDIR/test
    set(TEST_E2E_PROFILE_04_EXTERNAL_SLAVE_START_SCRIPT ${TEST_E2E_PROFILE_04_NAME}_external_slave_start.sh)
    copy_to_builddir(
        ${PROJECT_SOURCE_DIR}/test/e2e_tests/${TEST_E2E_PROFILE_04_EXTERNAL_SLAVE_START_SCRIPT}
        ${PROJECT_BINARY_DIR}/test/${TEST_E2E_PROFILE_04_EXTERNAL_SLAVE_START_SCRIPT}
        ${TEST_E2E_PROFILE_04_SERVICE}
    )
endif()
```

这个 CMake 片段主要用于构建和配置 End-to-End (E2E) 测试场景，特别是针对 "Profile 04" 的测试。它定义了一些测试的可执行文件，并为它们配置了必要的文件，包括 JSON 配置文件和启动脚本。下面是对该片段的逐行分析：

### 1. 条件判断
```cmake
if(NOT ${TESTS_BAT} AND ${TEST_E2E_PROFILE_04})
```
- 这个条件判断确保只有在 `TESTS_BAT` 未定义或为 `OFF` 且 `TEST_E2E_PROFILE_04` 为 `ON` 时，才会执行下面的代码块。

### 2. 定义测试名称
```cmake
set(TEST_E2E_PROFILE_04_NAME e2e_profile_04_test)
set(TEST_E2E_PROFILE_04_SERVICE e2e_profile_04_test_service)
set(TEST_E2E_PROFILE_04_CLIENT e2e_profile_04_test_client)
```
- 定义了 E2E Profile 04 测试的名称及其关联的服务端和客户端可执行文件名称。

### 3. 创建服务端可执行文件并链接库
```cmake
add_executable(${TEST_E2E_PROFILE_04_SERVICE} e2e_tests/${TEST_E2E_PROFILE_04_SERVICE}.cpp)
target_link_libraries(${TEST_E2E_PROFILE_04_SERVICE}
    vsomeip3
    ${Boost_LIBRARIES}
    ${DL_LIBRARY}
    ${TEST_LINK_LIBRARIES}
)
```
- 生成服务端的可执行文件 `e2e_profile_04_test_service`，它由 `e2e_tests/e2e_profile_04_test_service.cpp` 文件编译生成。
- 链接了一系列库，包括 `vsomeip3`（通常用于 SOME/IP 通信）、Boost 库、动态链接库（`DL_LIBRARY`）和其他可能的测试相关库（`TEST_LINK_LIBRARIES`）。

### 4. 创建客户端可执行文件并链接库
```cmake
add_executable(${TEST_E2E_PROFILE_04_CLIENT} e2e_tests/${TEST_E2E_PROFILE_04_CLIENT}.cpp)
target_link_libraries(${TEST_E2E_PROFILE_04_CLIENT}
    vsomeip3
    ${Boost_LIBRARIES}
    ${DL_LIBRARY}
    ${TEST_LINK_LIBRARIES}
)
```
- 生成客户端的可执行文件 `e2e_profile_04_test_client`，它由 `e2e_tests/e2e_profile_04_test_client.cpp` 文件编译生成。
- 同样链接了与服务端相同的一系列库。

### 5. 配置并复制服务端配置文件
```cmake
set(TEST_E2E_PROFILE_04_SERVICE_CONFIG_FILE_EXTERNAL ${TEST_E2E_PROFILE_04_NAME}_service_external.json)
configure_file(
    ${PROJECT_SOURCE_DIR}/test/e2e_tests/conf/${TEST_E2E_PROFILE_04_SERVICE_CONFIG_FILE_EXTERNAL}.in
    ${PROJECT_SOURCE_DIR}/test/e2e_tests/${TEST_E2E_PROFILE_04_SERVICE_CONFIG_FILE_EXTERNAL}
    @ONLY)
copy_to_builddir(
    ${PROJECT_SOURCE_DIR}/test/e2e_tests/${TEST_E2E_PROFILE_04_SERVICE_CONFIG_FILE_EXTERNAL}
    ${PROJECT_BINARY_DIR}/test/${TEST_E2E_PROFILE_04_SERVICE_CONFIG_FILE_EXTERNAL}
    ${TEST_E2E_PROFILE_04_SERVICE}
)
```
- 先将服务端的外部配置文件 `.in` 模板处理成实际的 JSON 文件。`configure_file()` 命令会替换 `.in` 文件中的占位符变量并生成目标文件。
- 之后使用 `copy_to_builddir` 函数将生成的 JSON 配置文件从源目录复制到构建目录中，并确保这个操作与服务端的构建相关联（即构建服务端时需要确保配置文件已复制）。

### 6. 配置并复制客户端配置文件
```cmake
set(TEST_E2E_PROFILE_04_CLIENT_CONFIG_FILE_EXTERNAL ${TEST_E2E_PROFILE_04_NAME}_client_external.json)
configure_file(
    ${PROJECT_SOURCE_DIR}/test/e2e_tests/conf/${TEST_E2E_PROFILE_04_CLIENT_CONFIG_FILE_EXTERNAL}.in
    ${PROJECT_SOURCE_DIR}/test/e2e_tests/${TEST_E2E_PROFILE_04_CLIENT_CONFIG_FILE_EXTERNAL}
    @ONLY)
copy_to_builddir(
    ${PROJECT_SOURCE_DIR}/test/e2e_tests/${TEST_E2E_PROFILE_04_CLIENT_CONFIG_FILE_EXTERNAL}
    ${PROJECT_BINARY_DIR}/test/${TEST_E2E_PROFILE_04_CLIENT_CONFIG_FILE_EXTERNAL}
    ${TEST_E2E_PROFILE_04_SERVICE}
)
```
- 与服务端类似，配置并复制客户端的 JSON 配置文件到构建目录。

### 7. 复制服务端启动脚本
```cmake
set(TEST_E2E_PROFILE_04_EXTERNAL_MASTER_START_SCRIPT ${TEST_E2E_PROFILE_04_NAME}_external_master_start.sh)
copy_to_builddir(
    ${PROJECT_SOURCE_DIR}/test/e2e_tests/${TEST_E2E_PROFILE_04_EXTERNAL_MASTER_START_SCRIPT}
    ${PROJECT_BINARY_DIR}/test/${TEST_E2E_PROFILE_04_EXTERNAL_MASTER_START_SCRIPT}
    ${TEST_E2E_PROFILE_04_SERVICE}
)
```
- 将服务端的启动脚本复制到构建目录中。此脚本用于在执行 E2E 测试时启动服务端。

### 8. 复制客户端启动脚本
```cmake
set(TEST_E2E_PROFILE_04_EXTERNAL_SLAVE_START_SCRIPT ${TEST_E2E_PROFILE_04_NAME}_external_slave_start.sh)
copy_to_builddir(
    ${PROJECT_SOURCE_DIR}/test/e2e_tests/${TEST_E2E_PROFILE_04_EXTERNAL_SLAVE_START_SCRIPT}
    ${PROJECT_BINARY_DIR}/test/${TEST_E2E_PROFILE_04_EXTERNAL_SLAVE_START_SCRIPT}
    ${TEST_E2E_PROFILE_04_SERVICE}
)
```
- 将客户端的启动脚本复制到构建目录中。此脚本用于在执行 E2E 测试时启动客户端。

vsomeip之后会添加gtest的依赖。

```cmake
add_dependencies(${TEST_E2E_PROFILE_04_SERVICE} gtest)
add_dependencies(${TEST_E2E_PROFILE_04_CLIENT} gtest)
```

之后又添加到build_tests的目标中，在构建的时候指定build_tests时候，就可以编译e2e相关的测试。

```cmake
add_dependencies(build_tests ${TEST_E2E_PROFILE_04_SERVICE})
add_dependencies(build_tests ${TEST_E2E_PROFILE_04_CLIENT})
```

最后使用add_test添加测试。

```cmake
add_test(NAME ${TEST_E2E_PROFILE_04_NAME}_external
COMMAND ${PROJECT_BINARY_DIR}/test/${TEST_E2E_PROFILE_04_EXTERNAL_MASTER_START_SCRIPT} e2e_profile_04_test_client_external.json)
set_tests_properties(${TEST_E2E_PROFILE_04_NAME}_external PROPERTIES TIMEOUT 180)
```

## cyclonedds中ctest的使用

cyclonedds因为使用了`gtest_add_tests()`，所以相对比较简单。

```cmake
find_package(GTest REQUIRED)

idlcxx_generate(TARGET ddscxx_test_types FILES
  data/Space.idl
  data/KeylessSpace.idl
  data/HelloWorldData.idl
  data/Serialization.idl
  data/CdrDataModels.idl
  data/CdrDataModels_pragma.idl
  data/EntityProperties.idl
  data/EntityProperties_pragma.idl
  data/ExtendedTypesModels.idl
  data/RegressionModels.idl
  data/RegressionModels_pragma.idl
  data/ExternalModels.idl
  data/KeyHashModels.idl
  data/TraitsModels.idl
  WARNINGS no-implicit-extensibility)

configure_file(
  config_simple.xml.in config_simple.xml @ONLY)

if(ENABLE_ICEORYX)
  # Packages required packages for Iceoryx tests
  find_package(iceoryx_posh REQUIRED)
  find_package(iceoryx_posh_testing REQUIRED)
endif()

set(sources
  Bounded.cpp
  EntityStatus.cpp
  Listener.cpp
  ListenerStress.cpp
  DomainParticipant.cpp
  Exception.cpp
  Conversions.cpp
  FindDataWriter.cpp
  FindDataReader.cpp
  Topic.cpp
  Publisher.cpp
  Serdata.cpp
  Subscriber.cpp
  DataWriter.cpp
  DataReader.cpp
  DataReaderSelector.cpp
  DataReaderManipulatorSelector.cpp
  Duration.cpp
  Time.cpp
  Query.cpp
  WaitSet.cpp
  Qos.cpp
  Condition.cpp
  Util.cpp
  CDRStreamer.cpp
  GeneratedEntities.cpp
  ExtendedTypes.cpp
  Regression.cpp
  External.cpp
  DataModels.cpp
  SampleInfo.cpp
  DeferredDestruction.cpp
  KeyHash.cpp)

if (ENABLE_TYPELIB AND ENABLE_TOPIC_DISCOVERY)
  # Add topic/type discovery tests
  list(APPEND sources
    FindTopic.cpp
    TopicTypeDiscovery.cpp)
endif()

if (ENABLE_QOS_PROVIDER)
  list(APPEND sources
    QosProvider.cpp)
endif()

if(ENABLE_ICEORYX)
  list(APPEND sources Iceoryx.cpp)
endif()

add_executable(ddscxx_tests ${sources})

# Disable the static analyzer in GCC to avoid crashing the GNU C++ compiler
# on Azure Pipelines
if(DEFINED ENV{SYSTEM_TEAMFOUNDATIONSERVERURI})
  if(CMAKE_C_COMPILER_ID STREQUAL "GNU" AND ANALYZER STREQUAL "on")
    target_compile_options(ddscxx_tests PRIVATE -fno-analyzer)
  endif()
endif()

target_include_directories(
  ddscxx_tests
  PRIVATE
    "$<BUILD_INTERFACE:${CYCLONE_SOURCE_DIR}/src/core/ddsc/src>")


set_property(TARGET ddscxx_tests PROPERTY CXX_STANDARD ${cyclonedds_cpp_std_to_use})
target_link_libraries(
  ddscxx_tests PRIVATE
    CycloneDDS-CXX::ddscxx
    GTest::GTest
    GTest::Main
    ddscxx_test_types)

if(ENABLE_ICEORYX)
  target_link_libraries(
    ddscxx_tests PRIVATE
      iceoryx_posh::iceoryx_posh_roudi
      iceoryx_posh_testing::iceoryx_posh_testing
      # iceoryx_posh_testing depends on GoogleMock but does not propagate the
      # the dependency. Workaround it by requiring the package here.
      GTest::GMock
      GTest::GMockMain)
endif()

target_link_libraries(ddscxx_tests ${TEST_LINK_LIBS})

gtest_add_tests(TARGET ddscxx_tests SOURCES ${sources} TEST_LIST tests)

# Ensure shared libraries are found
if(WIN32)
  set(sep ";")
  set(var "PATH")
elseif(APPLE)
  set(sep ":")
  set(var "DYLD_LIBRARY_PATH")
else()
  set(sep ":")
  set(var "LD_LIBRARY_PATH")
endif()

get_target_property(cyclonedds_lib CycloneDDS::ddsc LOCATION)
get_target_property(gtest_lib GTest::GTest IMPORTED_LOCATION)
get_target_property(gtest_main_lib GTest::Main IMPORTED_LOCATION)
if(ENABLE_ICEORYX)
  get_target_property(gmock_lib GTest::GMock IMPORTED_LOCATION)
  get_target_property(gmock_main_lib GTest::GMockMain IMPORTED_LOCATION)
endif()

# Ignore false positives due to gtest not being compiled with asan
if(SANITIZER MATCHES "address")
  foreach(test ${tests})
    set_property(TEST ${test} APPEND PROPERTY ENVIRONMENT "ASAN_OPTIONS=detect_container_overflow=0")
  endforeach()
endif()

foreach(lib ${cyclonedds_lib} ${gtest_lib} ${gtest_main_lib} ${gmock_lib} ${gmock_main_lib})
  get_filename_component(libdir "${lib}" PATH)
  file(TO_NATIVE_PATH "${libdir}" libdir)

  foreach(test ${tests})
    get_property(envvars TEST ${test} PROPERTY ENVIRONMENT)
    list(LENGTH envvars n)
    set(add TRUE)
    if(n GREATER 0)
      math(EXPR n "${n} - 1")
      foreach(i RANGE 0 ${n})
        list(GET envvars ${i} envvar)
        if(envvar MATCHES "^${var}=")
          list(REMOVE_AT envvars ${i})
          set_property(TEST ${test} PROPERTY ENVIRONMENT "${envvars}")
          string(REGEX REPLACE "^${var}=" "" paths "${envvar}")
          string(REPLACE ";" "\\;" paths "${var}=${libdir}${sep}${paths}")
          set_property(TEST ${test} APPEND PROPERTY ENVIRONMENT "${paths}")
          set(add FALSE)
          break()
        endif()
      endforeach()
    endif()
    if(add)
      set_property(TEST ${test} APPEND PROPERTY ENVIRONMENT "${var}=${libdir}")
    endif()
  endforeach()
endforeach()
```

这个 CMake 片段主要用于设置和构建一个名为 `ddscxx_tests` 的测试可执行文件，并处理与测试框架（如 Google Test 和 Iceoryx）相关的配置。它还涉及到一些环境变量的设置和特殊编译器选项的处理。以下是逐步的分析：

### 1. 引入必要的包和依赖

```cmake
find_package(GTest REQUIRED)
```
- 查找并引入 Google Test 库，确保项目可以使用 GTest 进行测试。

### 2. 生成 IDL 文件

```cmake
idlcxx_generate(TARGET ddscxx_test_types FILES
  ...
  WARNINGS no-implicit-extensibility)
```
- 使用 `idlcxx_generate` 命令生成目标 `ddscxx_test_types`，这是一个基于一组 IDL 文件的类型库，可能用于 DDS（数据分发服务）系统中。
- `WARNINGS no-implicit-extensibility` 表示在生成类型时不允许隐式的可扩展性（通常用于确保数据类型的稳定性）。

### 3. 配置 XML 文件

```cmake
configure_file(
  config_simple.xml.in config_simple.xml @ONLY)
```
- 配置并生成 `config_simple.xml` 文件，使用 `.in` 文件中的内容和定义的变量进行替换（`@ONLY` 选项表示只替换 `@VAR@` 形式的变量）。

### 4. 条件性引入 Iceoryx

```cmake
if(ENABLE_ICEORYX)
  find_package(iceoryx_posh REQUIRED)
  find_package(iceoryx_posh_testing REQUIRED)
endif()
```
- 如果启用了 Iceoryx，则引入必要的 Iceoryx 相关库（`iceoryx_posh` 和 `iceoryx_posh_testing`）。Iceoryx 是一种用于零拷贝数据传输的中间件。

### 5. 定义测试源文件

```cmake
set(sources
  ...
  DeferredDestruction.cpp
  KeyHash.cpp)
```
- 列举了一系列用于构建 `ddscxx_tests` 可执行文件的源文件。

### 6. 条件性增加源文件

```cmake
if (ENABLE_TYPELIB AND ENABLE_TOPIC_DISCOVERY)
  list(APPEND sources FindTopic.cpp TopicTypeDiscovery.cpp)
endif()

if (ENABLE_QOS_PROVIDER)
  list(APPEND sources QosProvider.cpp)
endif()

if(ENABLE_ICEORYX)
  list(APPEND sources Iceoryx.cpp)
endif()
```
- 根据特定的编译选项（如 `ENABLE_TYPELIB`、`ENABLE_TOPIC_DISCOVERY`、`ENABLE_QOS_PROVIDER` 和 `ENABLE_ICEORYX`），向源文件列表中添加额外的源文件。

### 7. 创建可执行文件并设置编译选项

```cmake
add_executable(ddscxx_tests ${sources})

if(DEFINED ENV{SYSTEM_TEAMFOUNDATIONSERVERURI})
  if(CMAKE_C_COMPILER_ID STREQUAL "GNU" AND ANALYZER STREQUAL "on")
    target_compile_options(ddscxx_tests PRIVATE -fno-analyzer)
  endif()
endif()
```
- 创建 `ddscxx_tests` 可执行文件，并在特定条件下（比如在 Azure Pipelines 上构建时使用 GNU 编译器）禁用静态分析器（`-fno-analyzer`），以避免编译器崩溃。

### 8. 设置包含目录

```cmake
target_include_directories(
  ddscxx_tests
  PRIVATE
    "$<BUILD_INTERFACE:${CYCLONE_SOURCE_DIR}/src/core/ddsc/src>")
```
- 设置 `ddscxx_tests` 的私有包含目录，确保它可以访问 Cyclone DDS 的核心代码。

### 9. 链接库

```cmake
target_link_libraries(
  ddscxx_tests PRIVATE
    CycloneDDS-CXX::ddscxx
    GTest::GTest
    GTest::Main
    ddscxx_test_types)
```
- 链接 `ddscxx_tests` 所需的库，包括 Cyclone DDS C++ 库、Google Test 库和生成的 `ddscxx_test_types` 库。

### 10. 条件性链接 Iceoryx 相关库

```cmake
if(ENABLE_ICEORYX)
  target_link_libraries(
    ddscxx_tests PRIVATE
      iceoryx_posh::iceoryx_posh_roudi
      iceoryx_posh_testing::iceoryx_posh_testing
      GTest::GMock
      GTest::GMockMain)
endif()
```
- 如果启用了 Iceoryx，则额外链接 Iceoryx 相关的库，并解决 Google Mock 的依赖问题。

### 11. 添加测试

```cmake
gtest_add_tests(TARGET ddscxx_tests SOURCES ${sources} TEST_LIST tests)
```
- 使用 `gtest_add_tests` 自动扫描和添加测试，并生成一个测试列表 `tests`。

### 12. 设置环境变量以确保找到共享库

```cmake
if(WIN32)
  set(sep ";")
  set(var "PATH")
elseif(APPLE)
  set(sep ":")
  set(var "DYLD_LIBRARY_PATH")
else()
  set(sep ":")
  set(var "LD_LIBRARY_PATH")
endif()
```
- 设置适当的环境变量名称和路径分隔符，以便在不同操作系统上找到共享库。

### 13. 获取库路径并设置测试环境

```cmake
get_target_property(cyclonedds_lib CycloneDDS::ddsc LOCATION)
get_target_property(gtest_lib GTest::GTest IMPORTED_LOCATION)
get_target_property(gtest_main_lib GTest::Main IMPORTED_LOCATION)
if(ENABLE_ICEORYX)
  get_target_property(gmock_lib GTest::GMock IMPORTED_LOCATION)
  get_target_property(gmock_main_lib GTest::GMockMain IMPORTED_LOCATION)
endif()
```
- 获取 CycloneDDS、GTest 和 GMock 库的路径，用于设置环境变量。

```cmake
foreach(lib ${cyclonedds_lib} ${gtest_lib} ${gtest_main_lib} ${gmock_lib} ${gmock_main_lib})
  get_filename_component(libdir "${lib}" PATH)
  file(TO_NATIVE_PATH "${libdir}" libdir)

  foreach(test ${tests})
    get_property(envvars TEST ${test} PROPERTY ENVIRONMENT)
    list(LENGTH envvars n)
    set(add TRUE)
    if(n GREATER 0)
      math(EXPR n "${n} - 1")
      foreach(i RANGE 0 ${n})
        list(GET envvars ${i} envvar)
        if(envvar MATCHES "^${var}=")
          list(REMOVE_AT envvars ${i})
          set_property(TEST ${test} PROPERTY ENVIRONMENT "${envvars}")
          string(REGEX REPLACE "^${var}=" "" paths "${envvar}")
          string(REPLACE ";" "\\;" paths "${var}=${libdir}${sep}${paths}")
          set_property(TEST ${test} APPEND PROPERTY ENVIRONMENT "${paths}")
          set(add FALSE)
          break()
        endif()
      endforeach()
    endif()
    if(add)
      set_property(TEST ${test} APPEND PROPERTY ENVIRONMENT "${var}=${libdir}")
    endif()
  endforeach()
endforeach()
```
- 这段代码确保在运行测试时，测试环境中包含了必要的库路径。它遍历每个测试并为其设置正确的环境变量，以便动态链接库在运行时可以被找到。
