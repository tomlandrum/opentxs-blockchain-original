#[[
// clang-format off
]]#
# Copyright (c) 2019 The Open-Transactions developers
# This Source Code Form is subject to the terms of the Mozilla Public
# License, v. 2.0. If a copy of the MPL was not distributed with this
# file, You can obtain one at http://mozilla.org/MPL/2.0/.

cmake_minimum_required(VERSION 3.12)
project(opentxs-blockchain)
include(GNUInstallDirs)

#[[
if(NOT ANDROID)
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM LAST)
find_package(Git)
if(GIT_FOUND)
    execute_process(COMMAND git "describe" OUTPUT_VARIABLE GIT_VERSION WORKING_DIRECTORY ${CMAKE_SOURCE_DIR} OUTPUT_STRIP_TRAILING_WHITESPACE)
    string(REGEX REPLACE "^([0-9]+)\\..*" "\\1" VERSION_MAJOR "${GIT_VERSION}")
    string(REGEX REPLACE "^${VERSION_MAJOR}\\.([0-9]+).*" "\\1" VERSION_MINOR "${GIT_VERSION}")
    string(REGEX REPLACE "^${VERSION_MAJOR}\\.${VERSION_MINOR}\\.([0-9]+).*" "\\1" VERSION_BUGFIX "${GIT_VERSION}")
    string(REGEX REPLACE "^${VERSION_MAJOR}\\.${VERSION_MINOR}\\.${VERSION_BUGFIX}-([0-9]+)-.*" "\\1" VERSION_NEW_COMMITS "${GIT_VERSION}")
    string(REGEX REPLACE "^${VERSION_MAJOR}\\.${VERSION_MINOR}\\.${VERSION_BUGFIX}-${VERSION_NEW_COMMITS}-(.*)" "\\1" VERSION_SHA1 "${GIT_VERSION}")
    if("${VERSION_NEW_COMMITS}" STREQUAL "${GIT_VERSION}")
        set(VERSION_STRING "${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_BUGFIX}")
        set(VERSION_NEW_COMMITS "")
        set(VERSION_SHA1 "")
    else()
        set(VERSION_STRING "${VERSION_MAJOR}.${VERSION_MINOR}.${VERSION_BUGFIX}-${VERSION_NEW_COMMITS}-${VERSION_SHA1}")
    endif()
    if ("${VERSION_STRING}" STREQUAL "..")
        message(FATAL_ERROR "Version string missing. Did you git clone using the --depth parameter?")
    endif()
else()
  message(FATAL_ERROR "Git not found.")
endif()
endif() # NOT ANDROID
]]

#-----------------------------------------------------------------------------
# Options for building

if(ANDROID)
  option(BUILD_TESTS         "Build the unit tests." OFF)
else()
  option(BUILD_TESTS         "Build the unit tests." ON)
endif()

option(BUILD_VERBOSE       "Verbose build output." ON)
option(USE_CCACHE          "Use ccache." OFF)
option(OT_SANITIZE         "Enable sanitize options" ON)

option(BUILD_SHARED_LIBS   "Build shared libraries." ON)

set(PACKAGE_CONTACT        ""                             CACHE <TYPE>  "Package Maintainer")
set(PACKAGE_VENDOR         "Open Transactions Developers" CACHE <TYPE>  "Package Vendor")

if(BUILD_VERBOSE)
  set(CMAKE_VERBOSE_MAKEFILE true)
endif()

#-----------------------------------------------------------------------------
# Collect all binaries into bin subdirectory

set(LIBRARY_OUTPUT_PATH  ${CMAKE_CURRENT_BINARY_DIR}/lib)


#-----------------------------------------------------------------------------
# Check which compiler to use

