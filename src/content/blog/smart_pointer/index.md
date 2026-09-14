---
title: 'Rust中的智能指针'
publishDate: 'September 14, 2026'
updatedDate: 'September 14, 2026'
description: '只有将一切打碎，才能重新组合。'
tags:
  - Rust
  - Programming language
  - smart pointer
language: 'Chinese'
heroImage: { src: 'highpower.png', color: '#559ce8' }
---

# 智能指针

指针（pointer）是一个包含内存地址的变量。Rust 中最常见的指针是引用（reference），引用用 `&` 表示，借用它指向的数据，除了保存地址之外没有额外的运行时开销。

智能指针（smart pointer）通常是一个拥有数据所有权的数据结构。它的行为类似指针，但还会维护额外的元数据，并提供额外功能。标准库中的 `Box<T>`、`Rc<T>`、`RefCell<T>`、`Weak<T>` 都属于常见的智能指针。

智能指针通常会实现两个重要 trait：

- `Deref` 允许智能指针通过 `*` 表现得像引用，并支持解引用强制转换。
- `Drop` 允许类型自定义离开作用域时的清理逻辑。

本章重点讨论以下几种类型：

- `Box<T>`：将值放在堆上，并提供间接访问。
- `Rc<T>`：通过引用计数，让一个值拥有多个所有者。
- `RefCell<T>`：在运行时检查借用规则，实现内部可变性。
- `Weak<T>`：不拥有数据的弱引用，常用于打破引用循环。

**讨论：** 引用只借用数据，智能指针往往拥有数据，因此两者的生命周期和使用方式不同。智能指针不是“绕过”所有权系统，而是把所有权、借用检查或清理行为封装在类型中。选择哪种智能指针，应由数据的所有权关系和线程模型决定。

## 使用 Box<T> 指向堆数据

最简单直接的智能指针是 `Box<T>`。它将 `T` 分配在堆上，而 `Box<T>` 自身通常只占用一个指针大小的栈空间：

```rust
fn main() {
    let b = Box::new(5);
    println!("b = {b}");
}
```

当 `b` 离开作用域时，先释放 `Box`，再释放它指向的堆数据。除了堆分配和间接访问之外，`Box<T>` 没有引用计数等额外开销。

常见使用场景包括：

- 类型大小在编译时未知，但当前位置要求类型具有确定大小；
- 转移大量数据的所有权时，不希望复制整个数据，只复制一个指针；
- 使用 trait 对象，只关心值实现了哪些 trait，而不关心它的具体类型。

### 在堆上存储数据

对于单个 `i32` 来说，放入 `Box` 通常没有必要，因为栈上的存储更简单。`Box` 的真正价值在于递归类型、大型数据或 trait 对象：

```rust
trait Draw {
    fn draw(&self);
}

fn draw_all(items: &[Box<dyn Draw>]) {
    for item in items {
        item.draw();
    }
}
```

`dyn Draw` 是动态大小类型，不能直接作为普通局部变量存储；`Box<dyn Draw>` 通过一个已知大小的指针间接指向堆上的具体实现。

**讨论：** `Box<T>` 不会让数据自动变成共享数据，也不会让数据可以同时可变借用。它只解决“数据放在哪里”和“如何提供一层间接”的问题。需要共享所有权时，应考虑 `Rc<T>` 或线程安全的 `Arc<T>`，而不是把多个 `Box<T>` 当成共享指针。

### Box 创建递归类型

递归类型的值可以包含另一个同类型的值。典型例子是函数式语言中的 cons list：

```rust
#[allow(dead_code)]
enum List {
    Cons(i32, List),
    Nil,
}
```

上面的定义不能编译。编译器会报告 `E0072: recursive type List has infinite size`，因为 `Cons` 直接包含一个 `List`，而计算 `List` 大小时又必须继续计算其中的 `List`。

#### cons list

cons list 的每个节点包含当前值和下一个节点，末尾用 `Nil` 表示：

```text
Cons(1, Cons(2, Cons(3, Nil)))
```

在 Rust 中，可以用 `Box` 保存下一个节点：

```rust
use crate::List::{Cons, Nil};

#[derive(Debug)]
enum List {
    Cons(i32, Box<List>),
    Nil,
}

fn main() {
    let list = Cons(1, Box::new(Cons(2, Box::new(Cons(3, Box::new(Nil))))));
    println!("{list:?}");
}
```

