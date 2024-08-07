---
title: leveldb file io-randomio
date: 2024-08-07 20:25:00 +0800
categories: [Blogging, leveldb]
tags: [writing]
---

RandomAccessFile 是一个抽象基类，定义随机读取文件的接口。它声明了一个纯虚函数 Read，强制子类实现这个方法。

只读的话，是支持多线程并发访问的，逐步看下实现

```cpp
Status NewRandomAccessFile(const std::string& f,
                            RandomAccessFile** r) override {
  return target_->NewRandomAccessFile(f, r);
}

Status NewRandomAccessFile(const std::string& filename,
                            RandomAccessFile** result) override {
  *result = nullptr;
  int fd = ::open(filename.c_str(), O_RDONLY | kOpenBaseFlags);
  if (fd < 0) {
    return PosixError(filename, errno);
  }

  if (!mmap_limiter_.Acquire()) {
    *result = new PosixRandomAccessFile(filename, fd, &fd_limiter_);
    return Status::OK();
  }

  uint64_t file_size;
  Status status = GetFileSize(filename, &file_size);
  if (status.ok()) {
    void* mmap_base =
        ::mmap(/*addr=*/nullptr, file_size, PROT_READ, MAP_SHARED, fd, 0);
    if (mmap_base != MAP_FAILED) {
      *result = new PosixMmapReadableFile(filename,
                                          reinterpret_cast<char*>(mmap_base),
                                          file_size, &mmap_limiter_);
    } else {
      status = PosixError(filename, errno);
    }
  }
  ::close(fd);
  if (!status.ok()) {
    mmap_limiter_.Release();
  }
  return status;
}
```

可以看到的是，在创建过程中，有两层limit，第一层来自mmap limit， 64位下默认是1000，否则就是0。如果超过了limit，就创建`PosixRandomAccessFile`

## PosixRandomAccessFile

这里的构造里能看到第二层limit, fd, 注意这里的fd在传入构造的时候是打开的fd。

```cpp
// The new instance takes ownership of |fd|. |fd_limiter| must outlive this
// instance, and will be used to determine if .
PosixRandomAccessFile(std::string filename, int fd, Limiter* fd_limiter)
    : has_permanent_fd_(fd_limiter->Acquire()),
      fd_(has_permanent_fd_ ? fd : -1),
      fd_limiter_(fd_limiter),
      filename_(std::move(filename)) {
  if (!has_permanent_fd_) {
    assert(fd_ == -1);
    ::close(fd);  // The file will be opened on every read.
  }
}

~PosixRandomAccessFile() override {
  if (has_permanent_fd_) {
    assert(fd_ != -1);
    ::close(fd_);
    fd_limiter_->Release();
  }
}
```

此时，如果拿不到fd的limit，就close掉fd，因此其实每次读都有一个open的操作。这里的limit怎么拿，在posix下可以看下, 不设置默认值的话，一般是从getrlimit里拿，无线就拿INT_MAX, 否则就拿rlim_cur的20%。

```cpp
// Return the maximum number of read-only files to keep open.
int MaxOpenFiles() {
  if (g_open_read_only_file_limit >= 0) {
    return g_open_read_only_file_limit;
  }
#ifdef __Fuchsia__
  // Fuchsia doesn't implement getrlimit.
  g_open_read_only_file_limit = 50;
#else
  struct ::rlimit rlim;
  if (::getrlimit(RLIMIT_NOFILE, &rlim)) {
    // getrlimit failed, fallback to hard-coded default.
    g_open_read_only_file_limit = 50;
  } else if (rlim.rlim_cur == RLIM_INFINITY) {
    g_open_read_only_file_limit = std::numeric_limits<int>::max();
  } else {
    // Allow use of 20% of available file descriptors for read-only files.
    g_open_read_only_file_limit = rlim.rlim_cur / 5;
  }
#endif
  return g_open_read_only_file_limit;
}
```

主要看下read的实现

