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

# 确保 libtorch 路径正确
if(TORCH)
    set(CMAKE_PREFIX_PATH "/path/to/libtorch")  # 修改为 libtorch 的实际路径
    find_package(Torch REQUIRED)
endif()

# 包含必要的路径
include(libtorch)  # 确保 libtorch.cmake 文件中配置了正确的路径
include(openfst)
include(wetextprocessing)

include_directories(
  ${CMAKE_SOURCE_DIR}
  ${CMAKE_SOURCE_DIR}/kaldi
  ${build_DIR}/pytorch_android-1.13.0.aar/headers  # 添加 Pytorch 头文件路径
)

add_subdirectory(utils)
add_subdirectory(frontend)
add_subdirectory(post_processor)
add_subdirectory(kaldi)  # kaldi: wfst based decoder
add_subdirectory(decoder)
add_dependencies(post_processor wetextprocessing)

link_libraries(frontend decoder android)

# 添加共享库目标
add_library(${TARGET} SHARED wenet.cc)

# 链接所需的库，包含 Torch 库
target_link_libraries(${TARGET} PRIVATE 
    frontend
    decoder
    android
    libc++_shared.so 
    ${TORCH_LIBRARIES}  # 确保链接 Torch 库
)

add_executable(decoder_main bin/decoder_main.cc)
target_link_libraries(decoder_main PUBLIC 
    libc++_shared.so 
    ${TORCH_LIBRARIES}  # 确保链接 Torch 库
)

# 强制使用 lld 链接器以避免符号冲突
target_link_options(${TARGET} PRIVATE -fuse-ld=lld)

set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--allow-multiple-definition")

# Debug information
message(STATUS "ANDROID_ABI: ${ANDROID_ABI}")
if(TORCH)
    message(STATUS "Torch Libraries: ${TORCH_LIBRARIES}")
endif()
message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")
message(STATUS "CMAKE_LIBRARY_OUTPUT_DIRECTORY: ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}")
