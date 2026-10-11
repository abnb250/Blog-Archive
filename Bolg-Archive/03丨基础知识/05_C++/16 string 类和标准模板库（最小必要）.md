# 16 string 类和标准模板库（最小必要）
***
作者：abnb250/山中鹤鸣  
博客：https://www.cnblogs.com/abnb250  
交流群：1062464078  
[回到主目录点这里](https://www.cnblogs.com/abnb250/articles/23230010)  

<br>

<br>

你还在用 `char name[20]` 存名字？用 `int arr[100]` 存数据？用两个平行数组做查找表？

这些手工方式不是不能用，而是坑太多：字符串拼接怕越界，动态数组怕忘记 `delete[]`，查找得自己写循环，插入删除还得挪元素。现代 C++ 早就给你准备好了工具：`std::string`、`std::vector`、`std::map`、迭代器、lambda。这一章咱们就从这些痛点出发，一个个把它们用起来。

<br>
<br>

## 知识点 1：std::string——别再手写 char[] 了

***
<br>

### 问题

<br>

你写 `char name[20] = "hello";`，想拼接、查找、截取，得用 `strcat`、`strstr`、`strncpy`，还得时刻担心缓冲区溢出。有没有一个字符串类型，能自动管理内存，还提供现成的查找、截取、拼接？

<br>

### 方案

<br>

用 **`std::string`**。它自动管理内存，提供丰富的成员函数：构造、长度、访问、查找、子串、修改、遍历，全都有。你不用再手动 `new char[]` 和 `delete[]`。

<br>

### 最小示例

<br>

```cpp
#include <iostream>
#include <string>

int main() {
    std::string s = "hello";
    std::string s2("world");
    std::string s3(5, 'a');          // "aaaaa"
    std::string s4 = s + " " + s2;   // 拼接

    std::cout << "s4 = " << s4 << std::endl;
    std::cout << "s.size() = " << s.size() << std::endl;
    std::cout << "s.empty() = " << s.empty() << std::endl;

    std::cout << "s[0] = " << s[0] << std::endl;
    std::cout << "s.at(0) = " << s.at(0) << std::endl;
    std::cout << "s.front() = " << s.front() << std::endl;
    std::cout << "s.back() = " << s.back() << std::endl;

    std::cout << "s.find(\"ll\") = " << s.find("ll") << std::endl;
    std::cout << "s.rfind(\"l\") = " << s.rfind("l") << std::endl;

    std::cout << "s.substr(1, 3) = " << s.substr(1, 3) << std::endl;

    s.append("!");
    std::cout << "after append: " << s << std::endl;
    s.insert(0, "say ");
    std::cout << "after insert: " << s << std::endl;
    s.erase(0, 4);
    std::cout << "after erase: " << s << std::endl;
    s.replace(0, 5, "hi");
    std::cout << "after replace: " << s << std::endl;

    for (char c : s) {
        std::cout << c;
    }
    std::cout << std::endl;

    return 0;
}
```

运行结果：

```text
s4 = hello world
s.size() = 5
s.empty() = 0
s[0] = h
s.at(0) = h
s.front() = h
s.back() = o
s.find("ll") = 2
s.rfind("l") = 3
s.substr(1, 3) = ell
after append: hello!
after insert: say hello!
after erase: hello!
after replace: hi!
hi!
```

**易错点**：`s.find("xyz")` 找不到时返回 **`std::string::npos`**，不要直接拿它当下标用。`s.at(100)` 越界会抛 `std::out_of_range` 异常，而 `s[100]` 不检查，行为未定义。

<br>

### 一句话总结

<br>

**`std::string` 就是自动管理内存的字符串，拼接、查找、截取都方便，不用再担心越界。**

<br>

### 自测

<br>

`s.find("xyz")` 找不到返回什么？`s[100]` 和 `s.at(100)` 有什么区别？

<br>
<br>

## 知识点 2：vector——别再手写动态数组了

***
<br>

### 问题

<br>

你用 `new int[n]` 创建动态数组，得手动 `delete[]`，还得自己记录大小，扩容更麻烦。有没有一个数组，能自动扩容、自动释放、还知道自己的大小？

<br>

### 方案

<br>

用 **`std::vector`**。它是动态数组，自动管理内存，支持 `push_back`、`pop_back`、`size`、`empty`，可以用 `[]` 和 `at()` 访问，还能直接传给 `std::sort` 排序。

<br>

### 最小示例

<br>

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> v = {1, 2, 3};
    v.push_back(4);
    v.push_back(5);

    std::cout << "size = " << v.size() << std::endl;
    std::cout << "empty = " << v.empty() << std::endl;
    std::cout << "v[0] = " << v[0] << std::endl;
    std::cout << "v.at(0) = " << v.at(0) << std::endl;
    std::cout << "front = " << v.front() << std::endl;
    std::cout << "back = " << v.back() << std::endl;

    v.pop_back();
    std::cout << "after pop_back, back = " << v.back() << std::endl;

    std::sort(v.begin(), v.end());
    for (int x : v) {
        std::cout << x << " ";
    }
    std::cout << std::endl;

    auto it = std::find(v.begin(), v.end(), 3);
    if (it != v.end()) {
        std::cout << "found 3 at index " << (it - v.begin()) << std::endl;
    }

    return 0;
}
```

运行结果：

```text
size = 5
empty = 0
v[0] = 1
v.at(0) = 1
front = 1
back = 5
after pop_back, back = 4
1 2 3 4
found 3 at index 2
```

**易错点**：`v[100]` 不检查越界，`v.at(100)` 会抛异常。另外，`push_back` 可能导致 vector 扩容，**原有的迭代器会失效**，别再继续用旧的迭代器。

<br>

### 一句话总结

<br>

**`vector` 就是会自动扩容、自动释放的数组，替代 `new[]`/`delete[]`，还能直接配合算法用。**

<br>

### 自测

<br>

`vector` 的 `[]` 和 `at()` 有什么区别？`push_back` 后原来的迭代器还能用吗？

<br>
<br>

## 知识点 3：map 和 unordered_map——别再手写查找表了

***
<br>

### 问题

<br>

你想根据名字查分数，用两个数组平行存放：`names[i]` 和 `scores[i]`。查找得遍历，插入删除还得挪元素。有没有现成的“字典”？

<br>

### 方案

<br>

用 **`std::map`** 或 **`std::unordered_map`**。

- `std::map`：红黑树实现，**有序**，查找 O(log n)。
- `std::unordered_map`：哈希表实现，**无序**，平均查找 O(1)。

它们都提供 `insert`、`find`、`erase`、`[]` 等操作。

<br>

### 最小示例

<br>

```cpp
#include <iostream>
#include <map>
#include <unordered_map>
#include <string>

