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

# 收集当前目录下非 Exclude 指定目录下的文件(源文件和头文件)
CollectSourceFiles(
  ${CMAKE_CURRENT_SOURCE_DIR}
  PRIVATE_SOURCES
  # Exclude
  ${CMAKE_CURRENT_SOURCE_DIR}/Debugging
  ${CMAKE_CURRENT_SOURCE_DIR}/Platform
  ${CMAKE_CURRENT_SOURCE_DIR}/Collision
  ${CMAKE_CURRENT_SOURCE_DIR}/Navigation
  ${CMAKE_CURRENT_SOURCE_DIR}/PrecompiledHeaders
)

# 依据构建 WORLD SERVER 或者是 MAP TOOLS 的配置重新计算一遍源文件
if (BUILD_APPLICATION_WORLDSERVER OR BUILD_TOOLS_MAPS)
  unset(PRIVATE_SOURCES)
  CollectSourceFiles(
    ${CMAKE_CURRENT_SOURCE_DIR}
    PRIVATE_SOURCES
    # Exclude
    ${CMAKE_CURRENT_SOURCE_DIR}/Debugging
    ${CMAKE_CURRENT_SOURCE_DIR}/Platform
    ${CMAKE_CURRENT_SOURCE_DIR}/PrecompiledHeaders
  )
endif()

# Manually set sources for Debugging directory as we don't want to include WheatyExceptionReport in common project
# It needs to be included both in authserver and worldserver for the static global variable to be properly initialized
# and to handle crash logs on windows
list(APPEND PRIVATE_SOURCES
  ${CMAKE_CURRENT_SOURCE_DIR}/Debugging/Errors.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/Debugging/Errors.h)

if (USE_COREPCH)
  set(PRIVATE_PCH_HEADER PrecompiledHeaders/commonPCH.h)
endif()

# IDE支持
GroupSources(${CMAKE_CURRENT_SOURCE_DIR})

# PRIVATE_SOURCES 默认情况下会包含头文件，但是 cmake 会帮我们过滤掉
add_library(
  common
  ${PRIVATE_SOURCES}
)

# 收集当前目录下(非 Exclude 指定目录)的所有文件夹
CollectIncludeDirectories(
  ${CMAKE_CURRENT_SOURCE_DIR}
  PUBLIC_INCLUDES
  # Exclude
  ${CMAKE_CURRENT_SOURCE_DIR}/PrecompiledHeaders)

# 添加包含目录
target_include_directories(common
  PUBLIC
    # Provide the binary dir for all child targets
    ${CMAKE_BINARY_DIR}
    ${PUBLIC_INCLUDES}
  PRIVATE
    ${CMAKE_CURRENT_BINARY_DIR})

# 添加链接项；PUBLIC 下的 target 通过 add_subdirectory(deps) 暴露出来
target_link_libraries(common
  PRIVATE
    acore-core-interface
  PUBLIC
    boost
    argon2
    sfmt
    utf8cpp
    openssl
    threads
    jemalloc
    stdfs
    fmt)

# 构建 WORLD 或者 MMAP TOOL 需要引入 3D 引擎库
if (BUILD_APPLICATION_WORLDSERVER OR BUILD_TOOLS_MAPS)
  target_link_libraries(common
    PUBLIC
      g3dlib
      Detour)
endif()

# IDE支持
set_target_properties(common
  PROPERTIES
    FOLDER
      "server")

if(BUILD_SHARED_LIBS)
  if(UNIX)
    install(TARGETS common
      LIBRARY
        DESTINATION lib)
  elseif(WIN32)
    install(TARGETS common
      RUNTIME
        DESTINATION "${CMAKE_INSTALL_PREFIX}")
  endif()
endif()

# precompile header 支持
# Generate precompiled header
if (USE_COREPCH)
  add_cxx_pch(common ${PRIVATE_PCH_HEADER})
endif ()