INCLUDE(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG(-std=c++17 HAVE_STD17)

if(HAVE_STD17)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++17")
else()
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++1z")
endif()

if(${CMAKE_CXX_COMPILER_ID} MATCHES GNU)
  set(GCC_WARN_FLAGS "-Wno-attributes -Wnoexcept -Wstrict-null-sentinel -Wlogical-op")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthread")

  if(ANDROID)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fexceptions -frtti")
  endif()
elseif(${CMAKE_CXX_COMPILER_ID} MATCHES Clang)
  set(CMAKE_COMPILER_IS_CLANGXX 1)

  if(APPLE)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -stdlib=libc++")
    set(CMAKE_CXX_LINK_FLAGS "${CMAKE_CXX_LINK_FLAGS} -stdlib=libc++")
  else()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthread")
  endif()
endif()

if(BUILD_SHARED_LIBS)
  # Windows would complain about unknown flag -fPIC
  if(NOT WIN32)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC")
  endif()
endif()

if(WIN32)
  add_definitions(-D_UNICODE)
else()
  set(PEDANTIC_CXX_FLAGS "-Wall -Werror -Wextra -pedantic -Wno-missing-braces -Wcast-align -Wctor-dtor-privacy -Wdisabled-optimization -Winit-self -Wmissing-include-dirs -Wswitch-default -Wold-style-cast -Woverloaded-virtual -Wmissing-declarations -Wno-deprecated-declarations -Wno-unused-parameter -Wno-old-style-cast -Wunused-macros -Wundef -Wformat=2 -Wno-format-nonliteral -Wno-disabled-optimization ${GCC_WARN_FLAGS}")
endif()

if(ANDROID)
  set(PEDANTIC_CXX_FLAGS "${PEDANTIC_CXX_FLAGS} -Wno-missing-field-initializers")
  add_definitions(-D__STDC_LIMIT_MACROS)
endif()

if(CMAKE_COMPILER_IS_CLANGXX)
  set(PEDANTIC_CXX_FLAGS "${PEDANTIC_CXX_FLAGS} -Wno-unused-parameter -Wno-old-style-cast")
endif()

if(CMAKE_BUILD_TYPE MATCHES "Debug" AND OT_SANITIZE)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=undefined,address -Wformat -Werror=format-security -Werror=array-bounds -fno-optimize-sibling-calls -fno-omit-frame-pointer")
  set(CMAKE_CXX_LINK_FLAGS "${CMAKE_CXX_LINK_FLAGS} -fsanitize=undefined,address")
  if(${CMAKE_CXX_COMPILER_ID} MATCHES GNU)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fuse-ld=gold")
    set(CMAKE_CXX_LINK_FLAGS "${CMAKE_CXX_LINK_FLAGS} -fuse-ld=gold")
  endif() #GNU LINKER
endif() # DEBUG

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${PEDANTIC_CXX_FLAGS}")

message(STATUS "General -------------------------------------")
message(STATUS "opentxs-blockchain version:  ${VERSION_STRING}")
message(STATUS "CMake version:          ${CMAKE_VERSION}")
message(STATUS "System:                 ${CMAKE_SYSTEM}")
message(STATUS "Processor:              ${CMAKE_SYSTEM_PROCESSOR}")
message(STATUS "C++ Compiler ID:        ${CMAKE_CXX_COMPILER_ID}")
message(STATUS "C++ Flags:              ${CMAKE_CXX_FLAGS}")
message(STATUS "C++ link flags:         ${CMAKE_CXX_LINK_FLAGS}")
message(STATUS "Verbose:                ${BUILD_VERBOSE}")
message(STATUS "Testing:                ${BUILD_TESTS}")

#-----------------------------------------------------------------------------
# Third party libraries

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

if(BUILD_TESTS AND NOT OT_BUNDLED_OPENTXS_BLOCKCHAIN)
  find_package(GTest REQUIRED)
  enable_testing()
endif()

#-----------------------------------------------------------------------------
# Force out-of-source build

if(${CMAKE_CURRENT_SOURCE_DIR} STREQUAL ${CMAKE_CURRENT_BINARY_DIR})
message(FATAL_ERROR "CMake generation is not allowed within the source directory!
Remove the CMakeCache.txt file and try again from another folder, e.g.:

   rm CMakeCache.txt
   mkdir build
   cd build
   cmake ..
")
endif()

#-----------------------------------------------------------------------------
# Use ccache

if (USE_CCACHE)
  find_program(CCACHE "ccache")
  if (CCACHE)
    SET_PROPERTY(GLOBAL PROPERTY RULE_LAUNCH_COMPILE "CCACHE_CPP2=yes ${CCACHE}")
    SET_PROPERTY(GLOBAL PROPERTY RULE_LAUNCH_LINK ${CCACHE})
  endif(CCACHE)
endif (USE_CCACHE)

#-----------------------------------------------------------------------------
# Default to Release build

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release CACHE STRING
      "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel."
      FORCE)
endif()


#-----------------------------------------------------------------------------
# Source Definitions

if(WIN32)
    if(NOT IDC_STATIC)
        add_definitions(-DIDC_STATIC=-1)
    endif()

    add_definitions(-DOT_COMPANYNAME="Open Transactions Community")
    add_definitions(-DOT_PRODUCTNAME="Open Transactions")
    add_definitions(-DOT_PRODUCTVERSION="${VERSION_MAJOR},${VERSION_MINOR},${VERSION_BUGFIX},0")
    add_definitions(-DOT_API_NAME="Open Transactions Blockchain API")
    add_definitions(-DOT_LIBRARY_NAME="Open Transactions Blockchain Library")
    add_definitions(-DOT_PROMPT_NAME="Open Transactions Blockchain Prompt")
endif()

#-----------------------------------------------------------------------------
# Build source

#Fix RPATH
if(APPLE)
    # use, i.e. don't skip the full RPATH for the build tree
    SET(CMAKE_SKIP_BUILD_RPATH  FALSE)

    # when building, don't use the install RPATH already
    # (but later on when installing)
    SET(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE)

    SET(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")

    # add the automatically determined parts of the RPATH
    # which point to directories outside the build tree to the install RPATH
    SET(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

    # the RPATH to be used when installing, but only if it's not a system directory
    LIST(FIND CMAKE_PLATFORM_IMPLICIT_LINK_DIRECTORIES "${CMAKE_INSTALL_PREFIX}/lib" isSystemDir)
    IF("${isSystemDir}" STREQUAL "-1")
    SET(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
    ENDIF("${isSystemDir}" STREQUAL "-1")
endif()

function(set_lib_property lib)
  if (NOT ANDROID)
    set_target_properties(${lib} PROPERTIES VERSION ${VERSION_STRING} SOVERSION ${VERSION_MAJOR})
  endif()
endfunction(set_lib_property)

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)
include_directories(SYSTEM ${CMAKE_CURRENT_BINARY_DIR}/src)

add_subdirectory(src)

#-----------------------------------------------------------------------------
# Build Unit tests

if(GTEST_FOUND AND NOT OT_BUNDLED_OPENTXS_BLOCKCHAIN)
  add_subdirectory(tests)
endif()

#-----------------------------------------------------------------------------
# Produce a cmake-package

if((NOT ANDROID) AND (NOT OT_BUNDLED_OPENTXS_BLOCKCHAIN))
  include(CMakePackageConfigHelpers)
  set(CMAKE_CONFIG_DEST "${CMAKE_INSTALL_LIBDIR}/opentxs-blockchain")
  set(OPENTXS_BLOCKCHAIN_INCLUDE_DIRS "${CMAKE_INSTALL_PREFIX}/include")

  if(BUILD_SHARED_LIBS)
    set(OPENTXS_BLOCKCHAIN_LIBRARIES "${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}/${CMAKE_SHARED_LIBRARY_PREFIX}opentxs-blockchain${CMAKE_SHARED_LIBRARY_SUFFIX}")
  else()
    set(OPENTXS_BLOCKCHAIN_LIBRARIES "${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}/${CMAKE_STATIC_LIBRARY_PREFIX}opentxs-blockchain${CMAKE_STATIC_LIBRARY_PREFIX}")
  endif()

  configure_package_config_file(
         "cmake/opentxs-blockchainConfig.cmake.in"
         "${CMAKE_BINARY_DIR}/opentxs-blockchainConfig.cmake"
         INSTALL_DESTINATION ${CMAKE_CONFIG_DEST}
         PATH_VARS OPENTXS_BLOCKCHAIN_INCLUDE_DIRS OPENTXS_BLOCKCHAIN_LIBRARIES)

  write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/opentxs-blockchainConfigVersion.cmake"
    VERSION 1
    COMPATIBILITY AnyNewerVersion)

  install(FILES
          "${CMAKE_CURRENT_BINARY_DIR}/opentxs-blockchainConfig.cmake"
          "${CMAKE_CURRENT_BINARY_DIR}/opentxs-blockchainConfigVersion.cmake"
          DESTINATION ${CMAKE_CONFIG_DEST}
          COMPONENT dev)

  install(EXPORT opentxs-blockchain-targets
          DESTINATION ${CMAKE_CONFIG_DEST})
endif()

#-----------------------------------------------------------------------------
# Produce a pkg-config file

string(REGEX REPLACE "^${CMAKE_INSTALL_PREFIX}(.*)" "\\1" LIB_INSTALL_DIR "${CMAKE_INSTALL_LIBDIR}")
configure_file("cmake/opentxs-blockchain.pc.in" "opentxs-blockchain.pc" @ONLY)
  install(FILES "${CMAKE_CURRENT_BINARY_DIR}/opentxs-blockchain.pc"
          DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig"
          COMPONENT main)

#-----------------------------------------------------------------------------
# Install

install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/include/
        DESTINATION opentxs
        COMPONENT dev
        FILES_MATCHING PATTERN "*.hpp")

#-----------------------------------------------------------------------------
# Uninstal
configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/uninstall.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/uninstall.cmake"
    IMMEDIATE @ONLY)

if(NOT OT_BUNDLED_OPENTXS_BLOCKCHAIN)
  add_custom_target(uninstall
      COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/uninstall.cmake)
endif()
#[[
// clang-format on
]]#
