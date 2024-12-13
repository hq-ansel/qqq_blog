---
date: 2024-12-13T21:12:56+08:00
tags:
  - cpp
  - STL
title: cpp Lambda表达式与自定义哈希函数与decltype
share: true
canonicalURL: ""
keywords: 
description: 
series: 系列
lastmod: 
lang: cn
cover:
  image: 
author: heqi
dir: posts
math: "true"
---
在leetcode 49中的官方解答存在lambda表达式，与decltype与自定已哈希函数的用法的例子

```c++
auto arrayHash = [fn = hash<int>{}] (const array<int, 26>& arr) -> size_t {
    return accumulate(arr.begin(), arr.end(), 0u, [&](size_t acc, int num) {
        return (acc << 1) ^ fn(num);
    });
};

```

### **1 自定义哈希函数**

#### **背景：**

在 `std::unordered_map` 或 `std::unordered_set` 中，键需要能够被哈希以支持高效查找。C++ 标准库对基本类型（如 `int`、`std::string` 等）和部分标准容器（如 `std::pair`）提供了默认的哈希实现。但对于一些自定义类型（如 `std::array<int, 26>`），没有默认的哈希函数，需要开发者提供。

#### **定义自定义哈希函数：**

自定义哈希函数通常是一个函数对象或 lambda 表达式，用于对键进行哈希计算。

- 示例：
    
    ```cpp
    auto customHash = [](const std::array<int, 26>& arr) -> size_t {
        return std::accumulate(arr.begin(), arr.end(), 0u, [](size_t acc, int num) {
            return acc ^ std::hash<int>{}(num);
        });
    };
    ```
    

#### **作用：**

通过自定义哈希函数，开发者可以定义如何为复杂类型生成哈希值。这种能力尤其重要，用于支持非标准类型作为键的容器。

---

### **2 Lambda 表达式捕获**

Lambda 表达式是 C++ 中的一种轻量级语法，用于定义**匿名函数**。它可以在代码中直接定义并使用，没有名字，也无需单独声明，可以捕获周围作用域中的变量。这使得它非常适合需要临时定义简单函数的场景。

Lambda 表达式在 C++11 中引入，并在后续标准（C++14、C++17 等）中得到了增强。

#### **Lambda 表达式的基本结构**

Lambda 表达式的基本语法如下：

```cpp
[捕获列表](参数列表) -> 返回类型 {
    函数体
};
```

#### **分解各部分：**

1. **捕获列表 `[]`**:
    - 用于捕获外部作用域中的变量，可以按值或按引用捕获，也可以进行初始化。
2. **参数列表 `()`**:
    - 类似于普通函数的参数列表，可以为空。
3. **返回类型 `->`**:
    - 明确指定函数的返回类型，可省略（在大多数情况下可以自动推导）。
4. **函数体 `{}`**:
    - 包含函数的逻辑，类似于普通函数的实现部分。

#### **示例代码**

#### **简单例子：**

```cpp
auto add = [](int a, int b) -> int {
    return a + b;
};

int result = add(3, 5); // result = 8
```

- **解释：**
    - `[ ]`：捕获列表为空，表示不使用外部变量。
    - `(int a, int b)`：函数接受两个参数 `a` 和 `b`。
    - `-> int`：函数返回类型是 `int`。
    - `{ return a + b; }`：函数体计算 `a + b` 并返回结果。

#### **捕获外部变量：**

```cpp
int x = 10;
int y = 20;

auto multiply = [x, &y](int z) -> int {
    y += 10; // y 按引用捕获，可以被修改
    return x * y * z;
};

int result = multiply(2); // result = 10 * 30 * 2 = 600
```

- **解释：**
    - `[x, &y]`：按值捕获 `x`，按引用捕获 `y`。
    - `y` 在 lambda 内可以被修改，而 `x` 不会被改变。

#### **省略返回类型：**

C++14 开始支持自动返回类型推导，因此返回类型可以省略。

```cpp
auto subtract = [](int a, int b) {
    return a - b;
};
```

---

