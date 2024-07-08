# sylar学习13：关于工程

在拿到这样的代码之后，自己确实是有些手足无措的，不知道该如何下手。去想到底有没有主函数，或者在想其他乱七八糟的东西。后面发现是自己对从一堆文件到工程这个过程不是很熟悉，所以才出现这样的问题。于是在这里总结一下，也算是对sylar项目做一个整体的回顾。

针对这个项目，要做的是

```shell
cd build
cmake ..
make && make install
```

第一行都了解，表示的意思是创建一个文件。

### cmake

cmake是一个工具，自动生成makefile文件的工具，在多个平台都可以使用。

> 允许开发者编写一种平台无关的 **CMakeList.txt** 文件来定制整个编译流程。
>
> 然后再根据目标用户的平台进一步生成所需的本地化 Makefile 和工程文件，如 Unix 的 Makefile 或 Windows 的 Visual Studio 工程。从而做到“Write once, run everywhere”。

**关于CMakeList.txt：**

```cmake
# 表示cmake的最低版本
cmake_minimum_required (VERSION 2.6)

# 表示目前编译的项目
project (day07)

# 表示当前编译使用c++14版本来编译程序
set(CMAKE_CXX_STANDARD 14)

# 表示项目的执行程序， 括号中的day07 表示最终生成的执行程序名称，  后面的表示程序的源码文件
add_executable(day07 main.cpp stu.cpp)
```



### make&&make install

make，常指一条计算机指令 ，可以从一个名为Makefile的文件中获得如何构建程序的依赖关系。通常项目的编译规则就定义在makrfile 里面，比如： 规定先编译哪些文件，后编译哪些文件… 当编写一个程序时，可以为它编写一个makefile文件，不过在windows下的很多IDE 工具，内部都集成了这些编译的工作，只需要点击某一个按钮，一切就完成了。换算到手动操作的话，就需要编写一个makefile文件，然后使用make命令执行编译和后续的安装。

主要是构建依赖关系，然后一键编译。

