cmake_minimum_required(VERSION 3.10)
project(webgpu)

# Find Cargo
find_program(CARGO_EXECUTABLE cargo)
if(NOT CARGO_EXECUTABLE)
  message(FATAL_ERROR "Cargo not found. Make sure it is installed and in your PATH.")
endif()

# Define the output directory for the Rust build
set(RUST_BUILD_DIR "${CMAKE_BINARY_DIR}/rust")

# Ensure the output directory exists
file(MAKE_DIRECTORY ${RUST_BUILD_DIR})

# Add a custom command to build the Rust library
add_custom_command(
  OUTPUT ${RUST_BUILD_DIR}/libwgpu_native.a
  COMMAND ${CARGO_EXECUTABLE} build --release --manifest-path ${CMAKE_SOURCE_DIR}/wgpu-native/Cargo.toml
  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/wgpu-native
  COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/wgpu-native/target/release/libwgpu_native.a ${RUST_BUILD_DIR}/libwgpu_native.a
  COMMENT "Building wgpu-native with Cargo"
)

# Create a custom target that depends on the Rust build
add_custom_target(rust_build ALL
  DEPENDS ${RUST_BUILD_DIR}/libwgpu_native.a
)

# Add the Rust library as an imported target
add_library(wgpu_native STATIC IMPORTED GLOBAL)
set_target_properties(wgpu_native PROPERTIES
  IMPORTED_LOCATION ${RUST_BUILD_DIR}/libwgpu_native.a
)

# Ensure that the rust_build target is built before linking
add_dependencies(wgpu_native rust_build)

# Define your C++ library
add_library(${PROJECT_NAME} SHARED
  src/webgpu.cpp
)

# Link your C++ library with the Rust library
target_link_libraries(${PROJECT_NAME} PRIVATE wgpu_native)

# Add include directories if needed
target_include_directories(${PROJECT_NAME} PRIVATE ${CMAKE_SOURCE_DIR}/include)

