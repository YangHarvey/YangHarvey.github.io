---
title: fsanitize使用
date: 2024-07-11 20:45:60
tags: [cpp, fsanitize]
description: 2024-07-11-fsanitize使用
layout: post
---

## fsanitize使用

fsanitize可以很好的帮我们捕捉程序中的内存错误，例如段错误或者double free等等，但是我发现我之前在cpp项目中使用的方式不太正确，通常我是在cmake项目中传递`fsanitize=address`的标志给编译器

```cmake
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address")
```

但是这种情况下`lasan`库只传递给了编译器，没有传递给链接器(linker)，所以会某些情况下会报下面的错误（但是我很多时候没有传递给链接器也不会报错，这里我不太明白为什么）:

```shell
undefined reference to '_asan_init_v4'
```

正确的使用方式应该是下面这样:

```cmake
add_compile_options(-fsanitize=address)
add_link_options(-fsanitize=address)
```
上面的命令会为项目中编译出来的所有程序都添加`fsanitize=address`的标志（包括编译器和链接器）。如果不想要这样做，例如我只想给某一个程序编译时添加`fsanitize=address`，那么可以用下面的方式

```cmake
target_compile_options(lib/program PRIVATE -fsanitize=address)
target_link_options(lib/program PRIVATE -fsanitize=address)
```

## fsanitize-recover使用

`fsanitize=address`选项会在检测到内存错误时abort程序，但是很多时候我们需要让程序检测到内存错误时继续运行而不是直接崩溃，那么我们可以用`fsanitize-recover=address`，然后引入环境变量`ASAN_OPTIONS=halt_on_error=0`，用法如下:

```cmake 
add_compile_options(-fsanitize=address -fsanitize-recover=address)
add_link_options(-fsanitize=address -fsanitize-recover=address)
```
然后引入环境变量`export ASAN_OPTIONS=halt_on_error=0`，再运行程序，就不会再遇到内存错误时立刻崩溃了

如果感觉调试信息还不够，还可以加`-fno-omit-frame-pointer`选项，这个选项会禁用优化，保留帧指针，这有助于在发生错误时提供更准确的调用栈信息。

## 总结

上述选项都会对性能有影响，因此最好只在调试时使用




