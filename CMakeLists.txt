cmake_minimum_required(VERSION 3.10)
include(GNUInstallDirs)
project(webgpu
    VERSION 1.0.0
)

find_program(CARGO_EXECUTABLE cargo)
if(NOT CARGO_EXECUTABLE)
  message(FATAL_ERROR "Cargo not found. Make sure it is installed and in your PATH.")
endif()

set(LIBWEBGPU_A ${CMAKE_BINARY_DIR}/libwebgpu.a)
set(LIBWEBGPU_SO ${CMAKE_BINARY_DIR}/libwebgpu.so)

if(${CMAKE_BUILD_TYPE} MATCHES "Debug")
add_custom_command(
    OUTPUT ${LIBWEBGPU_A}
    COMMAND ${CARGO_EXECUTABLE} build --manifest-path ${CMAKE_SOURCE_DIR}/wgpu-native/Cargo.toml
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/wgpu-native
    COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/wgpu-native/target/${CARGO_BUILD_TYPE}/libwgpu_native.a ${LIBWEBGPU_A}
    COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/wgpu-native/target/${CARGO_BUILD_TYPE}/libwgpu_native.so ${LIBWEBGPU_SO}
    COMMENT "Building wgpu-native with Cargo"
)
else()
add_custom_command(
    OUTPUT ${LIBWEBGPU_A}
    COMMAND ${CARGO_EXECUTABLE} build --manifest-path ${CMAKE_SOURCE_DIR}/wgpu-native/Cargo.toml --release
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/wgpu-native
    COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/wgpu-native/target/${CARGO_BUILD_TYPE}/libwgpu_native.a ${LIBWEBGPU_A}
    COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/wgpu-native/target/${CARGO_BUILD_TYPE}/libwgpu_native.so ${LIBWEBGPU_SO}
    COMMENT "Building wgpu-native with Cargo"
endif()


add_custom_target(rust_build ALL
    DEPENDS ${LIBWEBGPU_A}
)

add_library(wgpu_native STATIC IMPORTED GLOBAL)
set_target_properties(wgpu_native PROPERTIES
  IMPORTED_LOCATION ${LIBWEBGPU_A}
)

add_dependencies(wgpu_native rust_build)
add_library(${PROJECT_NAME} INTERFACE)
target_link_libraries(${PROJECT_NAME} INTERFACE wgpu_native)

target_include_directories(${PROJECT_NAME}
    INTERFACE
    $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/..>
    $<BUILD_INTERFACE:${CMAKE_BINARY_DIR}/>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
)

install(
    TARGETS ${PROJECT_NAME}
    EXPORT ${PROJECT_NAME}_Exports
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    NAMELINK_SKIP
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

install(EXPORT ${PROJECT_NAME}_Exports DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}-${PROJECT_VERSION})

install(TARGETS ${PROJECT_NAME}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    NAMELINK_ONLY
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

install(FILES ${LIBWEBGPU_A} DESTINATION ${CMAKE_INSTALL_LIBDIR})
install(FILES ${LIBWEBGPU_SO} DESTINATION ${CMAKE_INSTALL_BINDIR})
install(DIRECTORY ${CMAKE_SOURCE_DIR}/include/ DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    FILES_MATCHING PATTERN "webgpu.hpp" PATTERN "webgpu.h" PATTERN "wgpu.h"
)

include(CMakePackageConfigHelpers)

configure_package_config_file(
    "${PROJECT_SOURCE_DIR}/cmake/${PROJECT_NAME}Config.cmake.in"
    "${PROJECT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
    INSTALL_DESTINATION
    ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}-${PROJECT_VERSION}
)

write_basic_package_version_file(
    "${PROJECT_NAME}ConfigVersion.cmake"
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion
)

install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake
    DESTINATION
    ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}-${PROJECT_VERSION}
)