```cpp
Status Read(uint64_t offset, size_t n, Slice* result,
            char* scratch) const override {
  int fd = fd_;
  if (!has_permanent_fd_) {
    fd = ::open(filename_.c_str(), O_RDONLY | kOpenBaseFlags);
    if (fd < 0) {
      return PosixError(filename_, errno);
    }
  }

  assert(fd != -1);

  Status status;
  ssize_t read_size = ::pread(fd, scratch, n, static_cast<off_t>(offset));
  *result = Slice(scratch, (read_size < 0) ? 0 : read_size);
  if (read_size < 0) {
    // An error: return a non-ok status.
    status = PosixError(filename_, errno);
  }
  if (!has_permanent_fd_) {
    // Close the temporary file descriptor opened earlier.
    assert(fd != fd_);
    ::close(fd);
  }
  return status;
}
```

区分是不是可持久fd，读是通过pread来读取，因此需要传入offset，和足够读的缓冲区。读完后，如果不是持久fd，就close掉。

## PosixMmapReadableFile

在之前构造函数的下半部分, 通过mmap来读取文件.

```cpp
uint64_t file_size;
Status status = GetFileSize(filename, &file_size);
if (status.ok()) {
  void* mmap_base =
      ::mmap(/*addr=*/nullptr, file_size, PROT_READ, MAP_SHARED, fd, 0);
  if (mmap_base != MAP_FAILED) {
    *result = new PosixMmapReadableFile(filename,
                                        reinterpret_cast<char*>(mmap_base),
                                        file_size, &mmap_limiter_);
  } else {
    status = PosixError(filename, errno);
  }
}
::close(fd);
if (!status.ok()) {
  mmap_limiter_.Release();
}
```

这里直接将文件映射到内存，随后关闭fd，mmap的prot和flags都标志了只读，映射的地址是0，这样就会由内核来选择合适的地址。

构造和析构显得平平无奇

```cpp
PosixMmapReadableFile(std::string filename, char* mmap_base, size_t length,
                      Limiter* mmap_limiter)
    : mmap_base_(mmap_base),
      length_(length),
      mmap_limiter_(mmap_limiter),
      filename_(std::move(filename)) {}

~PosixMmapReadableFile() override {
  ::munmap(static_cast<void*>(mmap_base_), length_);
  mmap_limiter_->Release();
}
```

这里的read，也是直接简单的内存offset读操作

```cpp
Status Read(uint64_t offset, size_t n, Slice* result,
            char* scratch) const override {
  if (offset + n > length_) {
    *result = Slice();
    return PosixError(filename_, EINVAL);
  }

  *result = Slice(mmap_base_ + offset, n);
  return Status::OK();
}
```

最后可以关注一下这里为什么要靠mmap做一个random access的实现。

### 1 避免系统调用

基于安全考虑操作系统划分了内核空间与用户空间。内核空间的概念如内核数据、内核代码、内核堆栈和内核堆段，都位于内核内存区域，对用户程序不可见，这称为内存分段管理机制。这种分段就是需要系统调用的原因。

系统调用可以防止用户程序跳转到内存中的任意位置并执行操作系统代码，如果用户程序需要调用内核代码，只能通过预定义的接口将控制权安全地移交给内核代码。当用户程序执行系统调用时，就会发生"环境切换"。

当我们使用 mmap 将文件映射到虚拟地址空间时，只需要一个系统调用：mmap(2) 。相反，如果我们使用 pread/pwrite 进行文件 I/O，则程序必须为每个读/写操作进行环境切换。这是为什么 mmap 应该提供更好的性能的原因。

环境切换缓慢的原因主要是由于 TLB(Translation Lookaside Buffer) 刷新。TLB 是 CPU 中利用局部性的最快的高速缓存。它缓存页表条目，这些条目存储在每个进程名为 的内存描述符中 mm_struct，该描述符包含进程的所有内存区域。因此，当发生环境切换时，TLB 的 PTE(Page Table Entry) 应该失效。