`Box<List>` 的大小是固定的，因为它本质上是一个指针；因此 `List` 的大小也可以计算出来：`Cons` 需要一个 `i32` 和一个指针，`Nil` 不需要数据。

**讨论：** 真实项目中，表示线性列表通常优先使用 `Vec<T>`。这里使用 cons list 是为了说明递归类型的布局规则：递归类型必须通过指针、引用或其他间接方式打断无限嵌套。`Box` 只提供间接存储，并不提供多个所有者或运行时借用检查。

#### 计算非递归类型的大小

对于枚举，Rust 会为它分配足够容纳最大变体的空间，再加上区分当前变体所需的判别信息：

```rust
#[allow(dead_code)]
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
}
```

`Message` 的大小至少要能容纳最大的变体。枚举值在同一时刻只有一个变体有效，所以不会把所有变体的大小简单相加。

**讨论：** “递归类型大小无限”并不是说运行时一定会构造无限长的数据，而是说编译器无法从类型定义本身计算出一个固定布局。使用 `Box` 后，节点内部只保存固定大小的地址，实际节点可以继续在堆上递归分配。

#### 获取一个已知大小的递归类型

修改后的 `List` 可以正常构造：

```rust
use crate::List::{Cons, Nil};

#[allow(dead_code)]
enum List {
    Cons(i32, Box<List>),
    Nil,
}

fn main() {
    let list = Cons(1, Box::new(Cons(2, Box::new(Nil))));
    let _ = list;
}
```

`Box<T>` 实现了 `Deref`，因此可以像访问引用一样访问其中的数据；它还实现了 `Drop`，离开作用域时会自动清理堆数据。

## 将智能指针视为常规引用

### 追踪引用的值

解引用运算符 `*` 可以取得引用指向的值：

```rust
fn main() {
    let x = 5;
    let y = &x;

    assert_eq!(x, 5);
    assert_eq!(*y, 5);
}
```

### 像引用一样使用 Box<T>

`Box<T>` 也可以使用 `*` 解引用：

```rust
fn main() {
    let x = 5;
    let y = Box::new(x);

    assert_eq!(x, 5);
    assert_eq!(*y, 5);
}
```

**讨论：** `*y` 得到的是 `y` 指向的值，但在比较等场景中 Rust 会根据表达式的使用方式自动处理借用。解引用不等于复制；对于没有实现 `Copy` 的类型，直接取出内部值可能会发生所有权移动。

### 定义我们自己的智能指针

下面的元组结构体保存一个 `T`：

```rust
struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(value: T) -> MyBox<T> {
        MyBox(value)
    }
}

fn main() {
    let x = 5;
    let y = MyBox::new(x);

    // 这行暂时不能编译：MyBox 尚未实现 Deref。
    // assert_eq!(*y, 5);
}
```

普通结构体不会因为内部保存了一个值，就自动支持 `*` 运算符。要让 `MyBox<T>` 像引用或 `Box<T>` 一样解引用，需要实现 `Deref`。

**讨论：** 自定义智能指针时，应该明确它是否拥有内部值、是否允许可变访问，以及清理时需要释放哪些资源。仅实现 `Deref` 只能提供访问语义，不能自动获得引用计数或线程安全能力。

### 实现 Deref trait

`Deref` 的 `Target` 指定解引用后得到的目标类型：

```rust
use std::ops::Deref;

struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(value: T) -> MyBox<T> {
        MyBox(value)
    }
}

impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

fn main() {
    let y = MyBox::new(5);
    assert_eq!(*y, 5);
}
```

当 Rust 看到 `*y` 且 `y` 的类型实现了 `Deref` 时，它会把操作理解为近似下面的形式：

```rust
*(y.deref())
```

`deref` 返回引用而不是直接返回内部值，是为了避免把值从智能指针中移出。`Deref` 的方法签名借用 `self`，所以解引用不会取得 `MyBox` 的所有权。

**讨论：** `Deref` 适合表达“这个类型可以自然地看作某个目标类型”，例如 `String` 可以看作 `str`。不要为了省略少量代码而随意实现它；错误的 `Deref` 语义会让方法调用和类型推导变得难以理解。

