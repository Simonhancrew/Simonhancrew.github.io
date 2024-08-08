---
title: leveldb file io 1 seqio
date: 2024-08-05 21:53:00 +0800
categories: [Blogging, leveldb]
tags: [writing]
---

## leveldb file io

env.h中抽象出了一些文件io的接口，针对这些抽象的接口，同时也抽象出了文件的抽象类，比如

1. `SequentialFile`
2. `RandomAccessFile`
3. `WritableFile`

### SequentialFile文件io接口

重要的接口就2个

```cpp
// Read up to "n" bytes from the file.  "scratch[0..n-1]" may be
// written by this routine.  Sets "*result" to the data that was
// read (including if fewer than "n" bytes were successfully read).
// May set "*result" to point at data in "scratch[0..n-1]", so
// "scratch[0..n-1]" must be live when "*result" is used.
// If an error was encountered, returns a non-OK status.
//
// REQUIRES: External synchronization
virtual Status Read(size_t n, Slice* result, char* scratch) = 0;

// Skip "n" bytes from the file. This is guaranteed to be no
// slower that reading the same data, but may be faster.
//
// If end of file is reached, skipping will stop at the end of the
// file, and Skip will return OK.
//
// REQUIRES: External synchronization
virtual Status Skip(uint64_t n) = 0;
```

具体的impl在`PosixSequentialFile`, 写的比较呆

```cpp
Status Read(size_t n, Slice* result, char* scratch) override {
  Status status;
  while (true) {
    ::ssize_t read_size = ::read(fd_, scratch, n);
    if (read_size < 0) {  // Read error.
      if (errno == EINTR) {
        continue;  // Retry
      }
      status = PosixError(filename_, errno);
      break;
    }
    *result = Slice(scratch, read_size);
    break;
  }
  return status;
}
```

scratch空间要开够，如果是nonblock会一直重试。

Skip的实现是直接用lseek的, SEEK_CUR表示从当前位置开始跳过 n 个字节

```cpp
Status Skip(uint64_t n) override {
  if (::lseek(fd_, n, SEEK_CUR) == static_cast<off_t>(-1)) {
    return PosixError(filename_, errno);
  }
  return Status::OK();
}
```

最后回头看一下构造跟析构的过程

```cpp
Status NewSequentialFile(const std::string& filename,
                          SequentialFile** result) override {
  int fd = ::open(filename.c_str(), O_RDONLY | kOpenBaseFlags);
  if (fd < 0) {
    *result = nullptr;
    return PosixError(filename, errno);
  }

  *result = new PosixSequentialFile(filename, fd);
  return Status::OK();
}

~PosixSequentialFile() override { close(fd_); }
```

这里类似一个工场函数，传入fd，filename

在析构的时候靠RAII来关闭fd，避免fd泄露导致core。