这种失效和随后的缓存重新填充是环境切换慢的原因。然而，这些都是古老的事实，现代计算机上的 TLB 是有标签 的，因此环境切换不一定会刷新计算机的 TLB。就像 armV8 文档中描述的那样：

**对于非全局条目，当更新 TLB 并将该条目标记为非全局时，除了正常的翻译信息之外，TLB 条目中还会存储一个值。该值称为地址空间 ID (ASID)，它是操作系统分配给每个单独任务的编号。如果当前 ASID 与条目中存储的 ASID 匹配，则后续 TLB 查找仅在该条目上匹配。这允许针对标记为非全局但具有不同 ASID 值的特定页面存在多个有效的 TLB 条目。换句话说，当进行环境切换时，我们不一定需要刷新 TLB。**

当发生环境切换时，CPU 可能会简单地更改活动 ASID，这就是 TLB 现在称为 Tagged TLB 的原因。这里的要点是，在现代计算机上，环境切换不像过去那么昂贵了，在 Intel 处理器上通常涉及以下步骤：

1. 将 SP（堆栈指针）切换到内核堆栈。
2. 保存用户模式 SP、PC（程序计数器）、特权模式
3. CPL（当前特权级别）从3更改为0。该级别与内存分段相关。
4. 将新 PC 设置为系统调用处理程序（通过代码映射）

在阿里云的 Intel(R) Xeon(R) Gold 6148 CPU @ 2.40GHz 云机器（2 核）上， 进程级别的环境切换平均需要约 1800ns，getpid 平均需要约 380ns

代码可以简单测测

```cpp
#include <assert.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/time.h>
#include <sys/wait.h>
#include <unistd.h>
#include <fcntl.h>
#include <sched.h>
#include <iostream>
#define N_ITERATIONS 1000000
long nanosec(struct timeval t) {
  return ((t.tv_sec * 1000000 + t.tv_usec) * 1000);
}
int main() {
  cpu_set_t set;
  CPU_ZERO( & set);
  CPU_SET(0, & set);
  long N_iterations = 1000000;
  // creating two pipes
  int p1[2]; // parent -> child
  int p2[2]; // child -> parent
  if (pipe(p1) == -1)
    exit(1);
  if (pipe(p2) == -1)
    exit(2);
  switch (fork()) {
  case -1: // probabaly due to resource limit
    exit(3);
  case 0:
    // in child process
    // see https://man7.org/linux/man-pages/man2/sched_setaffinity.2.html if you have any question
    if (sched_setaffinity(getpid(), sizeof(cpu_set_t), & set) == -1) {
      exit(4);
    }
    close(p1[1]);
    close(p2[0]);
    for (int i = 0; i < N_ITERATIONS; i++) {
      char buf[0];
      while (read(p1[0], & buf, sizeof(buf)) == -1) {}
      write(p2[1], & buf, sizeof(buf));
    }
    return 0;
  default:
    // in parent process
    if (sched_setaffinity(getpid(), sizeof(cpu_set_t), & set) == -1) {
      exit(5);
    }
    close(p1[0]);
    close(p2[1]);
    struct timeval start_time, end_time;
    int res;
    res = gettimeofday( & start_time, NULL);
    assert(res == 0);
    for (int i = 0; i < N_ITERATIONS; i++) {
      char buf[0];
      write(p1[1], & buf, sizeof(buf));
      while (read(p2[0], & buf, sizeof(buf)) == -1) {}
    }
    res = gettimeofday( & end_time, NULL);
    assert(res == 0);
    std::cout << "average nanoseconds to perform context switch is: " << (nanosec(end_time) - nanosec(start_time)) / N_iterations * 1.0 << std::endl;
    wait(NULL); // wait for child to exit
  }
  return 0;
}
```

