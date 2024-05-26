cmake_minimum_required(VERSION 3.10)
project(webgpu)

find_program(CARGO_EXECUTABLE cargo)
if(NOT CARGO_EXECUTABLE)
  message(FATAL_ERROR "Cargo not found. Make sure it is installed and in your PATH.")
endif()

if (${CMAKE_BUILD_TYPE} MATCHES "Debug")
add_custom_command(
  OUTPUT ${CMAKE_BINARY_DIR}/libwebgpu.a
  COMMAND ${CARGO_EXECUTABLE} build --manifest-path ${CMAKE_SOURCE_DIR}/wgpu-native/Cargo.toml
  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/wgpu-native
  COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/wgpu-native/target/debug/libwgpu_native.a ${CMAKE_BINARY_DIR}/libwebgpu.a
  COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/wgpu-native/target/debug/libwgpu_native.so ${CMAKE_BINARY_DIR}/libwebgpu.so
  COMMENT "Building wgpu-native with Cargo"
)
else()
add_custom_command(
  OUTPUT ${CMAKE_BINARY_DIR}/libwebgpu.a
  COMMAND ${CARGO_EXECUTABLE} build --release --manifest-path ${CMAKE_SOURCE_DIR}/wgpu-native/Cargo.toml
  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/wgpu-native
  COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/wgpu-native/target/release/libwgpu_native.a ${CMAKE_BINARY_DIR}/libwebgpu.a
  COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/wgpu-native/target/release/libwgpu_native.so ${CMAKE_BINARY_DIR}/libwebgpu.so
  COMMENT "Building wgpu-native with Cargo"
)
endif()


add_custom_target(rust_build ALL
	DEPENDS ${CMAKE_BINARY_DIR}/libwebgpu.a
)

add_library(wgpu_native STATIC IMPORTED GLOBAL)
set_target_properties(wgpu_native PROPERTIES
  IMPORTED_LOCATION ${RUST_BUILD_DIR}/libwgpu_native.a
)

add_dependencies(wgpu_native rust_build)
add_library(${PROJECT_NAME} INTERFACE)
target_link_libraries(${PROJECT_NAME} INTERFACE wgpu_native)
target_include_directories(${PROJECT_NAME} INTERFACE ${CMAKE_SOURCE_DIR}/include)

install(DIRECTORY ${CMAKE_SOURCE_DIR}/include DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
install(FILES ${CMAKE_BINARY_DIR}/libwebgpu.a DESTINATION ${CMAKE_INSTALL_LIBDIR})
install(FILES ${CMAKE_BINARY_DIR}/libwebgpu.so DESTINATION ${CMAKE_INSTALL_LIBDIR})
