---
title: 从开源库研究cmake的cpack
date: 2024-08-19 16:55 +0800
last_modified_at: 2024-08-19 16:55 +0800
author: fleetime0
categories: ["cmake"]
tags: ["c++", "cmake"]
pin: true
math: true
mermaid: true
---

CPack 是 CMake 中的一个模块，用于为 CMake 构建的项目生成软件包。它支持多种不同的平台和包管理器，如 DEB、RPM、NSIS、DMG 等，使得软件发布和分发变得更加容易。以下是 CPack 的一些关键点：

### 1. 基本工作流程
CPack 通常与 CMake 一起使用。你可以在 CMake 的 CMakeLists.txt 文件中进行配置，并通过 `cpack` 命令生成安装包。典型的工作流程包括以下步骤：

1. **配置 CPack**：在 CMakeLists.txt 文件中添加 CPack 相关的配置。
2. **生成构建文件**：使用 CMake 生成构建文件。
3. **构建项目**：使用构建工具（如 make、ninja）构建项目。
4. **生成包**：运行 `cpack` 命令生成安装包。

### 2. CPack 配置
在 CMakeLists.txt 中，你需要通过设置一些变量来配置 CPack。例如：

```cmake
# 设置项目名称和版本
set(CPACK_PACKAGE_NAME "MyApp")
set(CPACK_PACKAGE_VERSION "1.0.0")

# 设置安装路径
set(CPACK_INSTALL_PREFIX "/usr/local")

# 设置生成的安装包类型，如 DEB、RPM、NSIS 等
set(CPACK_GENERATOR "DEB;RPM")

# 其他配置选项，如维护者、许可证等
set(CPACK_PACKAGE_MAINTAINER "Your Name <your.email@example.com>")
set(CPACK_PACKAGE_DESCRIPTION "This is a description of MyApp")
```

### 3. 使用 CPack 生成安装包
在配置好 CMakeLists.txt 文件后，可以按以下步骤生成安装包：

1. **配置 CMake**：运行 `cmake` 命令生成构建文件。
2. **构建项目**：运行构建工具，如 `make`。
3. **生成安装包**：运行 `cpack` 命令生成指定类型的安装包。

例如，假设你已经生成了构建文件并完成了构建，可以运行以下命令：

```bash
cpack
```

这将根据你在 CMakeLists.txt 中的配置生成安装包。如果你想生成特定类型的包，可以指定生成器，如：

```bash
cpack -G DEB
```

### 4. 支持的生成器
CPack 支持多种包生成器，每种生成器用于不同的平台和需求。常见的生成器包括：

- **DEB**: 用于生成 Debian 系列 Linux 的 `.deb` 包。
- **RPM**: 用于生成 Red Hat 系列 Linux 的 `.rpm` 包。
- **NSIS**: 用于生成 Windows 上的安装程序。
- **DMG**: 用于生成 macOS 上的磁盘镜像文件。
- **TGZ/ZIP**: 生成压缩的归档文件。

### 5. 高级配置选项
CPack 提供了大量的配置选项来定制生成的安装包。这些选项可以设置安装路径、许可证信息、安装脚本等。例如：

- **CPACK_PACKAGE_FILE_NAME**: 自定义生成包的文件名。
- **CPACK_PACKAGE_CONTACT**: 设置联系信息。
- **CPACK_PACKAGE_LICENSE**: 指定软件包的许可证。
- **CPACK_DEBIAN_PACKAGE_DEPENDS**: 设置 `.deb` 包的依赖项。
- **CPACK_RPM_PACKAGE_REQUIRES**: 设置 `.rpm` 包的依赖项。

通过这些选项，你可以生成非常复杂和灵活的安装包，以满足不同平台和用户的需求。

### 总结
CPack 是一个强大且灵活的工具，能够帮助开发者轻松创建适用于多个平台的安装包。通过在 CMakeLists.txt 中进行简单的配置，你可以生成各种格式的安装包，并根据需求进行定制。

## cylonedds中cpack的使用

cyclonedds-cxx中单独定义了一个CMakeCPack.cmake，然后在主CMakeLists.txt通过`include()`引入。CMakeCpack.cmake的内容以及分析如下：

