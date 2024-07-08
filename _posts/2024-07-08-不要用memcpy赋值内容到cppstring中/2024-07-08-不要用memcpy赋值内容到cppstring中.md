---
title: 不要用memcpy赋值内容到std::string中
date: 2024-07-08 20:45:60
tags: [debug, cpp]
description: 不要用memcpy赋值内容到std::string中
layout: post
---

最近写代码的时候，用上了`copilot`，但是我发现有一天它给了我一个有明显错误的代码。
我要实现的功能是从字节流中反序列化出我的数据结构，其中有一部分是需要反序列化到`std::string`中，`copilot`给我的代码是：
```cpp
// 我需要从char *data_recv中
void deserialize(char *data_recv, int len) {
    std::string sql;
    memcpy(sql.data(), data_recv, len);
}
```
这一段代码有很明显的问题，使用`memcpy`给`std::string`赋值时，没有调用`std::string`的构造函数，直接向`std::string`底层的字符串地址`char *`写入数据。

不能全信`copilot`给的代码啊