---
title: '函数式编程特性：闭包和迭代器'
publishDate: 'August 12, 2026'
updatedDate: 'August 12, 2026'
description: '闭包和迭代器是 Rust 中两个非常强大的工具，它们使函数式编程成为可能。'
tags:
  - Rust
  - closure
  - iterator
  - functional programming

language: 'Chinese'
heroImage: { src: 'magician.png', color: '#9698C1' }
---



# 闭包

Rust 的闭包（closure）是可以保存在变量中、作为参数传递，或从函数返回的匿名函数。闭包与普通函数一样可以接收参数并返回值，但它还可以捕获定义位置周围作用域中的变量，因此特别适合把“某个操作”作为参数传入。

闭包的基本形式是 `|参数列表| 表达式`。参数和返回值的类型通常可以由编译器根据使用场景推断出来；如果需要，也可以像函数一样显式标注类型。

## 捕获环境

考虑一个送 T 恤的例子：用户可以指定喜欢的颜色；没有指定颜色时，就赠送库存最多的颜色。`unwrap_or_else` 接收一个无参数闭包，并且只在 `Option` 为 `None` 时调用它。

```rust
#[derive(Debug, PartialEq)]
enum ShirtColor {
    Red,
    Blue,
}

struct Inventory {
    shirts: Vec<ShirtColor>,
}

impl Inventory {
    fn giveaway(&self, user_preference: Option<ShirtColor>) -> ShirtColor {
        user_preference.unwrap_or_else(|| self.most_stocked())
    }

    fn most_stocked(&self) -> ShirtColor {
        let mut num_red = 0;
        let mut num_blue = 0;

        for color in &self.shirts {
            match color {
                ShirtColor::Red => num_red += 1,
                ShirtColor::Blue => num_blue += 1,
            }
        }

        if num_red > num_blue {
            ShirtColor::Red
        } else {
            ShirtColor::Blue
        }
    }
}

fn main() {
    let store = Inventory {
        shirts: vec![ShirtColor::Blue, ShirtColor::Blue, ShirtColor::Red],
    };

    let user_pref1 = Some(ShirtColor::Red);
    let giveaway1 = store.giveaway(user_pref1);
    assert_eq!(giveaway1, ShirtColor::Red);

    let user_pref2 = None;
    let giveaway2 = store.giveaway(user_pref2);
    assert_eq!(giveaway2, ShirtColor::Blue);
}
```

`|| self.most_stocked()` 是一个无参数闭包。它捕获了当前 `Inventory` 实例的不可变借用，因此可以在闭包体中使用 `self`。如果 `user_preference` 是 `Some`，`unwrap_or_else` 直接返回其中的值，闭包不会执行；只有 `None` 时才会调用闭包。

普通函数无法直接捕获定义位置的局部变量，必须把所需数据作为参数传入；闭包则可以把环境和行为一起打包。这使得标准库方法不必了解 `Inventory` 的具体类型，也能延迟执行调用者提供的逻辑。

**讨论**：闭包并不是“总会执行”的回调。像 `unwrap_or_else` 这样的惰性 API 只在需要默认值时调用闭包；如果无论如何都要先计算参数，应使用 `unwrap_or`。这既能避免不必要的计算，也能避免默认逻辑产生不需要的副作用。

## 推断和注解闭包类型

函数的参数和返回值是公开接口的一部分，所以通常必须写出类型；闭包往往只在局部上下文中使用，编译器可以从调用位置推断类型。下面的两个闭包分别把整数加一和把字符串转为大写：

```rust
fn main() {
    let add_one = |x: u32| -> u32 { x + 1 };
    assert_eq!(add_one(5), 6);

    let to_upper = |text: &str| -> String { text.to_uppercase() };
    assert_eq!(to_upper("rust"), "RUST");
}
```

在很多情况下，类型可以完全省略：

```rust
fn main() {
    let example_closure = |x| x;

    let value = example_closure(String::from("hello"));
    assert_eq!(value, "hello");
    // example_closure(5); // 错误：这个闭包的参数类型已经被推断为 String
}
```

闭包的匿名类型会在第一次使用时确定。上例中第一次调用传入了 `String`，所以之后不能再用 `i32` 调用同一个闭包。需要让同一段逻辑处理不同类型时，应定义泛型函数或针对不同类型定义不同闭包。

**讨论**：类型推断减少了局部代码的噪声，但公开 API 不应依赖读者猜测类型。给复杂闭包写参数和返回值注解通常更易读；同时，闭包的参数类型不能只凭“看起来合理”来改变，闭包变量本身不是自动泛型的。

## 捕获引用和移动所有权

闭包根据使用捕获变量的方式，可能以不可变借用、可变借用或取得所有权的方式捕获环境。下面的闭包只读取 `list`，因此捕获不可变借用：

