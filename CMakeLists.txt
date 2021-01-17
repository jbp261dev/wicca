# Copyright 2011-2012,2014,2017 Free Software Foundation, Inc.
#
# This file is part of Wicca project
#
########################################################################
if(${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})
    message(FATAL_ERROR "Prevented in-tree build. This is bad practice. ${CMAKE_SOURCE_DIR}")
endif(${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})

########################################################################
# Project setup
########################################################################
# Make sure this version matches ${CMAKE_MIN_VERSION} (a variable can't be
# used here).
cmake_minimum_required(VERSION 3.8)
project(wicca CXX C)
enable_testing()
option(BUILD_SHARED_LIBS "Build shared libraries" ON)

# Make sure our local CMake Modules path comes first
list(INSERT CMAKE_MODULE_PATH 0 ${CMAKE_SOURCE_DIR}/cmake/Modules)

# Select the release build type by default to get optimization flags
if(NOT CMAKE_BUILD_TYPE)
   SET(CMAKE_BUILD_TYPE "Release")
   message(STATUS "Build type not specified: defaulting to release.")
endif(NOT CMAKE_BUILD_TYPE)
SET(CMAKE_BUILD_TYPE ${CMAKE_BUILD_TYPE} CACHE STRING "")
message(STATUS "Build type set to ${CMAKE_BUILD_TYPE}.")

# Set the version information here
SET(VERSION_MAJOR 0)
SET(VERSION_API   0)
SET(VERSION_ABI   0)
SET(VERSION_PATCH 0)
#eventually, replace version.sh and fill in the variables below
set(MAJOR_VERSION ${VERSION_MAJOR})
set(API_COMPAT    ${VERSION_API})
set(MINOR_VERSION ${VERSION_ABI})
set(MAINT_VERSION ${VERSION_PATCH})
set(VERSION "${MAJOR_VERSION}.${API_COMPAT}.${MINOR_VERSION}.${MAINT_VERSION}")
set(LIBVER "${MAJOR_VERSION}.${API_COMPAT}.${MINOR_VERSION}")
########################################################################
# Extract the version string from git describe.
########################################################################
find_package(Git)

# Minimum dependency versions for central dependencies:
set(BOOST_MIN_VERSION "1.53")
#set(SWIG_MIN_VERSION "3.0.8")
set(CMAKE_MIN_VERSION "3.5.1")
#set(PYTHON_MIN_VERSION "2.7.6")
#set(PYTHON3_MIN_VERSION "3.6.5")
set(GCC_MIN_VERSION "4.8.4")
set(CLANG_MIN_VERSION "3.4.0")
set(APPLECLANG_MIN_VERSION "500")
set(MSVC_MIN_VERSION "1800")

# Enable generation of compile_commands.json for code completion engines
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

########################################################################
# Configure CMake policies
########################################################################
# This will suppress developer warnings during the cmake process that can occur
# if a newer cmake version than the minimum is used.
if(POLICY CMP0026)
    cmake_policy(SET CMP0026 NEW)
endif()
if(POLICY CMP0043)
    cmake_policy(SET CMP0043 NEW)
endif()
if(POLICY CMP0045)
    cmake_policy(SET CMP0045 NEW)
endif()
if(POLICY CMP0046)
    cmake_policy(SET CMP0046 NEW)
endif()

