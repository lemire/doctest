cmake_minimum_required(VERSION 3.10)

if(POLICY CMP0077)
    cmake_policy(SET CMP0077 NEW)
endif()

################################################################################
## DOCTEST
################################################################################

file(READ ${CMAKE_CURRENT_SOURCE_DIR}/scripts/version.txt ver)
project(doctest VERSION ${ver} LANGUAGES CXX)

# Determine if doctest is built as a subproject (using add_subdirectory) or if it is the main project.
set(MAIN_PROJECT OFF)
if(CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
    set(MAIN_PROJECT ON)
endif()

option(DOCTEST_WITH_TESTS               "Build tests/examples" ${MAIN_PROJECT})
option(DOCTEST_WITH_MAIN_IN_STATIC_LIB  "Build a static lib (cmake target) with a default main entry point" ON)
option(DOCTEST_NO_INSTALL  "Skip the installation process" OFF)
option(DOCTEST_USE_STD_HEADERS  "Use std headers" OFF)

add_library(${PROJECT_NAME} INTERFACE)
add_library(${PROJECT_NAME}::${PROJECT_NAME} ALIAS ${PROJECT_NAME})

if(NOT CMAKE_VERSION VERSION_LESS 3.8)
    target_compile_features(${PROJECT_NAME} INTERFACE cxx_std_11)
endif()

set(doctest_parts_folder "${CMAKE_CURRENT_SOURCE_DIR}/doctest/parts")
set(doctest_folder "${CMAKE_CURRENT_SOURCE_DIR}/") # in order to have the mpi extension files, not included into the doctest.h single header

if(MAIN_PROJECT)
    # use a special hidden version of the header which directly includes the 2 parts - proper reporting of file/line locations during dev
    target_include_directories(${PROJECT_NAME} INTERFACE
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/scripts/development_only/>
        $<BUILD_INTERFACE:${doctest_parts_folder}>
        $<BUILD_INTERFACE:${doctest_folder}>)

    # add a custom target that assembles the single header when any of the parts are touched
    add_custom_command(
        OUTPUT ${CMAKE_CURRENT_SOURCE_DIR}/doctest/doctest.h
        DEPENDS
            ${doctest_parts_folder}/doctest_fwd.h
            ${doctest_parts_folder}/doctest.cpp
        COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_SOURCE_DIR}/scripts/cmake/assemble_single_header.cmake
        COMMENT "assembling the single header")

    add_custom_target(assemble_single_header ALL DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/doctest/doctest.h)
else()
    target_include_directories(${PROJECT_NAME} INTERFACE $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/>)
endif()

if(CMAKE_BUILD_TYPE STREQUAL "Debug")
	if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
		if (NOT WIN32)
			set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -gdwarf-4 ")
		endif()
	endif()
endif()

# hack to support building on XCode 6 and 7 - propagate the definition to everything
if(DEFINED DOCTEST_THREAD_LOCAL)
    target_compile_definitions(${PROJECT_NAME} INTERFACE
        DOCTEST_THREAD_LOCAL=${DOCTEST_THREAD_LOCAL})
endif()

if(DOCTEST_USE_STD_HEADERS)
    target_compile_definitions(${PROJECT_NAME} INTERFACE DOCTEST_CONFIG_USE_STD_HEADERS)
endif()

################################################################################
## TESTS/EXAMPLES/HELPERS
################################################################################

if(${DOCTEST_WITH_MAIN_IN_STATIC_LIB})
    add_library(${PROJECT_NAME}_with_main STATIC EXCLUDE_FROM_ALL ${doctest_parts_folder}/doctest.cpp)
    add_library(${PROJECT_NAME}::${PROJECT_NAME}_with_main ALIAS ${PROJECT_NAME}_with_main)
    target_compile_definitions(${PROJECT_NAME}_with_main PRIVATE
        DOCTEST_CONFIG_IMPLEMENT_WITH_MAIN)
    set_target_properties(${PROJECT_NAME}_with_main PROPERTIES CXX_STANDARD 11 CXX_STANDARD_REQUIRED ON)
    target_link_libraries(${PROJECT_NAME}_with_main PUBLIC ${PROJECT_NAME})
endif()

if(MAIN_PROJECT AND DOCTEST_WITH_TESTS)
    include(scripts/cmake/common.cmake)

    add_subdirectory(examples/all_features)
    add_subdirectory(examples/exe_with_static_libs)
    add_subdirectory(examples/executable_dll_and_plugin)
    add_subdirectory(examples/combining_the_same_tests_built_differently_in_multiple_shared_objects)
    add_subdirectory(scripts/playground)
    add_subdirectory(examples/mpi)
endif()

set(DOCTEST_CMAKE_HELPER "${CMAKE_CURRENT_SOURCE_DIR}/scripts/cmake/doctest.cmake" CACHE PATH "Absolute path to to doctest helper file. `include(${DOCTEST_CMAKE_HELPER})`")

################################################################################
## PACKAGE SUPPORT
################################################################################

set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")

if(CMAKE_SYSTEM_NAME STREQUAL Linux)
    include(GNUInstallDirs)
    set(include_install_dir ${CMAKE_INSTALL_INCLUDEDIR})
    set(config_install_dir "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}")
else()
    set(include_install_dir "include")
    set(config_install_dir "lib/cmake/${PROJECT_NAME}")
endif()


set(version_config "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
set(targets_export_name "${PROJECT_NAME}Targets")
set(namespace "${PROJECT_NAME}::")

include(CMakePackageConfigHelpers)

# CMake automatically adds an architecture compatibility check to make sure
# 32 and 64 bit code is not accidentally mixed. For a header-only library this
# is not required. The check can be disabled by temporarily unsetting
# CMAKE_SIZEOF_VOID_P. In CMake 3.14 and later this can be achieved more cleanly
# with write_basic_package_version_file(ARCH_INDEPENDENT).
# TODO: Use this once a newer CMake can be required.
set(DOCTEST_SIZEOF_VOID_P ${CMAKE_SIZEOF_VOID_P})
unset(CMAKE_SIZEOF_VOID_P)
write_basic_package_version_file(
    "${version_config}" VERSION ${PROJECT_VERSION} COMPATIBILITY SameMajorVersion
)
set(CMAKE_SIZEOF_VOID_P ${DOCTEST_SIZEOF_VOID_P})

configure_file("scripts/cmake/Config.cmake.in" "${project_config}" @ONLY)

if(NOT ${DOCTEST_NO_INSTALL})
    install(
        TARGETS ${PROJECT_NAME}
        EXPORT "${targets_export_name}"
        INCLUDES DESTINATION "${include_install_dir}"
    )

    install(
        FILES "doctest/doctest.h"
        DESTINATION "${include_install_dir}/doctest"
    )
    install(
        DIRECTORY "doctest/extensions"
        DESTINATION "${include_install_dir}/doctest"
        FILES_MATCHING PATTERN "*.h"
    )

    install(
        FILES "${project_config}" "${version_config}"
        DESTINATION "${config_install_dir}"
    )

    install(
        FILES "scripts/cmake/doctest.cmake" "scripts/cmake/doctestAddTests.cmake"
        DESTINATION "${config_install_dir}"
    )

    install(
        EXPORT "${targets_export_name}"
        NAMESPACE "${namespace}"
        DESTINATION "${config_install_dir}"
    )
endif()
