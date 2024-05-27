# deepstream-startup
Show how to Use VSCode and CMake to make Deepstream development with C/C++ life easier

# Prequirements
* CMake > 3.25
```bash
sudo apt-get update
sudo apt-get install ca-certificates gpg wget
test -f /usr/share/doc/kitware-archive-keyring/copyright ||
wget -O - https://apt.kitware.com/keys/kitware-archive-latest.asc 2>/dev/null | gpg --dearmor - | sudo tee /usr/share/keyrings/kitware-archive-keyring.gpg >/dev/null

# For Ubuntu Jammy Jellyfish (22.04):
echo 'deb [signed-by=/usr/share/keyrings/kitware-archive-keyring.gpg] https://apt.kitware.com/ubuntu/ jammy main' | sudo tee /etc/apt/sources.list.d/kitware.list >/dev/null
sudo apt-get update

# For Ubuntu Focal Fossa (20.04):
echo 'deb [signed-by=/usr/share/keyrings/kitware-archive-keyring.gpg] https://apt.kitware.com/ubuntu/ focal main' | sudo tee /etc/apt/sources.list.d/kitware.list >/dev/null
sudo apt-get update

test -f /usr/share/doc/kitware-archive-keyring/copyright ||
sudo rm /usr/share/keyrings/kitware-archive-keyring.gpg

sudo apt-get install kitware-archive-keyring

sudo apt-get install cmake

```
* VSCode
* clangd
```bash
sudo apt-get install clangd-12
```
* package to run this example
```bash
sudo apt-get install libgstreamer-plugins-base1.0-dev libgstreamer1.0-dev \
   libgstrtspserver-1.0-dev libx11-dev libjson-glib-dev libyaml-cpp-dev
```

# Setup VSCode for development
* install VSCode extension 
CodeLLDB、clangd 、CMake 、CMake Tools
* Note: Do not install C/C++ extension pack from Microsoft

* copy deepstream-test5 to your workspace
deepstream-test5 is a complete example to modify deepstream-app，this will take the advantage of the module design of deepstream-app. You can add probe to customize the app. It have good config parser and error handling, those will save many effort.  
Following our workspace is in the copy of deepstream-test5 folder.

```bash
cp -r /opt/nvidia/deepstream/deepstream/apps/sample_apps/deepstream-test5/* /path/to/your/workspace

* create launch.json and task.json
```bash
* create launch.json and task.json for compile and debug
```

tasks.json use CMake to build  

```json
{
  "version": "2.0.0",
  "tasks": [
        {
            "type": "cmake",
            "label": "CMake: build",
            "command": "build",
            "targets": [
                "all"
            ],
            "group": "build",
            "problemMatcher": [],
            "detail": "CMake template build task"
        }
    ]
}
```

launch.json call task in task json to build


```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "type": "lldb",
            "request": "launch",
            "name": "Debug",
            "program": "${workspaceFolder}/deepstream-test5",//the path of the executable file deepstream-test5
            "args": ["-c", "configs/test5_config_file_src_infer.txt"],//arg for deepstream-test5
            "cwd": "${workspaceFolder}",//指定工作目錄
            "preLaunchTask": "CMake: build"
        }
    ]
}
```  


create settings.json for clangd to read compile_commands.json. clangd can generate code navigator.
```json
{
    "cmake.sourceDirectory": "${workspaceFolder}",
    "cmake.generator": "Unix Makefiles",
    "clangd.arguments": [
        "--compile-commands-dir=${workspaceFolder}/build",
        "--background-index",
        "-j=8",
        "--query-driver=/usr/bin/clang++",
        "--clang-tidy",
        "--clang-tidy-checks=performance-*,bugprone-*",
        "--all-scopes-completion",
        "--completion-style=detailed",
        "--function-arg-placeholders",
        "--header-insertion=iwyu",
        "--pch-storage=memory",
    ],
}
```

* Create CMAKELists.txt 
Create CMakeLists.txt under the workspace folder, CMake will help you find CUDA and nvcc, no more `CUDA_VER` env variable needed even in jetson.  
In this example，we need to compile deepstream-app first and use the object file in our project.So we use `ExternalProject_Add` to do this.

We need to change the permission for deepstream-app source folder in order to compile the deepstream-app.
```
sudo chmod -R 777 /opt/nvidia/deepstream/deepstream/sources/apps/
```

```CMakeLists
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
```
* click build button
![Screenshot from 2024-05-27 15-31-57](https://github.com/jenhaoyang/jenhaoyang.github.io/assets/7457532/4ec27faf-f33a-4177-a1d4-d38d731ca509)


* start debug and make a breakpoint
![Screenshot from 2024-05-27 15-45-50](https://github.com/jenhaoyang/jenhaoyang.github.io/assets/7457532/b59ce08f-4eba-499d-9de9-4774fb1c631e)