```rust
fn main() {
    let list = vec![1, 2, 3];

    let only_borrows = || println!("list: {list:?}");
    only_borrows();

    println!("list 仍然可用：{list:?}");
}
```

如果闭包修改捕获的变量，就需要可变借用，闭包变量也必须声明为 `mut`：

```rust
fn main() {
    let mut list = vec![1, 2, 3];

    let mut borrows_mutably = || list.push(4);
    borrows_mutably();

    assert_eq!(list, vec![1, 2, 3, 4]);
}
```

闭包定义时就会建立捕获关系。上例中，在 `borrows_mutably` 的定义和调用之间不能读取 `list`，因为此时 `list` 已经被可变借用；调用结束后，这次可变借用才可以结束。

如果希望无论闭包体是否需要所有权，都强制把环境中的值移动进闭包，可以在参数列表前使用 `move`：

```rust
fn main() {
    let list = vec![1, 2, 3];

    let takes_ownership = move || println!("list: {list:?}");
    takes_ownership();

    // println!("{list:?}"); // 错误：list 的所有权已经移入闭包
}
```

`move` 对在线程中运行的闭包尤其重要：新线程的执行时间可能超过创建它的函数，不能让线程持有一个已经失效的局部变量引用。

```rust
use std::thread;

fn main() {
    let list = vec![1, 2, 3];

    let handle = thread::spawn(move || {
        println!("来自线程：{list:?}");
    });

    handle.join().unwrap();
}
```

`thread::spawn` 要求闭包满足 `Send + 'static` 等约束。`move` 将 `list` 的所有权交给线程，使线程不再依赖创建线程的栈帧；线程结束后，闭包负责释放这份 `Vec`。

**讨论**：`move` 表示捕获方式优先采用移动，但不等于闭包只能调用一次。例如 `move || println!("{text}")` 仍可能实现 `Fn`，因为它只是反复读取已经移入闭包的 `String`。真正决定 `Fn`、`FnMut` 或 `FnOnce` 的，是闭包体调用时如何使用捕获的值。

## 将捕获的值移出闭包

闭包捕获值的方式和闭包体对值的操作，会共同决定闭包实现哪些 `Fn` trait。闭包可以不捕获环境，也可以捕获环境中的借用或所有权。

```rust
fn main() {
    let text = String::from("hello");

    // 只读取捕获的值：可以重复调用，实现 Fn。
    let print = || println!("{text}");
    print();
    print();

    let mut count = 0;
    // 修改捕获的值：需要可变调用，实现 FnMut。
    let mut increase = || {
        count += 1;
        count
    };
    assert_eq!(increase(), 1);
    assert_eq!(increase(), 2);

    let message = String::from("消耗我");
    // 把捕获的值移出闭包：只能调用一次，实现 FnOnce。
    let consume = || message;
    let result = consume();
    assert_eq!(result, "消耗我");
}
```

三个 trait 的关系可以概括为：

- `FnOnce`：闭包至少可以被调用一次。所有闭包都实现它；如果闭包把捕获的值移出自身，通常只能实现 `FnOnce`。
- `FnMut`：闭包不会移出捕获的值，但可能修改它。调用这类闭包需要可变访问，因此可以调用多次。
- `Fn`：闭包既不移出也不修改捕获的值，或者根本没有捕获环境。它可以通过不可变引用被多次调用。

这三个 trait 具有包含关系：`Fn` 闭包也实现 `FnMut` 和 `FnOnce`，`FnMut` 闭包也实现 `FnOnce`。因此只要求 `FnOnce` 的函数最灵活，只要求 `Fn` 的函数约束最严格。

标准库中 `Option<T>::unwrap_or_else` 的核心签名可以简化为：

```rust
impl<T> Option<T> {
    fn unwrap_or_else<F>(self, f: F) -> T
    where
        F: FnOnce() -> T,
    {
        match self {
            Some(value) => value,
            None => f(),
        }
    }
}
```

它最多调用一次 `f`，所以只需要 `FnOnce`。即使传入的闭包实现了 `Fn` 或 `FnMut`，也都满足 `FnOnce` 的约束。

