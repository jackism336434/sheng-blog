---

title: 'Python Lambda'

publishDate: 'March 19, 2026'

updatedDate: 'March 19, 2026'

description: 'The experiment of Lambda.'

tags:
-  Python

-  Function

-  Lambda

language: 'Chinese'

heroImage: { src: './thumbnail.jpg', color: '#9698C1' }

---


#### Lambda的定义

在 Python 中，**lambda** 是一种匿名函数写法，常用于简短、一次性的运算逻辑。其语法为：

lambda 参数1, 参数2, ... : 表达式


#### 为什么要有Lambda

Lambda函数是一种**匿名函数**，也称为[内联函数]或者[函数字面量]。通常用于那些**简单的、一次性的函数**，这样可以避免定义一个完整的函数。例如，如果你只是想对一个列表的每个元素进行平方操作，你可以使用Lambda函数，而不需要定义一个单独的函数。既然Lambda作为函数，当然允许你将函数作为**参数传递**给其他函数，这样你就可以在需要的时候创建简单的、匿名的函数。
#### 对Lambda的探索


1.实验一：对比普通函数

```python
# 普通函数：有名字、多行、完整结构
def square(x):
	return x * x 

 print(square(5))


# Lambda 函数：匿名、一行、极简 
square_lambda = lambda x: x * x 

print(square_lambda(5))

```

我们发现，这里Lambda函数的返回值是一个函数对象，但是通常情况下，我们一般不考虑这样写，而是直接写Lambda的表达式


2.实验二：带多个参数的Lambda表达式

```python
add_lambda = lambda a, b: a + b 

print(add_lambda(3, 4))
```


3.实验三：Lambda在实际项目中的应用(搭配高阶函数[[map，filter，reduce]])

需求：把列表里所有数字 ×2

```python
nums = [1, 2, 3, 4] 
# map(函数, 列表)：用函数批量处理列表所有元素 
result = map(lambda x: x * 2, nums) 

print(list(result))

```

需求：筛选出大于 3 的数字

```python

nums = [1, 2, 3, 4, 5] 

result = filter(lambda x: x > 3, nums) 

print(list(result))
```

需求：按列表里的**第二个元素**排序

``` python


data = [(1, 5), (3, 2), (2, 8)] 
# key=lambda 指定排序依据 

sorted_data = sorted(data, key=lambda x: x[1]) print(sorted_data)
```



4.动手操作（Lambda结合map，filter，sorted）

需求：

1. 有一个学生列表：`students = [("小明", 85), ("小红", 92), ("小刚", 76)]`
2. 用 Lambda + sorted 按**分数从高到低**排序
3. 用 Lambda + filter 筛选出**分数大于 80** 的学生



参考答案
``` python

students = [("小明", 85), ("小红", 92), ("小刚", 76)] 
# 1. 按分数降序排序 
sorted_stu = sorted(students, key=lambda x: x[1], reverse=True) print("分数排序：", sorted_stu) 
# 2. 筛选分数>80 
good_stu = filter(lambda x: x[1] > 80, students) 

print("高分学生：", list(good_stu))
```