```cmake
if(CMAKECPACK_INCLUDED)
  return()
endif()
set(CMAKECPACK_INCLUDED true)

include(GNUInstallDirs)
set(PROJECT_NAME_FULL "Eclipse Cyclone DDS ISO IEC C++ PSM")
# Set some convenience variants of the project-name
string(REPLACE " " "-" PROJECT_NAME_DASHED "${PROJECT_NAME_FULL}")

set(CPACK_PACKAGE_VERSION_MAJOR ${PROJECT_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${PROJECT_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH ${PROJECT_VERSION_PATCH})
set(CPACK_PACKAGE_VERSION_TWEAK ${PROJECT_VERSION_TWEAK})
set(CPACK_PACKAGE_VERSION ${PROJECT_VERSION})

set(CPACK_PACKAGE_NAME ${PROJECT_NAME})
set(CPACK_PACKAGE_VENDOR "Eclipse Cyclone DDS project")
set(CPACK_PACKAGE_CONTACT "https://github.com/eclipse-cyclonedds/cyclonedds-cxx")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "Eclipse Cyclone DDS ISO IEC C++ PSM")

file(COPY "${PROJECT_SOURCE_DIR}/LICENSE" DESTINATION "${CMAKE_BINARY_DIR}")
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_BINARY_DIR}/LICENSE")

# Packages could be generated on alien systems. e.g. Debian packages could be
# created on Red Hat Enterprise Linux, but since packages also need to be
# verified on the target platform, please refrain from doing so. Another
# reason for building installer packages on the target platform is to ensure
# the binaries are linked to the libc version shipped with that platform. To
# support "generic" Linux distributions, eventually compressed tarballs will
# be shipped.
#
# NOTE: Settings for different platforms are in separate control branches.
#       Although that does not make sense from a technical point-of-view, it
#       does help to clearify which settings are required for a platform.

set(CPACK_COMPONENTS_ALL dev lib idlcxx)
set(CPACK_COMPONENT_LIB_DISPLAY_NAME "${PROJECT_NAME_FULL} library")
set(CPACK_COMPONENT_LIB_DESCRIPTION  "Library used to run programs with ${PROJECT_NAME_FULL}")
set(CPACK_COMPONENT_DEV_DISPLAY_NAME "${PROJECT_NAME_FULL} development")
set(CPACK_COMPONENT_DEV_DESCRIPTION  "Development files for use with ${PROJECT_NAME_FULL}")
set(CPACK_COMPONENT_IDLCXX_DISPLAY_NAME "${PROJECT_NAME_FULL} compiler plugin for idlc")
set(CPACK_COMPONENT_IDLCXX_DESCRIPTION  "Idl compiler plugin library for ${PROJECT_NAME_FULL}")

if(WIN32 AND NOT UNIX)
  file(COPY "${PROJECT_SOURCE_DIR}/WiX/LICENSE.rtf" DESTINATION "${CMAKE_BINARY_DIR}")
  set(CPACK_WIX_LICENSE_RTF "${CMAKE_BINARY_DIR}/LICENSE.rtf")
  if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(__arch "win64")
  else()
    set(__arch "win32")
  endif()
  mark_as_advanced(__arch)

  set(CPACK_GENERATOR "WIX;ZIP;${CPACK_GENERATOR}" CACHE STRING "List of package generators")
  set(CPACK_PACKAGE_FILE_NAME "${PROJECT_NAME}-${CPACK_PACKAGE_VERSION}-${__arch}")
  set(WIX_DIR "${PROJECT_SOURCE_DIR}/WiX")
  set(CPACK_PACKAGE_INSTALL_DIRECTORY "${PROJECT_NAME_FULL}")
  set(CPACK_WIX_UI_REF "CustomUI_InstallDir")
  set(CPACK_WIX_PATCH_FILE "${WIX_DIR}/env.xml")
  set(CPACK_WIX_EXTRA_SOURCES "${WIX_DIR}/PathDlg.wxs" 
                              "${WIX_DIR}/DialogOrder.wxs")
  set(CPACK_WIX_CMAKE_PACKAGE_REGISTRY "${PROJECT_NAME}")
  set(CPACK_WIX_PRODUCT_ICON "${WIX_DIR}/icon.ico")
  set(CPACK_WIX_UI_BANNER "${WIX_DIR}/banner.png")
  set(CPACK_WIX_UI_DIALOG "${WIX_DIR}/dialog.png")
  # when updating the version number also generate a new GUID
	set(CPACK_WIX_UPGRADE_GUID "0D106C1A-D499-4DD4-ABD5-BE77415C1E8B")
  
  include(InstallRequiredSystemLibraries)
  set(CMAKE_INSTALL_UCRT_LIBRARIES TRUE)

elseif(CMAKE_SYSTEM_NAME MATCHES "Linux")
  set(CPACK_COMPONENTS_GROUPING "IGNORE")

  if(EXISTS "/etc/redhat-release")
    if(CMAKE_SIZEOF_VOID_P EQUAL 8)
      set(__arch "x86_64")
    else()
      set(__arch "i686")
    endif()

    set(CPACK_GENERATOR "RPM;TGZ;${CPACK_GENERATOR}" CACHE STRING "List of package generators")
    set(CPACK_RPM_PACKAGE_LICENSE "Eclipse Public License v2.0  http://www.eclipse.org/legal/epl-2.0")
    set(CPACK_RPM_COMPONENT_INSTALL ON)
    set(CPACK_RPM_PACKAGE_RELEASE 1)
    set(CPACK_RPM_PACKAGE_RELEASE_DIST ON)
    set(CPACK_RPM_LIB_PACKAGE_NAME "${PROJECT_NAME_DASHED}")
    set(CPACK_RPM_LIB_FILE_NAME "${CPACK_RPM_LIB_PACKAGE_NAME}-${CPACK_PACKAGE_VERSION}-${CPACK_RPM_PACKAGE_RELEASE}%{?dist}-${__arch}.rpm")
    set(CPACK_RPM_DEV_PACKAGE_NAME "${CPACK_RPM_LIB_PACKAGE_NAME}-devel")
    set(CPACK_RPM_DEV_FILE_NAME "${CPACK_RPM_DEV_PACKAGE_NAME}-${CPACK_PACKAGE_VERSION}-${CPACK_RPM_PACKAGE_RELEASE}%{?dist}-${__arch}.rpm")
    set(CPACK_RPM_DEV_PACKAGE_REQUIRES "${CPACK_RPM_LIB_PACKAGE_NAME} = ${CPACK_PACKAGE_VERSION}")
  elseif(EXISTS "/etc/debian_version")
    set(CPACK_DEB_COMPONENT_INSTALL ON)
    if(CMAKE_SIZEOF_VOID_P EQUAL 8)
      set(__arch "amd64")
    else()
      set(__arch "i386")
    endif()

    set(CPACK_GENERATOR "DEB;TGZ;${CPACK_GENERATOR}" CACHE STRING "List of package generators")

    string(TOLOWER "${PROJECT_NAME_DASHED}" CPACK_DEBIAN_LIB_PACKAGE_NAME)
    set(CPACK_DEBIAN_LIB_FILE_NAME "${CPACK_DEBIAN_LIB_PACKAGE_NAME}_${CPACK_PACKAGE_VERSION}_${__arch}.deb")
    set(CPACK_DEBIAN_DEV_PACKAGE_DEPENDS "${CPACK_DEBIAN_LIB_PACKAGE_NAME} (= ${CPACK_PACKAGE_VERSION}), libc6 (>= 2.23)")
    set(CPACK_DEBIAN_DEV_PACKAGE_NAME "${CPACK_DEBIAN_LIB_PACKAGE_NAME}-dev")
    set(CPACK_DEBIAN_DEV_FILE_NAME "${CPACK_DEBIAN_DEV_PACKAGE_NAME}_${CPACK_PACKAGE_VERSION}_${__arch}.deb")
  else()
    # Generic tgz package
    set(CPACK_GENERATOR "TGZ;${CPACK_GENERATOR}" CACHE STRING "List of package generators")
  endif()
else()
    # Fallback to zip package
    set(CPACK_GENERATOR "ZIP;${CPACK_GENERATOR}" CACHE STRING "List of package generators")
endif()

# This must always be last!
include(CPack)
```

