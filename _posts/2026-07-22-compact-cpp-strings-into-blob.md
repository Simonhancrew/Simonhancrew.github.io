---
title: 把 C++ 中的大量字符串压缩成一个二进制 Blob
date: 2026-07-22 02:30:00 +0800
categories: [Blogging, c++, optimization]
tags: [c++, binary-format, compression, serialization, package-size]
---

最近遇到一个很典型的包体积问题：一份构建期生成的 C++ 文件里保存了大量域名、IP 和配置字符串。代码本身没有复杂逻辑，主要内容却是这样的静态数据：

```cpp
constexpr const char* kHosts[] = {
    "api-01.edge.example.com",
    "api-02.edge.example.com",
    "api-03.edge.example.com",
    "log-01.edge.example.com",
    "log-02.edge.example.com",
};
```

这些字符串最终会进入可执行文件或动态库的只读数据段。编译器通常能合并完全相同的字符串字面量，却不会把 `api-01.edge.example.com` 和 `api-02.edge.example.com` 自动拆成公共前后缀。数据规模上来之后，重复字符会直接反映到产物大小中。

这类数据还有几个明显特点：

- 大量字符串拥有相同的前缀或后缀；
- 同一个字符串可能被多个业务表引用；
- 类型、区域、数量和索引通常都是小整数；
- 数据只在构建时变化，运行时以只读方式使用。

因此可以为它设计一个很小的领域专用二进制格式：构建时将字符串去重并拆分，生成紧凑 Blob；运行时由 C++ 按约定还原。它不是 gzip、zstd 这类通用压缩，而是一种针对数据形态的字典编码。

## 编译器为什么没有替我们完成这件事

完全相同的字符串，编译器和链接器有机会合并：

```cpp
const char* a = "edge.example.com";
const char* b = "edge.example.com";
```

但下面两个字符串只是部分相同：

```text
api-01.edge.example.com
api-02.edge.example.com
```

通常仍会在只读数据段中各保存一份完整内容。它们共有的 `.edge.example.com` 不会自然变成一个可复用字典项。与此同时，`const char*` 表还可能带来指针或重定位开销。

需要注意，我们真正关心的是最终 ELF、Mach-O、PE、动态库或安装包的体积，而不只是生成的 `.cpp` 有多少行。把 Blob 展开成 `0xab, 0xcd` 的 C++ 数组后，源文件文本甚至可能更大；收益来自编译后的字节数组更紧凑。如果连仓库体积和生成文件体积也需要控制，可以在构建期生成源码，或者直接嵌入二进制资源。

## 整体方案

整个流程可以分成构建期和运行时两部分：

```text
构建期：
JSON / YAML / 其他结构化配置
    -> 收集并去重字符串
    -> 选择公共前缀和后缀
    -> 拆分成 prefix + piece + suffix
    -> 整数转为 Varint
    -> 输出 Blob

运行时：
Blob
    -> 校验格式版本
    -> 读取三个字符串池
    -> 根据索引重建完整字符串
    -> 根据业务表中的字符串 ID 恢复配置
```

压缩收益主要来自四层：

1. 全局字符串去重；
2. 公共前后缀字典；
3. 业务表只保存字符串索引；
4. 小整数使用 Varint。

## 第一步：给完整字符串编号

先遍历所有配置，把每个不同字符串放入全局表：

```text
string ID 0 -> api-01.edge.example.com
string ID 1 -> api-02.edge.example.com
string ID 2 -> log-01.edge.example.com
```

如果一个字符串同时被多个区域或配置项使用，也只会进入字符串表一次。业务表中不再保存原文，只保存 ID：

```text
region A -> [0, 1]
region B -> [0, 2]
```

这一步解决的是“完全相同字符串重复出现”的问题。

## 第二步：寻找公共前缀和后缀

接下来处理“内容不同但结构相似”的字符串。以这三个值为例：

```text
api-01.edge.example.com
api-02.edge.example.com
log-01.edge.example.com
```

可以找出：

```text
公共前缀：api-
公共后缀：.edge.example.com
```

于是它们可以拆为：

