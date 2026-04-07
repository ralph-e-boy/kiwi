# --------------------------------------------------------------------------------------
# Copyright (c) 2013-2026, Nucleic Development Team.
#
# Distributed under the terms of the Modified BSD License.
#
# The full license is in the file LICENSE, distributed with this software.
# --------------------------------------------------------------------------------------

cmake_minimum_required(VERSION 3.15)

# Read version from version.h
file(READ "${CMAKE_CURRENT_SOURCE_DIR}/kiwi/version.h" VERSION_FILE)
string(REGEX MATCH "KIWI_MAJOR_VERSION ([0-9]+)" _ ${VERSION_FILE})
set(KIWI_MAJOR_VERSION ${CMAKE_MATCH_1})
string(REGEX MATCH "KIWI_MINOR_VERSION ([0-9]+)" _ ${VERSION_FILE})
set(KIWI_MINOR_VERSION ${CMAKE_MATCH_1})
string(REGEX MATCH "KIWI_MICRO_VERSION ([0-9]+)" _ ${VERSION_FILE})
set(KIWI_MICRO_VERSION ${CMAKE_MATCH_1})

project(kiwi 
    VERSION ${KIWI_MAJOR_VERSION}.${KIWI_MINOR_VERSION}.${KIWI_MICRO_VERSION}
    LANGUAGES CXX
    DESCRIPTION "A fast implementation of the Cassowary constraint solver"
)

# Determine if this is the top-level project
if(CMAKE_PROJECT_NAME STREQUAL PROJECT_NAME)
    set(KIWI_IS_TOP_LEVEL TRUE)
else()
    set(KIWI_IS_TOP_LEVEL FALSE)
endif()

# Options
option(KIWI_BUILD_BENCHMARKS "Build benchmarks" ${KIWI_IS_TOP_LEVEL})
option(KIWI_BUILD_TESTS "Build tests" ${KIWI_IS_TOP_LEVEL})
option(KIWI_SWIFT_WRAPPER "Enable Swift wrapper generation targets" OFF)

# C++ standard
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Header-only library
add_library(kiwi INTERFACE)
add_library(kiwi::kiwi ALIAS kiwi)

target_include_directories(kiwi
    INTERFACE
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<INSTALL_INTERFACE:include>
)

target_compile_features(kiwi INTERFACE cxx_std_11)

target_compile_options(kiwi INTERFACE
    $<$<CXX_COMPILER_ID:GNU,Clang,AppleClang>:
        -Wall 
        -Wextra
        -Wpedantic
    >
)

# Benchmarks
if(KIWI_BUILD_BENCHMARKS)
    add_subdirectory(benchmarks)
endif()

# Tests
if(KIWI_BUILD_TESTS)
    add_subdirectory(tests)
endif()

