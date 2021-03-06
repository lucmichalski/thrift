#
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements. See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership. The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License. You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied. See the License for the
# specific language governing permissions and limitations
# under the License.
#

cmake_minimum_required(VERSION 3.5)

include("cmake/HunterGate.cmake")
HunterGate(
    URL "https://github.com/ruslo/hunter/archive/v0.14.25.tar.gz"
    SHA1 "a7ba9775c1206ced7cac204db802b116736ca481"
)

project("Apache Thrift")

set(is_android ${ANDROID}) # syntax compatibility
string(COMPARE EQUAL "${CMAKE_OSX_SYSROOT}" "iphoneos" is_ios)
string(COMPARE EQUAL "${CMAKE_SYSTEM_NAME}" "Linux" is_linux)
string(COMPARE EQUAL "${CMAKE_GENERATOR}" "Xcode" is_xcode)

# See: cmake/Hunter/config.cmake
# First call to 'hunter_add_package' will contains read of Hunter/config.cmake
# Next variables will be used:
#   * is_ios
#   * is_linux
hunter_add_package(Sugar)
include("${SUGAR_ROOT}/cmake/Sugar")
include(sugar_include)

hunter_add_package(ZLIB)
find_package(ZLIB CONFIG REQUIRED)

hunter_add_package(Boost)
find_package(Boost CONFIG REQUIRED)

set(CMAKE_MODULE_PATH "${CMAKE_MODULE_PATH}" "${CMAKE_CURRENT_SOURCE_DIR}/build/cmake")

# TODO: add `git rev-parse --short HEAD`
# Read the version information from the Autoconf file
file (STRINGS "${CMAKE_CURRENT_SOURCE_DIR}/configure.ac" CONFIGURE_AC REGEX "AC_INIT\\(.*\\)" )

# The following variable is used in the version.h.in file
string(REGEX REPLACE "AC_INIT\\(\\[.*\\], \\[([0-9]+\\.[0-9]+\\.[0-9]+(-dev)?)\\]\\)" "\\1" PACKAGE_VERSION ${CONFIGURE_AC})
message(STATUS "Parsed Thrift package version: ${PACKAGE_VERSION}")

# These are internal to CMake
string(REGEX REPLACE "([0-9]+\\.[0-9]+\\.[0-9]+)(-dev)?" "\\1" thrift_VERSION ${PACKAGE_VERSION})
string(REGEX REPLACE "([0-9]+)\\.[0-9]+\\.[0-9]+" "\\1" thrift_VERSION_MAJOR ${thrift_VERSION})
string(REGEX REPLACE "[0-9]+\\.([0-9])+\\.[0-9]+" "\\1" thrift_VERSION_MINOR ${thrift_VERSION})
string(REGEX REPLACE "[0-9]+\\.[0-9]+\\.([0-9]+)" "\\1" thrift_VERSION_PATCH ${thrift_VERSION})
message(STATUS "Parsed Thrift version: ${thrift_VERSION} (${thrift_VERSION_MAJOR}.${thrift_VERSION_MINOR}.${thrift_VERSION_PATCH})")

# Some default settings
include(DefineCMakeDefaults)

# Build time options are defined here
include(DefineOptions)
include(DefineInstallationPaths)

# Based on the options set some platform specifics
include(DefinePlatformSpecifc)

# Generate the config.h file
include(ConfigureChecks)

# Package it
include(CPackConfig)


find_package(Threads)

include(CTest)
if(BUILD_TESTING)
  message(STATUS "Building with unittests")

  enable_testing()
  # Define "make check" as alias for "make test"
  add_custom_target(check COMMAND ctest)
else ()
  message(STATUS "Building without tests")
endif ()

if(BUILD_COMPILER)
    if(NOT EXISTS ${THRIFT_COMPILER})
        set(THRIFT_COMPILER $<TARGET_FILE:thrift-compiler>)
    endif()
    add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/compiler/cpp)
endif()

if(BUILD_CPP)
    add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/lib/cpp)
    if(BUILD_TUTORIALS)
        add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/tutorial/cpp)
    endif()
    if(BUILD_TESTING)
        if(WITH_LIBEVENT AND WITH_ZLIB AND WITH_OPENSSL)
            add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/test/cpp)
        else()
            message(WARNING "libevent and/or ZLIB and/or OpenSSL not found or disabled; will not build some tests")
        endif()
    endif()
endif()

if(BUILD_C_GLIB)
    add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/lib/c_glib)
endif()

if(BUILD_JAVA)
    add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/lib/java)
endif()

if(BUILD_PYTHON)
    add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/lib/py)
    if(BUILD_TESTING)
        add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/test/py)
    endif()
endif()

if(BUILD_HASKELL)
    add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/lib/hs)
    if(BUILD_TESTING)
        add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/test/hs)
    endif()
endif()

PRINT_CONFIG_SUMMARY()
