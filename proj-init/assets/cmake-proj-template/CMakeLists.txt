cmake_minimum_required(VERSION 3.16)

# CMP0091=NEW 启用 CMAKE_MSVC_RUNTIME_LIBRARY 变量控制 CRT 链接方式（/MT|/MTd），
# 替代旧的 CMAKE_CXX_FLAGS 方式。必须在 project() 之前设置，
# CMAKE_POLICY_DEFAULT_ 前缀确保子项目也继承该策略。
set(CMAKE_POLICY_DEFAULT_CMP0091 NEW)

project(ProjectName LANGUAGES CXX)

# 禁用 vcpkg 全局 MSBuild 集成（本项目使用 LUDASHI_PC_PUBLIC_ROOT 内部公共库，不使用 vcpkg）
set(VCPKG_APPLOCAL_DEPS OFF)
set_property(GLOBAL PROPERTY VS_GLOBAL_VcpkgEnabled false)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# 构建约束检查：平台工具集必须为 v141 (VS2017)
# Ninja 等非 IDE 生成器不支持 -T 参数，通过 -DEC_ENFORCE_V141_TOOLSET=OFF 关闭此检查
option(ENFORCE_V141_TOOLSET "强制要求 v141 工具集" ON)

if(ENFORCE_V141_TOOLSET AND MSVC AND NOT CMAKE_GENERATOR_TOOLSET STREQUAL "v141")
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

# 公共库路径（通过环境变量 LUDASHI_PC_PUBLIC_ROOT 指定）
if(NOT DEFINED ENV{LUDASHI_PC_PUBLIC_ROOT})
  message(FATAL_ERROR "环境变量 LUDASHI_PC_PUBLIC_ROOT 未设置，请设置为 public 公共库根目录，例如：\n"
    "  set LUDASHI_PC_PUBLIC_ROOT=D:/work/codeup.aliyun.com/public")
endif()
set(PUBLIC_DIR "$ENV{LUDASHI_PC_PUBLIC_ROOT}")
set(PUBLIC_INCLUDE_DIR "${PUBLIC_DIR}/include")
set(PUBLIC_LIB_2017_DIR "${PUBLIC_DIR}/lib_2017")

# 创建 GTest::{gtest,gtest_main,gmock} imported targets
foreach(name gtest gtest_main gmock)
  add_library(GTest::${name} STATIC IMPORTED)
  set_target_properties(GTest::${name} PROPERTIES
    IMPORTED_LOCATION             "${PUBLIC_LIB_2017_DIR}/${name}_2017_T32.lib"
    IMPORTED_LOCATION_DEBUG       "${PUBLIC_LIB_2017_DIR}/${name}_2017_T32d.lib"
    INTERFACE_INCLUDE_DIRECTORIES "${PUBLIC_INCLUDE_DIR}")
endforeach()
set_target_properties(GTest::gtest_main GTest::gmock PROPERTIES INTERFACE_LINK_LIBRARIES GTest::gtest)
enable_testing()

add_subdirectory(tests)
add_subdirectory(src)