这个 CMake 片段展示了如何使用 CPack 来配置并生成项目的安装包。以下是这个片段的详细分析：

### 1. 初始检查与设置
```cmake
if(CMAKECPACK_INCLUDED)
  return()
endif()
set(CMAKECPACK_INCLUDED true)
```
- 这部分代码检查是否已经包含了 CPack 配置（通过 `CMAKECPACK_INCLUDED` 变量）。如果已经包含，则直接返回，避免重复设置。

### 2. 引入和项目名称设置
```cmake
include(GNUInstallDirs)
set(PROJECT_NAME_FULL "Eclipse Cyclone DDS ISO IEC C++ PSM")
string(REPLACE " " "-" PROJECT_NAME_DASHED "${PROJECT_NAME_FULL}")
```
- `GNUInstallDirs` 模块提供了一些标准目录变量，例如 `CMAKE_INSTALL_BINDIR`、`CMAKE_INSTALL_LIBDIR` 等。
- `PROJECT_NAME_FULL` 被设置为项目的全名，并创建了一个用连字符 `-` 替换空格的版本 `PROJECT_NAME_DASHED`，便于在生成包文件名时使用。

### 3. 设置 CPack 的版本信息
```cmake
set(CPACK_PACKAGE_VERSION_MAJOR ${PROJECT_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${PROJECT_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH ${PROJECT_VERSION_PATCH})
set(CPACK_PACKAGE_VERSION_TWEAK ${PROJECT_VERSION_TWEAK})
set(CPACK_PACKAGE_VERSION ${PROJECT_VERSION})
```
- 这里设置了 CPack 的包版本信息，使用了 CMake 中预定义的项目版本变量 `PROJECT_VERSION_MAJOR` 等。

