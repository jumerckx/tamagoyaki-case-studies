cmake_minimum_required(VERSION 3.20.0)
project(tamagoyaki-case-studies VERSION 0.1.0 LANGUAGES CXX C)

# The two case studies built on Tamagoyaki:
#
#   herbie_mlir   floating-point accuracy, in the spirit of Herbie, with
#                 interval arithmetic from Rival 3
#   rover-mlir    datapath optimisation over CIRCT's comb/hw/datapath
#
# They live outside the Tamagoyaki repository because of what they drag in --
# CIRCT, Rust, Rival 3, a pinned Herbie and its whole Racket package closure --
# none of which the framework itself needs. Tamagoyaki arrives here the way any
# other dependency does, through find_package below and a pinned flake input.

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
if(CMAKE_EXPORT_COMPILE_COMMANDS)
  set(CMAKE_CXX_STANDARD_INCLUDE_DIRECTORIES
      ${CMAKE_CXX_IMPLICIT_INCLUDE_DIRECTORIES})
endif()

set(CMAKE_CXX_STANDARD 17 CACHE STRING "C++ standard to conform to")

include(GNUInstallDirs)

option(ENABLE_ASAN "Enable AddressSanitizer" OFF)
if(ENABLE_ASAN)
  add_compile_options(-fsanitize=address -fno-omit-frame-pointer)
  add_link_options(-fsanitize=address)
endif()

find_package(MLIR REQUIRED CONFIG)
message(STATUS "Using MLIRConfig.cmake in: ${MLIR_DIR}")
message(STATUS "Using LLVMConfig.cmake in: ${LLVM_DIR}")

# Brings the MLIREquivalence / MLIREmatch / TamagoyakiTiming imported targets,
# the add_dialect_tablegen() helper, and TAMAGOYAKI_TOOLS_DIR (where
# tamagoyaki-opt lives, which the lit suites run). Point Tamagoyaki_DIR at an
# install prefix's lib/cmake/tamagoyaki, or -- for the development loop against
# a local checkout -- at a Tamagoyaki build directory's.
find_package(Tamagoyaki REQUIRED CONFIG)
message(STATUS "Using TamagoyakiConfig.cmake in: ${Tamagoyaki_DIR}")

list(APPEND CMAKE_MODULE_PATH "${MLIR_CMAKE_DIR}")
list(APPEND CMAKE_MODULE_PATH "${LLVM_CMAKE_DIR}")

include(TableGen)
include(AddLLVM)
include(AddMLIR)
include(HandleLLVMOptions)

set(LLVM_RUNTIME_OUTPUT_INTDIR ${CMAKE_BINARY_DIR}/bin)
set(LLVM_LIBRARY_OUTPUT_INTDIR ${CMAKE_BINARY_DIR}/lib)
set(MLIR_BINARY_DIR ${CMAKE_BINARY_DIR})

include_directories(${LLVM_INCLUDE_DIRS})
include_directories(${MLIR_INCLUDE_DIRS})
link_directories(${LLVM_BUILD_LIBRARY_DIR})
separate_arguments(LLVM_DEFINITIONS_LIST NATIVE_COMMAND ${LLVM_DEFINITIONS})
add_definitions(${LLVM_DEFINITIONS_LIST})

# Tamagoyaki's own include directories are deliberately not added at directory
# scope: they are usage requirements of MLIREquivalence/MLIREmatch, exactly as
# they are inside the Tamagoyaki tree.

# ============================================================================
# Subprojects
# ============================================================================
# Each can be turned off, and the evaluations build exactly one: the Herbie
# pipeline has no use for CIRCT, and the Rover pipeline none for Racket.
option(BUILD_HERBIE_MLIR "Build herbie_mlir subproject" ON)
if(BUILD_HERBIE_MLIR)
  add_subdirectory(herbie_mlir)
endif()

option(BUILD_ROVER_MLIR "Build rover-mlir subproject" ON)
if(BUILD_ROVER_MLIR)
  find_package(CIRCT REQUIRED CONFIG)
  message(STATUS "Using CIRCTConfig.cmake in: ${CIRCT_DIR}")
  list(APPEND CMAKE_MODULE_PATH "${CIRCT_CMAKE_DIR}")
  include_directories(${CIRCT_INCLUDE_DIRS})
  add_subdirectory(rover-mlir)
endif()

# ============================================================================
# check-all
# ============================================================================
# Run every test suite that is enabled. A subproject without a lit suite is
# still pulled in as a build dep, so CI fails on a compile break instead of
# silently skipping it.
set(CHECK_ALL_DEPENDS)
if(BUILD_HERBIE_MLIR)
  list(APPEND CHECK_ALL_DEPENDS check-herbie-mlir)
endif()
if(BUILD_ROVER_MLIR)
  if(ROVER_MLIR_BUILD_TOOLS)
    list(APPEND CHECK_ALL_DEPENDS check-rover-mlir)
  else()
    list(APPEND CHECK_ALL_DEPENDS RoverDialect)
  endif()
endif()
add_custom_target(check-all DEPENDS ${CHECK_ALL_DEPENDS})
set_target_properties(check-all PROPERTIES FOLDER "Tests")
