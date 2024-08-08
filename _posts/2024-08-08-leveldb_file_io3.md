---
title: leveldb file3 io write
date: 2024-08-08 20:25:00 +0800
categories: [Blogging, leveldb]
tags: [writing]
---

这里主要看下`WritableFile`下的细节，这里只有四个方法

```cpp
virtual Status Append(const Slice& data) = 0;
virtual Status Close() = 0;
virtual Status Flush() = 0;
virtual Status Sync() = 0;
```

appendfile和writable file都是一类file，区别在于oflag不一致, `O_TRUNC`表示如果文件存在，就将文件长度截断为0

```cpp
// append
int fd = ::open(filename.c_str(),
              O_TRUNC | O_WRONLY | O_CREAT | kOpenBaseFlags, 0644);

// wirte
int fd = ::open(filename.c_str(),
                O_APPEND | O_WRONLY | O_CREAT | kOpenBaseFlags, 0644);
```

在append函数下，默认有个65536=<uint16_t>::max的buffer，只有缓冲区满才会将数据写入磁盘文件。

如果遇到大量短数据的写入，也会通过memcpy来拷贝数据到buffer中，然后再写入磁盘文件，减少对底层文件系统的调用次数，提高写操作的效率。

```cpp
Status Append(const Slice& data) override {
  size_t write_size = data.size();
  const char* write_data = data.data();

  // Fit as much as possible into buffer.
  size_t copy_size = std::min(write_size, kWritableFileBufferSize - pos_);
  std::memcpy(buf_ + pos_, write_data, copy_size);
  write_data += copy_size;
  write_size -= copy_size;
  pos_ += copy_size;
  if (write_size == 0) {
    return Status::OK();
  }

  // Can't fit in buffer, so need to do at least one write.
  Status status = FlushBuffer();
  if (!status.ok()) {
    return status;
  }

  // Small writes go to buffer, large writes are written directly.
  if (write_size < kWritableFileBufferSize) {
    std::memcpy(buf_, write_data, write_size);
    pos_ = write_size;
    return Status::OK();
  }
  return WriteUnbuffered(write_data, write_size);
}
```

`WriteUnbuffered`的实现就是通过系统调用write实现

```cpp
Status WriteUnbuffered(const char* data, size_t size) {
  while (size > 0) {
    ssize_t write_result = ::write(fd_, data, size);
    if (write_result < 0) {
      if (errno == EINTR) {
        continue;  // Retry
      }
      return PosixError(filename_, errno);
    }
    data += write_result;
    size -= write_result;
  }
  return Status::OK();
}
```

WritableFile 还提供了 Flush 接口，用于将内存缓冲区 buf_ 的数据写入文件，它内部也是通过调用 WriteUnbuffered 来实现。不过值得注意的是，这里 Flush 写磁盘成功，并不保证数据已经写入磁盘，甚至不能保证磁盘有足够的空间来存储内容。

如果要做数据写磁盘成功的保证，需要调用`sync()`

```cpp
Status Sync() override {
  // Ensure new files referred to by the manifest are in the filesystem.
  //
  // This needs to happen before the manifest file is flushed to disk, to
  // avoid crashing in a state where the manifest refers to files that are not
  // yet on disk.
  Status status = SyncDirIfManifest();
  if (!status.ok()) {
    return status;
  }

  status = FlushBuffer();
  if (!status.ok()) {
    return status;
  }

  return SyncFd(fd_, filename_);
}

Status SyncDirIfManifest() {
  Status status;
  if (!is_manifest_) {
    return status;
  }

  int fd = ::open(dirname_.c_str(), O_RDONLY | kOpenBaseFlags);
  if (fd < 0) {
    status = PosixError(dirname_, errno);
  } else {
    status = SyncFd(fd, dirname_);
    ::close(fd);
  }
  return status;
}


// Ensures that all the caches associated with the given file descriptor's
// data are flushed all the way to durable media, and can withstand power
// failures.
//
// The path argument is only used to populate the description string in the
// returned Status if an error occurs.
static Status SyncFd(int fd, const std::string& fd_path) {
#if HAVE_FULLFSYNC
  // On macOS and iOS, fsync() doesn't guarantee durability past power
  // failures. fcntl(F_FULLFSYNC) is required for that purpose. Some
  // filesystems don't support fcntl(F_FULLFSYNC), and require a fallback to
  // fsync().
  if (::fcntl(fd, F_FULLFSYNC) == 0) {
    return Status::OK();
  }
#endif  // HAVE_FULLFSYNC

#if HAVE_FDATASYNC
  bool sync_success = ::fdatasync(fd) == 0;
#else
  bool sync_success = ::fsync(fd) == 0;
#endif  // HAVE_FDATASYNC

  if (sync_success) {
    return Status::OK();
  }
  return PosixError(fd_path, errno);
}
```

这里核心是调用 SyncFd() 方法，确保文件描述符 fd 关联的所有缓冲数据都被同步到物理磁盘。

在darwin下使用了`fcntl()`函数的`F_FULLFSYNC`选项来确保数据被同步到物理磁盘。如果定义了`HAVE_FDATASYNC`，将使用 `fdatasync()`来同步数据。其他情况下，默认使用`fsync()`函数来实现同样的功能。

`SyncDirIfManifest`确保如果文件是`manifest`文件(以`MANIFEST`开始命名的文件)，相关的目录更改也得到同步。

mainfest 文件记录数据库文件的元数据，包括版本信息、合并操作、数据库状态等关键信息。

文件系统在创建新文件或修改文件目录项时，这些变更可能并不立即写入磁盘。在更新`manifest`文件前确保所在目录的数据已被同步到磁盘，防止系统崩溃时，`manifest`文件引用的文件尚未真正写入磁盘。