int main() {
    // map：有序
    std::map<std::string, int> m;
    m["apple"] = 1;
    m.insert({"banana", 2});
    m["cherry"] = 3;

    std::cout << "map size = " << m.size() << std::endl;

    auto it = m.find("apple");
    if (it != m.end()) {
        std::cout << it->first << ": " << it->second << std::endl;
    }

    for (const auto& pair : m) {
        std::cout << pair.first << " = " << pair.second << std::endl;
    }

    m.erase("apple");
    std::cout << "after erase, size = " << m.size() << std::endl;

    // unordered_map：无序，哈希
    std::unordered_map<std::string, int> um;
    um["one"] = 1;
    um["two"] = 2;
    um["three"] = 3;

    std::cout << "unordered_map size = " << um.size() << std::endl;
    for (const auto& pair : um) {
        std::cout << pair.first << " = " << pair.second << std::endl;
    }

    return 0;
}
```

运行结果示例：

```text
map size = 3
apple: 1
apple = 1
banana = 2
cherry = 3
after erase, size = 2
unordered_map size = 3
three = 3
two = 2
one = 1
```

**易错点**：`map` 用 `[]` 访问不存在的键会**自动插入一个默认值**（比如 `int` 就是 0）。如果你只是想查找，用 `find` 或 `at`。`unordered_map` 同理。

<br>

### 一句话总结

<br>

**`map` 是有序字典，`unordered_map` 是哈希字典，根据要不要排序选；用 `[]` 访问会插入默认值，查找用 `find`。**

<br>

### 自测

<br>

`map` 用 `[]` 访问不存在的键会怎样？`map` 和 `unordered_map` 有什么区别？

<br>
<br>

## 知识点 4：迭代器——统一遍历容器的方式

***
<br>

### 问题

<br>

`string` 用下标遍历，`vector` 用下标遍历，`map` 不能用下标遍历（因为键不是整数）。每个容器遍历方式都不一样，能不能统一？

<br>

### 方案

<br>

用**迭代器**。它像指针一样，`begin()` 指向第一个元素，`end()` 指向最后一个元素之后。`*it` 取元素，`++it` 前进。所有标准容器都支持迭代器，算法也通过迭代器操作容器。

<br>

### 最小示例

<br>

```cpp
#include <iostream>
#include <vector>
#include <map>
#include <string>

