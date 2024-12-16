---
date: 2024-12-16T20:12:40+08:00
tags:
  - cpp
  - 开发八股
  - STL
title: CppSTL map系列
share: true
canonicalURL: ""
keywords: 
description: 
series: 系列
lastmod: 
lang: cn
cover:
  image: 
author: qqq
dir: posts
math: "true"
---
 C++ STL 中 `map` 与 `unordered_map` 的使用笔记和比较

#### 1. **简介**

| 容器类型      | **`map`**           | **`unordered_map`**    |
| --------- | ------------------- | ---------------------- |
| **实现机制**  | 红黑树（自平衡二叉搜索树）       | 哈希表                    |
| **键的顺序**  | 按键的升序（或自定义排序）       | 无序（哈希值决定元素位置）          |
| **时间复杂度** | 插入、删除、查找均为 O(log⁡n) | 插入、删除、查找均为 O(1)（均摊复杂度） |
| **内存使用**  | 较少                  | 较多（需要额外的哈希表存储）         |
| **适用场景**  | 需要有序的数据（按键排序）       | 快速查找、插入，且不要求有序         |
| **线程安全性** | 非线程安全               | 非线程安全                  |

---

#### 2. **头文件**

```cpp
#include <map>
#include <unordered_map>
```

---

#### 3. **基本操作比较**

##### 定义和初始化

```cpp

std::map<KeyType, ValueType, Compare>
std::unordered_map<KeyType, ValueType, Hash, KeyEqual>


std::map<int, std::string> m;
std::unordered_map<int, std::string> um;

std::map<int, std::string> m2 = {{1, "one"}, {2, "two"}};
std::unordered_map<int, std::string> um2 = {{1, "one"}, {2, "two"}};
```

##### 插入元素

```cpp
m[1] = "hello";
um[1] = "hello";

m.insert({2, "world"});
um.insert({2, "world"});

m.emplace(3, "C++");
um.emplace(3, "C++");
```

##### 访问元素

```cpp
std::string val = m[1];    // 若键不存在，会创建默认值
std::string val = um[1];   // 若键不存在，也会创建默认值

std::string val = m.at(2); // 若键不存在，抛出异常
std::string val = um.at(2); // 同上
```

##### 查找元素

```cpp
if (m.find(3) != m.end()) {
    std::cout << "Found in map" << std::endl;
}

if (um.find(3) != um.end()) {
    std::cout << "Found in unordered_map" << std::endl;
}
```

##### 遍历容器

```cpp
// 遍历 map
for (const auto& [key, value] : m) {
    std::cout << "Key: " << key << ", Value: " << value << std::endl;
}

// 遍历 unordered_map
for (const auto& [key, value] : um) {
    std::cout << "Key: " << key << ", Value: " << value << std::endl;
}
```

##### 删除元素

```cpp
m.erase(1);        // 按键值删除
um.erase(1);       // 同上

m.erase(m.begin());  // 按迭代器删除
um.erase(um.begin());
```

##### 清空容器

```cpp
m.clear();
um.clear();
```

---

#### 4. **性能对比**

1. **时间复杂度**
    
    - `map`: 插入、删除、查找的时间复杂度为 O(log⁡n)O(\log n)，适合需要有序数据的场景。
    - `unordered_map`: 插入、删除、查找的均摊时间复杂度为 O(1)O(1)，适合快速查找的场景。
2. **内存占用**
    
    - `map`: 内存占用较低。
    - `unordered_map`: 由于哈希表的开销（如桶分配），内存占用较高。
3. **顺序性**
    
    - `map`: 数据按键升序存储，遍历结果有序。
    - `unordered_map`: 数据无序，遍历顺序与插入顺序无关。
4. **自定义排序**
    
    - `map`: 支持通过比较器自定义排序。
    - `unordered_map`: 不支持排序。

---

#### 5. **示例代码：性能和特性对比**

```cpp
#include <iostream>
#include <map>
#include <unordered_map>
#include <chrono>

int main() {
    const int n = 1000000;

    // 初始化
    std::map<int, int> m;
    std::unordered_map<int, int> um;

    // 插入性能
    auto start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < n; ++i) {
        m[i] = i;
    }
    auto end = std::chrono::high_resolution_clock::now();
    std::cout << "map insertion time: "
              << std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count()
              << " ms" << std::endl;

    start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < n; ++i) {
        um[i] = i;
    }
    end = std::chrono::high_resolution_clock::now();
    std::cout << "unordered_map insertion time: "
              << std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count()
              << " ms" << std::endl;

    // 查找性能
    start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < n; ++i) {
        m.find(i);
    }
    end = std::chrono::high_resolution_clock::now();
    std::cout << "map find time: "
              << std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count()
              << " ms" << std::endl;

    start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < n; ++i) {
        um.find(i);
    }
    end = std::chrono::high_resolution_clock::now();
    std::cout << "unordered_map find time: "
              << std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count()
              << " ms" << std::endl;

    return 0;
}
```

---

#### 6. **选择建议**

- **使用 `map` 的场景**：
    
    1. 需要保持键值对的有序性。
    2. 需要按顺序遍历键值对。
    3. 数据规模较小且性能要求不高。
- **使用 `unordered_map` 的场景**：
    
    1. 主要需求是快速查找、插入、删除。
    2. 不关心键值对的存储顺序。
    3. 数据规模较大，性能优先。