```text
api- + 01 + .edge.example.com
api- + 02 + .edge.example.com
       log-01 + .edge.example.com
```

生成三个字符串池：

```text
prefix pool:
  [0] "api-"

suffix pool:
  [0] ".edge.example.com"

piece pool:
  [0] "01"
  [1] "02"
  [2] "log-01"
```

完整字符串则变成三个索引：

```text
string ID 0 -> (prefix=1, suffix=1, piece=0)
string ID 1 -> (prefix=1, suffix=1, piece=1)
string ID 2 -> (prefix=0, suffix=1, piece=2)
```

前缀和后缀索引从 1 开始，因为 0 用来表示“不使用”；piece 一定存在，即使它是空字符串，所以可以直接使用从 0 开始的索引。

重建逻辑就是：

```cpp
std::string value;

if (prefix_index != 0) {
  value += prefixes[prefix_index - 1];
}

value += pieces[piece_index];

if (suffix_index != 0) {
  value += suffixes[suffix_index - 1];
}
```

## 哪些前后缀值得进入字典

并不是所有公共片段都值得保存。比如一个只有两个字符的前缀，存入字典后还要增加长度、内容和引用索引，可能反而让 Blob 变大。

一个简单的估算方式是：

```text
savings = 使用次数 × 片段长度 - 片段长度 - 使用次数
```

右侧分别代表：

```text
原先重复保存的字节数
- 字典项本身占用的字节数
- 每次引用大致需要的索引字节数
```

例如长度为 17 的后缀被 20 个字符串使用：

```text
savings = 20 × 17 - 17 - 20 = 303
```

它明显值得进入字典。实际实现还可以增加一些约束：

- 候选片段至少 4 字节；
- 至少被两个字符串使用；
- 只保留收益大于 0 的候选；
- 按收益排序，只保留固定数量的前缀和后缀；
- 一个字符串同时匹配多个候选时，优先使用最长项。

候选生成可以直接两两比较字符串。对于构建期工具，这种近似 `O(n²)` 的做法在数据量有限时足够简单可靠；数据规模更大时，可以换成 Trie、排序后比较相邻字符串，或使用后缀相关的数据结构。

这里的选择算法是一种启发式方法，不保证全局最优。它的价值在于实现简单、格式稳定，并且很适合命名规律明显的配置字符串。

## 第三步：用 Varint 保存整数

Blob 里有大量数量、长度、类型和索引。这些值通常很小，如果统一使用 32 位整数，每个值都要占 4 字节。

Varint 每个字节使用低 7 位保存数据，最高位表示后面是否还有字节：

| 数字 | 编码结果 |
| ---: | --- |
| 0 | `00` |
| 1 | `01` |
| 127 | `7f` |
| 128 | `80 01` |
| 300 | `ac 02` |

写入代码很短：

```python
def write_varint(buffer, value):
    while value >= 0x80:
        buffer.append((value & 0x7f) | 0x80)
        value >>= 7
    buffer.append(value)
```

如果字符串池不到 128 项，大多数索引都只占 1 字节。前面的一个完整字符串记录通常只需要三个字节：

```text
01 01 00
```

分别表示前缀 1、后缀 1、piece 0。

## Blob 的具体布局

一个最小版本可以这样安排：

```text
+-------------------------------+
| Magic + Version               |
+-------------------------------+
| Prefix String Pool            |
+-------------------------------+
| Suffix String Pool            |
+-------------------------------+
| Piece String Pool             |
+-------------------------------+
| String Assembly Table         |
+-------------------------------+
| Business Index Tables         |
+-------------------------------+
```

字符串池使用统一格式：

```text
string_count: varint

重复 string_count 次：
    utf8_byte_length: varint
    utf8_bytes[utf8_byte_length]
```

完整字符串拼装表是：

```text
string_count: varint

重复 string_count 次：
    prefix_index: varint
    suffix_index: varint
    piece_index:  varint
```

业务表则只需要保存结构信息和完整字符串 ID。例如：

```text
group_count: varint

重复 group_count 次：
    group_type: varint
    value_count: varint
    string_id[value_count]: varint
```

