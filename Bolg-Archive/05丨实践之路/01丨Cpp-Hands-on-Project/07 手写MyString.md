# 07 手写MyString
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[点击返回目录](https://www.cnblogs.com/abnb250/articles/23512533)  

<br>

<br>

## 前情提要
这一节手写一个 MyString 类，  
覆盖 RAII 构造析构、拷贝/移动语义、c_str/data/下标、at 边界检查、+=/+ 拼接、==/!=/< 比较以及 clear。  
注释版主要围绕"空指针兜底"展开：  
移动之后 data_ 变成 nullptr，c_str 要返回静态空串而不是直接返回 data_；  
operator+= 里不能直接 strlen(nullptr)/strcpy(dst, nullptr)；  
operator==/operator< 传给 strcmp 之前也要做非空判断；clear 写 data_[0] 之前要确认 data_ 不为空。  

<br>

<br>

## 代码（加注释版）
注释解释了两种场景：  
1. 代码有反直觉痕迹，自然写法就在旁边。
2. 代码看起来自然，但背后有隐形决策。

<br>

**MyString.h**  
```cpp
#ifndef MYSTRING_H
#define MYSTRING_H

#include <iostream>
#include <iomanip>
#include <utility>
#include <algorithm>
#include <stdexcept>

class MyString{
private:
    char* data_;
    std::size_t size_;
public:
    MyString() : data_(new char[1]{'\0'}), size_(0) {}
    MyString(const char* s);
    MyString(const char* s, int len);
    ~MyString();

    MyString(const MyString& other);
    MyString& operator=(const MyString& other);
    MyString(MyString&& other) noexcept;
    MyString& operator=(MyString&& other) noexcept;

    void Swap(MyString& other) noexcept;
    std::size_t size() const;
    bool empty() const;
    const char* c_str() const noexcept;
    const char* data() const noexcept;

    const char& operator[](std::size_t index) const;
    char& operator[](std::size_t index);
    char at(std::size_t index) const;
    char& at(std::size_t index);

    MyString& operator+=(const char* other) ;
    MyString& operator+=(const MyString& other);
    MyString operator+(const char* other) const;
    MyString operator+(const MyString& other) const;

    bool operator==(const MyString& other) const;
    bool operator!=(const MyString& other) const;
    bool operator<(const MyString& other) const;
    
    void clear();
};

#endif
```

<br>

**MyString.cpp**
```cpp
#include "MyString.h"
#include <utility>
#include <cstring>

MyString::MyString(const char *s) {
    if(s == nullptr){
        data_ = new char[1]{'\0'};
        size_ = 0;
        return;
    }
    std::size_t len = std::strlen(s);
    char* tmp = new char[len+1];
    for(std::size_t i = 0; i < len; i++){
        tmp[i] = s[i];
    }
    tmp[len] = '\0';
    data_ = tmp;
    size_ = len;
}

MyString::MyString(const char *s, int len) {
    if(len < 0 ) throw std::invalid_argument("negative length");
    if(!s && len > 0) throw std::invalid_argument("nullptr pointer");
    std::size_t l = static_cast<std::size_t>(len);
    char * tmp = new char[l+1];
    for(std::size_t i = 0; i < l; i++){
        tmp[i] = s[i];
    }
    tmp[l] = '\0';
    data_ = tmp;
    size_ = l;
}

MyString::~MyString() {
    delete[] data_;
}

MyString::MyString(const MyString &other) : 
    data_(nullptr), 
    size_(other.size_) {
    char* tmp = new char[size_ +1];
    for(std::size_t i = 0; i < size_; i++){
        tmp[i] = other.data_[i];
    }
    tmp[size_] = '\0';
    data_ = tmp;
}

MyString& MyString::operator=(const MyString &other) {
    if(this == &other){
        return *this;
    }
    // 不能先 delete[] data_ 再复制 other，复制中途抛异常时原字符串已丢
    MyString tmp(other);
    Swap(tmp);
    return *this;
}

MyString::MyString(MyString &&other) noexcept
    : data_(other.data_), 
    size_(other.size_) {
    // 不能只复制指针不置空 other，other 析构会 delete[] 同一块内存
    other.data_ = nullptr;
    other.size_ = 0;
    }

MyString& MyString::operator=(MyString &&other) noexcept {
    if(this == &other){
        return *this;
    }
    delete[] data_;
    size_ = other.size_;
    data_ = other.data_;
    other.data_ = nullptr;
    other.size_ = 0;
    return *this;
}

void MyString::Swap(MyString& other) noexcept {
    std::swap(data_, other.data_);
    std::swap(size_, other.size_);
}

std::size_t MyString::size() const {
    return size_;
}

bool MyString::empty() const {
    return size_ == 0;
}

const char * MyString::c_str() const noexcept{
    // 不能直接 return data_，移动后 data_ 为 nullptr，调用方按 C 串使用会解引用空指针
    static const char emp[1]{'\0'};
    return data_ ? data_ : emp;
}

const char * MyString::data() const noexcept {
    return c_str();
}

const char& MyString::operator[](std::size_t index) const {
    if(index >= size_ ){
        throw std::out_of_range("out of range");
    }
    return data_[index];
}

char& MyString::operator[](std::size_t index) {
    if(index >= size_ ){
        throw std::out_of_range("out of range");
    }
    return data_[index];
}

char MyString::at(std::size_t index) const {
    if(index >= size_){
        throw std::out_of_range("out of range");
    }
    return data_[index];
}

char& MyString::at(std::size_t index){
    if(index >= size_ ){
        throw std::out_of_range("out of range");
    }
    return data_[index];
}

MyString& MyString::operator+=(const char *other) {
    // 不能去掉 nullptr 检查，std::strlen(nullptr) 是 UB
    if(other == nullptr) return *this;
    std::size_t len = std::strlen(other);
    char* tmp = new char[size_+len + 1];
    // 不能直接 strcpy(tmp, data_)，移动后 data_ 为 nullptr
    std::strcpy(tmp, data_ ? data_ : "");
    std::strcpy(tmp + size_, other);
    delete[] data_;
    data_ = tmp;
    size_ += len;
    return *this;
}

MyString& MyString::operator+=(const MyString &other) {
    char* tmp = new char[size_ + other.size_ + 1];
    std::strcpy(tmp, data_ ? data_ : "");
    // 不能直接 strcpy(tmp + size_, other.data_)，other 可能是已移动对象，data_ 为 nullptr
    std::strcpy(tmp + size_, other.data_ ? other.data_ : "");
    delete[] data_;
    data_ = tmp;
    size_ += other.size_;
    return *this;
}

MyString MyString::operator+(const char *other) const {
    MyString nt(*this);
    nt += other;
    return nt;
}

MyString MyString::operator+(const MyString &other) const {
    MyString nt(*this);
    nt += other;
    return nt;
}

bool MyString::operator==(const MyString &other) const {
    // 不能直接 strcmp(data_, other.data_)，移动后 data_ 可能为 nullptr
    const char* a = data_ ? data_ : "";
    const char* b = other.data_ ? other.data_ : "";
    return std::strcmp(a, b) == 0;
}

bool MyString::operator!=(const MyString &other) const {
    return !(*this == other);
}

bool MyString::operator<(const MyString &other) const {
    // 同 operator==：不能直接 strcmp(data_, other.data_)，移动后可能为 nullptr
    const char* a = data_ ? data_ : "";
    const char* b = other.data_ ? other.data_ : "";
    return std::strcmp(a, b) < 0;
}

void MyString::clear() {
    size_ = 0;
    // 不能去掉 if(data_)，移动后 data_ 为 nullptr，写 data_[0] 是空指针解引用
    if(data_){
        data_[0] = '\0';
    }
}
```

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>
