cmake_minimum_required(VERSION 3.20)

cmake_path(GET CMAKE_SOURCE_DIR FILENAME SLIM_LIB)
string(TOLOWER "${SLIM_LIB}" SLIM_LIB_LOWER)
find_package(PkgConfig REQUIRED)
#set(CMAKE_FIND_DEBUG_MODE TRUE)

include(FetchContent)

set(SLIM_USE_LOCAL_SOURCE OFF CACHE BOOL "Use local library source instead of git")

if(SLIM_USE_LOCAL_SOURCE)
    set(GIT_TAG 0.0.0)
    set(SLIMLIB_GIT_HASH "none")
else()
    execute_process(
        COMMAND git ls-remote --tags --sort=-v:refname https://github.com/greergan/${SLIM_LIB}.git
        OUTPUT_VARIABLE TAGS_RAW
        OUTPUT_STRIP_TRAILING_WHITESPACE
        ERROR_QUIET
    )

    string(REGEX MATCH "refs/tags/([^\\^\\n]+)" _ "${TAGS_RAW}")
    set(GIT_TAG ${CMAKE_MATCH_1})
endif()

if(SLIM_USE_LOCAL_SOURCE)
    set(slimlibrarysrc_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR})
    message(STATUS "using LOCAL source (${slimlibrarysrc_SOURCE_DIR})")
else()
    FetchContent_Declare(
        SlimLibrarySrc
        GIT_REPOSITORY https://github.com/greergan/${SLIM_LIB}.git
        GIT_TAG        ${GIT_TAG}
    )

    message(STATUS "using GIT source (tag: ${GIT_TAG})")
    message(STATUS "fetched from https://github.com/greergan/${SLIM_LIB}.git")

    FetchContent_Populate(SlimLibrarySrc)

    execute_process(
        COMMAND git rev-parse --short HEAD
        OUTPUT_VARIABLE SLIMLIB_GIT_HASH
        OUTPUT_STRIP_TRAILING_WHITESPACE
        ERROR_QUIET
    )
endif()

project(${SLIM_LIB} VERSION ${GIT_TAG} LANGUAGES CXX)
set(SLIMLIB_CXX_STANDARD 23)

if(${SLIM_LIB} MATCHES "^SlimCommon")
    string(REGEX REPLACE "^SlimCommon" "" SLIM_COMMON_SUFFIX "${SLIM_LIB}")
    string(REGEX REPLACE "([A-Z])" "/\\1" SLIM_COMMON_PATH "${SLIM_COMMON_SUFFIX}")
    string(TOLOWER "${SLIM_COMMON_PATH}" SLIM_COMMON_PATH)
    string(REGEX REPLACE "^/" "" SLIM_COMMON_PATH "${SLIM_COMMON_PATH}")
    string(REGEX MATCH "[^/]+$" HEADER_FILE_PREFIX "${SLIM_COMMON_PATH}")
    string(REGEX REPLACE "/[^/]+$" "" SLIM_COMMON_PARENT "${SLIM_COMMON_PATH}")
    if(SLIM_COMMON_PARENT STREQUAL SLIM_COMMON_PATH)
        set(SLIM_INCLUDE_DIR "include/slim/common")
    else()
        set(SLIM_INCLUDE_DIR "include/slim/common/${SLIM_COMMON_PARENT}")
    endif()
else()
    string(REGEX REPLACE "^slim" "" HEADER_FILE_PREFIX "${SLIM_LIB_LOWER}")
    set(SLIM_INCLUDE_DIR "include/slim")
endif()
set(SLIM_METADATA_FILE_OUT "${SLIM_LIB_LOWER}.pc")

set(HPP_IN_FILE "${slimlibrarysrc_SOURCE_DIR}/${SLIM_INCLUDE_DIR}/${SLIM_LIB}.hpp.in")
set(H_IN_FILE "${slimlibrarysrc_SOURCE_DIR}/${SLIM_INCLUDE_DIR}/${HEADER_FILE_PREFIX}.h.in")

set(HPP_OUT_FILE "${CMAKE_CURRENT_BINARY_DIR}/${SLIM_INCLUDE_DIR}/${SLIM_LIB}.hpp")
set(H_OUT_FILE "${CMAKE_CURRENT_BINARY_DIR}/${SLIM_INCLUDE_DIR}/${HEADER_FILE_PREFIX}.h")

set(HPP_HEADER_ONLY OFF)

if(EXISTS ${HPP_IN_FILE})
    set(HPP_HEADER_ONLY ON)
    set(SLIM_LIB_DESCRIPTION "${SLIM_LIB} header-only C++ library")
    set(SLIM_METADATA_FILE_IN "slim_header_lib.pc.in")
