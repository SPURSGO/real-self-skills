cmake_minimum_required(VERSION 3.16)

# CMP0091=NEW 启用 CMAKE_MSVC_RUNTIME_LIBRARY 变量控制 CRT 链接方式（/MT|/MTd），
# 替代旧的 CMAKE_CXX_FLAGS 方式。必须在 project() 之前设置，
# CMAKE_POLICY_DEFAULT_ 前缀确保子项目也继承该策略。
set(CMAKE_POLICY_DEFAULT_CMP0091 NEW)

project(ProjectName LANGUAGES CXX)

# 禁用 vcpkg 全局 MSBuild 集成
set(VCPKG_APPLOCAL_DEPS OFF)
set_property(GLOBAL PROPERTY VS_GLOBAL_VcpkgEnabled false)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# 构建约束检查：平台工具集必须为 v141 (VS2017)
if(MSVC AND NOT CMAKE_GENERATOR_TOOLSET STREQUAL "v141")
  message(FATAL_ERROR "平台工具集必须为 v141，请在 cmake 命令行中指定：\n"
    "  cmake -S . -B build -A Win32 -T v141")
endif()

# 32 位构建、静态 CRT、不使用预编译头
if(MSVC)
  add_compile_options(/utf-8)
  add_compile_definitions(UNICODE _UNICODE)
  # 静态链接 CRT（/MT, /MTd）
  set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif()

# 公共库路径（通过环境变量 PUBLIC2_DIR 指定）
if(DEFINED ENV{PUBLIC2_DIR})
  set(PUBLIC2_DIR "$ENV{PUBLIC2_DIR}")
else()
  message(FATAL_ERROR "环境变量 PUBLIC2_DIR 未设置，请设置为 public2 公共库根目录，例如：\n"
    "  set PUBLIC2_DIR=your_public2_dir")
endif()
set(PUBLIC2_INCLUDE_DIR "${PUBLIC2_DIR}/include")
set(PUBLIC2_LIB_2017_DIR "${PUBLIC2_DIR}/lib_2017")


# GoogleTest（使用 PUBLIC2 公共库中 v141 预编译版本）
# 测试构建开关，CI Release 构建时可通过 -DBUILD_TESTS=OFF 关闭以跳过测试编译
option(BUILD_TESTS "构建单元测试和集成测试" ON)

if(BUILD_TESTS)
  # 创建 GTest::gtest imported target
  add_library(GTest::gtest STATIC IMPORTED)
  set_target_properties(GTest::gtest PROPERTIES
    IMPORTED_LOCATION "${PUBLIC2_LIB_2017_DIR}/gtest_2017_T32.lib"
    IMPORTED_LOCATION_DEBUG "${PUBLIC2_LIB_2017_DIR}/gtest_2017_T32d.lib"
    INTERFACE_INCLUDE_DIRECTORIES "${PUBLIC2_INCLUDE_DIR}"
  )

  # 创建 GTest::gtest_main imported target
  add_library(GTest::gtest_main STATIC IMPORTED)
  set_target_properties(GTest::gtest_main PROPERTIES
    IMPORTED_LOCATION "${PUBLIC2_LIB_2017_DIR}/gtest_main_2017_T32.lib"
    IMPORTED_LOCATION_DEBUG "${PUBLIC2_LIB_2017_DIR}/gtest_main_2017_T32d.lib"
    INTERFACE_LINK_LIBRARIES GTest::gtest
  )

  # 创建 GTest::gmock imported target
  add_library(GTest::gmock STATIC IMPORTED)
  set_target_properties(GTest::gmock PROPERTIES
    IMPORTED_LOCATION "${PUBLIC2_LIB_2017_DIR}/gmock_2017_T32.lib"
    IMPORTED_LOCATION_DEBUG "${PUBLIC2_LIB_2017_DIR}/gmock_2017_T32d.lib"
    INTERFACE_LINK_LIBRARIES GTest::gtest
  )

  enable_testing()
  add_subdirectory(tests)
endif()

add_subdirectory(third_party/jsoncpp)
add_subdirectory(src)
add_subdirectory(tools)
