cmake_minimum_required(VERSION 3.25)
project(deepstream-test5 LANGUAGES C CXX) #project(deepstream-test5 LANGUAGES CUDA C CXX)
set(CMAKE_CXX_STANDARD 14)
set (CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(CMAKE_BUILD_TYPE Debug)
find_package(PkgConfig REQUIRED)
pkg_check_modules(PKGS REQUIRED uuid x11 gstreamer-video-1.0 json-glib-1.0 gstreamer-1.0 gstreamer-rtsp-server-1.0 yaml-cpp)
find_package(CUDAToolkit)
message("CUDA_VER is ${CUDAToolkit_VERSION_MAJOR}.${CUDAToolkit_VERSION_MINOR}")

if(NOT DEFINED CMAKE_CUDA_STANDARD)
    set(CMAKE_CUDA_STANDARD 11)
    set(CMAKE_CUDA_STANDARD_REQUIRED True)
endif()

SET(DEEPSTREAMAPP
    /opt/nvidia/deepstream/deepstream/sources/apps
)

include(ExternalProject)
ExternalProject_Add(deepstreamapp  
    SOURCE_DIR ${DEEPSTREAMAPP}/sample_apps/deepstream-app 
    CONFIGURE_COMMAND ""
    INSTALL_COMMAND ""
    DOWNLOAD_COMMAND ""
    BUILD_COMMAND export CUDA_VER=${CUDAToolkit_VERSION_MAJOR}.${CUDAToolkit_VERSION_MINOR} && make
    BUILD_IN_SOURCE true
)

file(GLOB_RECURSE  DEEPSTREAMAPPCOMMONOBJ "/opt/nvidia/deepstream/deepstream/sources/apps/apps-common/src/*.o")
file(GLOB_RECURSE  DEEPSTREAMAPPOBJ "/opt/nvidia/deepstream/deepstream/sources/apps/sample_apps/deepstream-app/*.o")

list(REMOVE_ITEM DEEPSTREAMAPPOBJ "/opt/nvidia/deepstream/deepstream/sources/apps/sample_apps/deepstream-app/deepstream_app_main.o")

message(DEEPSTREAMAPPCOMMONOBJ="${DEEPSTREAMAPPCOMMONOBJ}")
message(DEEPSTREAMAPPOBJ="${DEEPSTREAMAPPOBJ}")


SET(OBJS
    ${DEEPSTREAMAPPCOMMONOBJ}
    ${DEEPSTREAMAPPOBJ}
    /opt/nvidia/deepstream/deepstream/sources/apps/apps-common/src/deepstream_config_file_parser.o
    )   

SET_SOURCE_FILES_PROPERTIES(
  ${OBJS}
  PROPERTIES
  EXTERNAL_OBJECT true
  GENERATED true
)


add_executable(${PROJECT_NAME}
    ${OBJS}
    deepstream_utc.c
    deepstream_test5_app_main.c
)

target_include_directories(${PROJECT_NAME} PRIVATE
    ${PKGS_INCLUDE_DIRS}
    ${CMAKE_CUDA_TOOLKIT_INCLUDE_DIRECTORIES}
    /opt/nvidia/deepstream/deepstream/sources/apps/apps-common/includes
    /opt/nvidia/deepstream/deepstream/sources/includes
    /opt/nvidia/deepstream/deepstream/sources/apps/sample_apps/deepstream-app
    ${CMAKE_CURRENT_SOURCE_DIR}/cparse
    ${CMAKE_CURRENT_SOURCE_DIR}
)

target_link_libraries(${PROJECT_NAME} PRIVATE
    ${PKGS_LIBRARIES}
    CUDA::cuda_driver
    nvds_utils nvdsgst_helper m nvdsgst_meta nvds_meta stdc++
    nvbufsurface nvbufsurftransform nvds_batch_jpegenc uuid jpeg png nvds_yml_parser
    nvdsgst_smartrecord nvds_msgbroker nvdsgst_customhelper
    pthread 
    PRIVATE CUDA::cudart
)

target_link_directories(${PROJECT_NAME} PRIVATE
    /opt/nvidia/deepstream/deepstream/lib/
)

target_link_options(${PROJECT_NAME} PRIVATE -Wl,-rpath,/opt/nvidia/deepstream/deepstream/lib/)