else()
    set(SLIM_LIB_DESCRIPTION "${SLIM_LIB} C++ library")
    set(SLIM_METADATA_FILE_IN "slim_common_lib.pc.in")
endif()

if(NOT SLIM_INCLUDE_DIR)
    message(FATAL_ERROR "empty SLIM_INCLUDE_DIR")
endif()
if(NOT HEADER_FILE_PREFIX)
    message(FATAL_ERROR "empty HEADER_FILE_PREFIX")
endif()
if(NOT SLIM_METADATA_FILE_OUT)
    message(FATAL_ERROR "empty SLIM_METADATA_FILE_OUT")
endif()

set(SLIMLIB_VERSION ${GIT_TAG})
set(SLIMLIB_VERSION_FULL "${SLIMLIB_VERSION}+${SLIMLIB_GIT_HASH}")

execute_process(
	COMMAND git config --get user.name
	OUTPUT_VARIABLE GIT_USER_NAME
	OUTPUT_STRIP_TRAILING_WHITESPACE
	ERROR_QUIET
)

if(NOT GIT_USER_NAME)
	set(GIT_USER_NAME "unknown")
endif()

execute_process(
	COMMAND git config --get user.email
	OUTPUT_VARIABLE GIT_USER_EMAIL
	OUTPUT_STRIP_TRAILING_WHITESPACE
	ERROR_QUIET
)

if(NOT GIT_USER_EMAIL)
	set(GIT_USER_EMAIL "unknown")
endif()

file(STRINGS "${CMAKE_SOURCE_DIR}/required_packages" PACKAGE_LINES
     REGEX "^[^#\n]")

set(FOUND_PACKAGES "")

foreach(line IN LISTS PACKAGE_LINES)
    string(REGEX MATCHALL "[^ \t]+" parts "${line}")

    list(GET parts 0 PKG_NAME)
    list(LENGTH parts part_count)

    if(part_count GREATER_EQUAL 3)
        list(GET parts 1 PKG_MIN)
        list(GET parts 2 PKG_MAX)
        find_package(${PKG_NAME} ${PKG_MIN}...${PKG_MAX} REQUIRED)

    elseif(part_count EQUAL 2)
        list(GET parts 1 PKG_MIN)
        find_package(${PKG_NAME} ${PKG_MIN} REQUIRED)

    else()
        find_package(${PKG_NAME} REQUIRED)
    endif()

    get_cmake_property(_all_imported IMPORTED_TARGETS)
    message(STATUS "  [${PKG_NAME}] imported targets: ${_all_imported}")


    string(TOUPPER ${PKG_NAME} PKG_NAME_UPPER)
    list(APPEND FOUND_PACKAGES ${PKG_NAME})
message(STATUS "FOUND_PACKAGES: ${FOUND_PACKAGES}")

    set(PKG_INFO "")
    set(CANDIDATE_TARGETS
        ${PKG_NAME}::${PKG_NAME}
        ${PKG_NAME_UPPER}::${PKG_NAME_UPPER}
        ${PKG_NAME}
    )
message(STATUS "CANDIDATE_TARGETS: ${CANDIDATE_TARGETS}")
    foreach(tgt IN LISTS CANDIDATE_TARGETS)
message(STATUS "TARGET: ${tgt}")
        if(TARGET ${tgt})
message(FATAL_ERROR)
            get_target_property(_incdirs  ${tgt} INTERFACE_INCLUDE_DIRECTORIES)
            get_target_property(_copts    ${tgt} INTERFACE_COMPILE_OPTIONS)
            get_target_property(_cdefs    ${tgt} INTERFACE_COMPILE_DEFINITIONS)
            get_target_property(_libs     ${tgt} INTERFACE_LINK_LIBRARIES)
            get_target_property(_location ${tgt} LOCATION)

            if(_incdirs AND NOT _incdirs MATCHES "NOTFOUND")
                string(REPLACE ";" ", " _incdirs_str "${_incdirs}")
                string(APPEND PKG_INFO " includes=[${_incdirs_str}]")
            endif()
            if(_copts AND NOT _copts MATCHES "NOTFOUND")
                string(REPLACE ";" ", " _copts_str "${_copts}")
                string(APPEND PKG_INFO " cflags=[${_copts_str}]")
            endif()
            if(_cdefs AND NOT _cdefs MATCHES "NOTFOUND")
                string(REPLACE ";" ", " _cdefs_str "${_cdefs}")
                string(APPEND PKG_INFO " defines=[${_cdefs_str}]")
            endif()
            if(_libs AND NOT _libs MATCHES "NOTFOUND")
                string(REPLACE ";" ", " _libs_str "${_libs}")
                string(APPEND PKG_INFO " ldflags=[${_libs_str}]")
            endif()
            if(_location AND NOT _location MATCHES "NOTFOUND")
                string(APPEND PKG_INFO " location=[${_location}]")
            endif()
            break()
        endif()
    endforeach()

    message(STATUS "package: ${PKG_NAME} ${PKG_INFO}")
    # Store per-package info in a named variable to avoid semicolon splitting
    set(PKG_INFO_${PKG_NAME} "${PKG_INFO}")
