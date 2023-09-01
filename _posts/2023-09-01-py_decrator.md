---
title: decorator
author: simon
date: 2023-09-01
category: tools
layout: post
---

## decorator

### 无参装饰器

```python
def decorator(func):
  print('processing decorator')
  def wrapper():
    print("before")
    func()
    print("after")
  return wrapper

@decorator
def test():
  print("test")
```
输出
```
processing decorator
```

无参装饰器，在装饰器函数内部定义一个内部函数，用于对原函数进行修饰。返回值要是一个函数的引用，这样才能替换原函数。

在用@装饰函数的时候，实际上执行了一次decorator函数，将test函数作为参数传入，然后将decorator函数的返回值作为test函数的引用。

执行逻辑:

1. 立刻执行decorator函数,将test函数作为参数传入, 返回内部函数的引用，等价于`decorated_test = decorator(test)`
2. 最后，解释器会使用装饰后的函数名重新绑定被装饰的函数，等价`test = decorated_test`，此时test函数已经被包装成了wrapper函数

### 有参装饰器
```python
def parameter_decorator(param):
  def decorator(func):
  # 在这里添加装饰逻辑
    return func # 返回被装饰的函数
  return decorator # 返回装饰器函数
 
@parameter_decorator('param_value')
def my_function():
   # 函数逻辑

```

1. 当解释器执行到装饰器部分时，会首先执行参数装饰器函数，将装饰器函数的返回值作为装饰器函数返回，等价`decorator = parameter_decorator('param_value')`
2. 然后，解释器会将被装饰的函数作为参数传递给装饰器函数，并执行装饰器函数，返回一个新的函数或方法作为装饰后的函数。等价`decorated_function = decorator(my_function)`
3. 最后，解释器会使用装饰后的函数名重新绑定它，等价`my_function = decorated_function`