### 在函数和方法中使用 Deref 强制转换

Deref 强制转换会把一种引用转换为另一种引用。例如，`String` 可以在需要 `&str` 的地方使用：

```rust
fn hello(name: &str) {
    println!("Hello, {name}!");
}

fn main() {
    let name = String::from("Rust");
    hello(&name); // &String 自动转换为 &str
}
```

自定义的 `MyBox<String>` 也可以使用这种转换：

```rust
use std::ops::Deref;

struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(value: T) -> Self {
        Self(value)
    }
}

impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &T {
        &self.0
    }
}

fn hello(name: &str) {
    println!("Hello, {name}!");
}

fn main() {
    let name = MyBox::new(String::from("Rust"));
    hello(&name);
}
```

这里的转换链可以理解为 `&MyBox<String>` → `&String` → `&str`。编译器会在函数或方法参数类型不匹配但存在合适的 `Deref` 实现时，自动尝试这样的转换。

**讨论：** Deref 强制转换只发生在引用之间，不会把拥有的 `String` 自动转换成拥有的 `str`，也不会调用任意类型转换逻辑。必要时可以显式写出 `&*value` 或 `value.as_str()`，但通常应优先保留清晰的参数类型。

### 处理可变引用的 Deref 强制类型转换

`DerefMut` 用于可变解引用：

```rust
use std::ops::{Deref, DerefMut};

struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(value: T) -> Self {
        Self(value)
    }
}

impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &T {
        &self.0
    }
}

impl<T> DerefMut for MyBox<T> {
    fn deref_mut(&mut self) -> &mut T {
        &mut self.0
    }
}

fn main() {
    let mut value = MyBox::new(String::from("Rust"));
    value.push_str(" 学习");
}
```

Rust 会在以下情况下执行 Deref 强制转换：

- `T: Deref<Target = U>` 时，`&T` 可以转换为 `&U`；
- `T: DerefMut<Target = U>` 时，`&mut T` 可以转换为 `&mut U`；
- `T: Deref<Target = U>` 时，`&mut T` 也可以转换为 `&U`。

可变引用可以转成不可变引用，但不可变引用不能转成可变引用，因为后者可能违反“可变引用必须是数据唯一引用”的借用规则。

**讨论：** `DerefMut` 应只在确实能够安全提供唯一可变访问时实现。像 `Rc<T>` 这类允许多个所有者的类型不能简单地实现 `DerefMut`，否则会把共享所有权错误地伪装成唯一可变借用。

## 使用 Drop trait 运行清理代码

`Drop` 允许类型定义值离开作用域时的清理逻辑：

```rust
struct CustomSmartPointer {
    data: String,
}

impl Drop for CustomSmartPointer {
    fn drop(&mut self) {
        println!("Dropping {}", self.data);
    }
}

fn main() {
    let _a = CustomSmartPointer {
        data: String::from("a"),
    };
    let _b = CustomSmartPointer {
        data: String::from("b"),
    };
    println!("CustomSmartPointers created.");
}
```

变量通常按照创建顺序的逆序销毁，因此 `b` 会先于 `a` 被丢弃。不能直接调用 `value.drop()`；如果确实需要提前释放值，应调用标准库函数 `drop(value)`：

```rust
let value = CustomSmartPointer {
    data: String::from("临时资源"),
};
drop(value); // 提前触发 Drop
```

**讨论：** Rust 会自动调用 `Drop::drop`，并在之后继续清理字段。实现 `Drop` 时通常应释放文件、锁、连接等资源，而不是手动释放内存。不要在 `drop` 中再次显式释放同一资源，否则可能造成重复释放；需要控制析构顺序时，可以使用内部作用域或 `drop` 函数。

## Rc<T> 引用计数智能指针

有些数据结构中，一个值需要同时被多个部分拥有，例如图中的一个节点可能被多条边指向。`Rc<T>`（reference counted）通过记录强引用数量，在最后一个所有者离开作用域时清理数据：

```rust
use std::rc::Rc;

fn main() {
    let a = Rc::new(String::from("共享数据"));
    let b = Rc::clone(&a);
    let c = Rc::clone(&a);

    println!("{a}, {b}, {c}");
    println!("strong count = {}", Rc::strong_count(&a));
}
```