协议里不依赖 C++ 结构体布局，也没有 padding。解析器维护一个游标，按照数量和长度连续向后读取。

## 一个逐字节示例

假设前缀池只有一个字符串：

```text
"api-"
```

它的 UTF-8 字节是：

```text
61 70 69 2d
```

前缀池在 Blob 中就是：

```text
01                // 前缀数量 = 1
04                // 第 0 个前缀长度 = 4
61 70 69 2d       // "api-"
```

如果前缀池为空，则只写一个字节：

```text
00
```

解析器读到数量 0 后，不读取任何前缀；下一个字节已经是后缀池的数量。Blob 不需要额外的分隔符，因为每一段都能通过 count 和 length 确定边界。

如果不希望最终二进制中直接出现可搜索的明文，可以在写入字符串时做逐字节变换，在读取时执行逆变换。但简单 XOR 只能算混淆，不是加密，也不会带来压缩收益。涉及真正的敏感信息时，不应把密钥和密文一起放进客户端程序。

## 一组完整可运行的代码

前面的代码只展示了局部逻辑。下面给出一组可以直接串起来运行的最小版本：Python 负责分析字符串并生成 `generated_blob.h`，C++17 负责读取这个头文件中的 Blob 并还原全部字符串。

这个示例只编码字符串模型，没有加入具体业务表。真实项目可以在拼装表后继续追加分组、区域、类型以及 `string_id`，编码方式仍然是 count、Varint 和索引引用。

### Python：生成 Blob

保存为 `generate_blob.py`：

```python
#!/usr/bin/env python3

from pathlib import Path


MAGIC = b"STB1"
MIN_AFFIX_LENGTH = 4
MAX_AFFIXES = 32


def write_varint(buffer, value):
    while value >= 0x80:
        buffer.append((value & 0x7F) | 0x80)
        value >>= 7
    buffer.append(value)


def common_prefix(lhs, rhs):
    end = 0
    while end < min(len(lhs), len(rhs)) and lhs[end] == rhs[end]:
        end += 1
    return lhs[:end]


def common_suffix(lhs, rhs):
    length = 0
    while length < min(len(lhs), len(rhs)) and lhs[-length - 1] == rhs[-length - 1]:
        length += 1
    return lhs[len(lhs) - length:] if length else ""


def choose_affixes(values, use_prefix):
    candidates = set()

    # 两两比较字符串，收集长度足够的公共前缀或后缀。
    for index, value in enumerate(values):
        for other in values[index + 1:]:
            match = common_prefix(value, other) if use_prefix else common_suffix(value, other)
            while len(match) >= MIN_AFFIX_LENGTH:
                candidates.add(match)
                match = match[:-1] if use_prefix else match[1:]

    scored = []
    for candidate in candidates:
        if use_prefix:
            uses = sum(value.startswith(candidate) for value in values)
        else:
            uses = sum(value.endswith(candidate) for value in values)

        byte_length = len(candidate.encode("utf-8"))
        savings = uses * byte_length - byte_length - uses
        if uses >= 2 and savings > 0:
            scored.append((savings, byte_length, candidate))

    scored.sort(reverse=True)

    # 嵌套候选只保留一个，避免同时选择 api- 和 api-a 等重叠项。
    selected = []
    for _, _, candidate in scored:
        if use_prefix:
            overlaps = any(
                candidate.startswith(item) or item.startswith(candidate)
                for item in selected
            )
        else:
            overlaps = any(
                candidate.endswith(item) or item.endswith(candidate)
                for item in selected
            )

        if not overlaps:
            selected.append(candidate)
        if len(selected) == MAX_AFFIXES:
            break

    return selected


def build_string_model(input_values):
    # dict 保留插入顺序，同时完成字符串去重。
    values = list(dict.fromkeys(input_values))
    prefixes = choose_affixes(values, use_prefix=True)
    suffixes = choose_affixes(values, use_prefix=False)

    # 0 表示没有前后缀，因此池内元素从 1 开始编号。
    prefix_ids = {value: index + 1 for index, value in enumerate(prefixes)}
    suffix_ids = {value: index + 1 for index, value in enumerate(suffixes)}
    pieces = []
    piece_ids = {}
    records = []

    for value in values:
        prefix = max(
            (item for item in prefixes if value.startswith(item)),
            key=len,
            default="",
        )
        stem = value[len(prefix):]

        # 必须在去掉前缀后的 stem 上匹配，避免前后缀相互覆盖。
        suffix = max(
            (item for item in suffixes if stem.endswith(item)),
            key=len,
            default="",
        )
        piece = stem[:-len(suffix)] if suffix else stem

        if piece not in piece_ids:
            piece_ids[piece] = len(pieces)
            pieces.append(piece)

        records.append((
            prefix_ids.get(prefix, 0),
            suffix_ids.get(suffix, 0),
            piece_ids[piece],
        ))

    return values, prefixes, suffixes, pieces, records


def write_string_pool(buffer, pool):
    write_varint(buffer, len(pool))
    for value in pool:
        raw = value.encode("utf-8")
        write_varint(buffer, len(raw))
        buffer.extend(raw)


def pack_strings(input_values):
    values, prefixes, suffixes, pieces, records = build_string_model(input_values)

    output = bytearray(MAGIC)
    write_string_pool(output, prefixes)
    write_string_pool(output, suffixes)
    write_string_pool(output, pieces)

    write_varint(output, len(values))
    for prefix_id, suffix_id, piece_id in records:
        write_varint(output, prefix_id)
        write_varint(output, suffix_id)
        write_varint(output, piece_id)

    return bytes(output)


def write_cpp_header(path, blob):
    rows = []
    for offset in range(0, len(blob), 12):
        chunk = blob[offset:offset + 12]
        rows.append("    " + ", ".join(f"0x{byte:02x}" for byte in chunk) + ",")

    content = "\n".join([
        "#pragma once",
        "#include <cstddef>",
        "",
        "inline constexpr unsigned char kStringBlob[] = {",
        *rows,
        "};",
        "inline constexpr std::size_t kStringBlobSize = sizeof(kStringBlob);",
        "",
    ])
    Path(path).write_text(content, encoding="utf-8")


if __name__ == "__main__":
    source_strings = [
        "api-a.edge.example.com",
        "api-b.edge.example.com",
        "log-a.edge.example.com",
    ]

    blob = pack_strings(source_strings)
    write_cpp_header("generated_blob.h", blob)

    raw_size = sum(len(value.encode("utf-8")) for value in source_strings)
    print(f"raw UTF-8 bytes: {raw_size}")
    print(f"blob bytes: {len(blob)}")
```

