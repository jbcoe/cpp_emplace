cmake_minimum_required(VERSION 3.22.4 FATAL_ERROR)
cmake_policy(SET CMP0127 NEW)
cmake_policy(SET CMP0135 NEW)

if(NOT DEFINED PROJECT_NAME)
    set(CPP_EMPLACE_IS_NOT_SUBPROJECT ON)
endif()

set(CPP_EMPLACE_VERSION 0.0.1)

project(emplace LANGUAGES CXX VERSION ${CPP_EMPLACE_VERSION})

include(CTest)
include(FetchContent)
include(GNUInstallDirs)
include(CMakeDependentOption)
include(CMakePackageConfigHelpers)

# Ensure code coverage is switched off for Mac OS until the code coverage library addresses the AppleClang issue
cmake_dependent_option(ENABLE_CODE_COVERAGE "Enable code coverage" ON "\"${CMAKE_CXX_COMPILER_ID}\" STREQUAL \"Clang\" OR \"${CMAKE_CXX_COMPILER_ID}\" STREQUAL \"GNU\"" OFF)
option(ENABLE_SANITIZERS "Enable Address Sanitizer and Undefined Behaviour Sanitizer if available" OFF)

add_library(emplace INTERFACE)
target_include_directories(emplace
    INTERFACE
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)
target_sources(emplace
    INTERFACE
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/emplace.h>
)
target_compile_features(emplace
    INTERFACE
        cxx_std_20
)

add_library(indirect INTERFACE)
target_include_directories(indirect
    INTERFACE
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)
target_sources(indirect
    INTERFACE
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/indirect.h>
)
target_compile_features(indirect
    INTERFACE
        cxx_std_20
)

add_library(polymorphic INTERFACE)
target_include_directories(polymorphic
    INTERFACE
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)
target_sources(emplace
    INTERFACE
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/emplace.h>
)
target_compile_features(emplace
    INTERFACE
        cxx_std_20
)

add_library(xyz::emplace ALIAS emplace)

