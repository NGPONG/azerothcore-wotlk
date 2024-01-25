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

# 设置 app(worldserver/authserver) 的 target 的编译信息还有安装信息
#   * authserver EXEC ->
#       * acore-core-interface
#       * shared
#   * worldserver EXEC ->
#       * acore-core-interface
#       * modules
#       * scripts
#       * game
#       * gsoap
#       * readline
#       * gperftools
add_subdirectory(apps)

# 设置 database 的 target
#   * database ${BUILD_SHARED_LIBS} ->
#       * acore-core-interface
#       * mysql
#       * common
if ((APPS_BUILD AND NOT APPS_BUILD STREQUAL "none") OR BUILD_TOOLS_DB_IMPORT)
  add_subdirectory(database)
endif()

# 设置 shared 的 target
#   * shared ${BUILD_SHARED_LIBS} ->
#       * database
if (BUILD_APPLICATION_AUTHSERVER OR BUILD_APPLICATION_WORLDSERVER)
  add_subdirectory(shared)
endif()

# 设置 game 的 target
#   * game-interface INTREACE ->
#       * shared
#   * game STATIC ->
#       * acore-core-interface
#       * game-interface
# 设置 scripts 的 target
#   * scripts_<script_dir_name> SHARED ->
#       * acore-core-interface
#       * game
#   * scripts STATIC ->
#       * acore-core-interface
#       * game
if (BUILD_APPLICATION_WORLDSERVER)
  add_subdirectory(game)
  add_subdirectory(scripts)
endif()
