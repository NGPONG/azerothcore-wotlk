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

# Make the script module list available in the current scope
GetApplicationsList(APPLICATIONS_BUILD_LIST)

if (APPS_BUILD STREQUAL "none")
  set(APPS_DEFAULT_BUILD "disabled")
else()
  set(APPS_DEFAULT_BUILD "enabled")
endif()

# Sets BUILD_APPS_USE_WHITELIST
# Sets BUILD_APPS_WHITELIST
if (APPS_BUILD MATCHES "-only")
  set(BUILD_APPS_USE_WHITELIST ON)

  if (APPS_BUILD STREQUAL "auth-only")
    list(APPEND BUILD_APPS_WHITELIST authserver)
  endif()

  if (APPS_BUILD STREQUAL "world-only")
    list(APPEND BUILD_APPS_WHITELIST worldserver)
  endif()
endif()

# Set the SCRIPTS_${BUILD_APP} variables from the
# variables set above
foreach(BUILD_APP ${APPLICATIONS_BUILD_LIST})
  ApplicationNameToVariable(${BUILD_APP} BUILD_APP_VARIABLE)

  if(${BUILD_APP_VARIABLE} STREQUAL "default")
    if(BUILD_APPS_USE_WHITELIST)
      list(FIND BUILD_APPS_WHITELIST "${BUILD_APP}" INDEX)
      if(${INDEX} GREATER -1)
        set(${BUILD_APP_VARIABLE} ${APPS_DEFAULT_BUILD})
      else()
        set(${BUILD_APP_VARIABLE} "disabled")
      endif()
    else()
      set(${BUILD_APP_VARIABLE} ${APPS_DEFAULT_BUILD})
    endif()
  endif()

  # Build the Graph values
  if(${BUILD_APP_VARIABLE} MATCHES "enabled")
    list(APPEND BUILD_APP_GRAPH_KEYS apps)
    set(BUILD_APP_VALUE_DISPLAY_apps apps)
    list(APPEND BUILD_APP_VALUE_CONTAINS_apps ${BUILD_APP})
  else()
    list(APPEND BUILD_APP_GRAPH_KEYS disabled)
    set(BUILD_APP_VALUE_DISPLAY_disabled disabled)
    list(APPEND BUILD_APP_VALUE_CONTAINS_disabled ${BUILD_APP})
  endif()
endforeach()

list(SORT BUILD_APP_GRAPH_KEYS)
list(REMOVE_DUPLICATES BUILD_APP_GRAPH_KEYS)

# Display the graphs
message("")
message("* Apps build list (${APPS_BUILD}):")
message("  |")

foreach(BUILD_APP_GRAPH_KEY ${BUILD_APP_GRAPH_KEYS})
  if(NOT BUILD_APP_GRAPH_KEY STREQUAL "disabled")
    message("  +- ${BUILD_APP_VALUE_DISPLAY_${BUILD_APP_GRAPH_KEY}}")
  else()
    message("  |  ${BUILD_APP_VALUE_DISPLAY_${BUILD_APP_GRAPH_KEY}}")
  endif()
  foreach(BUILD_APP_GRAPH_ENTRY ${BUILD_APP_VALUE_CONTAINS_${BUILD_APP_GRAPH_KEY}})
    message("  |   +- ${BUILD_APP_GRAPH_ENTRY}")
  endforeach()
  message("  |")
endforeach()

message("")

GroupSources(${CMAKE_CURRENT_SOURCE_DIR})