if (${CPP_EMPLACE_IS_NOT_SUBPROJECT})

    if (${BUILD_TESTING})
        FetchContent_Declare(
          googletest
          URL https://github.com/google/googletest/archive/03597a01ee50ed33e9dfd640b249b4be3799d395.zip
        )
        # For Windows: Prevent overriding the parent project's compiler/linker settings
        set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
        FetchContent_MakeAvailable(googletest)

        add_library(common_compiler_settings INTERFACE)
        target_compile_options(common_compiler_settings
            INTERFACE
                $<$<CXX_COMPILER_ID:MSVC>:/EHsc>
                $<$<CXX_COMPILER_ID:MSVC>:/W4>
                $<$<CXX_COMPILER_ID:MSVC>:/bigobj>
                $<$<OR:$<CXX_COMPILER_ID:GNU>,$<CXX_COMPILER_ID:AppleClang>,$<CXX_COMPILER_ID:Clang>>:-Werror;-Wall;-Wno-self-assign-overloaded;-Wno-unknown-warning-option>
        )
        set_target_properties(common_compiler_settings PROPERTIES
            CXX_STANDARD 20
            CXX_STANDARD_REQUIRED YES
            CXX_EXTENSIONS NO
        )

        add_executable(emplace_test "")
        target_sources(emplace_test
            PRIVATE
                emplace_test.cc
        )
        target_link_libraries(emplace_test
            PRIVATE
                emplace
                GTest::gtest_main
                common_compiler_settings
        )

        if (ENABLE_SANITIZERS)
            set(SANITIZER_FLAGS_ASAN "-fsanitize=address -fno-omit-frame-pointer")
            set(SANITIZER_FLAGS_UBSAN "-fsanitize=undefined")

            include(CheckCXXCompilerFlag)
            check_cxx_compiler_flag("${SANITIZER_FLAGS_ASAN}" COMPILER_SUPPORTS_ASAN)
            check_cxx_compiler_flag("${SANITIZER_FLAGS_UBSAN}" COMPILER_SUPPORTS_UBSAN)

            if (COMPILER_SUPPORTS_ASAN)
                add_library(asan INTERFACE IMPORTED)
                set_target_properties(asan PROPERTIES
                    INTERFACE_COMPILE_OPTIONS "${SANITIZER_FLAGS_ASAN}"
                    INTERFACE_LINK_OPTIONS "${SANITIZER_FLAGS_ASAN}"
                )
                target_link_libraries(emplace_test
                    PRIVATE
                        asan
                )
            endif(COMPILER_SUPPORTS_ASAN)

            if (COMPILER_SUPPORTS_UBSAN)
                add_library(ubsan INTERFACE IMPORTED)
                set_target_properties(ubsan PROPERTIES
                    INTERFACE_COMPILE_OPTIONS "${SANITIZER_FLAGS_UBSAN}"
                    INTERFACE_LINK_OPTIONS "${SANITIZER_FLAGS_UBSAN}"
                )
                target_link_libraries(emplace_test
                    PRIVATE
                        ubsan
                )
            endif(COMPILER_SUPPORTS_UBSAN)
        endif(ENABLE_SANITIZERS)

        enable_testing()
        add_test(
            NAME emplace_test
            COMMAND emplace_test
            WORKING_DIRECTORY ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})

        list(APPEND CMAKE_MODULE_PATH ${catch2_SOURCE_DIR}/contrib)
        include(GoogleTest)
        gtest_discover_tests(emplace_test)

        if (ENABLE_CODE_COVERAGE)
            FetchContent_Declare(
                codecoverage
                GIT_REPOSITORY https://github.com/RWTH-HPC/CMake-codecov.git
            )

            FetchContent_GetProperties(codecoverage)
            if(NOT codecoverage_POPULATED)
                FetchContent_Populate(codecoverage)
                list(APPEND CMAKE_MODULE_PATH ${codecoverage_SOURCE_DIR}/cmake)
            endif()

            set(ENABLE_COVERAGE ON CACHE BOOL "Enable coverage build." FORCE)
            find_package(codecov)
            add_coverage(emplace_test)
            list(APPEND LCOV_REMOVE_PATTERNS "'/usr/*'")
            coverage_evaluate()
        endif()
    endif(${BUILD_TESTING})

    install(
        FILES
            "${CMAKE_CURRENT_SOURCE_DIR}/indirect.h"
            "${CMAKE_CURRENT_SOURCE_DIR}/polymorphic.h"
        DESTINATION
            ${CMAKE_INSTALL_INCLUDEDIR}
    )

    install(
        FILES
            "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE.txt"
        DESTINATION
            "${CMAKE_INSTALL_DATAROOTDIR}/licenses/emplace"
    )

    install(
        TARGETS emplace
        EXPORT emplace-export-set
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    )

    install(
        EXPORT emplace-export-set
        NAMESPACE emplace::
        DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/emplace"
    )

    configure_package_config_file(
        ${CMAKE_CURRENT_LIST_DIR}/emplace-config.cmake.in
        ${CMAKE_CURRENT_BINARY_DIR}/emplace-config.cmake
        INSTALL_DESTINATION
            "${CMAKE_INSTALL_LIBDIR}/cmake/emplace"
    )

    write_basic_package_version_file(
        ${CMAKE_CURRENT_BINARY_DIR}/emplace-version.cmake
        VERSION ${CPP_EMPLACE_VERSION}
        COMPATIBILITY SameMajorVersion
        ARCH_INDEPENDENT
    )

    install(
        FILES
            ${CMAKE_CURRENT_BINARY_DIR}/emplace-config.cmake
            ${CMAKE_CURRENT_BINARY_DIR}/emplace-version.cmake
        DESTINATION
            "${CMAKE_INSTALL_LIBDIR}/cmake/emplace"
    )
endif()