```cpp
// syscall switch
#include <sys/time.h>
#include <unistd.h>
#include <assert.h>
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#define N_ITERATIONS 1000000
long nanosec(struct timeval t) {
  return ((t.tv_sec * 1000000 + t.tv_usec) * 1000);
}
int main() {
  int i, res;
  unsigned long pid;
  float avgTimeSysCall;
  struct timeval t1, t2;
  res = gettimeofday( & t1, NULL);
  assert(res == 0);
  for (i = 0; i < N_ITERATIONS; i++) {
    unsigned long syscall_code = 39; //syscall_64.tbl#L69
    asm("movq %1, %%rax\n" // use inline assembly here
      "syscall\n"
      "movq %%rax, %0\n": "=r"(pid): "m"(syscall_code): "rax", "rdi");
  }
  res = gettimeofday( & t2, NULL);
  assert(res == 0);
  printf("Average nanoseconds for System Call getpid(pid: %ld) is: %f\n", pid, (nanosec(t2) - nanosec(t1)) / (N_ITERATIONS * 1.0));
}
```

### 2 避免用户缓冲区

pread 需要用到 Linux 用户手册 read(2) 显示需要一个名为 buf 的用户空间缓冲区，这是数据复制到的用户空间缓冲区。

Linux 需要用户空间缓冲区的原因与处理器需要 L1 和 L2 缓存的原因相同：从文件系统读取（无论是通过网络（例如 NFS）还是本地磁盘）与从内存读取相比非常慢。如果数据库执行大量 I/O 密集型操作，那么使用 mmap 会更快，因为不涉及用户缓冲区，因此无需将数据从内核缓冲区高速缓存复制到用户缓冲区。

### 3 缺点

mmap 是由 SunOS 人员引入 Unix 的，目的是在进程之间共享库对象文件。当多个进程共享底层相同的库对象时，数据一致性就成为一个问题，这当然可以通过被称为 private sharing(copy-on-write and demand paging) 的方法来解决。但是这个并不是万能的。

+ mmap 创建内存映射区域并将它们添加到区域树中。
+ munmap 从树中删除区域并使硬件页表结构中的条目无效。
+ 页错误在进程的区域树中查找导致页错误的虚拟地址，如果虚拟地址已映射，则在页表中添加一条虚拟地址-物理地址的映射并恢复应用程序。

在[Is there a problem for prometheus to use mmap?](https://bo-er.github.io/2023-12-21-It-there-a-problem-for-prometheus-to-use-mmap/)中描述到，mmap 有一些缺点：

1. 事务安全
2. I/O 停顿
     + 没有异步IO接口。
     + IO 停顿是不可预测的，因为页面驱逐取决于操作系统。
3. 错误处理
4. 性能问题
    + 页表争用
    + 单线程页面驱逐
    + TLB 被击落

后面可以详细看下

## REF

1. <https://linux-kernel-labs.github.io/refs/heads/master/labs/memory_mapping.html>
2. <https://grafana.com/blog/2020/06/10/new-in-prometheus-v2.19.0-memory-mapping-of-full-chunks-of-the-head-block-reduces-memory-usage-by-as-much-as-40/>
3. <https://github.com/prometheus/prometheus>
4. <http://books.gigatux.nl/mirror/kerneldevelopment/0672327201/ch14lev1sec2.html>
5. <http://web.cs.ucla.edu/classes/honors/UPLOADS/kousha/thesis.pdf>
6. <http://www.bitsavers.org/pdf/sun/sunos/4.0/800-1731-07r50_Release_4.0_Change_Notes_for_the_Sun_Workstation_198701.pdf>
7. <https://db.cs.cmu.edu/papers/2022/cidr2022-p13-crotty.pdf>
8. <https://www.singlestore.com/blog/linux-off-cpu-investigation/>
9. <https://www.kernel.org/doc/html/v4.18/core-api/cachetlb.html>
10. <https://www.kernel.org/doc/gorman/html/understand/understand007.html>
11. <https://www.usenix.org/system/files/atc20-papagiannis.pdf>
12. <https://courses.cs.washington.edu/courses/cse551/15sp/notes/talk-rcu.pdf>
13. <https://www.kernel.org/doc/html/v4.18/core-api/cachetlb.html>
14. <https://valyala.medium.com/mmap-in-go-considered-harmful-d92a25cb161d>
