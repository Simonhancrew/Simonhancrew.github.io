---
title: binutils nm
date: 2025-03-22 13:41:00 +0800
categories: [Blogging, unix, binutils]
tags: [writing]
---

`nm` 是一个用于显示目标文件（如 `.o` 文件、静态库 `.a` 文件或共享库 `.so` 文件）中符号信息的工具。它可以帮助你查看符号的名称、类型、可见性等。

---

### **1. `nm` 输出的符号可见性**
`nm` 输出的符号可见性主要通过符号的类型字符来表示。以下是常见的符号类型及其含义：

| 符号类型 | 含义                                                          |
| -------- | ------------------------------------------------------------- |
| `A`      | 全局绝对符号（Absolute），通常用于未定义的符号。              |
| `B`      | 未初始化的全局数据（BSS）。                                   |
| `C`      | 未初始化的公共符号（Common）。                                |
| `D`      | 已初始化的全局数据（Data）。                                  |
| `G`      | 已初始化的全局数据（Small Data）。                            |
| `I`      | 间接引用其他符号的符号（Indirect）。                          |
| `N`      | 调试符号（Debugging symbol）。                                |
| `R`      | 只读数据（Read-only data）。                                  |
| `S`      | 未初始化的全局数据（Small BSS）。                             |
| `T`      | 全局函数或代码段中的符号（Text）。                            |
| `U`      | 未定义的符号（Undefined），通常表示需要从其他库中解析的符号。 |
| `V`      | 弱符号（Weak symbol），可以被其他符号覆盖。                   |
| `W`      | 未标记的弱符号（Weak symbol without a default value）。       |
| `-`      | 调试符号（Debugging symbol）。                                |
| `?`      | 未知符号类型。                                                |

---

### **2. 符号可见性的定义**
符号的可见性（Visibility）是指符号在链接时是否可以被其他目标文件或库访问。符号可见性通常分为以下几种：

#### **(1) 全局符号（Global）**
- **定义**：全局符号可以被其他目标文件或库访问。
- **`nm` 输出**：符号类型为大写字母（如 `T`、`D`、`B` 等）。
- **示例**：
  ```bash
  0000000000000000 T main
  ```
  `T` 表示 `main` 是一个全局函数。

#### **(2) 局部符号（Local）**
- **定义**：局部符号只能在定义它的目标文件中访问。
- **`nm` 输出**：符号类型为小写字母（如 `t`、`d`、`b` 等）。
- **示例**：
  ```bash
  0000000000000000 t helper_function
  ```
  `t` 表示 `helper_function` 是一个局部函数。

#### **(3) 弱符号（Weak）**
- **定义**：弱符号可以被其他符号覆盖，通常用于提供默认实现。
- **`nm` 输出**：符号类型为 `V` 或 `W`。
- **示例**：
  ```bash
  0000000000000000 V default_function
  ```
  `V` 表示 `default_function` 是一个弱符号。

#### **(4) 未定义符号（Undefined）**
- **定义**：未定义符号需要在链接时从其他目标文件或库中解析。
- **`nm` 输出**：符号类型为 `U`。
- **示例**：
  ```bash
                  U printf
  ```
  `U` 表示 `printf` 是一个未定义的符号。

---

### **3. `nm` 输出示例**
假设有一个目标文件 `example.o`，`nm` 的输出可能如下：
```bash
0000000000000000 T main
0000000000000010 t helper_function
0000000000000020 D global_variable
0000000000000030 B uninitialized_variable
                  U printf
0000000000000040 V weak_function
```

- `main` 是一个全局函数。
- `helper_function` 是一个局部函数。
- `global_variable` 是一个已初始化的全局变量。
- `uninitialized_variable` 是一个未初始化的全局变量。
- `printf` 是一个未定义的符号。
- `weak_function` 是一个弱符号。

---

### **4. 控制符号可见性**
在编程时，可以通过以下方式控制符号的可见性：

#### **(1) `static` 关键字**
将符号定义为 `static`，使其成为局部符号：
```c
static void helper_function() {
    // ...
}
```

#### **(2) `__attribute__((visibility("...")))`**
使用 GCC 的 `visibility` 属性显式控制符号的可见性：
```c
__attribute__((visibility("default"))) void public_function() {
    // ...
}

__attribute__((visibility("hidden"))) void private_function() {
    // ...
}
```

#### **(3) 链接器脚本**
通过链接器脚本控制符号的可见性。

---

### **5. 总结**
- 全局符号（大写字母）可以被其他目标文件或库访问。
- 局部符号（小写字母）只能在定义它的目标文件中访问。
- 弱符号（`V` 或 `W`）可以被其他符号覆盖。
- 未定义符号（`U`）需要在链接时解析。