`Rc::clone(&a)` 只增加引用计数，不会深拷贝字符串内容。`Rc<T>` 只提供共享的不可变访问；如果还需要修改内部数据，通常与 `RefCell<T>` 结合使用。

`Rc<T>` 只能用于单线程场景。跨线程共享所有权应使用线程安全的 `Arc<T>`，并根据需要配合 `Mutex<T>` 或 `RwLock<T>`。

**讨论：** `Rc<T>` 适合所有权关系确实是共享的场景，但引用计数会带来额外运行时开销，也可能因为引用循环造成内存泄漏。能用普通所有权或借用表达关系时，应优先使用普通所有权或借用。

### 共享数据

用 `Box<T>` 定义 cons list 时，构造第二个列表会移动第一个列表的所有权，因此不能让多个列表共享同一尾部：

```rust
use crate::List::{Cons, Nil};

#[allow(dead_code)]
enum List {
    Cons(i32, Box<List>),
    Nil,
}

fn main() {
    let a = Cons(5, Box::new(Cons(10, Box::new(Nil))));
    let _b = Cons(3, Box::new(a));
    // 不能再次使用 a：a 的所有权已经移动给 b。
}
```

使用 `Rc<List>` 后，多个列表可以共享尾部：

```rust
use std::rc::Rc;
use crate::List::{Cons, Nil};

#[derive(Debug)]
enum List {
    Cons(i32, Rc<List>),
    Nil,
}

fn main() {
    let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));
    let b = Cons(3, Rc::clone(&a));
    let c = Cons(4, Rc::clone(&a));

    println!("a = {a:?}");
    println!("b = {b:?}");
    println!("c = {c:?}");
}
```

**讨论：** `Rc::clone` 的命名强调它是廉价的引用计数操作，而不是深拷贝。共享并不意味着可变；`Rc<T>` 的设计刻意保留了不可变访问，以免多个所有者同时修改同一数据。

### 克隆会增加引用计数

引用计数会随着 `Rc::clone` 增加，并在 `Rc` 离开作用域时自动减少：

```rust
use std::rc::Rc;

fn main() {
    let a = Rc::new(String::from("data"));
    println!("count = {}", Rc::strong_count(&a));

    {
        let b = Rc::clone(&a);
        println!("count = {}", Rc::strong_count(&a));
        let c = Rc::clone(&a);
        println!("count = {}", Rc::strong_count(&a));
        let _ = (b, c);
    }

    println!("count = {}", Rc::strong_count(&a));
}
```

**讨论：** `strong_count` 只统计强引用。弱引用由 `weak_count` 统计，并不会阻止数据释放。引用计数只能管理“沿着计数关系可达”的数据，无法识别逻辑上的循环，因此设计树或图时要明确哪些边拥有节点。

## RefCell<T> 和内部可变性模式

内部可变性（interior mutability）允许一个对外不可变的值，在内部修改自己的数据。`RefCell<T>` 将借用规则从编译时推迟到运行时：

- `borrow()` 获取不可变借用；
- `borrow_mut()` 获取可变借用；
- 违反借用规则时会 panic；
- `try_borrow()` 和 `try_borrow_mut()` 可以返回 `Result`，避免直接 panic。

`RefCell<T>` 仍然只拥有一个内部值，并且只能用于单线程场景。

### 在运行时强制借用规则

```rust
use std::cell::RefCell;

fn main() {
    let value = RefCell::new(5);

    *value.borrow_mut() += 1;
    println!("{}", value.borrow());
}
```

`borrow()` 返回 `Ref<T>`，`borrow_mut()` 返回 `RefMut<T>`。两者都实现了 `Deref`，因此可以像引用一样使用：

```rust
use std::cell::RefCell;

let value = RefCell::new(5);
let first = value.borrow_mut();
// value.borrow_mut(); // 运行时 panic：已有可变借用
 drop(first);
```

在引用和 `Box<T>` 中，借用规则由编译器检查；在 `RefCell<T>` 中，规则由运行时检查。编译时检查通常更早发现问题、没有运行时借用检查开销，因此应作为默认选择。

**讨论：** `RefCell<T>` 适用于静态分析过于保守、但程序员能够保证运行时借用合法的场景。它不会消除借用规则，只是把错误从编译期延迟到运行期；如果逻辑无法保证不冲突，应使用 `try_borrow` 系列方法或重新设计所有权关系。

