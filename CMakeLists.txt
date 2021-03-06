############################################################################
#    Copyright (C) 2014 by Ahmed Charles - acharles@outlook.com            #
#    Copyright (C) 2015-2017 by Stephen Lyons - slysven@virginmedia.com    #
#                                                                          #
#    This program is free software; you can redistribute it and/or modify  #
#    it under the terms of the GNU General Public License as published by  #
#    the Free Software Foundation; either version 2 of the License, or     #
#    (at your option) any later version.                                   #
#                                                                          #
#    This program is distributed in the hope that it will be useful,       #
#    but WITHOUT ANY WARRANTY; without even the implied warranty of        #
#    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the         #
#    GNU General Public License for more details.                          #
#                                                                          #
#    You should have received a copy of the GNU General Public License     #
#    along with this program; if not, write to the                         #
#    Free Software Foundation, Inc.,                                       #
#    59 Temple Place - Suite 330, Boston, MA  02111-1307, USA.             #
############################################################################

# Should be called before PROJECT.
cmake_minimum_required(VERSION 3.1)

project(mudlet)

if(CMAKE_CONFIGURATION_TYPES)
  add_custom_target(check COMMAND ${CMAKE_CTEST_COMMAND}
                    --force-new-ctest-process --output-on-failure
                    --build-config "$<CONFIGURATION>")
else()
  add_custom_target(check COMMAND ${CMAKE_CTEST_COMMAND}
                    --force-new-ctest-process --output-on-failure)
endif()

enable_testing()

if(POLICY CMP0020)
  cmake_policy(SET CMP0020 NEW)
endif()

if(WIN32)
  set(APP_TARGET mudlet.exe)
elseif(APPLE)
  set(APP_TARGET Mudlet)
else()
  set(APP_TARGET mudlet)
endif()

# APP_BUILD should only be empty/null in official "release" builds,
# developers may like to set the MUDLET_VERSION_BUILD environment variable to
# their user and branch names to make it easier to tell different builds apart!
#
# Changing this pair of values affects: ctelnet.cpp, main.cpp, mudlet.cpp
# dlgAboutDialog.cpp and TLuaInterpreter.cpp.  It will not necessarily cause
# those files to be automatically rebuilt so you may need to 'touch' those files if
# the variables are changed and you are not doing a full, clean, rebuild!
# Use APP_VERSION, APP_BUILD and APP_TARGET defines in the source code if needed.
# IMPORTANT:
# To insure consistency please ensure the SAME of the first two values are also
# assigned to the "VERSION" and "BUILD" variables in the native qmake project
# file, which is NOW called: ./src/mudlet.pro
set(APP_VERSION 3.6.1)
if(DEFINED ENV{MUDLET_VERSION_BUILD} AND NOT $ENV{MUDLET_VERSION_BUILD} STREQUAL "")
  SET(APP_BUILD $ENV{MUDLET_VERSION_BUILD})
else()
  set(APP_BUILD "-dev")
endif()

set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

# Enable the built-in updater by default unless NO_INCLUDE_UPDATER is
# already defined. Linux packagers will find it useful to do this since
# package managers are responsible for updates there. Automatically exclude
# the update stuff from FreeBSD and Cygwin because if they ever get finished
# they have their own packaging system
if((DEFINED ENV{NO_INCLUDE_UPDATER} AND NOT $ENV{MUDLET_VERSION_BUILD} STREQUAL "") OR (CMAKE_SYSTEM_NAME STREQUAL "FreeBSD") OR (CMAKE_SYSTEM_NAME STREQUAL "Cygwin"))
  option(WITH_UPDATER "Include libraries and code to allow release versions to be updated on demand" OFF)
else()
  option(WITH_UPDATER "Include libraries and code to allow release versions to be updated on demand" ON)
endif()

if(APPLE)
  # Needed (just) on MacOs as an #include in luazip.h:
  add_subdirectory(3rdparty/luazip)
endif()

