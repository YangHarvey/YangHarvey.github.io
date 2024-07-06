---
title: 内存分配器buffer_allocator(cpp)
date: 2024-04-13 20:45:60
tags: [rdma, buffer allocator]
description: 2024-07-04-内存分配器buffer_allocator
layout: post
---

当我们使用RDMA编程时，通常需要事先申请一大块内存，注册到RDMA中，然后手动管理这部分内存。下面是一个简单的顺序内存分配管理器的实现，申请从write_offset开始申请，释放从read_offset开始释放。

```cpp
#include <iostream>
#include <mutex>
#include <cassert>

class BufferAlloc {
public:
    BufferAlloc(char *_start, char *_end) :
        start_(_start), end_(_end), max_size_(end_ - start_), 
        curr_write_offset_(0), curr_read_offset_(0), free_space_(max_size_){}

        inline     std::pair<bool, char*> Alloc(size_t size) {
        std::scoped_lock<std::mutex> lock(latch_);
        /*
            size == 0, return false;
        */
        if(size == 0) {
            return {false, nullptr};
        }

        if(size > max_size_) {
            std::cerr << "[Error]: exceed max size! [Location]: " << __FILE__  << ":" << __LINE__ << std::endl;
            return {false, nullptr};
        }

        /*
            back curr_write_offset and free space
        */
        size_t curr_write_offset_bak = curr_write_offset_;
        size_t free_space_bak = free_space_;

        /*
            找到起始位置
        */ 
        if(curr_write_offset_ + size > max_size_) {
            if(max_size_ - curr_write_offset_ > free_space_) {
                return {false, nullptr};
            }
            free_space_ -= (max_size_ - curr_write_offset_);
            curr_write_offset_ = 0;
        }

        /*
            free space < size, return false, no enough space
        */
        if(free_space_ < size) {
            curr_write_offset_  = curr_write_offset_bak;
            free_space_         = free_space_bak;
            return {false, nullptr};
        }

        /*
            当前写入位置合适
        */
        size_t write_offset = curr_write_offset_;
        free_space_ -= size;
        curr_write_offset_ = (curr_write_offset_ + size) % max_size_;  

        return std::make_pair(true, start_ + write_offset);
    }

    inline void Free(size_t size) {
        std::scoped_lock<std::mutex> lock(latch_);
        /*
            检查异常情况
        */
        if(size == 0) {
            return ;
        }
        if(size > max_size_) {
            std::cerr << "[Error]: exceed max size! [Location]: " << __FILE__  << ":" << __LINE__ << std::endl;
            return ;
        }

        /*
            备份
        */
        size_t curr_read_offset_bak = curr_read_offset_;
        size_t free_space_bak       = free_space_;

        /*
            获取当前读取位置
        */
        if(curr_read_offset_ + size > max_size_) {
            free_space_ += (max_size_ - curr_read_offset_);
            curr_read_offset_ = 0;
        }

        /*
            free失败
        */
        if(free_space_ + size > max_size_) {
            curr_read_offset_ = curr_read_offset_bak;
            free_space_ = free_space_bak;
            return ;
        }
        
        /*
            free成功
        */
        free_space_ += size;
        curr_read_offset_ = (curr_read_offset_ + size) % max_size_;

        if(curr_read_offset_ == curr_write_offset_) {
            assert(free_space_ == max_size_);
            curr_read_offset_ = 0;
            curr_write_offset_ = 0;
        }

        return ;
    }

    inline size_t getFreeSpace() {
        return free_space_;
    }

    inline size_t getUsedSpace() {
        return max_size_ - free_space_;
    }

    inline size_t get_curr_write_offset() {
        return curr_write_offset_;
    }
    
    inline size_t get_curr_read_offset() {
        return curr_read_offset_;
    }

private:
    char    *start_;
    char    *end_;

    size_t  max_size_;
    size_t  curr_write_offset_;
    size_t  curr_read_offset_;

    size_t  free_space_;

    std::mutex latch_;
};
```