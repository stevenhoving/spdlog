#
# Copyright(c) 2015 Ruslan Baratov.
# Distributed under the MIT License (http://opensource.org/licenses/MIT)
#

cmake_minimum_required(VERSION 3.1)
project(spdlog VERSION 1.3.0 LANGUAGES CXX)
include(CTest)
include(CMakeDependentOption)
include(GNUInstallDirs)

#---------------------------------------------------------------------------------------
# set default build to release
#---------------------------------------------------------------------------------------
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Choose Release or Debug" FORCE)
endif()

message(STATUS "Build type: " ${CMAKE_BUILD_TYPE})

#---------------------------------------------------------------------------------------
# compiler config
#---------------------------------------------------------------------------------------
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU" OR "${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
    add_compile_options("-Wall")
    add_compile_options("-Wextra")
    add_compile_options("-Wconversion")
    add_compile_options("-pedantic")
    add_compile_options("-Wfatal-errors")
    
endif()

#---------------------------------------------------------------------------------------
# address sanitizers check
#---------------------------------------------------------------------------------------
include(cmake/sanitizers.cmake)

#---------------------------------------------------------------------------------------
# spdlog target
#---------------------------------------------------------------------------------------
add_library(spdlog INTERFACE)
add_library(spdlog::spdlog ALIAS spdlog)

# Check if spdlog is being used directly or via add_subdirectory
set(SPDLOG_MASTER_PROJECT OFF)
if (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
  set(SPDLOG_MASTER_PROJECT ON)
endif()

option(SPDLOG_BUILD_EXAMPLES "Build examples" ${SPDLOG_MASTER_PROJECT})
option(SPDLOG_BUILD_BENCH "Build benchmarks" ${SPDLOG_MASTER_PROJECT})

cmake_dependent_option(SPDLOG_BUILD_TESTING
    "Build spdlog tests" ${SPDLOG_MASTER_PROJECT}
    "BUILD_TESTING" OFF
)

#---------------------------------------------------------------------------------------
# spdlog tweakme config
#---------------------------------------------------------------------------------------
option(SPDLOG_CLOCK_COARSE "Enable under Linux, the much faster CLOCK_REALTIME_COARSE clock can be used" OFF)
option(SPDLOG_NO_DATETIME "Enable if date/time logging is not needed and never appear in the log pattern" OFF)
option(SPDLOG_NO_THREAD_ID "Enable if thread id logging is not needed (i.e. no %t in the log pattern)" OFF)
option(SPDLOG_DISABLE_TID_CACHING "Enable to prevent spdlog from caching thread ids in thread local storage" OFF)
option(SPDLOG_NO_NAME "Enable if logger name logging is not needed" OFF)
option(SPDLOG_DEBUG_ON "Enable usuage of the SPDLOG_DEBUG macro" OFF)
option(SPDLOG_TRACE_ON "Enable usuage of the SPDLOG_TRACE macro" OFF)
option(SPDLOG_NO_ATOMIC_LEVELS "Enable to avoid spdlog's usage of atomic log levels" OFF)
option(SPDLOG_WCHAR_FILENAMES "Enable usage of wchar_t for file names on Windows" OFF)
option(SPDLOG_EOL "Enable to override default eol ('\n' or '\r\n' under Linux/Windows)" OFF)
option(SPDLOG_FMT_EXTERNAL "Enable to use your own copy of the fmt library instead of spdlog's copy" OFF)
option(SPDLOG_WCHAR_TO_UTF8_SUPPORT "Enable wchar_t support (convert to utf8)" OFF)
option(SPDLOG_PREVENT_CHILD_FD "Enable to prevent child processes from inheriting log file descriptors" OFF)
option(SPDLOG_ENABLE_MESSAGE_COUNTER "Enable the message counting feature" OFF)
option(SPDLOG_LEVEL_NAMES "xxx" OFF)
option(SPDLOG_DISABLE_DEFAULT_LOGGER "Enable to disable default logger creation" OFF)

# this is not entirely correct... but will had to do for now...
configure_file(
    "cmake/tweakme.h.in"
    "${CMAKE_CURRENT_LIST_DIR}/include/spdlog/tweakme.h"
    @ONLY
)

target_include_directories(
    spdlog
    INTERFACE
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/include>"
    "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>"
)

set(HEADER_BASE "${CMAKE_CURRENT_SOURCE_DIR}/include")

if(SPDLOG_BUILD_EXAMPLES)
    add_subdirectory(example)
endif()

if(SPDLOG_BUILD_TESTING)
    add_subdirectory(tests)
endif()

if(SPDLOG_BUILD_BENCH)
    add_subdirectory(bench)
endif()

#---------------------------------------------------------------------------------------
# Install/export targets and files
#---------------------------------------------------------------------------------------
# set files and directories
set(config_install_dir "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}")
set(include_install_dir "${CMAKE_INSTALL_INCLUDEDIR}")
set(pkgconfig_install_dir "${CMAKE_INSTALL_LIBDIR}/pkgconfig")
set(version_config "${CMAKE_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake")
set(project_config "${PROJECT_NAME}Config.cmake")
set(pkg_config "${CMAKE_BINARY_DIR}/${PROJECT_NAME}.pc")
set(targets_export_name "${PROJECT_NAME}Targets")
set(namespace "${PROJECT_NAME}::")

# generate package version file
include(CMakePackageConfigHelpers)
write_basic_package_version_file(
    "${version_config}" COMPATIBILITY SameMajorVersion
)

# configure pkg config file
configure_file("cmake/spdlog.pc.in" "${pkg_config}" @ONLY)

# install targets
install(
    TARGETS spdlog
    EXPORT "${targets_export_name}"
)

# install headers
install(
    DIRECTORY "${HEADER_BASE}/${PROJECT_NAME}"
    DESTINATION "${include_install_dir}"
)

# install project version file
install(
    FILES "${version_config}"
    DESTINATION "${config_install_dir}"
)

# install pkg config file
install(
    FILES "${pkg_config}"
    DESTINATION "${pkgconfig_install_dir}"
)

# install project config file
install(
    EXPORT "${targets_export_name}"
    NAMESPACE "${namespace}"
    DESTINATION "${config_install_dir}"
    FILE ${project_config}
)

# export build directory config file
export(
    EXPORT ${targets_export_name}
    NAMESPACE "${namespace}"
    FILE ${project_config}
)

# register project in CMake user registry
export(PACKAGE ${PROJECT_NAME})

file(GLOB_RECURSE spdlog_include_SRCS "${HEADER_BASE}/*.h")
add_custom_target(spdlog_headers_for_ide SOURCES ${spdlog_include_SRCS})
