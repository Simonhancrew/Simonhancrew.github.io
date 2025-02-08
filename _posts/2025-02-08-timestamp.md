---
title: 时间戳
date: 2025-02-08 11:28:00 +0800
categories: [Blogging, timepoint, c++]
tags: [writing]
---

代码里出现了用了不同的获取时间戳的函数，导致在time的计算时出现了问题

部分用的是c++的代码

```cpp
uint64_t CppCurrentTime(void) {
  return std::chrono::duration_cast<std::chrono::milliseconds>(
             std::chrono::steady_clock::now().time_since_epoch())
      .count();
}

uint64_t CppCurrentUs(void) {
  return std::chrono::duration_cast<std::chrono::microseconds>(
             std::chrono::steady_clock::now().time_since_epoch())
      .count();
}
```

这个代码的实现依赖steady_clock，具体的实现我从[llvm里捞一下](https://github.com/llvm/llvm-project/blob/83783e8bec0eb25e06eff57a22b0419cdc2bce2c/libcxx/src/chrono.cpp#L213)


```cpp
#if _LIBCPP_HAS_MONOTONIC_CLOCK

#  if defined(__APPLE__)

// On Apple platforms, only CLOCK_UPTIME_RAW, CLOCK_MONOTONIC_RAW or
// mach_absolute_time are able to time functions in the nanosecond range.
// Furthermore, only CLOCK_MONOTONIC_RAW is truly monotonic, because it
// also counts cycles when the system is asleep. Thus, it is the only
// acceptable implementation of steady_clock.
static steady_clock::time_point __libcpp_steady_clock_now() {
  struct timespec tp;
  if (0 != clock_gettime(CLOCK_MONOTONIC_RAW, &tp))
    __throw_system_error(errno, "clock_gettime(CLOCK_MONOTONIC_RAW) failed");
  return steady_clock::time_point(seconds(tp.tv_sec) + nanoseconds(tp.tv_nsec));
}

#  elif defined(_LIBCPP_WIN32API)

// https://msdn.microsoft.com/en-us/library/windows/desktop/ms644905(v=vs.85).aspx says:
//    If the function fails, the return value is zero. <snip>
//    On systems that run Windows XP or later, the function will always succeed
//      and will thus never return zero.

static LARGE_INTEGER __QueryPerformanceFrequency() {
  LARGE_INTEGER val;
  (void)QueryPerformanceFrequency(&val);
  return val;
}

static steady_clock::time_point __libcpp_steady_clock_now() {
  static const LARGE_INTEGER freq = __QueryPerformanceFrequency();

  LARGE_INTEGER counter;
  (void)QueryPerformanceCounter(&counter);
  auto seconds   = counter.QuadPart / freq.QuadPart;
  auto fractions = counter.QuadPart % freq.QuadPart;
  auto dur       = seconds * nano::den + fractions * nano::den / freq.QuadPart;
  return steady_clock::time_point(steady_clock::duration(dur));
}

#  elif defined(__MVS__)

static steady_clock::time_point __libcpp_steady_clock_now() {
  struct timespec64 ts;
  if (0 != gettimeofdayMonotonic(&ts))
    __throw_system_error(errno, "failed to obtain time of day");

  return steady_clock::time_point(seconds(ts.tv_sec) + nanoseconds(ts.tv_nsec));
}

#  elif defined(__Fuchsia__)

static steady_clock::time_point __libcpp_steady_clock_now() noexcept {
  // Implicitly link against the vDSO system call ABI without
  // requiring the final link to specify -lzircon explicitly when
  // statically linking libc++.
#    pragma comment(lib, "zircon")

  return steady_clock::time_point(nanoseconds(_zx_clock_get_monotonic()));
}

#  elif defined(_LIBCPP_HAS_TIMESPEC_GET)

static steady_clock::time_point __libcpp_steady_clock_now() {
  struct timespec ts;
  if (timespec_get(&ts, TIME_MONOTONIC) != TIME_MONOTONIC)
    __throw_system_error(errno, "timespec_get(TIME_MONOTONIC) failed");
  return steady_clock::time_point(seconds(ts.tv_sec) + microseconds(ts.tv_nsec / 1000));
}

#  elif defined(_LIBCPP_HAS_CLOCK_GETTIME)

static steady_clock::time_point __libcpp_steady_clock_now() {
  struct timespec tp;
  if (0 != clock_gettime(CLOCK_MONOTONIC, &tp))
    __throw_system_error(errno, "clock_gettime(CLOCK_MONOTONIC) failed");
  return steady_clock::time_point(seconds(tp.tv_sec) + nanoseconds(tp.tv_nsec));
}

#  else
#    error "Monotonic clock not implemented on this platform"
#  endif

_LIBCPP_DIAGNOSTIC_PUSH
_LIBCPP_CLANG_DIAGNOSTIC_IGNORED("-Wdeprecated")
const bool steady_clock::is_steady;
_LIBCPP_DIAGNOSTIC_POP

steady_clock::time_point steady_clock::now() noexcept { return __libcpp_steady_clock_now(); }

#endif // _LIBCPP_HAS_MONOTONIC_CLOCK
```

另一个拿时间戳的函数直接用的mach的接口，c++里使用的是posix api

```cpp
#include <chrono>
#include <cstdio>
#include <cstdlib>
#include <mach/mach.h>
#include <mach/mach_time.h>
#include <sys/time.h>
#include <time.h>
#include <unistd.h>

namespace {

uint32_t timebase_info_numer = 0;
uint32_t timebase_info_denom = 0;

} // namespace

#define LIKELY(x) __builtin_expect(!!(x), 1)
#define UNLIKELY(x) __builtin_expect(!!(x), 0)

void TimeInit() {
  struct mach_timebase_info info;
  kern_return_t err;

  err = mach_timebase_info(&info);
  if (err != KERN_SUCCESS) {
    abort();
  }
  timebase_info_numer = info.numer;
  timebase_info_denom = info.denom;
  if (timebase_info_numer == 0 || timebase_info_denom == 0) {
    abort();
  }
}

uint64_t TickNs() {
  uint64_t abs_val;
  uint64_t abs_product;

  abs_val = mach_absolute_time();
  abs_product = abs_val * timebase_info_numer;
  if (UNLIKELY(abs_product < abs_val)) {
    return static_cast<uint64_t>((abs_val / timebase_info_denom) * timebase_info_numer);
  }
  return static_cast<uint64_t>(abs_product / timebase_info_denom);
}

uint64_t CppCurrentTime(void) {
  return std::chrono::duration_cast<std::chrono::milliseconds>(
             std::chrono::steady_clock::now().time_since_epoch())
      .count();
}

uint64_t CppCurrentUs(void) {
  return std::chrono::duration_cast<std::chrono::microseconds>(
             std::chrono::steady_clock::now().time_since_epoch())
      .count();
}

int main() {
  TimeInit();
  uint64_t tick = TickNs();
  uint64_t cpp_tick = CppCurrentUs();
  printf("tick: %llu\n", tick);
  printf("cpp_tick: %llu\n", cpp_tick);
  return 0;
}
```

实际这两个拿到的时间有点区别

```shell
tick: 2506764865904000
cpp_tick: 2507524801880
```

最后统一用一个了
