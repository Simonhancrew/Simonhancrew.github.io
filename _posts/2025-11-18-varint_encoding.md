---
title: varint编码
date: 2025-11-18 17:01:00 +0800
categories: [Blogging, encoding, leveldb]
tags: [writing]
---

在leveldb的memtable里看到了存varint的字段，记一下这个。

在protobuf里其实也见到过

### 1. 原理

简单来讲，如果是32位数，会用1-5字节来表示这个数值，其中每个byte的most significant位，即数子表征的最大bit会作为是否需要继续往后扩充的依据

那么表征64位数的话，最大就要64 + 7 - 1 / 7 = 10字节来表示了

即每个byte，只有7bit是能用来有效表征数据的

### 2. 编码实现

直接看leveldb里一个64位的实现

```cpp
char* EncodeVarint64(char* dst, uint64_t v) {
  static const int B = 128;
  uint8_t* ptr = reinterpret_cast<uint8_t*>(dst);
  while (v >= B) {
    *(ptr++) = v | B;
    v >>= 7;
  }
  *(ptr++) = static_cast<uint8_t>(v);
  return reinterpret_cast<char*>(ptr);
}
```

这个B是128=1<<7, 只要比这个大就表示需要进位了，把v的低7位先写入

解码的就是凡过程来一次

```cpp
const char* GetVarint64Ptr(const char* p, const char* limit, uint64_t* value) {
  uint64_t result = 0;
  for (uint32_t shift = 0; shift <= 63 && p < limit; shift += 7) {
    uint64_t byte = *(reinterpret_cast<const uint8_t*>(p));
    p++;
    if (byte & 128) {
      // More bytes are present
      result |= ((byte & 127) << shift);
    } else {
      result |= (byte << shift);
      *value = result;
      return reinterpret_cast<const char*>(p);
    }
  }
  return nullptr;
}

bool GetVarint64(Slice* input, uint64_t* value) {
  const char* p = input->data();
  const char* limit = p + input->size();
  const char* q = GetVarint64Ptr(p, limit, value);
  if (q == nullptr) {
    return false;
  } else {
    *input = Slice(q, limit - q);
    return true;
  }
}
```

这里主要解码的步骤在`GetVarint64Ptr`, 因为最多10字节表征，所以左shift的最大是9，然后对每个byte值做判断要不要继续扩

最后返回终点右侧的指针

