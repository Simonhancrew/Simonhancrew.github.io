---
title: linux的page cache
date: 2024-07-03 12:44:00 +0800
categories: [Blogging, linux, page cache, os]
tags: [writing]
---

# Page cache

Linux的Page Cache（页面缓存）是内核中的一种机制，用于缓存从磁盘读取的文件内容，以及缓存即将写入磁盘的数据。

## 为什么使用

### 1 快

cpu其实没法直接操作磁盘数据，其实只能经过一次内存之后再操作，这中间是有copy的

这种缓存机制可以显著提高文件访问的速度，因为访问内存中的数据比访问磁盘上的数据要快得多。

| type          | access time | typical size | $/MB      |
| ------------- | ----------- | ------------ | --------- |
| register      | < 0.5 ns    | ~256byte     | $1000     |
| SRAM/Cache    | 5ns         | 1-4MB        | $100      |
| DRAM/Memory   | 50ns        | GBs          | $0.01     |
| Magnetic Disk | 5ms         | TBs          | $0.000001 |

### 2 资源共享

不同的进程访问相同的文件是常见的。比如访问根目录，编译器和IDE同时访问源代码文件。如果有page cache，那么不同的程序可以访问同一个page cache。

+ The root diretory inode
+ The system log file
+ IDE and compiler accessing the same source code file
+ Multiple browser windows accessing the same (locally cached) image

如果用page cache的话，其实可以避免内存的浪费，不同的进程，其实在访问一个page cache

### 3 接口 + 操作流程

### 3.1 read

这里主要考虑怎么做同时更改

Access to the cache implicity accesses the disk.

```c
int c = read(fd, buf, 512);
```

1. Kernel translates local file descriptor to (inode, file offset)
2. Is inode in page cache?
   + No? read inode from disk and put in page cache
3. Find data block number for offset from inode
4. Is block in page cache?
   + No? read block from disk and put in page cache
5. Copy 512 bytes of cached disk block to user supplied buffer

### 3.2 write back/through + write allocate/no-allocate

写回，默认是delay的， 是write back

By default, kernel marks written pages dirty and flushes after a delay:

```c
write(fd, ”hello world”, 11);
```

1. Kernel writes "hello world" to page for cached disk
2. Kernel adds the page to the "dirty list"—pages that have been modified but not yet written to disk
3. Periodically, kernel writes all pages in dirty list to disk

另外在`open`的时候，flag位置可以控制写的行为

比如设置`O_SYNC`，就是write through

```c
int fd = open(”myfile”, O_SYNC);
write(fd, ”hello world”, 11);
```

这个时候，同样操作`hello world`到文件的流程其实不等了，同步直接刷到磁盘(也更新了page cache)，当然这样性能其实就比较低了

下面这个主要是cpu写cache miss的时候的行为，write-allocate的行为，这个也是write默认的。

By default, kernel adds writes of new blocks to the cache

1. Kernel writes "hello world" to a new block allocated in a cache page (no write to disk yet)
2. Kernel modifies inode as necessary (e.g. increase file size field in inode)
3. Kernel adds both page containing new data and inode’s page to dirty list

data at the missed-write location is loaded to cache, followed by a write-hit operation. In this approach, write misses are similar to read misses.

当然你也可以通过flags调整他，设置`O_DIRECT`，就是no-allocate

```c
char buf[512] = "...";
int fd = open(”myfile”, O_DIRECT);
write(fd, buf, 512);
```

data at the missed-write location is not loaded to cache, and is written directly to the backing store. In this approach, only the reads are being cached。No write allocate

Writes do not go through the page cache, but directly transfer process memory to disk.

Writes-back all pages from all processes related to same pages.

Generally you want to use write no allocate when the written data will not be read by the CPU in the near future, and you want to use write-allocate when the data will be read again soon.

## REF

1. [page-cache](https://www.cs.princeton.edu/courses/archive/fall19/cos316/lectures/11-page-cache.pdf)