这段代码主要完成四件事：

1. `dict.fromkeys()` 对完整字符串去重并保持原始顺序；
2. `choose_affixes()` 找出有收益的公共前后缀；
3. `build_string_model()` 把每个字符串转换为三个索引；
4. `pack_strings()` 按固定顺序序列化，`write_cpp_header()` 再生成 C++ 数组。

### C++：解析并还原

保存为 `decode_blob.cpp`，并与生成的 `generated_blob.h` 放在同一目录：

```cpp
#include "generated_blob.h"

#include <cstddef>
#include <cstdint>
#include <iostream>
#include <limits>
#include <stdexcept>
#include <string>
#include <utility>
#include <vector>

class BlobReader {
 public:
  BlobReader(const unsigned char* data, std::size_t size)
      : current_(data), end_(data + size) {}

  std::uint8_t ReadByte() {
    if (current_ == end_) {
      throw std::runtime_error("unexpected end of blob");
    }
    return *current_++;
  }

  std::uint64_t ReadVarint() {
    std::uint64_t value = 0;
    for (unsigned index = 0; index < 10; ++index) {
      const std::uint8_t byte = ReadByte();

      // uint64_t 最多需要 10 个 Varint 字节，第 10 个只能使用最低位。
      if (index == 9 && (byte & 0xfe) != 0) {
        throw std::runtime_error("varint overflow");
      }

      value |= static_cast<std::uint64_t>(byte & 0x7f) << (index * 7);
      if ((byte & 0x80) == 0) {
        return value;
      }
    }
    throw std::runtime_error("invalid varint");
  }

  void Expect(const char* expected, std::size_t size) {
    for (std::size_t index = 0; index < size; ++index) {
      if (ReadByte() != static_cast<std::uint8_t>(expected[index])) {
        throw std::runtime_error("invalid blob header");
      }
    }
  }

  std::size_t Remaining() const {
    return static_cast<std::size_t>(end_ - current_);
  }

  bool AtEnd() const {
    return current_ == end_;
  }

 private:
  const unsigned char* current_;
  const unsigned char* end_;
};

std::size_t ReadSize(BlobReader& reader, std::size_t limit) {
  const std::uint64_t value = reader.ReadVarint();
  if (value > limit || value > std::numeric_limits<std::size_t>::max()) {
    throw std::runtime_error("blob size limit exceeded");
  }
  return static_cast<std::size_t>(value);
}

std::vector<std::string> ReadStringPool(BlobReader& reader) {
  constexpr std::size_t kMaxPoolItems = 1'000'000;
  constexpr std::size_t kMaxStringBytes = 1'000'000;

  const std::size_t count = ReadSize(reader, kMaxPoolItems);
  std::vector<std::string> pool;
  pool.reserve(count);

  for (std::size_t index = 0; index < count; ++index) {
    const std::size_t length = ReadSize(reader, kMaxStringBytes);
    if (length > reader.Remaining()) {
      throw std::runtime_error("truncated string");
    }

    std::string value(length, '\0');
    for (char& byte : value) {
      byte = static_cast<char>(reader.ReadByte());
    }
    pool.push_back(std::move(value));
  }

  return pool;
}

std::vector<std::string> DecodeStrings(
    const unsigned char* data,
    std::size_t size) {
  BlobReader reader(data, size);
  reader.Expect("STB1", 4);

  const auto prefixes = ReadStringPool(reader);
  const auto suffixes = ReadStringPool(reader);
  const auto pieces = ReadStringPool(reader);
  const std::size_t count = ReadSize(reader, 1'000'000);

  std::vector<std::string> strings;
  strings.reserve(count);

  for (std::size_t index = 0; index < count; ++index) {
    const std::uint64_t prefix_id = reader.ReadVarint();
    const std::uint64_t suffix_id = reader.ReadVarint();
    const std::uint64_t piece_id = reader.ReadVarint();

    if (prefix_id > prefixes.size() ||
        suffix_id > suffixes.size() ||
        piece_id >= pieces.size()) {
      throw std::runtime_error("invalid string model index");
    }

    std::string value;
    if (prefix_id != 0) {
      value += prefixes[prefix_id - 1];
    }
    value += pieces[piece_id];
    if (suffix_id != 0) {
      value += suffixes[suffix_id - 1];
    }
    strings.push_back(std::move(value));
  }

  if (!reader.AtEnd()) {
    throw std::runtime_error("unexpected trailing data");
  }
  return strings;
}

int main() {
  try {
    for (const auto& value : DecodeStrings(kStringBlob, kStringBlobSize)) {
      std::cout << value << '\n';
    }
  } catch (const std::exception& error) {
    std::cerr << "decode failed: " << error.what() << '\n';
    return 1;
  }
}
```