### 4. 基本包信息
```cmake
set(CPACK_PACKAGE_NAME ${PROJECT_NAME})
set(CPACK_PACKAGE_VENDOR "Eclipse Cyclone DDS project")
set(CPACK_PACKAGE_CONTACT "https://github.com/eclipse-cyclonedds/cyclonedds-cxx")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "Eclipse Cyclone DDS ISO IEC C++ PSM")
```
- 这些设置定义了包的基本信息，如名称、供应商、联系信息和描述。

### 5. 许可证文件处理
```cmake
file(COPY "${PROJECT_SOURCE_DIR}/LICENSE" DESTINATION "${CMAKE_BINARY_DIR}")
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_BINARY_DIR}/LICENSE")
```
- 该段代码将许可证文件从源目录复制到构建目录，并设置 CPack 使用此文件作为许可证文件。

### 6. 组件定义
```cmake
set(CPACK_COMPONENTS_ALL dev lib idlcxx)
set(CPACK_COMPONENT_LIB_DISPLAY_NAME "${PROJECT_NAME_FULL} library")
set(CPACK_COMPONENT_LIB_DESCRIPTION  "Library used to run programs with ${PROJECT_NAME_FULL}")
set(CPACK_COMPONENT_DEV_DISPLAY_NAME "${PROJECT_NAME_FULL} development")
set(CPACK_COMPONENT_DEV_DESCRIPTION  "Development files for use with ${PROJECT_NAME_FULL}")
set(CPACK_COMPONENT_IDLCXX_DISPLAY_NAME "${PROJECT_NAME_FULL} compiler plugin for idlc")
set(CPACK_COMPONENT_IDLCXX_DESCRIPTION  "Idl compiler plugin library for ${PROJECT_NAME_FULL}")
```
- 这里定义了安装包的组件（`dev`、`lib`、`idlcxx`），并为每个组件设置了显示名称和描述。这有助于创建细粒度的安装包，用户可以选择安装哪些组件。

### 7. Windows 特定配置
```cmake
if(WIN32 AND NOT UNIX)
  # Windows 特定配置代码...
endif()
```
- 该段代码处理 Windows 平台的特定配置，使用 WiX 生成器生成安装包，并设置了一些与 WiX 相关的参数（如图标、UI、补丁文件等）。
- 其中，还设置了针对 32 位和 64 位系统的架构标识。

### 8. Linux 特定配置
```cmake
elseif(CMAKE_SYSTEM_NAME MATCHES "Linux")
  # Linux 特定配置代码...
endif()
```
- 这段代码为 Linux 平台配置安装包，支持 RPM 和 DEB 两种包管理系统。
- 对于 Red Hat 系列，使用 RPM 生成器，定义了包的许可证、组件、文件名等。
- 对于 Debian 系列，使用 DEB 生成器，定义了包的依赖项、文件名等。
- 如果都不符合，则使用通用的 TGZ 格式包。

### 9. 其他平台配置
```cmake
else()
  # Fallback to zip package
  set(CPACK_GENERATOR "ZIP;${CPACK_GENERATOR}")
endif()
```
- 如果不是 Windows 也不是 Linux，那么使用 ZIP 格式作为默认的包格式。

### 10. 包含 CPack 模块
```cmake
include(CPack)
```
- 这是 CPack 配置的最后一步，将 CPack 模块包含进来。这一步会使用前面设置的所有 CPack 变量，生成配置生成器。
