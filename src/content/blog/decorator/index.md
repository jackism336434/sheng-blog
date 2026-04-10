---

title: 'Python decorator'

publishDate: 'March 10, 2026'

updatedDate: 'March 10, 2026'

description: 'The tutorial and insight of Python decorator.'

tags:
- Python

- Advanced

language: 'Chinese'

heroImage: { src: 'thumbnail.jpg', color: '#9698C1' }

---

### 什么是Python装饰器


是一种高阶函数，能在不改变原函数代码的前提下，动态地增加/修改原函数的功能。由此，我们知道这是一个函数，所以它会有参数，也就是原函数，还会有返回值，也就是新的增加了功能的函数。

### 装饰器有什么用

Python这一设计核心在于解决“在不修改源代码的情况下，为原函数添加或者修改功能”这一设计问题。

#### 一、核心设计原则：对扩展开放，对修改关闭

在没有装饰器的情况下，我们要为函数添加功能需要在函数内部修改或者增添源码，这样会破坏原来的函数，增加了风险。并且我们想要让多个函数有同一功能时，需要重复写同样的逻辑。
而有了装饰器，函数的核心功能和其他辅助职责解耦，能让我们更加关注核心。

#### 二、实现声明式编程

装饰器将“如何做”的指令性代码，转变为“要什么”的声明式标记，使代码意图一目了然,利于后期维护。
```Python

# 声明式：读起来就像功能描述
@login_required
@cache(expire=300)
@log_call
def get_user_data(user_id):
    # 纯净的业务逻辑
    return db.query(user_id)

# 指令式：
def get_user_data(user_id):
    if not current_user.is_authenticated:  # 权限检查
        raise ...
    start = time.time()                     # 性能记录
    key = f"user_{user_id}"                 # 缓存逻辑
    if key in cache:
        return cache[key]
    result = db.query(user_id)             # 业务逻辑（被淹没）
    cache[key] = result
    print(f"耗时: {time.time() - start}")
    return result
```


### 怎么使用装饰器（decorator）

#### 最简单的装饰器

```python
from functools import wraps

def basic_decorator(func):
    
    @wraps(func)  # 保留原函数元信息
    def wrapper(*args, **kwargs): #返回的新函数
        
        print(f"开始执行 {func.__name__}")
        
        # 调用原函数
        result = func(*args, **kwargs)
        
        
        print(f"结束执行 {func.__name__}")
        return result   #保证新函数返回值与原函数相同
    
    return wrapper

# 使用装饰器
@basic_decorator
def add(a, b):  #我们要修改的原函数
    """两数相加"""
    return a + b

# 测试
print(add(3, 5))  
```

@basic_decorator本质是一个Python的语法糖，我们可以理解为
`add=basic_decorator(add)`。代码本身二层嵌套的结构，但是如果我们想要让装饰器本身也传入参数，这就需要我们进一步拓展

#### 带参数的装饰器

```python

def decorator_with_args(arg1, arg2, ...):
    """第一层：接收装饰器参数"""
    def actual_decorator(func):
        """第二层：接收被装饰函数"""
        def wrapper(*args, **kwargs):
            """第三层：接收函数参数"""
            # 在这里可以使用arg1, arg2等装饰器参数
            result = func(*args, **kwargs)
            return result
        return wrapper
    return actual_decorator
```
这个代码本质是什么呢,我们来看一个具体的例子

``` python

# 定义带参数的装饰器
def repeat(n_times):
    def decorator(func):
        def wrapper(*args, **kwargs):
            for _ in range(n_times):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

# 使用装饰器（语法糖）
@repeat(n_times=3)
def say_hello(name):
    print(f"Hello, {name}!")

# 等价写法（实际执行过程）
def say_hello(name):
    print(f"Hello, {name}!")
temp = repeat(n_times=3)  # 1. 调用第一层，得到decorator函数
say_hello = temp(say_hello)  # 2. 用decorator装饰say_hello
# 等价于: say_hello = repeat(n_times=3)(say_hello)

# 调用结果
say_hello("Alice")  # 输出3次: Hello, Alice!
```

实际上，相比于原来的装饰器，就是多了一个传递参数的过程，所以多个一个嵌套结构，总体上是三层。