选择三种类型时可以参考：

- `Box<T>`：单一所有者，编译时检查借用；
- `Rc<T>`：多个所有者，只提供共享不可变访问；
- `RefCell<T>`：单一所有者，运行时检查可变或不可变借用。

### 使用内部可变性

内部可变性常用于测试替身（test double），例如 mock 对象需要在 `&self` 方法中记录调用信息。假设库的接口如下：

```rust
pub trait Messenger {
    fn send(&self, message: &str);
}

pub struct LimitTracker<'a, T: Messenger> {
    messenger: &'a T,
    value: usize,
    max: usize,
}

impl<'a, T: Messenger> LimitTracker<'a, T> {
    pub fn new(messenger: &'a T, max: usize) -> Self {
        Self {
            messenger,
            value: 0,
            max,
        }
    }

    pub fn set_value(&mut self, value: usize) {
        self.value = value;
        let ratio = value as f64 / self.max as f64;

        if ratio >= 1.0 {
            self.messenger.send("Error: you are over your quota!");
        } else if ratio >= 0.9 {
            self.messenger.send("Urgent warning: You've used up over 90%!");
        } else if ratio >= 0.75 {
            self.messenger.send("Warning: You've used up over 75%!");
        }
    }
}
```

`Messenger::send` 接收 `&self`，普通 `Vec<String>` 不能在其中直接修改：

```rust
struct MockMessenger {
    sent_messages: Vec<String>,
}

impl Messenger for MockMessenger {
    fn send(&self, message: &str) {
        // self.sent_messages.push(message.to_string());
        // 不能编译：不能通过不可变借用修改 Vec。
    }
}
```

将字段放进 `RefCell` 后，修改就可以安全地延迟到运行时检查：

```rust
use std::cell::RefCell;

struct MockMessenger {
    sent_messages: RefCell<Vec<String>>,
}

impl MockMessenger {
    fn new() -> Self {
        Self {
            sent_messages: RefCell::new(Vec::new()),
        }
    }
}

impl Messenger for MockMessenger {
    fn send(&self, message: &str) {
        self.sent_messages.borrow_mut().push(message.to_string());
    }
}

fn main() {
    let mock = MockMessenger::new();
    let mut tracker = LimitTracker::new(&mock, 100);
    tracker.set_value(80);

    assert_eq!(mock.sent_messages.borrow().len(), 1);
}
```

**讨论：** 这里的 `MockMessenger` 对外仍然可以通过 `&self` 使用，但内部借助 `RefCell` 修改了消息列表。内部可变性适合封装少量、明确的运行时状态；如果到处嵌套 `RefCell`，通常说明所有权或接口设计需要重新审视。

### 允许多个可变数据所有者

`Rc<RefCell<T>>` 将多个所有者和内部可变性组合起来：

```rust
use std::{cell::RefCell, rc::Rc};
use crate::List::{Cons, Nil};

#[derive(Debug)]
enum List {
    Cons(Rc<RefCell<i32>>, Rc<List>),
    Nil,
}

fn main() {
    let value = Rc::new(RefCell::new(5));
    let a = Rc::new(Cons(Rc::clone(&value), Rc::new(Nil)));
    let b = Cons(Rc::new(RefCell::new(4)), Rc::clone(&a));
    let c = Cons(Rc::new(RefCell::new(3)), Rc::clone(&a));

    *value.borrow_mut() += 10;
    println!("a = {a:?}");
    println!("b = {b:?}");
    println!("c = {c:?}");
}
```

`a`、`b`、`c` 共享同一个 `value`，所以修改 `value` 后，从三个列表观察到的值都会变化。

**讨论：** `Rc<RefCell<T>>` 的能力很强，但也把两种风险叠加在一起：引用循环导致泄漏，运行时借用冲突导致 panic。多线程场景不能直接使用它；可根据需求考虑 `Arc<Mutex<T>>` 或 `Arc<RwLock<T>>`。

## 引用循环会导致内存泄露

Rust 的所有权系统通常能自动清理内存，但它不保证完全杜绝内存泄漏。使用 `Rc<T>` 和 `RefCell<T>` 时，可以构造出引用循环：每个节点都被另一个节点的强引用持有，因此所有强引用计数永远不会降到零。

