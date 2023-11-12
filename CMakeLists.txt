cmake_minimum_required(VERSION 3.21)
project(my_application LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_FLAGS "-fPIC")
set(node_bridge "${CMAKE_CURRENT_BINARY_DIR}/node_bridge.cc")
set(operator_bridge "${CMAKE_CURRENT_BINARY_DIR}/operator_bridge.cc")
include(ExternalProject)
ExternalProject_Add(Dora
    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/dora
    GIT_REPOSITORY https://github.com/dora-rs/dora.git
    GIT_TAG main
    BUILD_IN_SOURCE True
    BUILD_ALAWYS True
    CONFIGURE_COMMAND ""
    BUILD_COMMAND
        cargo
        build
        --release
        --package dora-node-api-c
        --target-dir ${CMAKE_CURRENT_BINARY_DIR}/dora/build
        &&
        cargo
        build
        --release
        --package dora-operator-api-c
        --target-dir ${CMAKE_CURRENT_BINARY_DIR}/dora/build
        &&
        cargo
        build
        --release
        --package dora-node-api-cxx
        --target-dir ${CMAKE_CURRENT_BINARY_DIR}/dora/build
        &&
        cargo
        build
        --release
        --package dora-operator-api-cxx
        --target-dir ${CMAKE_CURRENT_BINARY_DIR}/dora/build
    INSTALL_COMMAND
        cp ${CMAKE_CURRENT_BINARY_DIR}/dora/build/cxxbridge/dora-node-api-cxx/src/lib.rs.cc
        ${node_bridge}
        &&
        cp ${CMAKE_CURRENT_BINARY_DIR}/dora/build/cxxbridge/dora-node-api-cxx/src/lib.rs.h
        ${CMAKE_CURRENT_BINARY_DIR}/dora-node-api.h
        &&
        cp ${CMAKE_CURRENT_BINARY_DIR}/dora/build/cxxbridge/dora-operator-api-cxx/src/lib.rs.cc
        ${operator_bridge}
        &&
        cp ${CMAKE_CURRENT_BINARY_DIR}/dora/build/cxxbridge/dora-operator-api-cxx/src/lib.rs.h
        ${CMAKE_CURRENT_BINARY_DIR}/dora-operator-api.h
    BUILD_BYPRODUCTS ${node_bridge} ${operator_bridge}
)

add_executable(node_c_api node-c-api/main.cc)
add_dependencies(node_c_api Dora)
target_include_directories(node_c_api PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/dora/src/Dora/apis/c)
target_link_directories(node_c_api PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/dora/build/release )
target_link_libraries(node_c_api dora_node_api_c)

add_executable(node_rust_api node-rust-api/main.cc ${node_bridge})
add_dependencies(node_rust_api Dora)
target_include_directories(node_rust_api PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
target_link_directories(node_rust_api PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/dora/build/release )
target_link_libraries(node_rust_api dora_node_api_cxx)

add_library(operator_c_api SHARED operator-c-api/operator.cc)
add_dependencies(operator_c_api Dora)
target_include_directories(operator_c_api PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/dora/src/Dora/apis/c)
target_link_directories(operator_c_api PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/dora/build/release )
target_link_libraries(operator_c_api dora_operator_api_c)

add_library(operator_rust_api SHARED operator-rust-api/operator.cc ${operator_bridge})
add_dependencies(operator_rust_api Dora)
target_include_directories(operator_rust_api PRIVATE ${CMAKE_CURRENT_BINARY_DIR} ${CMAKE_CURRENT_BINARY_DIR}/dora/src/Dora/apis/c ${CMAKE_CURRENT_SOURCE_DIR}/operator-rust-api)
target_link_directories(operator_rust_api PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/dora/build/release )
target_link_libraries(operator_rust_api dora_operator_api_cxx)

install(TARGETS node_c_api node_rust_api DESTINATION ${CMAKE_CURRENT_SOURCE_DIR}/bin)
install(TARGETS operator_c_api operator_rust_api DESTINATION ${CMAKE_CURRENT_SOURCE_DIR}/lib)