#### **捕获列表的形式**

1. **按值捕获 `[x]`**: 捕获外部变量的副本，lambda 内修改不会影响原变量。
    
    ```cpp
    int x = 10;
    auto lambda = [x]() { return x + 1; };
    ```
    
2. **按引用捕获 `[&x]`**: 捕获外部变量的引用，lambda 内修改会影响原变量。
    
    ```cpp
    int x = 10;
    auto lambda = [&x]() { x += 1; };
    lambda(); // x 被修改为 11
    ```
    
3. **捕获所有变量 `[=]` 或 `[&]`**:
    
    - `[=]`：按值捕获所有变量。
    - `[&]`：按引用捕获所有变量。
    
    ```cpp
    int x = 10, y = 20;
    auto lambda1 = [=]() { return x + y; }; // 按值捕获 x 和 y
    auto lambda2 = [&]() { x += y; };      // 按引用捕获 x 和 y
    ```
    
4. **初始化捕获 `[var = expr]`**: C++14 开始支持在捕获列表中初始化变量。
    
    ```cpp
    auto lambda = [sum = 10](int value) {
        return sum + value;
    };
    ```
    

---

#### **Lambda 表达式的用途**

1. **临时函数定义：** 使用 lambda 表达式可以避免单独声明一个函数，非常适合仅在特定上下文中使用的小型函数。
    
2. **与标准库算法结合：** 在使用 `std::for_each`、`std::sort` 等标准算法时，lambda 表达式常用作自定义逻辑的传递。
    
    ```cpp
    std::vector<int> nums = {1, 2, 3, 4};
    std::for_each(nums.begin(), nums.end(), [](int n) {
        std::cout << n << " ";
    });
    ```
    
3. **作为回调函数：** Lambda 表达式可以用作回调函数，例如事件处理或线程任务。
    
    ```cpp
    std::thread t([] {
        std::cout << "Thread running!" << std::endl;
    });
    t.join();
    ```
    
4. **捕获外部变量：** 可以灵活地使用捕获列表传递外部数据，避免显式传参的麻烦。
    
---

### **3 `decltype`**

#### **背景：**

`decltype` 是 C++11 引入的一个关键字，用于推导表达式的类型。它和 `auto` 类似，但更适合需要明确表达式类型的场景。

#### **用法：**

- 推导变量类型：
    
    ```cpp
    int x = 10;
    decltype(x) y = 20;  // y 的类型与 x 相同，为 int
    ```
    
- 用于函数返回值：
    
    ```cpp
    auto func = [](int a, int b) -> decltype(a + b) {
        return a + b;
    };
    ```
    

#### **在示例中的作用：**

`decltype(arrayHash)` 推导出 `arrayHash` 的类型（`std::function<size_t(const std::array<int, 26>&)>`），并将其用作 `unordered_map` 的第三个模板参数。

---

### **4 `std::accumulate`**

#### **背景：**

`std::accumulate` 是 C++ 标准库提供的算法，用于对一个范围内的所有元素进行累积操作。

#### **定义：**

```cpp
template<class InputIt, class T, class BinaryOperation>
T accumulate(InputIt first, InputIt last, T init, BinaryOperation op);
```

- **参数：**
    - `InputIt first, InputIt last`：范围的起始和结束迭代器。
    - `T init`：累积的初始值。
    - `BinaryOperation op`：二元操作函数，用于定义累积规则。

#### **用法：**

- 示例：累加一个整数数组：
    
    ```cpp
    std::vector<int> vec = {1, 2, 3};
    int sum = std::accumulate(vec.begin(), vec.end(), 0);  // sum = 6
    ```
    
- 示例：自定义操作规则：
    
    ```cpp
    int hashValue = std::accumulate(arr.begin(), arr.end(), 0u, [](size_t acc, int num) {
        return acc ^ std::hash<int>{}(num);
    });
    ```
    

#### **在示例中的作用：**

`std::accumulate` 被用来遍历 `std::array<int, 26>` 的所有元素，将每个元素的哈希值累积成一个最终的哈希值。
