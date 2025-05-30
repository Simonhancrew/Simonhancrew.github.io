---
title: windows encoding
date: 2025-05-30 23:18:00 +0800
categories: [Blogging, encoding]
tags: [writing]
---

除了windows，别的系统都默认utf-8编码

windows系统上通常是跟随系统设置，如果系统选择为中文地区的话，默认为GBK编码。

推荐的方案是把源码都保存成utf-8（不带bom）

### 跟win的api交互

推荐的方案是把字符转换成UTF-16，然后再去调用系统的api，尽量不要用ansi的，用带w的api。

调用系统的api最好用带w的，这样会避免获取到ansi的字符编码，然后你再转成utf-8放到程序内部

简单的判断，代码中跟文件系统相关且以ANSI字符串做出参/入参的方法都是不安全的，只要路径中包含当前字符编码无法表示的字符就会导致异常

### WideCharToMultiByte

UNICODE到ANSI这部分其实是可能会出错的，你看这个api的定义也知道了

```cpp
UINT WideCharToMultiByte(
  UINT CodePage,
  DWORD dwFlags,
  LPCWSTR lpWideCharStr,
  int cchWideChar,
  LPSTR lpMultiByteStr,
  int cbMultiByte,
  LPCSTR lpDefaultChar,
  LPBOOL lpUsedDefaultChar
);
```

`WideCharToMultiByte` 是 Windows API 中的一个函数，用于将宽字符字符串（通常是 UTF-16 编码）转换为多字节字符串（如 UTF-8 或 ANSI）。它是处理 Unicode 和多字节编码之间转换的核心工具之一.


#### 参数详解：

1. **`CodePage`**：
   - 指定目标多字节字符串的代码页（code page）。
   - 常见的代码页包括：
     - `CP_ACP`：系统默认的 ANSI 代码页（通常是 Windows-1252 或其他本地化代码页）。
     - `CP_UTF8`：UTF-8 编码。
     - 自定义代码页（如 `CP_OEMCP` 或其他系统支持的代码页）。

2. **`dwFlags`**：
   - 控制转换行为的标志位。
   - 常见标志包括：
     - `WC_COMPOSITECHECK`：尝试将复合字符（如带音标的字符）分解为基本字符和组合符。
     - `WC_DEFAULTCHAR`：如果无法转换某个字符，则使用默认字符替换。
     - `WC_NO_BEST_FIT_CHARS`：禁用最佳拟合字符映射（避免将宽字符映射到不合适的窄字符）。
     - 其他标志可以根据需求组合使用。

3. **`lpWideCharStr`**：
   - 指向要转换的宽字符字符串（`LPCWSTR`），通常是 UTF-16 编码。

4. **`cchWideChar`**：
   - 指定宽字符字符串的长度（以字符为单位）。
   - 如果为 `-1`，则假定字符串以 `\0` 结尾。

5. **`lpMultiByteStr`**：
   - 指向接收多字节字符串的缓冲区（`LPSTR`）。

6. **`cbMultiByte`**：
   - 指定缓冲区的大小（以字节为单位）。
   - 如果为 `-1`，函数将返回所需缓冲区大小而不执行实际转换。

7. **`lpDefaultChar`**：
   - 指定在转换失败时使用的默认字符。
   - 如果为 `NULL`，则使用系统默认字符。

8. **`lpUsedDefaultChar`**：
   - 一个布尔值指针，指示在转换过程中是否使用了默认字符。
   - 如果为 `NULL`，则忽略此功能。

---

### 2. **工作原理**

`WideCharToMultiByte` 的核心任务是将宽字符字符串（UTF-16 编码）转换为目标代码页的多字节字符串。以下是其工作流程：

#### （1）输入验证
- 检查 `lpWideCharStr` 是否有效。
- 确认 `cchWideChar` 和 `cbMultiByte` 是否合理。

#### （2）确定目标代码页
- 根据 `CodePage` 参数选择目标代码页。
- 如果是 `CP_UTF8`，则执行 UTF-16 到 UTF-8 的转换；如果是 `CP_ACP`，则根据系统区域设置进行转换。

#### （3）执行转换
- 遍历宽字符字符串中的每个字符。
- 将每个宽字符映射到目标代码页的对应多字节序列。
- 如果遇到无法映射的字符：
  - 如果启用了 `WC_DEFAULTCHAR` 标志，则使用默认字符代替。
  - 否则，返回错误或停止转换。

#### （4）存储结果
- 将生成的多字节字符串存储到 `lpMultiByteStr` 缓冲区中。
- 如果缓冲区不足，返回所需的最小缓冲区大小。


### 4. **常见问题与注意事项**

#### （1）缓冲区大小不足
- 如果缓冲区大小不足以容纳转换后的多字节字符串，`WideCharToMultiByte` 会返回所需的最小缓冲区大小。
- 你可以先调用一次 `WideCharToMultiByte` 并传递 `cbMultiByte = -1` 来获取所需缓冲区大小。

#### （2）默认字符的使用

- 如果某些宽字符无法映射到目标代码页，可以启用 `WC_DEFAULTCHAR` 标志并提供默认字符。
- 默认字符通常由系统决定，可以通过 `lpUsedDefaultChar` 检测是否使用了默认字符。

#### （3）性能优化

- 对于大规模字符串转换，建议一次性分配足够的缓冲区以避免多次调用。

#### （4）编码选择
- 如果目标是通用的文本处理，优先选择 `CP_UTF8` 以支持多种语言。
- 如果目标是本地化的 ANSI 文本，可以选择 `CP_ACP` 或自定义代码页。

### REF

1. [UTF-8 遍地开花](https://utf8everywhere.org/zh-cn#)
