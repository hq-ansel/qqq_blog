---
date: 2024-12-13T21:12:10+08:00
tags:
  - cpp
  - 开发八股
title: cpp STL pushback 与 emplace_back的区别
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


在 C++ 的 STL 容器（如 `std::vector`、`std::deque`）中，`push_back` 和 `emplace_back` 是用于在容器尾部插入元素的两种方法，它们的主要区别在于插入方式和效率。
###  **`push_back`**

- **功能**：将一个对象添加到容器的尾部。
    
- **实现方式**：
    
    - 需要一个已经构造好的对象。
    - 将该对象复制（或移动）到容器中。
- **使用示例**：
    
    ```cpp
    std::vector<std::pair<int, std::string>> vec;
    vec.push_back(std::make_pair(1, "example")); // 必须先创建 std::make_pair 对象
    ```
    
- **缺点**：
    
    - 如果对象的构造很昂贵，`push_back` 会产生额外的开销（复制或移动的开销）。
    - 无法直接在容器中原地构造对象。

---

###  **`emplace_back`**

- **功能**：在容器尾部**直接构造**对象。
    
- **实现方式**：
    
    - 通过传递构造函数的参数，直接在容器的存储位置构造对象。
    - 避免了多余的复制或移动。
- **使用示例**：
    
    ```cpp
    std::vector<std::pair<int, std::string>> vec;
    vec.emplace_back(1, "example"); // 直接在容器中构造对象
    ```
    
- **优点**：
    
    - 提高了效率，因为省略了中间的临时对象创建和销毁。
    - 更灵活，允许直接传递构造函数参数。

---

### **对比总结**

|特性|`push_back`|`emplace_back`|
|---|---|---|
|**调用方式**|需要一个已构造的对象|直接传递构造函数参数|
|**性能**|可能有额外的复制或移动开销|避免了复制或移动，效率更高|
|**使用场景**|已有一个现成的对象|需要直接在容器中构造对象|
|**可读性**|较简单，适合已有对象的插入|较复杂，但在某些场景下更加高效|

---

### **选择建议**

- 如果已有对象且不关心额外开销，可以使用 `push_back`。
- 如果希望避免临时对象的构造，或者希望通过构造函数直接初始化对象，使用 `emplace_back` 更合适。

### **注意事项**

- 如果构造参数传递有歧义（如多种重载构造函数），`emplace_back` 的行为可能更复杂，应小心使用。
- 对于简单类型（如 `int`、`double`），两者性能几乎无差别。

`emplace_back` 在某些情况下可能会导致危险行为，特别是当构造函数存在多个重载或隐式类型转换时，这可能导致程序选择了错误的构造函数，进而引发逻辑错误或意外行为。

### **危险例子**

假设我们有一个类 `Example`，其构造函数有多种重载：

```cpp
#include <vector>
#include <string>
#include <iostream>

class Example {
public:
    Example(int x) {
        std::cout << "Constructor with int: " << x << std::endl;
    }
    Example(const std::string& str) {
        std::cout << "Constructor with string: " << str << std::endl;
    }
};
```

我们尝试使用 `emplace_back` 向 `std::vector<Example>` 中插入对象：

```cpp
int main() {
    std::vector<Example> vec;

    vec.emplace_back(42);        // 调用 int 构造函数，输出：Constructor with int: 42
    vec.emplace_back("hello");   // 调用 string 构造函数，输出：Constructor with string: hello

    vec.emplace_back('a');       // 这里会调用哪个构造函数？
}
```

---

### **问题分析**

在最后一行 `vec.emplace_back('a');` 中：

- `'a'` 是一个字符（`char`）。
- `char` 类型可以隐式转换为 `int`，因此会选择 `Example(int)` 构造函数，而不是 `Example(const std::string&)` 构造函数。

这可能不是我们预期的行为，尤其是在类的构造函数设计复杂或含有隐式转换时。

---

### **运行结果**

输出如下：

```
Constructor with int: 42
Constructor with string: hello
Constructor with int: 97
```

注意到 `'a'` 被转换为了 ASCII 值 `97`，并调用了 `Example(int)` 构造函数。这可能会导致逻辑错误或意外行为。

---

### **如何避免？**

1. **显式构造函数**： 将构造函数声明为 `explicit`，避免隐式类型转换。
    
    ```cpp
    class Example {
    public:
        explicit Example(int x) {
            std::cout << "Constructor with int: " << x << std::endl;
        }
        explicit Example(const std::string& str) {
            std::cout << "Constructor with string: " << str << std::endl;
        }
    };
    ```
    
    如果尝试 `vec.emplace_back('a');`，编译器会报错，强制开发者明确指定意图。
    
2. **代码审查**： 在使用 `emplace_back` 时，确保传递的参数类型和构造函数匹配，避免隐式转换。
    
3. **使用 `push_back` 或直接构造**： 当逻辑复杂或类型歧义可能存在时，显式构造对象并使用 `push_back`，可提升代码可读性。
    
    ```cpp
    vec.push_back(Example('a'));
    ```
    