if(NOT EXISTS "${CMAKE_HOME_DIRECTORY}/3rdparty/edbee-lib/CMakeLists.txt")
  # The above CMakeList.txt is the top level one - we actually use the one
  # another level down in 3rdparty/edbee-lib/edbee-lib/CMakeLists.txt
  message(STATUS "git submodule for required edbee-lib editor widget missing from source code, will attempt to get it...")
  execute_process(COMMAND git submodule update --init 3rdparty/edbee-lib
    TIMEOUT 30
    WORKING_DIRECTORY ${CMAKE_HOME_DIRECTORY}
    OUTPUT_VARIABLE output_text
    ERROR_VARIABLE error_text)
  if(output_text OR error_text)
    message(STATUS ${output_text} ${error_text})
  endif()
endif()

if(WITH_UPDATER)
  if(NOT EXISTS "${CMAKE_HOME_DIRECTORY}/3rdparty/dblsqd/CMakeLists.txt")
    message(STATUS "git submodule for optional but wanted DBLSQD updater missing from source code, will attempt to get it...")
    execute_process(COMMAND git submodule update --init 3rdparty/dblsqd
      TIMEOUT 30
      WORKING_DIRECTORY ${CMAKE_HOME_DIRECTORY}
      OUTPUT_VARIABLE output_text
      ERROR_VARIABLE error_text)
    if(output_text OR error_text)
      message(STATUS ${output_text} ${error_text})
    endif()
  endif()
endif()

if(APPLE)
  if(WITH_UPDATER)
    if(NOT EXISTS "${CMAKE_HOME_DIRECTORY}/3rdparty/sparkle-glue/CMakeLists.txt")
      message(STATUS "git submodule for optional but wanted Sparkle glue for updater missing from source code, will attempt to get it...")
      execute_process(COMMAND git submodule update --init 3rdparty/sparkle-glue
        TIMEOUT 30
        WORKING_DIRECTORY ${CMAKE_HOME_DIRECTORY}
        OUTPUT_VARIABLE output_text
        ERROR_VARIABLE error_text)
      if(output_text OR error_text)
        message(STATUS ${output_text} ${error_text})
      endif()
    endif()
  endif()
endif()

if(EXISTS "${CMAKE_HOME_DIRECTORY}/3rdparty/edbee-lib/CMakeLists.txt")
  add_subdirectory(3rdparty/edbee-lib/edbee-lib)
elseif()
  message(FATAL_ERROR "Cannot locate edbee-lib editor widget submodule source code, build abandoned!")
endif()

if(WITH_UPDATER)
  if(EXISTS "${CMAKE_HOME_DIRECTORY}/3rdparty/dblsqd/CMakeLists.txt")
    add_subdirectory(3rdparty/dblsqd)
    add_definitions(-DINCLUDE_UPDATER)
  else()
    message(FATAL_ERROR "Cannot locate DBLSQD updater submodule source code, build abandoned!")
  endif()

  if(APPLE)
    if(EXISTS "${CMAKE_HOME_DIRECTORY}/3rdparty/sparkle-glue/CMakeLists.txt")
      add_subdirectory(3rdparty/sparkle-glue)
    else()
      message(FATAL_ERROR "Cannot locate Sparkle glue for updater submodule source code, build abandoned!")
    endif()
  endif()
endif()

if(APPLE AND WITH_UPDATER AND NOT EXISTS "${CMAKE_HOME_DIRECTORY}/3rdparty/cocoapods/Pods/Sparkle")
  message(STATUS "Sparkle CocoaPod is missing, running 'pod install' to get it...")
  execute_process(TIMEOUT 30
    WORKING_DIRECTORY "${CMAKE_HOME_DIRECTORY}/3rdparty/cocoapods"
    COMMAND pod install)
endif()

find_program(CCACHE_FOUND ccache)
if(CCACHE_FOUND)
  set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ccache)
  set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK ccache)
endif(CCACHE_FOUND)

add_subdirectory(src)
add_subdirectory(3rdparty/communi)
add_subdirectory(3rdparty/lua_yajl)