`sort_by_key` 则需要多次调用排序键闭包，因此约束为 `FnMut`：

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let mut list = [
        Rectangle { width: 30, height: 10 },
        Rectangle { width: 10, height: 20 },
        Rectangle { width: 20, height: 15 },
    ];

    let mut calls = 0;
    list.sort_by_key(|rectangle| {
        calls += 1;
        rectangle.width
    });

    assert_eq!(list[0].width, 10);
    assert!(calls > 0);
}
```

如果排序键闭包把某个值移出自身，就不能满足 `FnMut`，因为排序过程可能需要再次调用它：

```rust
// 不能作为 sort_by_key 的键函数：String 会在第一次调用时被移出闭包。
// let key = String::from("width");
// list.sort_by_key(|_| key);
```

**讨论**：选择 `Fn`、`FnMut` 还是 `FnOnce` 是 API 设计的重要部分。只调用一次的延迟操作应优先接受 `FnOnce`；需要重复调用但允许维护内部状态时使用 `FnMut`；需要共享、可重入行为时才要求 `Fn`。错误地要求 `Fn` 会拒绝本来安全的可变闭包，错误地要求 `FnOnce` 则无法表达“这个回调必须可重复调用”的意图。

# 迭代器

迭代器模式把“依次访问每一项并判断何时结束”的逻辑封装起来。Rust 中的迭代器是惰性的（lazy）：创建迭代器或调用迭代器适配器本身通常不会立即执行遍历，直到使用消费适配器、`for` 循环或手动调用 `next`。

```rust
fn main() {
    let v1 = vec![1, 2, 3];
    let v1_iter = v1.iter();

    for value in v1_iter {
        println!("{value}");
    }
}
```

`for` 循环会取得迭代器的所有权，并在内部反复调用 `next`。如果只创建迭代器而不消费它，则不会产生遍历元素的效果。

**讨论**：迭代器的惰性不是“没有工作”，而是把工作推迟到真正需要结果时。多个适配器可以先组合成一条处理流水线，最后一次性消费，既便于表达，也避免产生中间集合。

## Iterator trait 和 next 方法

`Iterator` trait 的核心要求是实现 `next` 方法：每次返回一个 `Some(item)`，遍历结束后返回 `None`。

```rust
fn main() {
    let values = vec![1, 2, 3];
    let mut iter = values.iter();

    assert_eq!(iter.next(), Some(&1));
    assert_eq!(iter.next(), Some(&2));
    assert_eq!(iter.next(), Some(&3));
    assert_eq!(iter.next(), None);
}
```

调用 `next` 会改变迭代器内部记录的位置，所以迭代器必须是可变的。`iter` 产生元素的不可变引用；`into_iter` 消费集合并产生拥有所有权的元素；`iter_mut` 产生可变引用：

```rust
fn main() {
    let values = vec![String::from("a"), String::from("b")];

    for value in values.iter() {
        println!("借用：{value}");
    }

    let owned_values = vec![String::from("a"), String::from("b")];
    for value in owned_values.into_iter() {
        println!("拥有：{value}");
    }
    // owned_values 已被 into_iter 消耗，不能再使用。
}
```

也可以为自定义类型实现 `Iterator`。下面的迭代器从 1 递增到 3：

```rust
struct Counter {
    current: u32,
}

impl Counter {
    fn new() -> Self {
        Self { current: 0 }
    }
}

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        self.current += 1;
        if self.current <= 3 {
            Some(self.current)
        } else {
            None
        }
    }
}
```

**讨论**：`next` 的 `&mut self` 说明迭代器通常是有状态的。`iter`、`iter_mut` 和 `into_iter` 的差异本质上是元素的所有权差异：选错方法常见的结果是借用检查错误，或者集合在循环后不能继续使用。需要保留原集合时通常使用 `iter`，需要消费集合时才使用 `into_iter`。

## 消费迭代器的方法

调用 `next` 并不能直接得到整个结果；一些方法会反复调用 `next`，直到返回 `None`，这类方法称为消费适配器。调用它们会消费迭代器，例如 `sum`、`collect`、`count` 和 `for_each`。

```rust
fn main() {
    let total: i32 = vec![1, 2, 3, 4].into_iter().sum();
    assert_eq!(total, 10);

    let values = vec![1, 2, 3];
    let count = values.iter().count();
    assert_eq!(count, 3);
}
```

`sum` 获取迭代器的所有权并累加元素。下面的测试使用 `collect` 把迭代器重新收集为 `Vec`：

```rust
fn main() {
    let result: Vec<i32> = vec![1, 2, 3]
        .into_iter()
        .map(|value| value * 2)
        .collect();

    assert_eq!(result, vec![2, 4, 6]);
}
```

`collect` 可以收集成多种类型，因此经常需要通过变量类型或涡轮鱼语法 `collect::<Vec<_>>()` 告诉编译器目标类型。

**讨论**：消费迭代器后，原迭代器不能再次使用，因为它的所有权已经被消费适配器取得。`iter().sum()` 与 `into_iter().sum()` 还会产生不同的元素类型：前者通常处理引用（并依赖相应的 `Sum` 实现），后者处理拥有所有权的值。明确选择借用还是移动，能减少不必要的克隆。

## 产生其他迭代器的方法

迭代器适配器不会立即消费当前迭代器，而是根据它创建一个新的迭代器。常用适配器包括 `map`、`filter`、`take`、`skip` 和 `zip`。

```rust
fn main() {
    let v1 = vec![1, 2, 3];
    let v2: Vec<_> = v1.iter().map(|value| value + 1).collect();

    assert_eq!(v2, vec![2, 3, 4]);
    assert_eq!(v1, vec![1, 2, 3]); // iter 只借用了 v1
}
```

上面的 `map` 接收一个闭包，对每个元素生成一个新值；它返回的仍是惰性迭代器，所以必须调用 `collect` 等消费适配器才能得到 `Vec`。

适配器可以链式组合：

```rust
fn main() {
    let result: Vec<i32> = (1..=10)
        .filter(|value| value % 2 == 0)
        .map(|value| value * value)
        .take(2)
        .collect();

    assert_eq!(result, vec![4, 16]);
}
```

**讨论**：适配器组合表达的是“转换流水线”，每一步只描述一个动作，最终由消费适配器触发执行。链式调用过长时可以拆成有意义的中间变量；不要为了追求函数式写法而牺牲可读性。适配器闭包的 trait 约束也各不相同，例如 `map` 通常需要 `FnMut`，因为它可能多次调用闭包。

## 使用捕获其环境的闭包

很多迭代器适配器都接收闭包。`filter` 的闭包接收一项并返回 `bool`：返回 `true` 的项会保留，返回 `false` 的项会被丢弃。闭包可以捕获环境中的变量，因此筛选条件不必硬编码。

```rust
#[derive(Debug, PartialEq)]
struct Shoe {
    size: u32,
    style: String,
}

