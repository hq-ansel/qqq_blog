---
date: 2024-12-13T21:12:27+08:00
tags:
  - cpp
  - 开发八股
title: and 符号在cpp中的用途
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
### **总结表**

|用途|含义|示例|
|---|---|---|
|地址运算符|获取变量的地址|`int* ptr = &x;`|
|引用声明|声明一个变量的引用|`int& ref = x;`|
|按位与|执行按位与操作|`int c = a & b;`|
|左值引用|函数接受一个左值引用|`void func(int& n);`|
|右值引用（`&&`）|函数接受一个右值引用|`void func(int&& n);`|
|函数返回类型|返回一个引用|`int& func();`|
|类型推导|使用 `decltype` 推导引用类型|`decltype(x)& ref = x;`|
|指针和引用类型|在类型声明中表示引用或地址操作|`int* ptr = &x;`|
|重载运算符|自定义按位与或按位与赋值运算符|`Example& operator&(Example);`|

### **1. 地址运算符**

`&` 用于获取变量的内存地址。

- **示例**：
    
    ```cpp
    int x = 10;
    int* ptr = &x; // 获取 x 的地址并存储在指针 ptr 中
    ```
    
- **作用**：获取变量的地址。
    

---

### **2. 引用声明**

`&` 用于声明一个变量的引用（reference）。

- **示例**：
    
    ```cpp
    int x = 10;
    int& ref = x; // ref 是 x 的引用
    ref = 20;     // 修改 ref 也会修改 x
    ```
    
- **作用**：为一个变量创建别名，操作引用会直接影响原变量。
    

---

### **3. 按位与运算符**

`&` 用于按位与（bitwise AND）操作。

- **示例**：
    
    ```cpp
    int a = 5;  // 二进制: 0101
    int b = 3;  // 二进制: 0011
    int c = a & b; // c = 1 (二进制: 0001)
    ```
    
- **作用**：逐位比较两个整数，每个位都执行 AND 操作。
    

---

### **4. 左值引用（Lvalue Reference）**

当 `&` 出现在函数形参中时，表示接受一个左值引用。

- **示例**：
    
    ```cpp
    void increment(int& n) { // 接受一个左值引用
        n++;
    }
    int x = 10;
    increment(x); // x 被修改为 11
    ```
    
- **作用**：允许直接操作传入变量。
    

---

### **5. 右值引用（Rvalue Reference，C++11 引入）**

当使用 `&&` 时，表示右值引用。

- **示例**：
    
    ```cpp
    void process(int&& n) { // 接受一个右值引用
        n++;
    }
    process(10); // 10 是右值
    ```
    
- **作用**：右值引用通常用于移动语义和完美转发。
    

---

### **6. 在类型声明中的符号**

`&` 出现在函数返回类型或参数类型中，用于表示返回值或参数是一个引用。

- **示例**：
    
    ```cpp
    int x = 10;
    int& getRef() { return x; } // 返回 x 的引用
    int& ref = getRef();        // ref 是 x 的引用
    ```
    

---

### **7. 结合 `decltype` 推导类型**

在类型推导中，`&` 表示引用类型。

- **示例**：
    
    ```cpp
    int x = 10;
    decltype(x)& ref = x; // ref 是 int 的引用
    ```
    

---

### **8. 用于指针与引用类型**

当 `&` 出现在类型声明中时，它可以表示引用或指针类型的操作。

- **示例**：
    
    ```cpp
    int x = 10;
    int* ptr = &x;  // 指针，存储地址
    int& ref = x;   // 引用，别名
    ```
    

---

### **9. 重载运算符**

`&` 和 `&=` 运算符可以被重载为类的成员或非成员函数。

- **示例**：
    
    ```cpp
    class Example {
    public:
        Example& operator&(const Example& other) {
            // 自定义按位与操作
            return *this;
        }
    };
    ```
    