![img](https://img-blog.csdnimg.cn/img_convert/3a751c0ee666ffc0188ad9fc0eff2e11.png)

### 导入第三方依赖

> 在C/C++中，项目最终都会分成两个部分内容，一个是 头文件( .h ) 一部分是源文件( .cpp ) 。 如果要编写好的功能给其他程序使用，通常会把源文件打包形成一个动态链接库文件( .so .a ） 文件 。 值得注意的是，头文件一般不会打包到链接库中，因为头文件仅仅只是声明而已。 链接库也增加了代码的重用性、提高编码的效率，也可看看成是对源码的一种保护。

这个自己其实见过，在调试项目的时候，会发现头文件都在，但是文件具体的内容都不在了。

关于库，库是写好的现有的，成熟的，可以复用的代码。现实中每个程序都要依赖很多基础的底层库，不可能每个人的代码都从零开始，因此库的存在意义非同寻常 。 本质上来说库是一种可执行代码的二进制形式，可以被操作系统载入内存执行。库有两种：静态库（.a） 和 动态库（.so）。



下面学习一下cmakelist.txt文件：

```cmake
cmake_minimum_required(VERSION 3.0) #版本
project(sylar-from-scratch) #指定工程名称

include (cmake/utils.cmake) #
#一般大写变量为CMake预定义的全局内建变量，直接用set修改值，有点像define
set(CMAKE_VERBOSE_MAKEFILE ON)

# 指定编译选项 $ENV{}用于获取环境变量，这里获取C++编译器的
# -Wall 告诉编译器 编译后显示所有警告。
# -Werror 告诉编译器视所有警告为错误，出现任何警告立即放弃编译
# -ggdb 使编译器生成gdb专用的更为丰富的调试信息。
set(CMAKE_CXX_FLAGS "$ENV{CXXFLAGS} -std=c++11 -O0 -ggdb -Wall -Werror")

# -rdynamic: 将所有符号都加入到符号表中，便于使用dlopen或者backtrace追踪到符号
# -fPIC: 生成位置无关的代码，便于动态链接
# 感觉这个和上面那个应该是可以叠加的一个状态，在上面使用的参数基础上再添加点参数
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -rdynamic -fPIC")

# -Wno-unused-function: 不要警告未使用函数
# -Wno-builtin-macro-redefined: 不要警告内置宏重定义，用于重定义内置的__FILE__宏
# -Wno-deprecated: 不要警告过时的特性
# -Wno-deprecated-declarations: 不要警告使用带deprecated属性的变量，类型，函数
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-unused-function -Wno-builtin-macro-redefined -Wno-deprecated -Wno-deprecated-declarations")
# 将包含目录添加到构建中。
include_directories(.)
# 条件编译，可以简单理解为将BUILD_TEST设为ON
option(BUILD_TEST "ON for complile test" ON)
# 使用第三方开源库的场景，需要找到，这里REQUIRED参数表示如果找不到就不继续进行了
find_package(Boost REQUIRED) 
if(Boost_FOUND) # 如果找到，然后添加
    include_directories(${Boost_INCLUDE_DIRS})
endif()

set(LIB_SRC
    sylar/log.cpp
    sylar/util.cpp
    sylar/mutex.cc
    sylar/env.cc
    sylar/config.cc
    sylar/thread.cc
    sylar/fiber.cc
    sylar/scheduler.cc
    sylar/iomanager.cc
    sylar/timer.cc
    sylar/fd_manager.cc
    sylar/hook.cc
    sylar/address.cc 
    sylar/socket.cc 
    sylar/bytearray.cc 
    sylar/tcp_server.cc 
    sylar/http/http-parser/http_parser.c 
    sylar/http/http.cc
    sylar/http/http_parser.cc 
    sylar/stream.cc 
    sylar/streams/socket_stream.cc
    sylar/http/http_session.cc 
    sylar/http/servlet.cc
    sylar/http/http_server.cc 
    sylar/uri.cc 
    sylar/http/http_connection.cc 
    sylar/daemon.cc 
    )
# 生成库，sylar为名字，SHARED表示是动态的，${LIB_SRC}表示构造库文件所需的源码文件，和上面的set呼应上了，最终动态库的名字会是libsylar。
add_library(sylar SHARED ${LIB_SRC})
# 这个引用了上面那个cmake文件：utils.cmake，是为了重定义目标源码的__FILE__宏，使用相对路径的形式，避免暴露敏感信息。
force_redefine_file_macro_for_sources(sylar)

set(LIBS
    sylar
    pthread
    dl
    yaml-cpp
)
# 这个也是将LIBS等价于后面的那些文件。
if(BUILD_TEST) # 这个和前面的option是对应的。
sylar_add_executable(test_log "tests/test_log.cpp" sylar "${LIBS}")# 这个就是根据test_log.cpp，${LIBS}(即上面等级的文件)生成名为test_log的可执行文件。
sylar_add_executable(test_util "tests/test_util.cpp" sylar "${LIBS}")
sylar_add_executable(test_env "tests/test_env.cc" sylar "${LIBS}")
sylar_add_executable(test_config "tests/test_config.cc" sylar "${LIBS}")
sylar_add_executable(test_thread "tests/test_thread.cc" sylar "${LIBS}")
sylar_add_executable(test_fiber "tests/test_fiber.cc" sylar "${LIBS}")
sylar_add_executable(test_fiber2 "tests/test_fiber2.cc" sylar "${LIBS}")
sylar_add_executable(test_scheduler "tests/test_scheduler.cc" sylar "${LIBS}")
sylar_add_executable(test_iomanager "tests/test_iomanager.cc" sylar "${LIBS}")
sylar_add_executable(test_timer "tests/test_timer.cc" sylar "${LIBS}")
sylar_add_executable(test_hook "tests/test_hook.cc" sylar "${LIBS}")
sylar_add_executable(test_address "tests/test_address.cc" sylar "${LIBS}")
sylar_add_executable(test_socket_tcp_server "tests/test_socket_tcp_server.cc" sylar "${LIBS}")
sylar_add_executable(test_socket_tcp_client "tests/test_socket_tcp_client.cc" sylar "${LIBS}")
sylar_add_executable(test_bytearray "tests/test_bytearray.cc" sylar "${LIBS}")
sylar_add_executable(test_tcp_server "tests/test_tcp_server.cc" sylar "${LIBS}")
sylar_add_executable(test_http "tests/test_http.cc" sylar "${LIBS}")
sylar_add_executable(test_http_parser "tests/test_http_parser.cc" sylar "${LIBS}")
sylar_add_executable(test_http_server "tests/test_http_server.cc" sylar "${LIBS}")
sylar_add_executable(test_uri "tests/test_uri.cc" sylar "${LIBS}")
sylar_add_executable(test_http_connection "tests/test_http_connection.cc" sylar "${LIBS}")
sylar_add_executable(test_daemon "tests/test_daemon.cc" sylar "${LIBS}")
endif()
# 设置可执行文件的输出目录
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)
# 设置库文件的输出目录
set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)
```

另外关于`sylar_add_executable`其实也是一种封装，在另一个cmake文件`utils.cmake`里可以看到：

```cmake
# 重定义目标源码的__FILE__宏，使用相对路径的形式，避免暴露敏感信息

# This function will overwrite the standard predefined macro "__FILE__".
# "__FILE__" expands to the name of the current input file, but cmake
# input the absolute path of source file, any code using the macro 
# would expose sensitive information, such as MORDOR_THROW_EXCEPTION(x),
# so we'd better overwirte it with filename.
function(force_redefine_file_macro_for_sources targetname)
    get_target_property(source_files "${targetname}" SOURCES)
    foreach(sourcefile ${source_files})
        # Get source file's current list of compile definitions.
        get_property(defs SOURCE "${sourcefile}"
            PROPERTY COMPILE_DEFINITIONS)
        # Get the relative path of the source file in project directory
        get_filename_component(filepath "${sourcefile}" ABSOLUTE)
        string(REPLACE ${PROJECT_SOURCE_DIR}/ "" relpath ${filepath})
        list(APPEND defs "__FILE__=\"${relpath}\"")
        # Set the updated compile definitions on the source file.
        set_property(
            SOURCE "${sourcefile}"
            PROPERTY COMPILE_DEFINITIONS ${defs}
            )
    endforeach()
endfunction()

# wrapper for add_executable
function(sylar_add_executable targetname srcs depends libs)
    add_executable(${targetname} ${srcs})
    add_dependencies(${targetname} ${depends})
    force_redefine_file_macro_for_sources(${targetname})
    target_link_libraries(${targetname} ${libs})
endfunction()
```

`sylar_add_executable`函数包括以下操作。以`sylar_add_executable(test_log "tests/test_log.cpp" sylar "${LIBS}")`为例。

add_executable根据tests/test_log.cpp生成可执行文件test_log。test_log 的生成需要链接库LIBS，即sylar、pthread、dl、yaml-cpp。所以我们需要target_link_libraries(\${targetname} \${libs})。但这样编译的时候会报错，一些符号的定义找不到，针对test_log我们知道是在库sylar中，所以在上面两条中间需要加上add_dependencies(\${targetname} \${depends})。

这样就没有问题了。

关于force_redefine_file_macro_for_sources，因为上面只对tests/test_log.cpp和sylar使用了，相对应的对sylar、pthread、dl、yaml-cpp没有使用，猜测是对相对路径和绝对路径的处理。



所以其实也没有很难对吧，就是先把那些.c/.cpp/.cc文件生成一个动态库，然后再进行链接生成可执行文件。之前的那些库应该也是这样生成的。所以自己设想的像nginx那样的是不会实现的，因为没有一个main.cpp文件可以生成可执行文件。