# Swift package (opt-in)
if(KIWI_SWIFT_WRAPPER)
    find_program(XCRUN xcrun)
    if(NOT XCRUN)
        message(FATAL_ERROR "xcrun not found — Swift toolchain required for KIWI_SWIFT_WRAPPER")
    endif()

    find_package(Python3 REQUIRED COMPONENTS Interpreter)

    # Wrapper generation
    add_custom_target(generate-swift-wrapper
        COMMAND ${Python3_EXECUTABLE}
            ${CMAKE_CURRENT_SOURCE_DIR}/Sources/CxxKiwi/generate_wrapper.py
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        COMMENT "Generating KiwiWrapper.h for Swift package"
    )
    add_custom_target(validate-swift-wrapper
        COMMAND ${Python3_EXECUTABLE}
            ${CMAKE_CURRENT_SOURCE_DIR}/Sources/CxxKiwi/generate_wrapper.py
            --validate-only
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        COMMENT "Validating KiwiWrapper.h against kiwi headers"
    )

    # Swift build / test / run
    add_custom_target(swift-build
        COMMAND ${XCRUN} swift build
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        COMMENT "Building Swift package"
    )
    add_custom_target(swift-build-release
        COMMAND ${XCRUN} swift build -c release
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        COMMENT "Building Swift package (release)"
    )
    add_custom_target(swift-test
        COMMAND ${XCRUN} swift test
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        COMMENT "Running Swift tests"
    )
    add_custom_target(swift-run
        COMMAND ${XCRUN} swift run KiwiTest
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        DEPENDS swift-build
        COMMENT "Running KiwiTest"
    )
    add_custom_target(swift-clean
        COMMAND ${XCRUN} swift package clean
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        COMMENT "Cleaning Swift build artifacts"
    )

    # Cross-platform builds (via xcodebuild)
    find_program(XCODEBUILD xcodebuild)
    if(XCODEBUILD)
        set(_SWIFT_SCHEME KiwiSolver)
        set(_SWIFT_XBUILD ${XCODEBUILD} -scheme ${_SWIFT_SCHEME} -quiet)

        add_custom_target(swift-build-ios
            COMMAND ${_SWIFT_XBUILD} -destination "generic/platform=iOS" build
            WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
            COMMENT "Building KiwiSolver for iOS"
        )
        add_custom_target(swift-build-ios-simulator
            COMMAND ${_SWIFT_XBUILD} -destination "generic/platform=iOS Simulator" build
            WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
            COMMENT "Building KiwiSolver for iOS Simulator"
        )
        add_custom_target(swift-build-tvos
            COMMAND ${_SWIFT_XBUILD} -destination "generic/platform=tvOS" build
            WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
            COMMENT "Building KiwiSolver for tvOS"
        )
        add_custom_target(swift-build-watchos
            COMMAND ${_SWIFT_XBUILD} -destination "generic/platform=watchOS" build
            WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
            COMMENT "Building KiwiSolver for watchOS"
        )
        add_custom_target(swift-build-visionos
            COMMAND ${_SWIFT_XBUILD} -destination "generic/platform=visionOS" build
            WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
            COMMENT "Building KiwiSolver for visionOS"
        )
        # Aggregate: always-available platforms (macOS + iOS)
        add_custom_target(swift-doc
            COMMAND ${XCRUN} swift package generate-documentation
                --target KiwiSolver
            WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
            COMMENT "Generating DocC documentation for KiwiSolver"
        )
        add_custom_target(swift-doc-preview
            COMMAND ${XCRUN} swift package preview-documentation
                --target KiwiSolver
            WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
            COMMENT "Previewing DocC documentation for KiwiSolver"
        )
        set(_SWIFT_API_IFACE ${CMAKE_CURRENT_SOURCE_DIR}/docs/build/KiwiSolver.swiftinterface)
        file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/swift-api.sh
            "#!/bin/sh\nmkdir -p \"$(dirname '${_SWIFT_API_IFACE}')\"\n${XCRUN} swift build -Xswiftc -emit-module-interface-path -Xswiftc '${_SWIFT_API_IFACE}' -Xswiftc -enable-library-evolution --target KiwiSolver 2>/dev/null || true\necho \"Wrote ${_SWIFT_API_IFACE}\"\n")
        add_custom_target(swift-api
            COMMAND sh ${CMAKE_CURRENT_BINARY_DIR}/swift-api.sh
            WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
            COMMENT "Generating Swift public interface (.swiftinterface)"
        )
        add_custom_target(swift-build-all-platforms
            DEPENDS swift-build swift-build-ios swift-build-ios-simulator
            COMMENT "Building KiwiSolver for macOS + iOS (add tvOS/watchOS/visionOS individually if SDK installed)"
        )
    else()
        message(STATUS "  xcodebuild not found — cross-platform Swift targets disabled")
    endif()
endif()

# Print configuration summary
message(STATUS "")
message(STATUS "Kiwi Configuration Summary:")
message(STATUS "  Version: ${PROJECT_VERSION}")
message(STATUS "  C++ Standard: C++${CMAKE_CXX_STANDARD}")
message(STATUS "  Build benchmarks: ${KIWI_BUILD_BENCHMARKS}")
message(STATUS "  Build tests: ${KIWI_BUILD_TESTS}")
message(STATUS "  Swift wrapper: ${KIWI_SWIFT_WRAPPER}")
message(STATUS "")