########################################################################
# Compiler version setup
########################################################################
# Append -O2 optimization flag for Debug builds (Not on MSVC since conflicts with RTC1 flag)
IF (NOT MSVC)
    SET(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -O2")
    SET(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -O2")
ENDIF()

# Check compiler version against our minimum
IF(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    IF(DEFINED CMAKE_CXX_COMPILER_VERSION)
        IF(${CMAKE_CXX_COMPILER_VERSION} VERSION_LESS ${GCC_MIN_VERSION})
            MESSAGE(WARNING "\nThe compiler selected to build Wicca (GCC version ${CMAKE_CXX_COMPILER_VERSION} : ${CMAKE_CXX_COMPILER}) is older than that officially supported (${GCC_MIN_VERSION} minimum). This build may or not work. We highly recommend using a more recent GCC version.")
        ENDIF()
    ELSE()
        MESSAGE(WARNING "\nCannot determine the version of the compiler selected to build Wicca (GCC : ${CMAKE_CXX_COMPILER}). This build may or not work. We highly recommend using GCC version ${GCC_MIN_VERSION} or more recent.")
    ENDIF()
ELSEIF(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    EXECUTE_PROCESS(COMMAND
        ${CMAKE_CXX_COMPILER} -v
        RESULT_VARIABLE _res ERROR_VARIABLE _err
        ERROR_STRIP_TRAILING_WHITESPACE)
    IF(${_res} STREQUAL "0")
        # output is in error stream
        STRING(REGEX MATCH "^Apple.*" IS_APPLE ${_err})
        IF("${IS_APPLE}" STREQUAL "")
            SET(MIN_VERSION ${CLANG_MIN_VERSION})
            SET(APPLE_STR "")
            # retrieve the compiler's version from it
            STRING(REGEX MATCH "clang version [0-9.]+" CLANG_OTHER_VERSION ${_err})
            STRING(REGEX MATCH "[0-9.]+" CLANG_VERSION ${CLANG_OTHER_VERSION})
        ELSE()
            SET(MIN_VERSION ${APPLECLANG_MIN_VERSION})
            SET(APPLE_STR "Apple ")
            # retrieve the compiler's version from it
            STRING(REGEX MATCH "(clang-[0-9.]+)" CLANG_APPLE_VERSION ${_err})
            STRING(REGEX MATCH "[0-9.]+" CLANG_VERSION ${CLANG_APPLE_VERSION})
        ENDIF()
        IF(${CLANG_VERSION} VERSION_LESS "${MIN_VERSION}")
            MESSAGE(WARNING "\nThe compiler selected to build Wicca (${APPLE_STR}Clang version ${CLANG_VERSION} : ${CMAKE_CXX_COMPILER}) is older than that officially supported (${MIN_VERSION} minimum). This build may or not work. We highly recommend using Apple Clang version ${APPLECLANG_MIN_VERSION} or more recent, or Clang version ${CLANG_MIN_VERSION} or more recent.")
        ENDIF()
    ELSE()
        MESSAGE(WARNING "\nCannot determine the version of the compiler selected to build Wicca (${APPLE_STR}Clang : ${CMAKE_CXX_COMPILER}). This build may or not work. We highly recommend using Apple Clang version ${APPLECLANG_MIN_VERSION} or more recent, or Clang version ${CLANG_MIN_VERSION} or more recent.")
    ENDIF()
ELSE()
    MESSAGE(status "Skipping compiler version check.")
ENDIF()

# Configure C++ standard if not externally specified (will actually be
# set after CppUnit check below). Use the variable CMAKE_CXX_STANDARD
# since it will actually be used for this purposes starting in CMake 3.1.

SET(CMAKE_C_EXTENSIONS OFF)
SET(CMAKE_CXX_EXTENSIONS OFF)

IF(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    SET(CMAKE_CXX_STANDARD 11)
ELSEIF(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    SET(CMAKE_CXX_STANDARD 11)
ELSEIF(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    SET(CMAKE_CXX_STANDARD 11)
ELSE()
    message(WARNING "C++ standard could not be set because compiler is not GNU, Clang or MSVC.")
ENDIF()

IF(CMAKE_C_COMPILER_ID STREQUAL "GNU")
    SET(CMAKE_C_STANDARD 11)
ELSEIF(CMAKE_C_COMPILER_ID MATCHES "Clang")
    SET(CMAKE_C_STANDARD 11)
ELSEIF(CMAKE_C_COMPILER_ID STREQUAL "MSVC")
    SET(CMAKE_C_STANDARD 11)
ELSE()
    message(WARNING "C standard could not be set because compiler is not GNU, Clang or MSVC.")
ENDIF()

#######################################################################
# Setup Boost and handle some system specific things
########################################################################

set(BOOST_REQUIRED_COMPONENTS
    date_time
    program_options
    filesystem
    system
    regex
    thread
    atomic
    chrono
    log
    log_setup
)

if(ENABLE_TESTING)
    list(APPEND BOOST_REQUIRED_COMPONENTS unit_test_framework)
endif(ENABLE_TESTING)

if(UNIX AND NOT BOOST_ROOT AND EXISTS "/usr/lib64")
    list(APPEND BOOST_LIBRARYDIR "/usr/lib64") #fedora 64-bit fix
endif(UNIX AND NOT BOOST_ROOT AND EXISTS "/usr/lib64")

if(MSVC)
    if (NOT DEFINED BOOST_ALL_DYN_LINK)
        set(BOOST_ALL_DYN_LINK TRUE)
    endif()
    set(BOOST_ALL_DYN_LINK "${BOOST_ALL_DYN_LINK}" CACHE BOOL "boost enable dynamic linking")
    if(BOOST_ALL_DYN_LINK)
        add_definitions(-DBOOST_ALL_DYN_LINK) #setup boost auto-linking in msvc
    else(BOOST_ALL_DYN_LINK)
        unset(BOOST_REQUIRED_COMPONENTS) #empty components list for static link
    endif(BOOST_ALL_DYN_LINK)
endif(MSVC)

# This does not allow us to disable specific versions. It is used
# internally by cmake to know the formation newer versions. As newer
# Boost version beyond what is shown here are produced, we must extend
# this list. To disable Boost versions, see below.

set(Boost_ADDITIONAL_VERSIONS
    "1.53" "1.54.0" "1.54"
    "1.55.0" "1.55" "1.56.0" "1.56" "1.57.0" "1.57" "1.58.0" "1.58" "1.59.0" "1.59"
    "1.60.0" "1.60" "1.61.0" "1.61" "1.62.0" "1.62" "1.63.0" "1.63" "1.64.0" "1.64"
    "1.65.0" "1.65" "1.66.0" "1.66" "1.67.0" "1.67" "1.68.0" "1.68" "1.69.0" "1.69"
    "1.71.0" "1.71" "1.72.0" "1.72" "1.73.0" "1.73" "1.74.0" "1.74" "1.75.0" "1.75"
)

find_package(Boost ${BOOST_MIN_VERSION} COMPONENTS ${BOOST_REQUIRED_COMPONENTS})

# Boost 1.52 disabled, see https://svn.boost.org/trac/boost/ticket/7669
# Similar problems with Boost 1.46 and 1.47.

OPTION(ENABLE_BAD_BOOST "Enable known bad versions of Boost" OFF)
if(ENABLE_BAD_BOOST)
  MESSAGE(STATUS "Enabling use of known bad versions of Boost.")
endif(ENABLE_BAD_BOOST)

# For any unsuitable Boost version, add the version number below in
# the following format: XXYYZZ
# Where:
#     XX is the major version ('10' for version 1)
#     YY is the minor version number ('46' for 1.46)
#     ZZ is the patcher version number (typically just '00')
set(Boost_NOGO_VERSIONS
  104600 104601 104700 105200
  )

foreach(ver ${Boost_NOGO_VERSIONS})
  if("${Boost_VERSION}" STREQUAL "${ver}")
    if(NOT ENABLE_BAD_BOOST)
      MESSAGE(STATUS "WARNING: Found a known bad version of Boost (v${Boost_VERSION}). Disabling.")
      set(Boost_FOUND FALSE)
    else(NOT ENABLE_BAD_BOOST)
      MESSAGE(STATUS "WARNING: Found a known bad version of Boost (v${Boost_VERSION}). Continuing anyway.")
      set(Boost_FOUND TRUE)
    endif(NOT ENABLE_BAD_BOOST)
  endif("${Boost_VERSION}" STREQUAL "${ver}")
endforeach(ver)

########################################################################
# Compiler specific setup
########################################################################

include(TestBigEndian)
TEST_BIG_ENDIAN(IS_BIG_ENDIAN)
if(IS_BIG_ENDIAN)
    add_definitions(-DIS_BIG_ENDIAN)
endif(IS_BIG_ENDIAN)

if(MSVC)
    include_directories(${CMAKE_SOURCE_DIR}/cmake/msvc) #missing headers
    add_definitions(-D_USE_MATH_DEFINES) #enables math constants on all supported versions of MSVC
    add_definitions(-D_WIN32_WINNT=0x0502) #Minimum version: "Windows Server 2003 with SP1, Windows XP with SP2"
    add_definitions(-DNOMINMAX) #disables stupidity and enables std::min and std::max
    add_definitions( #stop all kinds of compatibility warnings
        -D_SCL_SECURE_NO_WARNINGS
        -D_CRT_SECURE_NO_WARNINGS
        -D_CRT_SECURE_NO_DEPRECATE
        -D_CRT_NONSTDC_NO_DEPRECATE
    )
    add_definitions(-DHAVE_CONFIG_H)
    add_compile_options(/MP) #build with multiple processors
    add_compile_options(/bigobj) #allow for larger object files
endif(MSVC)

if(WIN32)
    add_definitions(-D_USE_MATH_DEFINES)
    if (CMAKE_SIZEOF_VOID_P EQUAL 8)
        add_definitions(-DMS_WIN64)
    endif(CMAKE_SIZEOF_VOID_P EQUAL 8)
endif(WIN32)

# Record Compiler Info for record
STRING(TOUPPER ${CMAKE_BUILD_TYPE} CBTU)
SET(COMPILER_INFO "")
IF(MSVC)
    IF(MSVC_VERSION LESS "${MSVC_MIN_VERSION}")
        MESSAGE(FATAL_ERROR "MSVC Versions < minimum version ${MSVC_MIN_VERSION}")
    ENDIF()
    SET(cmake_c_compiler_version ${MSVC_VERSION})
    SET(cmake_cxx_compiler_version ${MSVC_VERSION})
ELSE()
    execute_process(COMMAND ${CMAKE_C_COMPILER} --version
            OUTPUT_VARIABLE cmake_c_compiler_version)
    execute_process(COMMAND ${CMAKE_CXX_COMPILER} --version
            OUTPUT_VARIABLE cmake_cxx_compiler_version)
ENDIF(MSVC)

# Convert to a C string to compile and display properly
string(STRIP "${cmake_c_compiler_version}" cmake_c_compiler_version)
string(STRIP "${cmake_cxx_compiler_version}" cmake_cxx_compiler_version)
MESSAGE(STATUS "Compiler Version: ${cmake_c_compiler_version}")
MESSAGE(STATUS "Compiler Flags: ${COMPILER_INFO}")
string(REPLACE "\n" " \\n" cmake_c_compiler_version ${cmake_c_compiler_version})
string(REPLACE "\n" " \\n" cmake_cxx_compiler_version ${cmake_cxx_compiler_version})


########################################################################
# Add subdirectories (in order of deps)
########################################################################
add_subdirectory(src)

########################################################################
# Print summary
########################################################################
message(STATUS "Building for version: ${VERSION} / ${LIBVER}")