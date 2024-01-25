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

# Enforce compileparameters for corebuilds under GCC
# This to stop a few silly crashes that could have been avoided IF people
# weren't doing some -O3 psychooptimizations etc.

# Specified files for Windows
if (WIN32)
  # Crash logs
  set(winDebugging
    ${CMAKE_SOURCE_DIR}/src/common/Debugging/WheatyExceptionReport.cpp
    ${CMAKE_SOURCE_DIR}/src/common/Debugging/WheatyExceptionReport.h)

  # Service
  set(winService
    ${CMAKE_SOURCE_DIR}/src/common/Platform/ServiceWin32.cpp
    ${CMAKE_SOURCE_DIR}/src/common/Platform/ServiceWin32.h)
endif()

if(CMAKE_COMPILER_IS_GNUCXX AND NOT MINGW)
  add_definitions(-fno-delete-null-pointer-checks)
endif()

# 为 src/cmake/genrev.cmake 中生成的 revision.h 文件创建 target
# 为什么不在 genrev.cmake 文件里面直接创建呢？
add_subdirectory(genrev)

# 设置 server 目录下的 target，编译信息还有安装信息
add_subdirectory(server)

if (TOOLS_BUILD AND NOT TOOLS_BUILD STREQUAL "none")
  add_subdirectory(tools)
endif()
