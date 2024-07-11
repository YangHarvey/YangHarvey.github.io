---
title: fsanitize使用
date: 2024-07-11 20:45:60
tags: [cpp, fsanitize]
description: 2024-07-11-fsanitize使用
layout: post
---

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


