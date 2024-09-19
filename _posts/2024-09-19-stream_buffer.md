---
title: stream buffer设计
date: 2024-09-19 20:01:10 +0800
categories: [Blogging, stream buffer, muduo]
tags: [writing]
---

主要看下流式传输中的buffer设计，怎么做到高效的读写

这里以muduo的buffer类设计为例子

```ascii
/// @code
/// +-------------------+------------------+------------------+
/// | prependable bytes |  readable bytes  |  writable bytes  |
/// |                   |     (CONTENT)    |                  |
/// +-------------------+------------------+------------------+
/// |                   |                  |                  |
/// 0      <=      readerIndex   <=   writerIndex    <=     size
/// @endcode
```

prependable bytes大小8字节，一般选择是放整个整个buffer的长度，但是也有就放自定义header的

```cpp
void prepend(const void* /*restrict*/ data, size_t len)
{
  assert(len <= prependableBytes());
  readerIndex_ -= len;
  const char* d = static_cast<const char*>(data);
  std::copy(d, d+len, begin()+readerIndex_);
}
```

readable就是当前已经写入的有意义的数据，writeable就是还能写的缓存空间

当要发送整个buffer走的时候，要发送的是peek到后面readable的数据，然后`retrieveAll`，恢复可写长度。

```cpp
const char* peek() const
{ return begin() + readerIndex_; }


void retrieveAll()
{
  readerIndex_ = kCheapPrepend;
  writerIndex_ = kCheapPrepend;
}

static const size_t kCheapPrepend = 8;
```

实际扩容的时候，len已经超过writeable的长度，这时候分两种情况

1. 看看readerIndex到prependableBytes之间的数据是否能够放下len长度的数据，如果能放下，就把readerIndex到writerIndex之间的数据往前移动，然后把writerIndex往后移动len长度
2. 在后面直接扩容len长度

```cpp
void append(const char* /*restrict*/ data, size_t len)
{
  ensureWritableBytes(len);
  std::copy(data, data+len, beginWrite());
  hasWritten(len);
}

void ensureWritableBytes(size_t len)
{
  if (writableBytes() < len)
  {
    makeSpace(len);
  }
  assert(writableBytes() >= len);
}

void makeSpace(size_t len)
{
  if (writableBytes() + prependableBytes() < len + kCheapPrepend)
  {
    // FIXME: move readable data
    buffer_.resize(writerIndex_+len);
  }
  else
  {
    // move readable data to the front, make space inside buffer
    assert(kCheapPrepend < readerIndex_);
    size_t readable = readableBytes();
    std::copy(begin()+readerIndex_,
              begin()+writerIndex_,
              begin()+kCheapPrepend);
    readerIndex_ = kCheapPrepend;
    writerIndex_ = readerIndex_ + readable;
    assert(readable == readableBytes());
  }
}

size_t readableBytes() const
{ return writerIndex_ - readerIndex_; }

size_t writableBytes() const
{ return buffer_.size() - writerIndex_; }

```

然后就是读fd的数据部分的代码走的是readv，这里能读完就全部到buffer里，读不完就把buffer先读满，然后剩下的部分数据从extrabuf扩容到buffer里

```cpp
ssize_t Buffer::readFd(int fd, int* savedErrno)
{
  // saved an ioctl()/FIONREAD call to tell how much to read
  char extrabuf[65536];
  struct iovec vec[2];
  const size_t writable = writableBytes();
  vec[0].iov_base = begin()+writerIndex_;
  vec[0].iov_len = writable;
  vec[1].iov_base = extrabuf;
  vec[1].iov_len = sizeof extrabuf;
  // when there is enough space in this buffer, don't read into extrabuf.
  // when extrabuf is used, we read 128k-1 bytes at most.
  const int iovcnt = (writable < sizeof extrabuf) ? 2 : 1;
  const ssize_t n = sockets::readv(fd, vec, iovcnt);
  if (n < 0)
  {
    *savedErrno = errno;
  }
  else if (implicit_cast<size_t>(n) <= writable)
  {
    writerIndex_ += n;
  }
  else
  {
    writerIndex_ = buffer_.size();
    append(extrabuf, n - writable);
  }
  // if (n == writable + sizeof extrabuf)
  // {
  //   goto line_30;
  // }
  return n;
}
```