# Generates the actual apps projects
foreach(APPLICATION_NAME ${APPLICATIONS_BUILD_LIST})
  # 获取 app 的目录路径
  GetPathToApplication(${APPLICATION_NAME} SOURCE_APP_PATH)

  ApplicationNameToVariable(${APPLICATION_NAME} BUILD_APP_VARIABLE)

  if (${BUILD_APP_VARIABLE} STREQUAL "disabled")
    continue()
  endif()

  # 收集当前目录下非 Exclude 指定目录下的文件(源文件和头文件)
  unset(APP_PRIVATE_SOURCES)
  CollectSourceFiles(
    ${SOURCE_APP_PATH}
    APP_PRIVATE_SOURCES
    # Exclude
    ${SOURCE_APP_PATH}/PrecompiledHeaders)

  # windows 平台需要额外添加一些文件
  if (WIN32)
    list(APPEND APP_PRIVATE_SOURCES ${winDebugging})

    if (${APPLICATION_NAME} MATCHES "worldserver")
      list(APPEND APP_PRIVATE_SOURCES ${winService})
    endif()

    if (MSVC)
      list(APPEND APP_PRIVATE_SOURCES ${SOURCE_APP_PATH}/${APPLICATION_NAME}.rc)
    endif()
  endif()

  GetProjectNameOfApplicationName(${APPLICATION_NAME} APP_PROJECT_NAME)

  # 构建可执行文件 target [worldserver or authserver]
  add_executable(${APP_PROJECT_NAME}
    ${APP_PRIVATE_SOURCES})

  # 这里的 revision.h 是一个 target
  add_dependencies(${APP_PROJECT_NAME} revision.h)

  target_link_libraries(${APP_PROJECT_NAME}
    PRIVATE
      acore-core-interface)

  # authserver 需要依赖 shared 文件夹下的东西
  if (${APP_PROJECT_NAME} MATCHES "authserver")
    target_link_libraries(${APP_PROJECT_NAME}
      PUBLIC
        shared)
  # worldserver 依赖 scripts modules game 还有一些第三方库
  elseif(${APP_PROJECT_NAME} MATCHES "worldserver")
    target_link_libraries(${APP_PROJECT_NAME}
      PUBLIC
        modules
        scripts
        game
        gsoap
        readline
        gperftools)

    if (UNIX AND NOT NOJEM)
      set(${APP_PROJECT_NAME}_LINK_FLAGS "-pthread -lncurses ${${APP_PROJECT_NAME}_LINK_FLAGS}")
    endif()

    set_target_properties(${APP_PROJECT_NAME} PROPERTIES LINK_FLAGS "${${APP_PROJECT_NAME}_LINK_FLAGS}")

    # 此行代码不知道干啥的
    # Add all dynamic projects as dependency to the worldserver
    if (WORLDSERVER_DYNAMIC_SCRIPT_MODULES_DEPENDENCIES)
      add_dependencies(${APP_PROJECT_NAME} ${WORLDSERVER_DYNAMIC_SCRIPT_MODULES_DEPENDENCIES})
    endif()
  endif()

  # 收集当前目录下(非 Exclude 指定目录)的所有文件夹
  unset(APP_PUBLIC_INCLUDES)
  CollectIncludeDirectories(
    ${SOURCE_APP_PATH}
    APP_PUBLIC_INCLUDES
    # Exclude
    ${SOURCE_APP_PATH}/PrecompiledHeaders)

  # 添加头文件包含
  target_include_directories(${APP_PROJECT_NAME}
    PUBLIC
      ${CMAKE_CURRENT_SOURCE_DIR})

  target_include_directories(${APP_PROJECT_NAME}
    PUBLIC
      ${APP_PUBLIC_INCLUDES}
    PRIVATE
      ${CMAKE_CURRENT_BINARY_DIR}/${APPLICATION_NAME})

  # ide 支持
  set_target_properties(${APP_PROJECT_NAME}
    PROPERTIES
      FOLDER
        "server")

  # Install config
  CopyApplicationConfig(${APP_PROJECT_NAME} ${APPLICATION_NAME})

  if (UNIX)
    install(TARGETS ${APP_PROJECT_NAME} DESTINATION bin)
  elseif (WIN32)
    install(TARGETS ${APP_PROJECT_NAME} DESTINATION "${CMAKE_INSTALL_PREFIX}")
  endif()

  set(PATH_TO_PCH ${SOURCE_APP_PATH}/PrecompiledHeaders/${APPLICATION_NAME}PCH.h)

  # precompile header 支持
  # Generate precompiled header
  if (USE_COREPCH AND EXISTS ${PATH_TO_PCH})
    add_cxx_pch(${APP_PROJECT_NAME} ${PATH_TO_PCH})
  endif()
endforeach()
