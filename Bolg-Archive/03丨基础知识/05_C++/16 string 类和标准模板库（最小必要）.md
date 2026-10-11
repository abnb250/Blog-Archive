# 16 string 类和标准模板库（最小必要）
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[回到主目录点这里](https://www.cnblogs.com/abnb250/articles/23230010)  

<br>

<br>

## 一句话概括
***
**一句话**：`std::string` 替代 `char[]`，STL 容器（`vector`、`map`）替代手工数组，是现代 C++ 项目的标配。

<br>

<br>

## 核心概念
***
### std::string 常用操作
```cpp
#include <string>

std::string s = "hello";
std::string s2("world");
std::string s3(5, 'a');      // "aaaaa"
std::string s4 = s + " " + s2; // 拼接

// 长度
s.size();   // 5
s.empty();  // false

// 访问
s[0];           // 'h'
s.at(0);        // 'h'，越界抛异常
s.front();      // 'h'
s.back();       // 'o'

// 查找
s.find("ll");      // 2
s.rfind("l");      // 3
s.find("xyz");     // std::string::npos

// 子串
s.substr(1, 3);    // "ell"

// 修改
s.append("!");       // "hello!"
s.insert(0, "say "); // "say hello!"
s.erase(0, 4);       // "hello!"
s.replace(0, 5, "hi"); // "hi!"

// 遍历
for (char c : s) {
    std::cout << c;
}
```

<br>

### vector 常用操作
```cpp
#include <vector>

std::vector<int> v = {1, 2, 3};
v.push_back(4);        // {1,2,3,4}
v.pop_back();          // {1,2,3}
v.size();              // 3
v.empty();             // false
v[0];                  // 1
v.at(0);               // 1，越界抛异常
v.front();             // 1
v.back();              // 3

// 遍历
for (int x : v) {
    std::cout << x << " ";
}
for (auto it = v.begin(); it != v.end(); ++it) {
    std::cout << *it << " ";
}

// 排序
#include <algorithm>
std::sort(v.begin(), v.end());

// 查找
auto it = std::find(v.begin(), v.end(), 2);
if (it != v.end()) {
    std::cout << "found at " << (it - v.begin()) << std::endl;
}
```

<br>

### map 常用操作
```cpp
#include <map>

std::map<std::string, int> m;
m["apple"] = 1;              // 插入或修改
m.insert({"banana", 2});     // 插入
m.size();                    // 2
m.empty();                   // false

// 查找
auto it = m.find("apple");
if (it != m.end()) {
    std::cout << it->first << ": " << it->second << std::endl;
}

// 遍历
for (const auto& pair : m) {
    std::cout << pair.first << ": " << pair.second << std::endl;
}

// 删除
m.erase("apple");
```

<br>

### unordered_map
- 哈希表实现，查找 O(1)，不保持顺序。
- 用法和 `map` 类似，但无序。
```cpp
#include <unordered_map>
std::unordered_map<std::string, int> um;
```

<br>

### 迭代器
- 像指针一样遍历容器。
- `begin()` 指向第一个元素，`end()` 指向最后一个元素之后。
- `*it` 解引用，`++it` 前进。
```cpp
for (auto it = v.begin(); it != v.end(); ++it) {
    std::cout << *it << " ";
}
```

<br>

### lambda 基础
```cpp
std::sort(v.begin(), v.end(), [](int a, int b) {
    return a > b; // 降序
});
```

<br>

<br>

## 最小代码
***
```cpp
#include <iostream>
#include <string>
#include <vector>
#include <map>
#include <algorithm>

int main() {
    // string
    std::string s = "hello world";
    std::cout << s.find("world") << std::endl;
    std::cout << s.substr(0, 5) << std::endl;

    // vector
    std::vector<int> v = {3, 1, 4, 1, 5};
    std::sort(v.begin(), v.end());
    for (int x : v) std::cout << x << " ";
    std::cout << std::endl;

    // map
    std::map<std::string, int> m;
    m["one"] = 1;
    m["two"] = 2;
    for (const auto& p : m) {
        std::cout << p.first << "=" << p.second << " ";
    }
    std::cout << std::endl;

    return 0;
}
```

<br>

<br>

## 易错点
***
- `s.find()` 返回 `npos` 时不要直接使用。
- `vector` 的 `[]` 不检查越界，`at()` 会抛异常。
- `map` 用 `[]` 访问不存在的键会插入默认值。
- 迭代器失效：`push_back` 可能使 `vector` 迭代器失效。

<br>

<br>

## 项目用途
***
- `std::string` 处理所有文本。
- `vector` 替代动态数组。
- `map` / `unordered_map` 做字典、缓存、计数。
- lambda 用于排序、查找等算法。

<br>

<br>

## 自测
***
1. `string::find` 找不到返回什么？
2. `vector` 的 `[]` 和 `at()` 区别？
3. `map` 和 `unordered_map` 的区别？
4. 范围 `for` 遍历时如何修改元素？

<br>

<br>

<br>

<br>

<br>

<br>

<br>

<br>