int main() {
    std::vector<int> v = {10, 20, 30};
    for (auto it = v.begin(); it != v.end(); ++it) {
        std::cout << *it << " ";
    }
    std::cout << std::endl;

    std::map<std::string, int> m = {{"a", 1}, {"b", 2}};
    for (auto it = m.begin(); it != m.end(); ++it) {
        std::cout << it->first << "=" << it->second << " ";
    }
    std::cout << std::endl;

    // 用 std::find 查找
    auto it = std::find(v.begin(), v.end(), 20);
    if (it != v.end()) {
        std::cout << "found at index " << (it - v.begin()) << std::endl;
    }

    return 0;
}
```

运行结果：

```text
10 20 30 
a=1 b=2 
found at index 1
```

**易错点**：`end()` 不指向任何元素，不能解引用。`++it` 后如果越过了 `end()`，行为未定义。另外，容器修改后（如 `push_back`），原来的迭代器可能失效。

<br>

### 一句话总结

<br>

**迭代器就是容器里的“通用指针”，让算法和容器解耦，`begin()` 到 `end()` 半开区间遍历。**

<br>

### 自测

<br>

`begin()` 和 `end()` 分别指向哪里？`*it` 和 `++it` 是什么意思？

<br>
<br>

## 知识点 5：lambda——让算法更灵活

***
<br>

### 问题

<br>

`std::sort` 默认升序，你想降序，得单独写一个比较函数，太麻烦。能不能就地写个小函数，直接传给 `sort`？

<br>

### 方案

<br>

用 **lambda 表达式**。它就是一个匿名函数，语法是 `[捕获](参数) { 函数体 }`。可以直接写在算法调用里，不用跑到外面定义函数。

<br>

### 最小示例

<br>

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> v = {3, 1, 4, 1, 5};

    // 降序排序
    std::sort(v.begin(), v.end(), [](int a, int b) {
        return a > b;
    });

    for (int x : v) {
        std::cout << x << " ";
    }
    std::cout << std::endl;

    // 捕获外部变量
    int threshold = 3;
    auto it = std::find_if(v.begin(), v.end(), [threshold](int x) {
        return x < threshold;
    });

    if (it != v.end()) {
        std::cout << "first element < " << threshold << " is " << *it << std::endl;
    }

    return 0;
}
```

运行结果：

```text
5 4 3 1 1 
first element < 3 is 1
```

**易错点**：`[]` 不捕获任何外部变量，`[&]` 按引用捕获，`[=]` 按值捕获。按引用捕获时，要保证 lambda 执行时被引用的变量还活着。

<br>

### 一句话总结

<br>

**lambda 就是就地写的小函数，让算法调用更灵活，捕获列表决定它能用哪些外部变量。**

<br>

### 自测

<br>

lambda 的捕获列表有什么用？`[]` 和 `[&]` 有什么区别？

<br>
<br>

## 完整可运行代码
***

下面把这一章的主要知识点整合到一个程序里：`std::string`、`std::vector`、`std::map`、`std::unordered_map`、迭代器、lambda。你可以直接复制运行。

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <map>
#include <unordered_map>
#include <algorithm>

int main() {
    std::cout << "=== std::string ===" << std::endl;
    std::string s = "hello world";
    std::cout << "find world: " << s.find("world") << std::endl;
    std::cout << "substr(0,5): " << s.substr(0, 5) << std::endl;
    s.append("!");
    std::cout << "append: " << s << std::endl;

    std::cout << "\n=== std::vector ===" << std::endl;
    std::vector<int> v = {3, 1, 4, 1, 5};
    v.push_back(9);
    std::sort(v.begin(), v.end());
    for (int x : v) std::cout << x << " ";
    std::cout << std::endl;

    auto it = std::find(v.begin(), v.end(), 4);
    if (it != v.end()) {
        std::cout << "found 4 at index " << (it - v.begin()) << std::endl;
    }

    std::cout << "\n=== std::map ===" << std::endl;
    std::map<std::string, int> m;
    m["one"] = 1;
    m["two"] = 2;
    m["three"] = 3;
    for (const auto& p : m) {
        std::cout << p.first << "=" << p.second << " ";
    }
    std::cout << std::endl;

    std::cout << "\n=== std::unordered_map ===" << std::endl;
    std::unordered_map<std::string, int> um;
    um["apple"] = 10;
    um["banana"] = 20;
    for (const auto& p : um) {
        std::cout << p.first << "=" << p.second << " ";
    }
    std::cout << std::endl;

    std::cout << "\n=== lambda ===" << std::endl;
    std::vector<int> nums = {5, 2, 8, 1, 9};
    std::sort(nums.begin(), nums.end(), [](int a, int b) {
        return a > b;
    });
    for (int x : nums) std::cout << x << " ";
    std::cout << std::endl;

    int limit = 5;
    auto lit = std::find_if(nums.begin(), nums.end(), [limit](int x) {
        return x < limit;
    });
    if (lit != nums.end()) {
        std::cout << "first < " << limit << " is " << *lit << std::endl;
    }

    return 0;
}
```

运行结果示例：

```text
=== std::string ===
find world: 6
substr(0,5): hello
append: hello world!

=== std::vector ===
1 1 3 4 5 9 
found 4 at index 3

=== std::map ===
one=1 three=3 two=2 

=== std::unordered_map ===
banana=20 apple=10 

=== lambda ===
9 8 5 2 1 
first < 5 is 2
```

<br>
<br>

## 自测三问
***

1. `string::find` 找不到返回什么？`vector` 的 `[]` 和 `at()` 有什么区别？
2. `map` 和 `unordered_map` 的区别是什么？`map` 用 `[]` 访问不存在的键会怎样？
3. 迭代器的 `begin()` 和 `end()` 分别指向哪里？lambda 的捕获列表有什么用？

<br>
<br>

## 一句话总结全章
***

**`std::string` 替代 `char[]`，`std::vector` 替代手工数组，`map`/`unordered_map` 替代查找表，迭代器统一遍历，lambda 让算法更灵活——这些是现代 C++ 项目的标配，早用早省心。**

<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>