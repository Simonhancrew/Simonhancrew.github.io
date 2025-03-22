---
title: binutils objcopy
date: 2025-03-22 13:42:00 +0800
categories: [Blogging, unix, binutils]
tags: [writing]
---


`objcopy` 是 GNU `binutils` 工具集中的一个实用工具，用于 **复制和转换目标文件**。它可以将目标文件从一种格式转换为另一种格式，同时支持对目标文件进行各种操作，如删除或添加段、修改符号表、提取或插入二进制数据等。

---

### **1. 主要功能**
- **文件格式转换**：将目标文件从一种格式（如 ELF）转换为另一种格式（如二进制）。
- **段操作**：删除、添加或重命名段。
- **符号表操作**：修改或删除符号表。
- **数据提取与插入**：从文件中提取二进制数据，或将二进制数据插入到文件中。
- **调试信息处理**：删除或保留调试信息。

---

### **2. 常用选项**

| 选项                                | 描述                                                |
| ----------------------------------- | --------------------------------------------------- |
| `-I <输入格式>`                     | 指定输入文件的格式（如 `binary`、`elf64-x86-64`）。 |
| `-O <输出格式>`                     | 指定输出文件的格式（如 `binary`、`elf64-x86-64`）。 |
| `-B <架构>`                         | 指定目标文件的架构（如 `i386`、`arm`）。            |
| `-S`                                | 删除调试信息。                                      |
| `-R <段名>`                         | 删除指定的段。                                      |
| `--add-section <段名>=<文件>`       | 添加一个新的段，并将其内容从指定文件中读取。        |
| `--remove-section <段名>`           | 删除指定的段。                                      |
| `--rename-section <旧名>=<新名>`    | 重命名段。                                          |
| `--set-section-flags <段名>=<标志>` | 设置段的标志（如 `alloc`、`readonly`）。            |
| `--strip-all`                       | 删除所有符号表和重定位信息。                        |
| `--strip-debug`                     | 只删除调试信息。                                    |
| `--keep-symbol <符号>`              | 保留指定的符号，删除其他符号。                      |
| `--only-keep-debug`                 | 生成一个只包含调试信息的文件。                      |
| `--extract-symbol`                  | 提取符号表并生成一个符号文件。                      |

---

### **3. 使用示例**

#### **(1) 将 ELF 文件转换为二进制文件**
```bash
objcopy -O binary my_program my_program.bin
```
- 将 `my_program`（ELF 格式）转换为 `my_program.bin`（二进制格式）。

#### **(2) 删除调试信息**
```bash
objcopy -S my_program my_program_stripped
```
- 删除 `my_program` 中的调试信息，并保存为 `my_program_stripped`。

#### **(3) 删除指定的段**
```bash
objcopy -R .comment my_program my_program_no_comment
```
- 删除 `my_program` 中的 `.comment` 段，并保存为 `my_program_no_comment`。

#### **(4) 添加新的段**
```bash
objcopy --add-section .new_section=section_data.bin my_program my_program_new_section
```
- 从 `section_data.bin` 文件中读取数据，并将其作为 `.new_section` 添加到 `my_program` 中，保存为 `my_program_new_section`。

#### **(5) 重命名段**
```bash
objcopy --rename-section .data=.mydata my_program my_program_renamed
```
- 将 `my_program` 中的 `.data` 段重命名为 `.mydata`，并保存为 `my_program_renamed`。

#### **(6) 设置段的标志**
```bash
objcopy --set-section-flags .mydata=alloc,readonly my_program my_program_flags
```
- 将 `my_program` 中的 `.mydata` 段设置为可分配和只读，并保存为 `my_program_flags`。

#### **(7) 删除所有符号表和重定位信息**
```bash
objcopy --strip-all my_program my_program_stripped_all
```
- 删除 `my_program` 中的所有符号表和重定位信息，并保存为 `my_program_stripped_all`。

#### **(8) 只保留指定的符号**
```bash
objcopy --keep-symbol=main my_program my_program_main_only
```
- 只保留 `my_program` 中的 `main` 符号，删除其他符号，并保存为 `my_program_main_only`。

#### **(9) 生成只包含调试信息的文件**
```bash
objcopy --only-keep-debug my_program my_program.debug
```
- 从 `my_program` 中提取调试信息，并保存为 `my_program.debug`。

#### **(10) 提取符号表**
```bash
objcopy --extract-symbol my_program my_program.sym
```
- 从 `my_program` 中提取符号表，并保存为 `my_program.sym`。

---

### **4. 综合示例**
假设有一个 ELF 可执行文件 `my_program`，可以使用以下命令进行多种操作：

#### **(1) 转换为二进制文件并删除调试信息**
```bash
objcopy -O binary -S my_program my_program.bin
```

#### **(2) 删除 `.comment` 段并重命名 `.data` 段**
```bash
objcopy -R .comment --rename-section .data=.mydata my_program my_program_modified
```

#### **(3) 添加新的段并设置段标志**
```bash
objcopy --add-section .new_section=section_data.bin --set-section-flags .new_section=alloc,readonly my_program my_program_new
```

---

### **5. 如何分离调试信息**

cmake编译的时候使用RelWithDebugInfo, 这个时候是带-g的，具有调试信息

随后分离，常见的操作分为三部分

```shell
# 复制调试信息
objcopy --only-keep-debug a.out a.out.debug
# 删除调试信息
objcopy --strip-debug a.out
# 关联两者，方便gdb自动load
objcopy --add-gnu-debuglink=a.out.debug a.out
```