fn shoes_in_size(shoes: Vec<Shoe>, shoe_size: u32) -> Vec<Shoe> {
    shoes
        .into_iter()
        .filter(|shoe| shoe.size == shoe_size)
        .collect()
}

fn main() {
    let shoes = vec![
        Shoe { size: 10, style: String::from("运动鞋") },
        Shoe { size: 13, style: String::from("凉鞋") },
        Shoe { size: 10, style: String::from("靴子") },
    ];

    let result = shoes_in_size(shoes, 10);
    assert_eq!(result.len(), 2);
}
```

`shoes_in_size` 通过 `into_iter` 获取 `shoes` 的所有权，闭包从环境中借用 `shoe_size`，再用每只鞋的 `size` 与它比较。最后 `collect` 把过滤后的元素收集到新的 `Vec` 中。由于 `into_iter` 产生拥有所有权的 `Shoe`，返回集合不需要从原集合复制元素。

需要注意 `filter` 的闭包参数通常是“迭代器元素的引用”，即使迭代器的元素类型本身是值，也可能看到多一层引用；这里的比较表达式正是根据该参数形式编写的。

**讨论**：捕获环境让 `filter` 这样的通用方法拥有具体业务含义，但也要关注所有权：使用 `iter` 会保留原集合并返回引用，使用 `into_iter` 则会移动元素并适合构造拥有数据的新集合。若闭包需要修改外部状态，还必须保证外部变量可变，并满足该适配器对 `FnMut` 的要求。

## 性能比较：循环 or 迭代器

可以用循环和迭代器分别完成同一件事：

```rust
fn sum_with_loop(values: &[i32]) -> i32 {
    let mut total = 0;
    for value in values {
        total += value;
    }
    total
}

fn sum_with_iterator(values: &[i32]) -> i32 {
    values.iter().copied().sum()
}

fn main() {
    let values = [1, 2, 3, 4];
    assert_eq!(sum_with_loop(&values), sum_with_iterator(&values));
}
```

在许多情况下，迭代器是零成本抽象（zero-cost abstraction）：优化后的代码可以与手写循环大致相同，不会因为使用闭包和适配器就自动产生额外的运行时开销。编译器仍可能进行循环展开、消除边界检查等优化。

> In general, C++ implementations obey the zero-overhead principle: What you don’t use, you don’t pay for. And further: What you do use, you couldn’t hand code any better.

零成本并不意味着所有迭代器代码在任何情况下都绝对同速。调试构建、复杂闭包、额外分配（例如不必要的 `collect`）和无法内联的动态分发都可能影响性能；应使用基准测试验证实际热点，而不是凭代码表面形式判断。

**讨论**：循环与迭代器首先是表达方式的选择：循环适合复杂的分支和需要逐步调试的逻辑，迭代器适合清晰的转换与聚合流水线。只要避免无意义的克隆和中间集合，迭代器通常能同时获得较好的可读性与性能。闭包和迭代器共同体现了 Rust 的零成本抽象目标：用高层结构表达意图，而把优化交给编译器完成。