endforeach()
message(STATUS "OS: ${CMAKE_SYSTEM_NAME} ${CMAKE_SYSTEM_VERSION}")
message(STATUS "Processor: ${CMAKE_SYSTEM_PROCESSOR}")
message(STATUS "Project dir: ${CMAKE_SOURCE_DIR}")
message(STATUS "Build dir:   ${CMAKE_BINARY_DIR}")
message(STATUS "Dist dir: ${CPACK_OUTPUT_FILE_PREFIX}")
message(STATUS "Local INSTALL_PREFIX: ${CMAKE_INSTALL_PREFIX}")
list(LENGTH FOUND_PACKAGES number_of_packages)
message(STATUS "Found packages: ${number_of_packages}")
foreach(pkg IN LISTS FOUND_PACKAGES)
    message(STATUS "Found package:  ${pkg}:${PKG_INFO_${pkg}}")
endforeach()
message(STATUS "slim library name: ${SLIM_LIB}")
message(STATUS "slim include path: ${SLIM_INCLUDE_DIR}")
if(HPP_HEADER_ONLY)
    message(STATUS "install .hpp only: ${HPP_HEADER_ONLY}")
    message(STATUS ".hpp.in file: ${HPP_IN_FILE}")
    message(STATUS ".hpp file: ${HPP_OUT_FILE}")
endif()
# Compiler flags
set(SLIM_CXX_FLAGS "")

if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang|AppleClang")
    list(APPEND SLIM_CXX_FLAGS
        -std=c++${SLIMLIB_CXX_STANDARD}
        -Wall
        -Wextra
        -Wpedantic
        -Wshadow
        -Wconversion
        -Wsign-conversion
    )
    if(CMAKE_BUILD_TYPE STREQUAL "RELEASE")
        list(APPEND SLIM_CXX_FLAGS -O2 -DNDEBUG)
    elseif(CMAKE_BUILD_TYPE STREQUAL "COMPACT")
        list(APPEND SLIM_CXX_FLAGS -O3)
    elseif(CMAKE_BUILD_TYPE STREQUAL "DEBUG")
        list(APPEND SLIM_CXX_FLAGS -g -O0)
    endif()
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    list(APPEND SLIM_CXX_FLAGS
        /std:c++${SLIMLIB_CXX_STANDARD}
        /W4
        /WX-
        /permissive-
    )
    if(CMAKE_BUILD_TYPE STREQUAL "RELEASE")
        list(APPEND SLIM_CXX_FLAGS /O2 /DNDEBUG)
    elseif(CMAKE_BUILD_TYPE STREQUAL "COMPACT")
        list(APPEND SLIM_CXX_FLAGS /O1 /DNDEBUG)
    elseif(CMAKE_BUILD_TYPE STREQUAL "DEBUG")
        list(APPEND SLIM_CXX_FLAGS /Od /Zi)
    endif()
else()
    message(WARNING "Unknown compiler '${CMAKE_CXX_COMPILER_ID}' — no flags set")
endif()

list(JOIN SLIM_CXX_FLAGS " " SLIM_CXX_FLAGS_STR)
message(STATUS "Compiler:      ${CMAKE_CXX_COMPILER_ID} ${CMAKE_CXX_COMPILER_VERSION}")
message(STATUS "CXX standard:  C++${SLIMLIB_CXX_STANDARD}")
message(STATUS "Build type:    ${CMAKE_BUILD_TYPE}")
message(STATUS "CXX flags:     ${SLIM_CXX_FLAGS_STR}")
message(STATUS "version tag: ${GIT_TAG}")
message(STATUS "version hash: ${SLIMLIB_GIT_HASH}")
message(STATUS "full version: ${SLIMLIB_VERSION_FULL}")
message(STATUS "Package maintainer: ${GIT_USER_NAME} <${GIT_USER_EMAIL}>")

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

