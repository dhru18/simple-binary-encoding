#
# Copyright 2015 - 2016 Real Logic Ltd.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

cmake_minimum_required(VERSION 3.0.2 FATAL_ERROR)
cmake_policy(VERSION 3.0.2)

file(STRINGS version.txt SBE_VERSION_TXT LIMIT_COUNT 1 REGEX "^[0-9]+(\\.[0-9]+)+")
string(REGEX REPLACE "^([0-9]+(\\.[0-9]+)+).*$" "\\1" SBE_VERSION_NUMERIC "${SBE_VERSION_TXT}")

project("sbe" VERSION "${SBE_VERSION_NUMERIC}")

include(ExternalProject)
include(CheckLibraryExists)

enable_testing()

# default built type is Release
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Choose the type of build" FORCE)
endif(NOT CMAKE_BUILD_TYPE)

ExternalProject_Add(
	gtest
	URL ${CMAKE_CURRENT_SOURCE_DIR}/sbe-tool/src/test/lib/gtest-1.7.0.zip
	URL_MD5 2d6ec8ccdf5c46b05ba54a9fd1d130d7
	INSTALL_COMMAND ""
	LOG_CONFIGURE ON
	LOG_BUILD ON
)

ExternalProject_Get_Property(gtest source_dir)
set(GTEST_SOURCE_DIR ${source_dir})
ExternalProject_Get_Property(gtest binary_dir)
set(GTEST_BINARY_DIR ${binary_dir})

find_package(Threads)

if(UNIX)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -fexceptions -g -Wall -Wpedantic")
    set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -O0")
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -Ofast")
endif()

if(APPLE)
	# -Wall doesn't enable everything we want to see
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wsign-compare")
	add_definitions(-DDarwin)
elseif(MSVC)
	add_definitions(-DWIN32)

    set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /MTd /Od /Zi")
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MT")
else()
	add_definitions(-D_GNU_SOURCE)
endif()

set(LIBPREFIX "${CMAKE_STATIC_LIBRARY_PREFIX}")
set(LIBSUFFIX "${CMAKE_STATIC_LIBRARY_SUFFIX}")

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/binaries")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/lib")

set(SBE_JAR "${CMAKE_CURRENT_SOURCE_DIR}/sbe-all/build/libs/sbe-all-${SBE_VERSION_TXT}.jar")
if(EXISTS "${SBE_JAR}")
	message("latest SBE jar found: ${SBE_JAR}")
else()
	message(FATAL_ERROR "could not find latest SBE jar in ${CMAKE_CURRENT_SOURCE_DIR}/sbe-all/build/libs")
endif()

set(CODEC_TARGET_DIR "${CMAKE_CURRENT_BINARY_DIR}/generated")
set(CODEC_SCHEMA_DIR "${CMAKE_CURRENT_SOURCE_DIR}/sbe-tool/src/test/resources")
set(CODEC_PERF_SCHEMA_DIR "${CMAKE_CURRENT_SOURCE_DIR}/sbe-benchmarks/src/main/resources")
set(CODEC_EXAMPLES_SCHEMA_DIR "${CMAKE_CURRENT_SOURCE_DIR}/sbe-samples/src/main/resources")

include_directories(
	${GTEST_SOURCE_DIR}/include
	${CMAKE_CURRENT_SOURCE_DIR}/sbe-tool/src/main/cpp
	${CODEC_TARGET_DIR}
)

add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/sbe-tool/src/main/cpp)
add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/sbe-tool/src/test/cpp)
add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/sbe-samples/src/main/cpp)
add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/sbe-benchmarks/src/main/cpp)

add_dependencies(sbe ir_codecs)
