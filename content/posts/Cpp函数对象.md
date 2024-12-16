---
date: 2024-12-16T21:12:17+08:00
tags:
  - cpp
  - 开发八股
title: Cpp函数对象
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


**函数对象**（Function Objects）、**Lambda 表达式** 和 **类的成员函数** 是 C++ 中调用函数行为的三种实现方式，它们在语法、用途和底层实现上有所不同。
### **1. 函数对象（Function Objects）**

**定义**：函数对象是一个定义了 `operator()` 的类或结构体实例，可以像普通函数一样调用。

#### 示例：

```cpp
struct MyFunctionObject {
    int operator()(int x) const {
        return x * x;
    }
};
```

使用：

```cpp
MyFunctionObject obj;
int result = obj(5); // 调用 operator()
```

#### **底层实现**

- 本质是一个类或结构体，通过重载 `operator()` 实现。
- 编译器会将函数对象的调用解析为对 `operator()` 的直接调用。
- 因为函数对象是类实例，所以可以在内部保存状态（成员变量）。

**特点**：

- **有状态**：可以通过成员变量在对象中存储状态。
- **灵活性高**：可以自定义更多逻辑、配置。
- **效率高**：可以被内联优化，减少调用开销。

---

### **2. Lambda 表达式**

**定义**：Lambda 是一种匿名函数，可以捕获外部作用域的变量，并立即调用或传递。

#### 示例：

```cpp
auto lambda = [](int x) { return x * x; };
int result = lambda(5);
```

#### **底层实现**

- Lambda 是一个编译器生成的匿名类，其行为类似于函数对象。
- 编译器为每个 Lambda 生成一个唯一的类，类中定义了 `operator()`。
- 如果 Lambda 捕获了变量，这些变量会被存储为该类的成员变量。

例如：

```cpp
int a = 10;
auto lambda = [a](int x) { return x + a; }; // 捕获 a
```

等价于：

```cpp
class AnonymousLambda {
    int a; // 捕获变量
public:
    AnonymousLambda(int a) : a(a) {}
    int operator()(int x) const {
        return x + a;
    }
};
```

**特点**：

- **无状态或有状态**：根据捕获列表决定是否存储状态。
- **灵活性高**：简化代码，常用于临时逻辑。
- **可内联**：与函数对象类似，编译器可以内联优化。

---

### **3. 类的成员函数**

**定义**：类的成员函数是定义在类中的函数，可以访问类的成员变量和方法。

#### 示例：

```cpp
class MyClass {
public:
    int square(int x) { return x * x; }
};
```

使用：

```cpp
MyClass obj;
int result = obj.square(5);
```

#### **底层实现**

- 成员函数本质上是一个普通函数，但带有隐式的 `this` 指针。
- 在调用成员函数时，编译器会传递当前对象的地址 `this`，以便访问类的成员变量和其他方法。

例如：

```cpp
class MyClass {
    int a;
public:
    void print() {
        std::cout << a;
    }
};
```

底层实现等价于：

```cpp
void MyClass::print(MyClass* this) {
    std::cout << this->a;
}
```

**特点**：

- **绑定到对象**：必须通过对象或指针调用。
- **有状态**：依赖对象的成员变量和方法。
- **效率相对较高**：调用成本仅增加一个 `this` 指针。

---

### **对比总结**

|特性|函数对象|Lambda 表达式|类成员函数|
|---|---|---|---|
|**底层实现**|重载 `operator()` 的类|编译器生成匿名类，重载 `operator()`|普通函数，隐式传递 `this` 指针|
|**状态管理**|可以有状态（通过成员变量）|捕获外部变量决定是否有状态|依赖类的成员变量和 `this` 指针|
|**调用方式**|类实例调用 `()`|匿名对象调用 `()`|必须通过类对象调用|
|**效率**|高（可内联优化）|高（可内联优化）|高（普通函数调用 + `this`）|
|**灵活性**|高|更高，适合临时定义|依赖类，灵活性较低|

---

### **适用场景**

1. **函数对象**：
    
    - 适合需要反复调用且逻辑较复杂的场景。
    - 适用于存储状态的情况下（如计数器、配置参数）。
2. **Lambda 表达式**：
    
    - 适合临时的、一次性使用的函数逻辑。
    - 捕获外部变量时非常方便，代码更简洁。
3. **类成员函数**：
    
    - 适合与类绑定的逻辑，依赖类的成员变量或方法。
    - 通常用于面向对象设计中的方法。

---

### **代码示例对比**

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

// 函数对象
struct FunctionObject {
    int operator()(int x) const {
        return x * x;
    }
};

// Lambda
auto lambda = [](int x) { return x * x; };

// 类成员函数
class MyClass {
public:
    int square(int x) const { return x * x; }
};

int main() {
    std::vector<int> nums = {1, 2, 3, 4};

    // 函数对象
    FunctionObject funcObj;
    std::transform(nums.begin(), nums.end(), nums.begin(), funcObj);

    // Lambda 表达式
    std::transform(nums.begin(), nums.end(), nums.begin(), lambda);

    // 类成员函数
    MyClass obj;
    std::transform(nums.begin(), nums.end(), nums.begin(),
                   [&](int x) { return obj.square(x); });

    for (int n : nums) {
        std::cout << n << " ";
    }

    return 0;
}
```

---

### **总结**

- **底层实现**上，函数对象和 Lambda 表达式非常相似，Lambda 本质上是一个匿名的函数对象。
- **类成员函数**的底层实现则依赖 `this` 指针，绑定到特定对象。
- 三者的选择取决于使用场景和代码需求，Lambda 表达式在现代 C++ 中更常用，特别是在临时逻辑场景下。