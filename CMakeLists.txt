cmake_minimum_required(VERSION 3.10)
include(GNUInstallDirs)
project(webgpu VERSION 1.0.0)

# Find Cargo executable
find_program(CARGO_EXECUTABLE cargo)
if(NOT CARGO_EXECUTABLE)
  message(FATAL_ERROR "Cargo not found. Make sure it is installed and in your PATH.")
endif()

# Set library output paths
set(LIBWEBGPU_A ${CMAKE_BINARY_DIR}/libwgpu_native.a)
set(LIBWEBGPU_SO ${CMAKE_BINARY_DIR}/libwgpu_native.so)

# Add custom commands based on build type
if(${CMAKE_BUILD_TYPE} MATCHES "Debug")
    add_custom_command(
        OUTPUT ${LIBWEBGPU_A}
        COMMAND ${CARGO_EXECUTABLE} build --manifest-path ${CMAKE_SOURCE_DIR}/wgpu-native/Cargo.toml
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/wgpu-native
        COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/wgpu-native/target/debug/libwgpu_native.a ${LIBWEBGPU_A}
        COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/wgpu-native/target/debug/libwgpu_native.so ${LIBWEBGPU_SO}
        COMMENT "Building wgpu-native with Cargo"
    )
else()
    add_custom_command(
        OUTPUT ${LIBWEBGPU_A}
        COMMAND ${CARGO_EXECUTABLE} build --manifest-path ${CMAKE_SOURCE_DIR}/wgpu-native/Cargo.toml --release
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/wgpu-native
        COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/wgpu-native/target/release/libwgpu_native.a ${LIBWEBGPU_A}
        COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/wgpu-native/target/release/libwgpu_native.so ${LIBWEBGPU_SO}
        COMMENT "Building wgpu-native with Cargo"
    )
endif()

# Define custom target
add_custom_target(rust_build ALL DEPENDS ${LIBWEBGPU_A})

# Add imported library for wgpu_native
add_library(wgpu_native STATIC IMPORTED GLOBAL)
set_target_properties(wgpu_native PROPERTIES IMPORTED_LOCATION ${LIBWEBGPU_A})

# Add dependencies
add_dependencies(wgpu_native rust_build)

# Define webgpu target
add_library(${PROJECT_NAME} INTERFACE)
target_link_libraries(${PROJECT_NAME} INTERFACE wgpu_native)

# Include directories
target_include_directories(${PROJECT_NAME} INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/..>
    $<BUILD_INTERFACE:${CMAKE_BINARY_DIR}/>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

# Install targets
install(TARGETS ${PROJECT_NAME}
    EXPORT ${PROJECT_NAME}_Exports
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

# Install export configuration
install(EXPORT ${PROJECT_NAME}_Exports DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}-${PROJECT_VERSION})

# Manually install wgpu_native files
install(FILES ${LIBWEBGPU_A} DESTINATION ${CMAKE_INSTALL_LIBDIR})
install(FILES ${LIBWEBGPU_SO} DESTINATION ${CMAKE_INSTALL_BINDIR})

# Install header files
install(DIRECTORY ${CMAKE_SOURCE_DIR}/include/ DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    FILES_MATCHING PATTERN "webgpu.hpp" PATTERN "webgpu.h" PATTERN "wgpu.h"
)

# Include CMake package config helpers
include(CMakePackageConfigHelpers)

# Configure package configuration file
configure_package_config_file(
    "${PROJECT_SOURCE_DIR}/cmake/${PROJECT_NAME}Config.cmake.in"
    "${PROJECT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}-${PROJECT_VERSION}
)

# Write package version file
write_basic_package_version_file(
    "${PROJECT_NAME}ConfigVersion.cmake"
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion
)

# Configure pkg-config file
configure_file("${PROJECT_SOURCE_DIR}/cmake/webgpu.pc.in" webgpu.pc @ONLY)

# Install CMake configuration files
install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}-${PROJECT_VERSION}
)

