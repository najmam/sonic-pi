# This CMakeLists builds all the external libraries required for a Sonic Pi
cmake_minimum_required(VERSION 3.13)

message(STATUS "Externals Builder")

project(ExternalsBuilder)

if(WIN32)
  set(ERLANG_INCLUDE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/../../prebuilt/windows/headers/erlang CACHE STRING "Path to the Erlang header files")
elseif(${CMAKE_SYSTEM_NAME} MATCHES Darwin) # macOS
  set(ERLANG_INCLUDE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/../../prebuilt/macos/headers/erlang CACHE STRING "Path to the Erlang header files")
else()
  execute_process(
    COMMAND erl -noshell -eval "io:format(\"~s~n\", [filename:join([lists:concat([code:root_dir(), \"/erts-\", erlang:system_info(version)]), \"include\"])]), init:stop(0)."
    OUTPUT_VARIABLE FOUND_ERLANG_INCLUDE_PATH OUTPUT_STRIP_TRAILING_WHITESPACE)
  set(ERLANG_INCLUDE_PATH ${FOUND_ERLANG_INCLUDE_PATH} CACHE STRING "Path to the Erlang header files")
endif()

option(USE_SYSTEM_LIBS "Use system libraries instead of bundled libraries when building" OFF)

message(STATUS "ERLANG_INCLUDE_PATH: ${ERLANG_INCLUDE_PATH}")
message(STATUS "USE_SYSTEM_LIBS: ${USE_SYSTEM_LIBS}")

include(GNUInstallDirs)
include(ExternalProject)

set(CMAKE_OSX_DEPLOYMENT_TARGET '10.15')

# sp_midi
ExternalProject_Add(sp_midi
    PREFIX sp_midi-prefix
    SOURCE_DIR ${CMAKE_CURRENT_LIST_DIR}/sp_midi
    INSTALL_DIR ${CMAKE_CURRENT_SOURCE_DIR}/../server/beam/tau/priv
    CMAKE_ARGS
        -DCMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_SOURCE_DIR}/../server/beam/tau/priv
        -DERLANG_INCLUDE_PATH=${ERLANG_INCLUDE_PATH}
        -DCMAKE_OSX_DEPLOYMENT_TARGET=${CMAKE_OSX_DEPLOYMENT_TARGET}
        -DUSE_SYSTEM_RTMIDI=${USE_SYSTEM_LIBS}
    BUILD_COMMAND ${CMAKE_COMMAND} --build . --config Release
    )

ExternalProject_Add(sp_link
    PREFIX sp_link-prefix
    SOURCE_DIR ${CMAKE_CURRENT_LIST_DIR}/sp_link
    INSTALL_DIR ${CMAKE_CURRENT_SOURCE_DIR}/../server/beam/tau/priv
    CMAKE_ARGS
        -DCMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_SOURCE_DIR}/../server/beam/tau/priv
        -DERLANG_INCLUDE_PATH=${ERLANG_INCLUDE_PATH}
        -DCMAKE_OSX_DEPLOYMENT_TARGET=${CMAKE_OSX_DEPLOYMENT_TARGET}

    BUILD_COMMAND ${CMAKE_COMMAND} --build . --config Release
    )

find_package(PkgConfig REQUIRED)

# Kick off a generation by making a dummy/empty project
add_library(ExternalsBuilder STATIC externals.cpp)

if(WIN32)
  if(CMAKE_SIZEOF_VOID_P EQUAL 8) # 64-bit
      add_custom_command(
        TARGET ExternalsBuilder PRE_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_CURRENT_SOURCE_DIR}/../../prebuilt/windows/x64/ ${CMAKE_CURRENT_SOURCE_DIR}/../server/native/
      )
  elseif(CMAKE_SIZEOF_VOID_P EQUAL 4) # 32-bit
      add_custom_command(
        TARGET ExternalsBuilder PRE_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_CURRENT_SOURCE_DIR}/../../prebuilt/windows/x86/ ${CMAKE_CURRENT_SOURCE_DIR}/../server/native/
      )
  endif()
elseif(${CMAKE_SYSTEM_NAME} MATCHES Darwin) # macOS
  add_custom_command(
    TARGET ExternalsBuilder PRE_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_CURRENT_SOURCE_DIR}/../../prebuilt/macos/x64/ ${CMAKE_CURRENT_SOURCE_DIR}/../server/native/
    COMMAND ${CMAKE_COMMAND} -E create_symlink ${CMAKE_CURRENT_SOURCE_DIR}/../server/native/supercollider/scsynth ${CMAKE_CURRENT_SOURCE_DIR}/../server/native/scsynth
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_CURRENT_SOURCE_DIR}/../server/native/supercollider/extra-plugins/ ${CMAKE_CURRENT_SOURCE_DIR}/../server/native/supercollider/plugins/
    COMMAND ${CMAKE_COMMAND} -E rm -rf ${CMAKE_CURRENT_SOURCE_DIR}/../server/native/supercollider/extra-plugins
  )
endif()
