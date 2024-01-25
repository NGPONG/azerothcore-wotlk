#
# This file is part of the AzerothCore Project. See AUTHORS file for Copyright information
#
# This file is free software; as a special exception the author gives
# unlimited permission to copy and/or distribute it, with or without
# modifications, as long as this notice is preserved.
#
# This program is distributed in the hope that it will be useful, but
# WITHOUT ANY WARRANTY, to the extent permitted by law; without even the
# implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
#

# Require version Cmake
cmake_minimum_required(VERSION 3.16...3.22)

message(STATUS "CMake version: ${CMAKE_VERSION}")

# CMake policies (can not be handled elsewhere)
cmake_policy(SET CMP0005 NEW)

# Set projectname (must be done AFTER setting configurationtypes)
project(AzerothCore VERSION 3.0.0 LANGUAGES CXX C)

# 导出 compile_commands.json
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# 禁止在源码内构建项目
# add this options before PROJECT keyword
set(CMAKE_DISABLE_SOURCE_CHANGES ON)
set(CMAKE_DISABLE_IN_SOURCE_BUILD ON)

# 设置 build 的 target 不使用 RPATH，除非我们 intsall 它
# Set RPATH-handing (CMake parameters)
set(CMAKE_SKIP_BUILD_RPATH 0)
set(CMAKE_BUILD_WITH_INSTALL_RPATH 0)
set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH 1)

set(AC_PATH_ROOT "${CMAKE_SOURCE_DIR}")

# set macro-directory
list(APPEND CMAKE_MODULE_PATH
        "${CMAKE_SOURCE_DIR}/src/cmake/macros")

include(CMakePrintHelpers)
include(CheckCXXSourceRuns)
include(CheckIncludeFiles)
# 映射一些帮助函数
include(ConfigureScripts)
# 映射一些帮助函数
include(ConfigureModules)
# 映射一些帮助函数，并声明了 BUILD_APPLICATION_AUTHSERVER BUILD_APPLICATION_WORLDSERVER 变量
include(ConfigureApplications)
# 映射一些帮助函数，并声明了 BUILD_TOOLS_MAPS BUILD_TOOLS_DB_IMPORT 变量
include(ConfigureTools)

# 映射一些帮助函数或宏
# some utils for cmake
include(deps/acore/cmake-utils/utils.cmake)

# 映射一些帮助函数或宏
include(src/cmake/ac_macros.cmake)

# 设置一些变量的初始化，参考 doc_cmake
# set default buildoptions and print them
include(conf/dist/config.cmake)

# 用户自定义配置
# load custom configurations for cmake if exists
if(EXISTS "${CMAKE_SOURCE_DIR}/conf/config.cmake")
    include(conf/config.cmake)
endif()

#
# Loading dyn modules
#

# 为 MODULES 添加子模块的相关处理，并且还包含禁用模块变量的处理
# add modules and dependencies
CU_SUBDIRLIST(sub_DIRS  "${CMAKE_SOURCE_DIR}/modules" FALSE FALSE)
FOREACH(subdir ${sub_DIRS})

    get_filename_component(MODULENAME ${subdir} NAME)

    if (";${DISABLED_AC_MODULES};" MATCHES ";${MODULENAME};")
        continue()
    endif()

    STRING(REPLACE "${CMAKE_SOURCE_DIR}/" "" subdir_rel ${subdir})
    if(EXISTS "${subdir}/CMakeLists.txt")
        add_subdirectory("${subdir_rel}")
    endif()
ENDFOREACH()

CU_RUN_HOOK("AFTER_LOAD_CONF")

# 设置编译类型
# build in Release-mode by default if not explicitly set
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "RelWithDebInfo")
endif()

# turn off PCH totally if enabled (hidden setting, mainly for devs)
if( NOPCH )
    set(USE_COREPCH 0)
    set(USE_SCRIPTPCH 0)
endif()

