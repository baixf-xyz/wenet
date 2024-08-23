cmake_minimum_required(VERSION 3.4.1)
set(TARGET wenet)
project(${TARGET} CXX)
set(CMAKE_CXX_STANDARD 14)
include(ExternalProject)
option(TORCH "whether to build with Torch" ON)
option(ONNX "whether to build with ONNX" OFF)
option(ITN "whether to use WeTextProcessing" ON)
set(CMAKE_VERBOSE_MAKEFILE on)
set(build_DIR ${CMAKE_SOURCE_DIR}/../../../build)
list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)
string(REPLACE "-Wl,--exclude-libs,libgcc_real.a" "" CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS}")

# PyTorch 配置
include(libtorch)  # 确保libtorch.cmake文件中配置了正确的路径
if(NOT DEFINED PYTORCH_LIBRARY)
  message(FATAL_ERROR "PYTORCH_LIBRARY not found. Please check libtorch.cmake")
endif()

# FBJNI 配置
set(FBJNI_DIR ${build_DIR}/fbjni)  # 设置 FBJNI 库的路径
find_library(FBJNI_LIBRARY fbjni PATHS ${FBJNI_DIR}/lib NO_CMAKE_FIND_ROOT_PATH)
if(NOT FBJNI_LIBRARY)
  message(FATAL_ERROR "FBJNI_LIBRARY not found. Please check the path: ${FBJNI_DIR}")
endif()

include(openfst)
include(wetextprocessing)

include_directories(
  ${CMAKE_SOURCE_DIR}
  ${CMAKE_SOURCE_DIR}/kaldi
  ${build_DIR}/pytorch_android-1.13.0.aar/headers  # PyTorch 头文件路径
  ${FBJNI_DIR}/include  # FBJNI 头文件路径
)

add_subdirectory(utils)
add_subdirectory(frontend)
add_subdirectory(post_processor)
add_subdirectory(kaldi)  # kaldi: wfst based decoder
add_subdirectory(decoder)

add_dependencies(post_processor wetextprocessing)

link_libraries(frontend decoder android ${FBJNI_LIBRARY})

add_library(${TARGET} SHARED wenet.cc)

add_executable(decoder_main bin/decoder_main.cc)
target_link_libraries(decoder_main PUBLIC libc++_shared.so ${TORCH_LIBRARIES} ${FBJNI_LIBRARY})

# 打印调试信息
message(STATUS "PYTORCH_LIBRARY: ${PYTORCH_LIBRARY}")
message(STATUS "FBJNI_LIBRARY: ${FBJNI_LIBRARY}")