### 编译和运行

先生成 Blob 头文件，再编译 C++：

```bash
uv run generate_blob.py
clang++ -std=c++17 -Wall -Wextra -Wpedantic decode_blob.cpp -o decode_blob
./decode_blob
```

这组示例的实际输出是：

```text
raw UTF-8 bytes: 66
blob bytes: 50

api-a.edge.example.com
api-b.edge.example.com
log-a.edge.example.com
```

这里只有三个字符串，Blob 还承担了 4 字节魔数、三个池的数量、各字符串长度和拼装表等固定成本，仍然从 66 字节降到了 50 字节。随着共享后缀的字符串数量增加，字典项只保存一次，固定成本被摊薄，收益会更加明显。

这里的 66 字节只统计原始 UTF-8 内容，没有计算 C 字符串结尾的 `\0`、指针数组、对齐或重定位；50 字节则是完整字符串模型 Blob 的实际长度。它适合解释算法，但最终包体收益仍应按照下一节的方法在真实编译产物上测量。

## 包体积收益应该怎样测

不要只比较 JSON 和生成 `.cpp` 的文件大小。更有意义的是固定编译器、编译参数和链接参数，比较以下数据：

1. 动态库或可执行文件的未压缩大小；
2. `.rodata` 或对应只读段的大小；
3. strip 前后的产物大小；
4. APK、AAB、IPA、zip 等最终压缩包中的增量；
5. 解码后的峰值内存和启动耗时。

