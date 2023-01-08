## cmake 基本规则

### 预定义变量

- `PROJECT_NAME` 项目名称
- `PROJECT_SOURCE_DIR` 工程的根目录
- `PROJECT_BINARY_DIR` 执行 cmake 命令的目录
- `PROJECT_BINARY_DIR` 执行 cmake 命令的目录
- `CMAKE_CURRENT_SOURCE_DIR` 当前 CMakeLists.txt 文件所在目录
- `CMAKE_C_FLAGS` 设置 C 编译选项
- `CMAKE_CXX_FLAGS` 设置 C++ 编译选项
- `CMAKE_C_COMPILER` 设置 C 编译器
- `CMAKE_CXX_COMPILER` 设置 C++ 编译器
- `EXECUTABLE_OUTPUT_PATH` 设置编译后可执行文件目录
- `LIBRARY_OUTPUT_PATH` 设置生成的库文件目录

### 常用规则

- `cmake_minimum_required(VERSION 3.16)` 指令 cmake 版本
- `project(hello_world)` 设置工程名
- `include_directories(${PROJECT_SOURCE_DIR}/include)` 添加头文件路径
- `link_directories(${PROJECT_SOURCE_DIR}/lib)` 添加链接库的路径
- `add_subdirectory(module)`添加 module 子目录, 此目录下也要有 CMakeLists.txt 文件
- `add_executable(project1 main.c)`指定编译的可执行文件
- `add_library(lib1 SHARED library.c library.h)`指定生成的库文件，**SHARED** 是生成动态库，**STATIC** 后生成静态库 `add_compile_options()` 添加编译选项
- `target_link_libraries()`指定动态链接库
- `install()`指定 make install 的目录
- `set(XXXX YYYYYY)`用于设置和修改变量
- `${XXXX}` 使用变量

### 判断平台

```cmake
IF (WIN32)
    MESSAGE(STATUS "current platform: WIN32")
ELSEIF (ANDROID)
    MESSAGE(STATUS "current platform: ANDROID")
ELSEIF (APPLE)
    MESSAGE(STATUS "current platform: APPLE")
ELSEIF (UNIX)
    MESSAGE(STATUS "current platform: UNIX")
ELSEIF (MSVC)
    MESSAGE(STATUS "current platform: MSVC")
ELSE ()
    MESSAGE(STATUS "other platform: ${CMAKE_SYSTEM_NAME}")
ENDIF ()
```