### 制造引用循环

下面的 `List` 允许修改尾部链接：

```rust
use std::{cell::RefCell, rc::Rc};
use crate::List::{Cons, Nil};

#[derive(Debug)]
enum List {
    Cons(i32, RefCell<Rc<List>>),
    Nil,
}

impl List {
    fn tail(&self) -> Option<&RefCell<Rc<List>>> {
        match self {
            Cons(_, item) => Some(item),
            Nil => None,
        }
    }
}

fn main() {
    let a = Rc::new(Cons(5, RefCell::new(Rc::new(Nil))));
    let b = Rc::new(Cons(10, RefCell::new(Rc::clone(&a))));

    if let Some(link) = a.tail() {
        *link.borrow_mut() = Rc::clone(&b);
    }

    println!("a strong count = {}", Rc::strong_count(&a));
    println!("b strong count = {}", Rc::strong_count(&b));
    // a -> b -> a，两个节点的强引用计数都不会归零。
}
```

变量 `b` 指向 `a`，随后又把 `a` 的尾部改为指向 `b`，于是形成 `a -> b -> a`。`main` 结束时，局部变量被丢弃只会让计数各减少一次，循环内部的强引用仍然存在，因此堆数据无法释放。

如果递归打印这个循环，程序还可能不断沿着 `a -> b -> a` 展开，最终栈溢出。内存泄漏本身不违反内存安全，但会持续占用不再可用的资源。

**讨论：** 引用循环是逻辑错误，不是借用检查器能够自动发现的错误。设计图、树或双向结构时，应区分“拥有关系”和“观察关系”：拥有关系使用 `Rc`，反向或观察关系使用 `Weak`。同时可以通过测试、代码审查和限制递归遍历来降低风险。

### 使用 Weak<T> 防止引用循环

`Rc::downgrade(&rc)` 会创建 `Weak<T>`，增加 `weak_count`，但不会增加 `strong_count`。弱引用不拥有数据，因此不会阻止数据释放。使用它之前必须调用 `upgrade()`：

```rust
use std::{cell::RefCell, rc::{Rc, Weak}};

#[derive(Debug)]
struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,
    children: RefCell<Vec<Rc<Node>>>,
}

fn main() {
    let leaf = Rc::new(Node {
        value: 3,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(Vec::new()),
    });

    println!("leaf parent = {:?}", leaf.parent.borrow().upgrade());

    let branch = Rc::new(Node {
        value: 5,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![Rc::clone(&leaf)]),
    });

    *leaf.parent.borrow_mut() = Rc::downgrade(&branch);

    println!("branch value = {}", branch.value);
    println!("branch strong count = {}", Rc::strong_count(&branch));
    println!("branch weak count = {}", Rc::weak_count(&branch));

    if let Some(parent) = leaf.parent.borrow().upgrade() {
        println!("leaf parent value = {}", parent.value);
    }
}
```

在这个树结构中，父节点通过 `Rc<Node>` 拥有子节点；子节点通过 `Weak<Node>` 观察父节点。父节点被释放后，子节点中的 `upgrade()` 会返回 `None`，而不是返回悬垂指针。

**讨论：** `Weak<T>` 的核心语义是“可能已经不存在的观察者引用”，所以每次使用都必须处理 `Option<Rc<T>>`。树结构通常让父节点拥有子节点、子节点弱引用父节点；图结构则需要根据实际生命周期分别选择强引用和弱引用。`Weak<T>` 不是普通借用的替代品，它仍然是引用计数系统中的一种智能指针。

## 总结

- `Box<T>` 提供堆分配和间接存储，适合递归类型、较大数据和 trait 对象。
- `Deref` 和 `DerefMut` 让智能指针能够像引用一样访问目标类型，并支持解引用强制转换。
- `Drop` 会在值离开作用域时自动运行，用于清理资源。
- `Rc<T>` 通过引用计数提供单线程多所有权；跨线程应考虑 `Arc<T>`。
- `RefCell<T>` 在运行时检查借用规则，实现内部可变性。
- `Rc<RefCell<T>>` 可以同时提供共享所有权和可变访问，但要警惕运行时 panic 与引用循环。
- `Weak<T>` 不拥有数据，适合表示反向关系或观察关系，可以防止引用循环造成的内存泄漏。