Linux/Android 环境可以使用：

```bash
size libexample.so
readelf -S -W libexample.so
nm -S --size-sort libexample.so
```

还可以分别构建改造前后的产物，然后使用相同压缩参数：

```bash
gzip -9 -c before.so > before.so.gz
gzip -9 -c after.so  > after.so.gz
```

自定义字典编码对未压缩二进制的收益通常比较直接，但最终安装包本身还会经过 deflate 或其他压缩。通用压缩器也能发现重复后缀，因此安装包里的净收益可能小于 `.so` 的减少量，必须以实际产物为准。

另外要防止把调试符号、链接顺序变化、LTO 设置变化等噪声算进结果。最可靠的方式是只改变数据表示，再对相同 section 和符号做比较。

## 运行时成本与取舍

压缩不是免费的。原本的字符串字面量可以直接引用，Blob 则需要解析和重建。需要考虑：

- 解码发生在启动阶段还是首次使用时；
- 重建后的 `std::string` 是否长期驻留；
- 是否可以只保留索引，在真正使用时按需拼接；
- 是否会因为大量小对象分配抵消内存收益；
- 多线程首次访问时如何保证初始化安全。

如果解码后长期保留所有完整字符串，磁盘包体积仍然会下降，但运行时内存未必同等下降。进一步优化可以使用连续字符缓冲区、offset 表和 `std::string_view`，减少每个 `std::string` 的对象与分配开销；也可以保留拆分形式，在使用某个字符串时才拼接。

对于多数静态配置，一个常见折中是：

```text
构建期尽可能压缩
-> 启动时一次性校验并解码
-> 结果作为只读配置保存
```

这样实现最简单，也容易测试。如果数据特别大或启动时间敏感，再考虑惰性解码。

## 格式演进和安全性

自定义格式最危险的地方不是编码，而是以后修改编码却忘了同步解码器。Blob 头部至少应该包含：

```text
magic
format version
payload size
```

根据数据来源，还可以增加 CRC32 或更强的完整性校验。解析器必须检查：

- 魔数和版本是否支持；
- Varint 是否溢出；
- count 和 length 是否超过合理上限；
- 每次读取是否越过 Blob 末尾；
- 所有前缀、后缀、piece 和完整字符串索引是否合法；
- 解析结束后是否还有未预期的尾部数据。

即使 Blob 当前只来自本地编译产物，也不应该依赖未检查的指针移动。边界检查不仅是安全要求，也能让编码器和解码器版本不一致时更快暴露问题。

测试方面，最重要的是往返测试：

```text
原始结构化数据
-> encode
-> decode
-> 与原始数据逐字段比较
```

还应覆盖空字符串、空表、非 ASCII UTF-8、超出单字节范围的 Varint、错误索引、截断 Blob 和未知版本。

## 什么时候值得这样做

这种方案适合字符串多、命名规律强、数据变化频率低，并且对包体积有明确要求的客户端组件。它的优势是格式简单、依赖少、运行时解码可控。

如果数据没有明显结构，或者规模已经大到值得引入成熟压缩库，那么 zstd、Brotli 等通用方案通常更合适。也可以组合使用：先做字符串去重和结构化编码，再对整个 payload 做一次通用压缩，不过这会增加解压代码、内存和启动成本。

这次优化最关键的思路，不是把 C++ 代码换成一串看不懂的十六进制，而是先看清数据的重复模式：完整重复用字符串池解决，部分重复用前后缀字典解决，结构引用用编号解决，小整数用 Varint 解决。Blob 只是把这些决定稳定地序列化下来。