if(HPP_HEADER_ONLY)
    configure_file("${HPP_IN_FILE}" "${HPP_OUT_FILE}" @ONLY)
else()
    configure_file("${H_IN_FILE}" "${H_OUT_FILE}" @ONLY)
endif()

if(NOT HPP_HEADER_ONLY)

add_library(${SLIM_LIB_LOWER}_shared SHARED
    ${slimlibrarysrc_SOURCE_DIR}/src/main.cpp
)

# Fix #1: SOVERSION should be major version only, not full semver
string(REGEX MATCH "^([0-9]+)" SLIM_VERSION_MAJOR "${GIT_TAG}")

set_target_properties(${SLIM_LIB_LOWER}_shared PROPERTIES
    OUTPUT_NAME ${SLIM_LIB_LOWER}
    VERSION     ${GIT_TAG}
    SOVERSION   ${SLIM_VERSION_MAJOR}
)

add_library(${SLIM_LIB_LOWER}_static STATIC
    ${slimlibrarysrc_SOURCE_DIR}/src/main.cpp
)
set_target_properties(${SLIM_LIB_LOWER}_static PROPERTIES OUTPUT_NAME ${SLIM_LIB_LOWER})

foreach(t ${SLIM_LIB_LOWER}_shared ${SLIM_LIB_LOWER}_static)
    target_include_directories(${t}
        PUBLIC
            $<BUILD_INTERFACE:${slimlibrarysrc_SOURCE_DIR}/include>
            $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>
            $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
    )
    target_compile_options(${t} PUBLIC ${SLIM_CXX_FLAGS})
    # Keep this so consumers inherit the standard requirement via CMake's feature system
    target_compile_features(${t} PUBLIC cxx_std_${SLIMLIB_CXX_STANDARD})
endforeach()

add_library(${SLIM_LIB_LOWER} ALIAS ${SLIM_LIB_LOWER}_shared)

add_custom_command(TARGET ${SLIM_LIB_LOWER}_shared POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E echo "Shared lib: $<TARGET_FILE_NAME:${SLIM_LIB_LOWER}_shared>"
)

add_custom_command(TARGET ${SLIM_LIB_LOWER}_static POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E echo "Static lib: $<TARGET_FILE_NAME:${SLIM_LIB_LOWER}_static>"
)

install(TARGETS ${SLIM_LIB_LOWER}_shared ${SLIM_LIB_LOWER}_static
    EXPORT ${SLIM_LIB}Targets
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

endif() # NOT HPP_HEADER_ONLY

# Branch 1: src/test.cpp — always compiled when present
set(TEST_CPP "${CMAKE_CURRENT_SOURCE_DIR}/src/test.cpp")

if(EXISTS "${TEST_CPP}")
    add_executable(slim_test "${TEST_CPP}")
    target_compile_options(slim_test PRIVATE ${SLIM_CXX_FLAGS})
    if(HPP_HEADER_ONLY)
        target_include_directories(slim_test
            PRIVATE
                $<BUILD_INTERFACE:${slimlibrarysrc_SOURCE_DIR}/include>
                $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>
        )
    else()
        target_link_libraries(slim_test PRIVATE ${SLIM_LIB_LOWER}_shared)
    endif()
    message(STATUS "Testing: with src/test.cpp")
    add_custom_command(TARGET slim_test POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E echo "Running slim_test..."
        COMMAND $<TARGET_FILE:slim_test>
        COMMENT "Running slim_test"
    )
else()
    message(STATUS "Testing: src/test.cpp not found, skipping")
endif()

# Branch 2: tests/ directory — Catch2, always checked
if(EXISTS "${slimlibrarysrc_SOURCE_DIR}/tests")
    message(STATUS "Catch2 Testing: enabled")

    file(GLOB_RECURSE SLIM_TEST_SOURCES "${slimlibrarysrc_SOURCE_DIR}/tests/*.cpp")

    FetchContent_Declare(
        Catch2
        GIT_REPOSITORY https://github.com/catchorg/Catch2.git
        GIT_TAG v3.5.0
    )

    FetchContent_MakeAvailable(Catch2)

    enable_testing()

    add_executable(slim_tests ${SLIM_TEST_SOURCES})
    target_compile_options(slim_tests PRIVATE ${SLIM_CXX_FLAGS})
    if(HPP_HEADER_ONLY)
        target_include_directories(slim_tests
            PRIVATE
                $<BUILD_INTERFACE:${slimlibrarysrc_SOURCE_DIR}/include>
                $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>
        )
    else()
        target_link_libraries(slim_tests PRIVATE ${SLIM_LIB_LOWER})
    endif()

    target_link_libraries(slim_tests PRIVATE Catch2::Catch2WithMain)

    include(CTest)
    include(Catch)
    catch_discover_tests(slim_tests)

    add_custom_command(TARGET slim_tests POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E echo "Running slim_tests..."
        COMMAND $<TARGET_FILE:slim_tests>
        COMMENT "Running slim_tests"
    )
else()
    message(STATUS "Catch2 Testing: disabled (no tests directory found)")
endif()

string(REGEX REPLACE "^include/" "" SLIM_INSTALL_DIR "${SLIM_INCLUDE_DIR}")

if(HPP_HEADER_ONLY)
    install(FILES "${HPP_OUT_FILE}" DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${SLIM_INSTALL_DIR})
else()
    install(FILES "${H_OUT_FILE}" DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${SLIM_INSTALL_DIR})
endif()

if(NOT HPP_HEADER_ONLY)
    install(EXPORT ${SLIM_LIB}Targets
        FILE ${SLIM_LIB}Targets.cmake
        NAMESPACE Slim::
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${SLIM_LIB}
    )

    write_basic_package_version_file(
        "${CMAKE_CURRENT_BINARY_DIR}/${SLIM_LIB}ConfigVersion.cmake"
        VERSION ${GIT_TAG}
        COMPATIBILITY AnyNewerVersion
    )

    file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/${SLIM_LIB}Config.cmake"
"include(\"\${CMAKE_CURRENT_LIST_DIR}/${SLIM_LIB}Targets.cmake\")\n")

    install(FILES
        "${CMAKE_CURRENT_BINARY_DIR}/${SLIM_LIB}Config.cmake"
        "${CMAKE_CURRENT_BINARY_DIR}/${SLIM_LIB}ConfigVersion.cmake"
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${SLIM_LIB}
    )