# 1. 支持 c++20 编译
# 2. 声明一些 INTERFACE TARGET，这些 target 会被多个地方引用(包括deps内的依赖项)，而且会根据当前操作系统环境进行修改
#   * acore-warning-interface
#   * acore-compile-option-interface
#   * acore-default-interface ->
#       * acore-compile-option-interface
#   * acore-no-warning-interface
#   * acore-hidden-symbols-interface
#   * acore-dependency-interface ->
#       * acore-default-interface
#       * acore-no-warning-interface
#       * acore-hidden-symbols-interface
#   * acore-core-interface ->
#       * acore-default-interface
#       * acore-warning-interface
include(ConfigureBaseTargets)
# 检查平台相关
# 配置一些基础变量
#   * CONF_DIR:${CMAKE_INSTALL_PREFIX}/etc
#   * LIBSDIR:${CMAKE_INSTALL_PREFIX}/lib
# 配置卸载功能，配置文件位于：${CMAKE_SOURCE_DIR}/src/cmake/platform/cmake_uninstall.in.cmake
# 配置编译时相关的一些设置
#   * 检查编译器版本
#   * 设置一些编译指令，如：警告、分析器相关设置(MSAN,TSAN)、动态库相关、添加一些预定义宏；设置的大部分是上一步暴露出来的 INTERFACE target
include(CheckPlatform)
# 暴露 GroupSources 宏，该宏用于一些 IDE 使用 cmake 生成工程时候的目录树文件组织
include(GroupSources)
# 暴露一些 Collect* 相关的函数
include(AutoCollect)
# 暴露一些 Copy* 相关的函数；主要是用于 cmake install 的时候拷贝 core 的配置文件的
include(ConfigInstall)

CU_RUN_HOOK("AFTER_LOAD_CMAKE_MODULES")

find_package(PCHSupport)
find_package(MySQL REQUIRED)

if(UNIX AND WITH_PERFTOOLS)
    find_package(Gperftools)
endif()

# Git package 用于确保安装的 git 版本，似乎作用于一些 dirty-testing？
if(NOT WITHOUT_GIT)
    find_package(Git)
endif()

# 1. 初始化 BUILDDIR 变量
# 2. 用于 "${BUILDDIR}/revision.h" 文件的创建
# Find revision ID and hash of the sourcetree
include(src/cmake/genrev.cmake)

# print out the results before continuing
include(src/cmake/showoptions.cmake)

#
# Loading framework
#

# 1. 设置依赖项子目录
# 2. 依赖项内的每个模块要不就是暴露一些 IMPORTED target 或是现场直接编译一个 target
add_subdirectory(deps)

# 1. 如果支持则设置 precompile header
# 2. 添加 common 的构建 target
add_subdirectory(src/common)

#
# Loading application sources
#

CU_RUN_HOOK("BEFORE_SRC_LOAD")

# 1. 初始化 windows 环境下的一些变量
# 2. 添加 src/genrev src/server src/tools 子目录
# add core sources
add_subdirectory(src)

# 添加 modules 子目录
if (BUILD_APPLICATION_WORLDSERVER)
  # add modules sources
  add_subdirectory(modules)
endif()

CU_RUN_HOOK("AFTER_SRC_LOAD")

# 单元测试编译支持
if (BUILD_TESTING AND BUILD_APPLICATION_WORLDSERVER)
    # we use these flags to get code coverage
    set(UNIT_TEST_CXX_FLAGS "-fprofile-arcs -ftest-coverage -fno-inline")

    # enable additional flags for GCC.
    if ( CMAKE_CXX_COMPILER_ID MATCHES GNU )
        set(UNIT_TEST_CXX_FLAGS "${UNIT_TEST_CXX_FLAGS} -fno-inline-small-functions -fno-default-inline")
    endif()

    message("Unit tests code coverage: enabling ${UNIT_TEST_CXX_FLAGS}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${UNIT_TEST_CXX_FLAGS}")

    include(src/cmake/googletest.cmake)
    fetch_googletest(
            ${PROJECT_SOURCE_DIR}/src/cmake
            ${PROJECT_BINARY_DIR}/googletest
    )

    enable_testing()
    add_subdirectory(src/test)

    add_custom_target(coverage DEPENDS coverage_command)

    add_custom_command(OUTPUT coverage_command
        # Run unit tests.
        COMMAND ctest
        # Run the graphical front-end for code coverage.
        COMMAND lcov    --directory src --capture --output-file coverage.info
        COMMAND lcov    --remove coverage.info '/usr/*' '${CMAKE_BINARY_DIR}/googletest/*' '${CMAKE_CURRENT_SOURCE_DIR}/src/test/*' --output-file coverage.info
        COMMAND genhtml -o ${CMAKE_CURRENT_SOURCE_DIR}/coverage-report coverage.info
        WORKING_DIRECTORY "${CMAKE_BINARY_DIR}"
        )
endif()
