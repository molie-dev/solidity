cmake_minimum_required(VERSION 3.13.0)

set(ETH_CMAKE_DIR   "${CMAKE_CURRENT_LIST_DIR}/cmake"   CACHE PATH "The path to the cmake directory")
list(APPEND CMAKE_MODULE_PATH ${ETH_CMAKE_DIR})

# Set the build type, if none was specified.
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
	if(EXISTS "${PROJECT_SOURCE_DIR}/.git")
		set(DEFAULT_BUILD_TYPE "RelWithDebInfo")
	else()
		set(DEFAULT_BUILD_TYPE "Release")
	endif()
	set(CMAKE_BUILD_TYPE "${DEFAULT_BUILD_TYPE}" CACHE STRING "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel" FORCE)
	set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "RelWithDebInfo" "MinSizeRel")
endif()

include(EthToolchains)

# Set cmake_policies
include(EthPolicy)
eth_policy()

# project name and version should be set after cmake_policy CMP0048
set(PROJECT_VERSION "0.8.29")
# OSX target needed in order to support std::visit
set(CMAKE_OSX_DEPLOYMENT_TARGET "10.14")
project(solidity VERSION ${PROJECT_VERSION} LANGUAGES C CXX)

include(TestBigEndian)
TEST_BIG_ENDIAN(IS_BIG_ENDIAN)
if (IS_BIG_ENDIAN)
	message(FATAL_ERROR "${PROJECT_NAME} currently does not support big endian systems.")
endif()

option(SOLC_LINK_STATIC "Link solc executable statically on supported platforms" OFF)
option(SOLC_STATIC_STDLIBS "Link solc against static versions of libgcc and libstdc++ on supported platforms" OFF)
option(STRICT_Z3_VERSION "Require the exact version of Z3 solver expected by our test suite." ON)
option(PEDANTIC "Enable extra warnings and pedantic build flags. Treat all warnings as errors." ON)
option(PROFILE_OPTIMIZER_STEPS "Output performance metrics for the optimiser steps." OFF)
option(
	IGNORE_VENDORED_DEPENDENCIES
	"Ignore libraries provided as submodules of the repository and allow CMake to look for \
them in the typical locations, including system-wide dirs."
	OFF
)
option(
	ONLY_BUILD_SOLIDITY_LIBRARIES
	"Only build library targets that can be statically linked against. Do not build executables or tests."
	OFF
)
mark_as_advanced(PROFILE_OPTIMIZER_STEPS)
mark_as_advanced(IGNORE_VENDORED_DEPENDENCIES)
mark_as_advanced(ONLY_BUILD_SOLIDITY_LIBRARIES)

# Setup cccache.
include(EthCcache)

# Let's find our dependencies
include(EthDependencies)
if (NOT IGNORE_VENDORED_DEPENDENCIES)
  include(fmtlib)
  include(nlohmann-json)
  include(range-v3)
endif()

find_package(Threads)

if(NOT PEDANTIC)
  message(WARNING "-- Pedantic build flags turned off. Warnings will not make compilation fail. This is NOT recommended in development builds.")
endif()

if (PROFILE_OPTIMIZER_STEPS)
    add_definitions(-DPROFILE_OPTIMIZER_STEPS)
endif()

# Figure out what compiler and system are we using
include(EthCompilerSettings)

# Include utils
include(EthUtils)

# Create license.h from LICENSE.txt and template
# Converting to char array is required due to MSVC's string size limit.
file(READ ${PROJECT_SOURCE_DIR}/LICENSE.txt LICENSE_TEXT HEX)
string(REGEX MATCHALL ".." LICENSE_TEXT "${LICENSE_TEXT}")
string(REGEX REPLACE ";" ",\n\t0x" LICENSE_TEXT "${LICENSE_TEXT}")
set(LICENSE_TEXT "0x${LICENSE_TEXT}")

configure_file("${PROJECT_SOURCE_DIR}/cmake/templates/license.h.in" include/license.h)

include(EthOptions)
configure_project(TESTS)

if(EMSCRIPTEN)
    set(TESTED_Z3_VERSION "4.13.3")
    set(MINIMUM_Z3_VERSION "4.8.16")
    find_package(Z3)
    if (${Z3_FOUND})
      add_definitions(-DEMSCRIPTEN_BUILD)
      if (${STRICT_Z3_VERSION})
        if (NOT ("${Z3_VERSION_STRING}" VERSION_EQUAL ${TESTED_Z3_VERSION}))
          message(
            FATAL_ERROR
            "SMTChecker tests require Z3 ${TESTED_Z3_VERSION} for all tests to pass.\n"
          )
        endif()
      else()
        if ("${Z3_VERSION_STRING}" VERSION_LESS ${MINIMUM_Z3_VERSION})
          message(
            FATAL_ERROR
            "Solidity requires Z3 ${MINIMUM_Z3_VERSION} or newer."
          )
        endif()
      endif()
    else()
        message(FATAL_ERROR "Solidity requires Z3 for emscripten build.")
    endif()
endif()

add_subdirectory(libsolutil)
add_subdirectory(liblangutil)
add_subdirectory(libsmtutil)
add_subdirectory(libevmasm)
add_subdirectory(libyul)
add_subdirectory(libsolidity)
add_subdirectory(libsolc)
add_subdirectory(libstdlib)

if (NOT ONLY_BUILD_SOLIDITY_LIBRARIES)
	add_subdirectory(tools)

	if (NOT EMSCRIPTEN)
		add_subdirectory(solc)
	endif()

	if (TESTS AND NOT EMSCRIPTEN)
		add_subdirectory(test)
	endif()
endif()