endif()

configure_file(
    "${slimlibrarysrc_SOURCE_DIR}/${SLIM_METADATA_FILE_IN}"
    "${CMAKE_CURRENT_BINARY_DIR}/${SLIM_METADATA_FILE_OUT}"
    @ONLY
)

install(FILES
    "${CMAKE_CURRENT_BINARY_DIR}/${SLIM_METADATA_FILE_OUT}"
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
)

string(TOLOWER "${CMAKE_SYSTEM_PROCESSOR}" ARCH)
if(ARCH MATCHES "x86_64|amd64")
    set(ARCH_NAME "amd64")
elseif(ARCH MATCHES "i386|i686")
    set(ARCH_NAME "x86")
elseif(ARCH MATCHES "aarch64|arm64")
    set(ARCH_NAME "arm64")
elseif(ARCH MATCHES "armv7|arm")
    set(ARCH_NAME "arm")
else()
    set(ARCH_NAME "unknown")
endif()

set(CPACK_GENERATOR "DEB;RPM")

set(CPACK_PACKAGE_NAME ${SLIM_LIB_LOWER})
set(CPACK_PACKAGE_VERSION ${GIT_TAG})
set(CPACK_PACKAGE_CONTACT "${GIT_USER_NAME} <${GIT_USER_EMAIL}>")
set(CPACK_PACKAGE_FILE_NAME "${PROJECT_NAME}-${PROJECT_VERSION}-${ARCH_NAME}")

set(CPACK_DEBIAN_PACKAGE_MAINTAINER ${GIT_USER_NAME})
set(CPACK_DEBIAN_PACKAGE_SECTION "devel")
set(CPACK_DEBIAN_PACKAGE_PRIORITY "optional")
set(CPACK_DEBIAN_PACKAGE_ARCHITECTURE ${ARCH_NAME})
set(CPACK_PACKAGING_INSTALL_PREFIX "/usr")

set(CPACK_RPM_PACKAGE_NAME ${SLIM_LIB_LOWER})
set(CPACK_RPM_PACKAGE_VERSION ${GIT_TAG})
set(CPACK_RPM_PACKAGE_RELEASE "1")
set(CPACK_RPM_PACKAGE_LICENSE "MIT")
set(CPACK_RPM_PACKAGE_GROUP "Development/Libraries")
set(CPACK_RPM_PACKAGE_SUMMARY ${SLIM_LIB_DESCRIPTION})
set(CPACK_RPM_PACKAGE_ARCHITECTURE ${ARCH_NAME})
set(CPACK_RPM_PACKAGE_PREFIX "${CMAKE_INSTALL_PREFIX}")

include(CPack)