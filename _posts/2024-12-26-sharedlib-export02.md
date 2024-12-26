---
title: 动态库符号导出02
date: 2024-12-26 12:10:00 +0800
categories: [Blogging, compile]
tags: [writing]
---

这里跟原来的文章[动态库符号导出](/posts/2023-12-14-sharedlib-export)一脉相承，只是补充了一些导出c++的类的最佳实践

c++里可以借助虚函数，导出的时候只要导出一个c的create当前类的实例的函数(避免name mangling), 然后通过这个函数创建实例，然后调用虚函数就可以了，外部需要知道这个抽象类的结构

使用者跟DLL提供者共用一个抽象类的头文件，使用者依赖于DLL的东西很少，只需要知道抽象类的接口，以及获取对象指针的导出函数，对象内存空间的申请是在DLL模块中做的，释放也在DLL模块中完成,最后记得要调用释放对象的函数。

上述其实能用智能指针接管，比较方便

实际的代码

```cpp
// The abstract interface for Xyz object.  
// No extra specifiers required.  
struct IXyz  
{  
    virtual int Foo(int n) = 0;  
    virtual void Release() = 0;  
};  
  
// Factory function that creates instances of the Xyz object.  
extern "C" XYZAPI IXyz* APIENTRY GetXyz(); 
```

实际使用的

```cpp

#include "XyzLibrary.h"  
  
...  
IXyz* pXyz = ::GetXyz();  
  
if(pXyz)  
{  
    pXyz->Foo(42);  
  
    pXyz->Release();  
    pXyz = NULL;  
}
```

这个背后的想法是：

+ 一个由纯虚函数组成的成员很少的类只不过是一个虚函数表，一个函数指针数组，在DLL范围内这个函数指针数组被它的作者填充任何他认为必需的东西。这样这个指针数组在DLL外部使用就是调用接口的实际上的实现。

### REF

1. [DLL导出类和函数](https://huangwang.github.io/2018/06/15/DLL%E5%AF%BC%E5%87%BA%E7%B1%BB%E5%92%8C%E5%87%BD%E6%95%B0/)
