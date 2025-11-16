cmake_minimum_required(VERSION 3.20)
project(hand_on_cutlass LANGUAGES CUDA CXX)

set(_libtorch_desc "Optional path to a LibTorch installation (prefix containing share/cmake/Torch)")
set(LIBTORCH_ROOT "" CACHE PATH "${_libtorch_desc}")
if(NOT LIBTORCH_ROOT AND DEFINED ENV{LIBTORCH_ROOT})
  set(LIBTORCH_ROOT $ENV{LIBTORCH_ROOT})
endif()

if(NOT LIBTORCH_ROOT)
  set(_pixi_candidates
      ${CMAKE_SOURCE_DIR}/.pixi/env
      ${CMAKE_SOURCE_DIR}/.pixi/envs/default)
  get_filename_component(_source_parent "${CMAKE_SOURCE_DIR}" DIRECTORY)
  if(NOT _source_parent STREQUAL "${CMAKE_SOURCE_DIR}")
    list(APPEND _pixi_candidates
         ${_source_parent}/.pixi/env
         ${_source_parent}/.pixi/envs/default)
  endif()
  foreach(_pixi_hint IN LISTS _pixi_candidates)
    if(EXISTS "${_pixi_hint}/share/cmake/Torch/TorchConfig.cmake")
      set(_libtorch_auto_hint "${_pixi_hint}")
      break()
    endif()
  endforeach()
  if(_libtorch_auto_hint)
    set(LIBTORCH_ROOT "${_libtorch_auto_hint}" CACHE PATH "${_libtorch_desc}" FORCE)
    message(STATUS "Detected LibTorch under pixi environment: ${LIBTORCH_ROOT}")
  endif()
endif()

if(LIBTORCH_ROOT)
  list(APPEND CMAKE_PREFIX_PATH ${LIBTORCH_ROOT})
endif()

find_package(CUDAToolkit REQUIRED COMPONENTS cudart cublas)

find_package(Torch REQUIRED)

# Workaround for conda-forge libtorch which exports non-namespaced targets.
# We create aliases to maintain compatibility with modern CMake scripts that
# expect namespaced targets (e.g., Torch::Torch).
if(TARGET torch AND NOT TARGET Torch::Torch)
  message(STATUS "Creating ALIAS Torch::Torch for target torch")
  add_library(Torch::Torch ALIAS torch)
endif()

# Also check for other common torch libraries that might be used.
if(TARGET c10 AND NOT TARGET c10::c10)
  message(STATUS "Creating ALIAS c10::c10 for target c10")
  add_library(c10::c10 ALIAS c10)
endif()

if(TARGET torch_cuda AND NOT TARGET Torch::torch_cuda)
  message(STATUS "Creating ALIAS Torch::torch_cuda for target torch_cuda")
  add_library(Torch::torch_cuda ALIAS torch_cuda)
endif()

find_package(Python3 COMPONENTS Interpreter Development REQUIRED)
  execute_process(
    COMMAND "${Python3_EXECUTABLE}" -c "import torch; print(torch.utils.cmake_prefix_path)"
    OUTPUT_VARIABLE TORCH_CMAKE_PREFIX
    OUTPUT_STRIP_TRAILING_WHITESPACE
  )
  if(TORCH_CMAKE_PREFIX)
    list(APPEND CMAKE_PREFIX_PATH ${TORCH_CMAKE_PREFIX})
    # 重新查找以更新变量
    find_package(Torch REQUIRED)
  endif()

# Allow users to point CMake at a CUTLASS checkout. The directory should
# contain an `include/` folder with both `cute/` and `cutlass/` subfolders.
set(CUTLASS_DIR "" CACHE PATH "Path to CUTLASS repository root (with include/)")

# Fall back to environment variable if cache variable not set
if(NOT CUTLASS_DIR AND DEFINED ENV{CUTLASS_DIR})
  set(CUTLASS_DIR $ENV{CUTLASS_DIR})
endif()

# Try a few common local paths if still unset
if(NOT CUTLASS_DIR)
  foreach(CANDIDATE
          ${CMAKE_SOURCE_DIR}/third_party/cutlass
          ${CMAKE_SOURCE_DIR}/extern/cutlass
          ${CMAKE_SOURCE_DIR}/external/cutlass
          ${CMAKE_SOURCE_DIR}/deps/cutlass)
    if(EXISTS ${CANDIDATE}/include/cute/tensor.hpp)
      set(CUTLASS_DIR ${CANDIDATE})
      break()
    endif()
  endforeach()
endif()

if(CUTLASS_DIR)
  message(STATUS "Using CUTLASS from: ${CUTLASS_DIR}")
else()
  message(WARNING "CUTLASS not found. Set -DCUTLASS_DIR=/path/to/cutlass (must contain include/cute). Build may fail.")
endif()

add_subdirectory(01-minimal-gemm)
add_subdirectory(cute_api)
