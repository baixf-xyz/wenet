cmake_minimum_required(VERSION 3.4.1)
set(TARGET wenet)
project(${TARGET} CXX)
set(CMAKE_CXX_STANDARD 14)
include(ExternalProject)

# Options
option(TORCH "whether to build with Torch" ON)
option(ONNX "whether to build with ONNX" OFF)
option(ITN "whether to use WeTextProcessing" ON)

# Global settings
set(CMAKE_VERBOSE_MAKEFILE ON)
set(build_DIR ${CMAKE_SOURCE_DIR}/../../../build)
list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

# Remove excluded libs
string(REPLACE "-Wl,--exclude-libs,libgcc_real.a" "" CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS}")

# PyTorch configuration
if(TORCH)
  include(libtorch)
  if(NOT DEFINED PYTORCH_LIBRARY)
    message(FATAL_ERROR "PYTORCH_LIBRARY not found. Please check libtorch.cmake")
  endif()
endif()

# FBJNI configuration
set(FBJNI_DIR ${build_DIR}/fbjni)
find_library(FBJNI_LIBRARY fbjni 
             PATHS 
               ${FBJNI_DIR}/lib 
               ${FBJNI_DIR}/jni/${ANDROID_ABI}
             NO_CMAKE_FIND_ROOT_PATH
)
if(NOT FBJNI_LIBRARY)
  message(FATAL_ERROR "FBJNI_LIBRARY not found. Searched in:\n  ${FBJNI_DIR}/lib\n  ${FBJNI_DIR}/jni/${ANDROID_ABI}\nPlease ensure FBJNI is installed correctly.")
endif()

# Other dependencies
include(openfst)
if(ITN)
  include(wetextprocessing)
endif()

# Include directories
include_directories(
  ${CMAKE_SOURCE_DIR}
  ${CMAKE_SOURCE_DIR}/kaldi
  ${build_DIR}/pytorch_android-1.13.0.aar/headers
  ${FBJNI_DIR}/include
)

# Add subdirectories
add_subdirectory(utils)
add_subdirectory(frontend)
add_subdirectory(post_processor)
add_subdirectory(kaldi)
add_subdirectory(decoder)

# Dependencies
if(ITN)
  add_dependencies(post_processor wetextprocessing)
endif()

# Link libraries
link_libraries(frontend decoder android ${FBJNI_LIBRARY})
if(TORCH)
  link_libraries(${TORCH_LIBRARIES})
endif()

# Main library
add_library(${TARGET} SHARED wenet.cc)

# Executable
add_executable(decoder_main bin/decoder_main.cc)
target_link_libraries(decoder_main 
  PUBLIC 
    libc++_shared.so
    ${FBJNI_LIBRARY}
)
if(TORCH)
  target_link_libraries(decoder_main PUBLIC ${TORCH_LIBRARIES})
endif()

# Debug information
message(STATUS "ANDROID_ABI: ${ANDROID_ABI}")
message(STATUS "FBJNI_LIBRARY: ${FBJNI_LIBRARY}")
if(TORCH)
  message(STATUS "PYTORCH_LIBRARY: ${PYTORCH_LIBRARY}")
endif()
message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")
