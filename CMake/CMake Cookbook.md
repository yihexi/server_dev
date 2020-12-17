#### CMake中的列表型参数

定义 

```cmake
set(Foo a b c)
```

当Foo被传给一个命令时，会自动展开：

```cmake
command(${Foo}) == command(a b c)
```

如果不想展开，可以使用双引号：

```cmake 
command("${Foo}") == command("a b c")
```

使用list类型的变量，可以更优雅的写编译文件列表，比如

```cmake
add_executable(Hello Hello.c File2.c File3.c File4.c)
```

可以写成

```cmake
set (HELLO_SRCS Hello.c File2.c File3.c File4.c)

add_executable(HELLO ${HELLO_SRCS})
```

更重要的是，通过增加一个变量可以实现条件编译。

```cmake
if (WIN32) #如果是Mac就if(APPLE)
    set (HELLO_SRCS Hello.c File2.c)
else()
    set (HELLO_SRCS Hello.c File4.c)
endif()
```



#### CMake访问环境变量

```cmake
$ENV{VAR}
```

访问Window注册表需要转义 \

```cmake
[HKEY_CURRENT_USER]\\Software\\Path1\\Path2\\;key]
```



#### 为CMake选择编译器

有3种方式：

在generator中指定，在环境变量中指定，爱cache中指定。

给予Makefile的generator会寻找编译器，在下列文件中：

```
Modules/CMakeDeterminCCompiler.cmake
Modules/CMakeDeterminCXXCompiler.cmake
```

这个设置的优先级没有设置环境变量高。CC和CXX两个环境变量。

还可以在运行CMake命令的时候指定：

```
DCMAKE_CXX_COMPILER=cl
```



#### 如何查看目标的依赖

在如下四个文件中：

```
depend.make, flags.make, build.make, DependInfo.cmake
```

depend.make 保存了项目的依赖信息；flags.make保存了编译flags；DependInfo.cmake保存了编译的cpp文件和语言信息。



#### 如何指定编译静态库还是动态库？

```cmake
add_library(foo STATIC foo1.c)
```

STATIC有3种选项：STATIC， SHARED和MODULE。STATIC表示静态库，SHARED表示动态库，MODULE表示组件，是不会被链接到其它目标中的插件，但是可能会在运行时使用dlopen-系列的函数。

如果没有指定任何类型，cmake根据BUILD_SHARED_LIBS变量的值来决定是STATIC还是SHARED。



#### 如何打印CMake正在处理的文件——LISTFILE_STACK的用途





#### 

