############################################################################
# Copyright (c) 2017, Sylvain Corlay and Johan Mabille                     #
#                                                                          #
# Distributed under the terms of the BSD 3-Clause License.                 #
#                                                                          #
# The full license is in the file LICENSE, distributed with this software. #
############################################################################

cmake_minimum_required(VERSION 3.1)

if (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
    project(xeus-test)

    enable_testing()

    find_package(xeus REQUIRED CONFIG)
    find_package(nlohmann_json QUIET CONFIG)
endif ()

message(STATUS "Forcing tests build type to Release")
set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)

include(CheckCXXCompilerFlag)

string(TOUPPER "${CMAKE_BUILD_TYPE}" U_CMAKE_BUILD_TYPE)

if(nlohmann_json_FOUND)
  add_definitions(-DHAVE_NLOHMANN_JSON)
endif()

if(CMAKE_CXX_COMPILER_ID MATCHES Clang OR CMAKE_CXX_COMPILER_ID MATCHES GNU OR CMAKE_CXX_COMPILER_ID MATCHES Intel)
    add_compile_options(-Wunused-parameter -Wextra -Wreorder -Wconversion -Wsign-conversion)

    CHECK_CXX_COMPILER_FLAG(-march=native HAS_MARCH_NATIVE)
    if (HAS_MARCH_NATIVE)
        add_compile_options(-march=native)
    endif()
endif()

if(CMAKE_CXX_COMPILER_ID MATCHES MSVC)
    add_compile_options(/EHsc /MP /bigobj)
    set(CMAKE_EXE_LINKER_FLAGS /MANIFEST:NO)
endif()

if(DOWNLOAD_GTEST OR GTEST_SRC_DIR)
    if(DOWNLOAD_GTEST)
        # Download and unpack googletest at configure time
        configure_file(downloadGTest.cmake.in googletest-download/CMakeLists.txt)
    else()
        # Copy local source of googletest at configure time
        configure_file(copyGTest.cmake.in googletest-download/CMakeLists.txt)
    endif()
    execute_process(COMMAND ${CMAKE_COMMAND} -G "${CMAKE_GENERATOR}" .
                    RESULT_VARIABLE result
                    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/googletest-download )
    if(result)
        message(FATAL_ERROR "CMake step for googletest failed: ${result}")
    endif()
    execute_process(COMMAND ${CMAKE_COMMAND} --build .
                    RESULT_VARIABLE result
                    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/googletest-download )
    if(result)
        message(FATAL_ERROR "Build step for googletest failed: ${result}")
    endif()

    # Add googletest directly to our build. This defines
    # the gtest and gtest_main targets.
    add_subdirectory(${CMAKE_CURRENT_BINARY_DIR}/googletest-src
                     ${CMAKE_CURRENT_BINARY_DIR}/googletest-build)

    set(GTEST_INCLUDE_DIRS "${gtest_SOURCE_DIR}/include")
    set(GTEST_BOTH_LIBRARIES  gtest_main gtest)
else()
    find_package(GTest REQUIRED)
endif()

find_package(Threads)

include_directories(${GTEST_INCLUDE_DIRS})

set(XEUS_TESTS
    test_xin_memory_history_manager.cpp
)

if(nlohmann_json_FOUND)
    # Version up to 3.1.2 export the target `nlohmann_json`
    if(TARGET nlohmann_json)
      set(nlohmann_json_TARGET nlohmann_json)
    # Newer versions export the namespaced target `nlohmann_json::nlohmann_json`
    elseif(TARGET nlohmann_json::nlohmann_json)
      set(nlohmann_json_TARGET nlohmann_json::nlohmann_json)
    endif()
endif()

foreach(filename IN LISTS XEUS_TESTS)
    get_filename_component(targetname ${filename} NAME_WE)

    add_executable(${targetname} ${filename})
    target_link_libraries(${targetname} xeus ${GTEST_BOTH_LIBRARIES} Threads::Threads ${nlohmann_json_TARGET})

    add_test(NAME ${targetname} COMMAND ${targetname})
endforeach()

set(XEUS_TARGET test_xeus)

add_executable(${XEUS_TARGET} ${XEUS_TESTS} ${XEUS_HEADERS})
if(DOWNLOAD_GTEST OR GTEST_SRC_DIR)
    add_dependencies(${XEUS_TARGET} gtest_main)
endif()
target_link_libraries(${XEUS_TARGET} xeus ${GTEST_BOTH_LIBRARIES} Threads::Threads ${nlohmann_json_TARGET})

add_custom_target(xtest COMMAND test_xeus DEPENDS ${XEUS_TARGET})
