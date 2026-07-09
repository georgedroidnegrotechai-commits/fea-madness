cmake_minimum_required(VERSION 3.16)
project(fea_madness VERSION 0.1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Options
option(FEA_MADNESS_BUILD_EXAMPLES "Build example executables" ON)
option(FEA_MADNESS_BUILD_TESTS "Build tests (future)" OFF)

# Find or fetch Eigen (user should have Eigen3 installed, or we can add FetchContent later)
find_package(Eigen3 3.3 REQUIRED NO_MODULE)

# Core library (placeholder for now)
add_library(fea_madness_core INTERFACE)
target_include_directories(fea_madness_core INTERFACE
    ${CMAKE_CURRENT_SOURCE_DIR}/src
)
target_link_libraries(fea_madness_core INTERFACE Eigen3::Eigen)

# Examples
if(FEA_MADNESS_BUILD_EXAMPLES)
    add_subdirectory(examples)
endif()

# Future: tests, docs, install rules

message(STATUS "fea-madness configured. Eigen found at ${EIGEN3_INCLUDE_DIR}")
