---
title: 'rust中面向对象的设计方法'
publishDate: 'September 24, 2026'
updatedDate: 'September 24, 2026'
description: 'world is never enough'
tags:
  - Rust
  - Programming language
  - OOP
language: 'Chinese'
heroImage: { src: 'flower_sea.png', color: '#559ce8' }
---

## 面向对象语言的特征

面向对象编程（Object-Oriented Programming，OOP）并没有唯一的定义。常见特征包括：对象将数据和行为组合起来、封装实现细节，以及通过继承或接口实现代码复用与多态。Rust 支持其中的许多思想，但不提供传统的类继承。

### 对象包含数据和行为

《设计模式》（Design Patterns: Elements of Reusable Object-Oriented Software）的四位作者（Gang of Four）这样定义面向对象编程：

> Object-oriented programs are made up of objects. An object packages both data and the procedures that operate on that data. The procedures are typically called methods or operations.

在这个定义下，Rust 的结构体和枚举可以存放数据，`impl` 块可以提供操作这些数据的方法。

```rust
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    fn new(width: u32, height: u32) -> Self {
        Self { width, height }
    }

    fn area(&self) -> u32 {
        self.width * self.height
    }

    fn resize(&mut self, width: u32, height: u32) {
        self.width = width;
        self.height = height;
    }
}

fn main() {
    let mut rectangle = Rectangle::new(3, 4);
    assert_eq!(rectangle.area(), 12);
    rectangle.resize(5, 6);
    assert_eq!(rectangle.area(), 30);
}
```

**讨论：**

- `new` 没有 `self` 参数，是关联函数；它只是惯用命名，不是语言内置的构造器。
- `&self` 借用数据，`&mut self` 独占借用并允许修改数据，`self` 则取得所有权。这些接收者形式把方法对对象的使用方式写进了 API。
- 结构体及其 `impl` 块在语法上分开，但仍然能够组合数据和行为。不要把所有带方法的值与后文的“trait 对象”混为一谈。

### 封装了底层的实现细节

封装（encapsulation）意味着调用方通过公开 API 使用类型，而不直接依赖内部表示。Rust 用模块和可见性控制实现封装：`pub struct` 公开类型，但不会自动公开结构体字段。

下面的集合缓存平均值，并通过私有字段保证缓存与数据同步。

```rust
mod statistics {
    pub struct AveragedCollection {
        list: Vec<i32>,
        average: Option<f64>,
    }

    impl AveragedCollection {
        pub fn new() -> Self {
            Self {
                list: Vec::new(),
                average: None,
            }
        }

        pub fn add(&mut self, value: i32) {
            self.list.push(value);
            self.update_average();
        }

        pub fn remove(&mut self) -> Option<i32> {
            let result = self.list.pop();
            if result.is_some() {
                self.update_average();
            }
            result
        }

        pub fn average(&self) -> Option<f64> {
            self.average
        }

        fn update_average(&mut self) {
            self.average = if self.list.is_empty() {
                None
            } else {
                let total: f64 = self.list.iter().map(|&value| f64::from(value)).sum();
                Some(total / self.list.len() as f64)
            };
        }
    }
}

fn main() {
    let mut numbers = statistics::AveragedCollection::new();
    assert_eq!(numbers.average(), None);
    numbers.add(10);
    numbers.add(20);
    assert_eq!(numbers.average(), Some(15.0));
    assert_eq!(numbers.remove(), Some(20));
    assert_eq!(numbers.average(), Some(10.0));
}
```

**讨论：**

- 空集合的平均值没有定义，因此用 `Option<f64>` 表示，而不是除以零或把 `0.0` 当成“没有平均值”。先转成 `f64` 再求和可避免 `i32` 求和溢出，但浮点计算仍可能存在精度误差。
- 私有性的边界主要是模块，不是单个 `impl`。定义模块及其子模块可以访问私有成员；示例把类型放在子模块中，使外部调用者不能直接修改 `list`。
- 在模块外访问 `numbers.list` 会得到 E0616：``field `list` of struct `AveragedCollection` is private``。
- 缓存让查询变成 O(1)，这里的每次有效修改仍需 O(n) 重新求和；也可以缓存总和来优化，但要承担更多不变量维护工作。
- 封装允许更换内部表示，但**方法签名不变不等于行为兼容**。例如把 `Vec<i32>` 换成 `HashSet<i32>` 会去重，并失去“移除最后加入的值”的语义，不能视为无影响的替换。

### 作为类型系统与代码共享的继承

继承通常有两个用途：复用实现，以及让子类型用在父类型的位置。Rust 不提供结构体之间的类继承；宏可以生成重复代码，但并不会给语言增加类继承或子类型关系。

Rust 更常使用组合与委托复用数据和实现，使用 trait 的默认方法共享行为。

```rust
trait Summary {
    fn title(&self) -> &str;

    fn summarize(&self) -> String {
        format!("摘要：{}", self.title())
    }
}

struct Article {
    title: String,
}

impl Summary for Article {
    fn title(&self) -> &str {
        &self.title
    }
}

struct Newsletter {
    article: Article, // 组合：拥有一个 Article，而不是继承它
}

impl Summary for Newsletter {
    fn title(&self) -> &str {
        self.article.title() // 显式委托
    }

    fn summarize(&self) -> String {
        format!("本期推荐：{}", self.title()) // 覆盖默认实现
    }
}

fn print_summary(item: &impl Summary) {
    println!("{}", item.summarize());
}
```

**讨论：**

- 默认方法可以调用 trait 中的其他方法，但不能假设实现者有某个字段。共享数据通常通过组合完成。
- `trait Child: Parent` 是 supertrait（超 trait）约束，要求实现 `Child` 的类型也实现 `Parent`；它不是继承父结构体的字段。
- 多态不只发生在运行时：泛型通常通过单态化实现静态分发，trait 对象通过动态分发实现运行时多态。
- 组合避免继承整套不需要的行为，但可能需要编写委托方法。应根据复用和扩展需求选择设计，而不是强行模拟类层次。

## 使用trait object来抽象出共享行为

当库无法预先知道所有实现类型，却需要统一调用它们的行为时，可以使用 trait 对象。例如 GUI 库只要求组件能绘制，而用户可以增加自己的组件类型。

trait 对象的常见形式如下：

| 形式 | 含义 |
| --- | --- |
| `&dyn Draw` | 借用一个实现了 `Draw` 的值 |
| `&mut dyn Draw` | 独占借用，允许调用接收 `&mut self` 的方法 |
| `Box<dyn Draw>` | 拥有被装箱值的所有权 |
| `Rc<dyn Draw>` | 单线程共享所有权 |
| `Arc<dyn Draw + Send + Sync>` | 可在线程间共享的所有权，且实现类型满足线程安全约束 |

**讨论：** `dyn Draw` 是动态大小类型，通常放在引用或智能指针之后使用。`dyn` 不代表动态类型：能否转换、能调用哪些方法，都由编译器检查。`Arc` 本身不会让内部类型自动变得线程安全。

### 定义通用行为的trait

先定义一个只关心绘制行为的接口，以及保存异构组件的容器：

```rust
pub trait Draw {
    fn draw(&self);
}

pub struct Screen {
    pub components: Vec<Box<dyn Draw>>,
}

impl Screen {
    pub fn run(&self) {
        for component in &self.components {
            component.draw();
        }
    }
}
```

与之对比，泛型容器可以这样定义（与上面的定义放在一起）：

```rust
pub struct GenericScreen<T: Draw> {
    pub components: Vec<T>,
}

impl<T: Draw> GenericScreen<T> {
    pub fn run(&self) {
        for component in &self.components {
            component.draw();
        }
    }
}
```

**讨论：**

- `Vec<Box<dyn Draw>>` 的元素类型始终是 `Box<dyn Draw>`，但每个元素背后的具体类型可以不同。
- `GenericScreen<Button>` 中每个元素都是 `Button`，不能直接混入 `SelectBox`。不同的 `GenericScreen<T>` 实例可以选择不同的 `T`。
- 泛型并不意味着永远只能表示一种业务组件：也可以让 `T` 是枚举，再由枚举实现 `Draw`。关键是同一个 `Vec<T>` 只有一种元素类型。
- trait 对象隐藏具体类型，但值的数据仍然存在。通过对象只能调用所暴露的 trait 方法，不能直接访问具体类型的字段或额外方法。

### 实现trait

下面代码接在 `Draw` 和 `Screen` 定义之后。`SelectBox` 可以由库用户定义，而 `Screen::run` 不需要修改。

```rust
pub struct Button {
    pub width: u32,
    pub height: u32,
    pub label: String,
}

impl Draw for Button {
    fn draw(&self) {
        println!("按钮：{}（{} × {}）", self.label, self.width, self.height);
    }
}

struct SelectBox {
    options: Vec<String>,
}

impl Draw for SelectBox {
    fn draw(&self) {
        println!("下拉框：{:?}", self.options);
    }
}

fn main() {
    let screen = Screen {
        components: vec![
            Box::new(Button {
                width: 80,
                height: 30,
                label: String::from("确定"),
            }),
            Box::new(SelectBox {
                options: vec![String::from("是"), String::from("否")],
            }),
        ],
    };

    screen.run();
}
```

**讨论：**

- 根据目标类型，`Box<Button>` 和 `Box<SelectBox>` 可以强制转换为 `Box<dyn Draw>`，不需要手写 `as`。
- 这类似“只关心行为”的鸭子类型，但 Rust 要求显式实现 trait。仅仅定义一个同名的固有方法 `draw` 并不够。
- 如果放入 `Box::new(String::from("hello"))`，会得到 E0277：``the trait bound `String: Draw` is not satisfied``。
- 库用户可以为自己定义的类型实现库里的 `Draw`。但不能任意为外部类型实现外部 trait，仍需遵守孤儿规则；必要时用本地 newtype 包装外部类型。

### trait对象执行动态分发

泛型通常通过单态化为使用到的具体类型生成代码，调用目标在编译期确定；trait 对象通常借助虚函数表（vtable）在运行时选择实现。

```rust
fn draw_static<T: Draw>(value: &T) {
    value.draw();
}

fn draw_dynamic(value: &dyn Draw) {
    value.draw();
}

fn example(button: &Button, select_box: &SelectBox) {
    draw_static(button);
    draw_static(select_box);

    let components: [&dyn Draw; 2] = [button, select_box];
    for component in components {
        draw_dynamic(component);
    }
}
```

**讨论：**

- 可以把 trait 对象指针理解为“数据指针 + vtable 元数据”。vtable 提供方法实现等信息，具体布局不是可依赖的稳定 ABI。
- 动态分发通常增加一次间接调用，并可能妨碍内联；若优化器能够推断目标，也可能去虚拟化。因此不能断言它绝不可能被内联。
- 动态分发与堆分配是两件事：上面的 `&dyn Draw` 不需要额外堆分配；`Box` 则用于拥有装箱值。
- 静态分发便于优化，但大量单态化可能增大代码体积。trait 对象以一定运行时成本换取统一接口和开放的实现类型集合。

### dyn兼容性与常见限制

并非所有 trait 都能用作 `dyn Trait`。这个要求称为 **dyn 兼容性（dyn compatibility）**，旧资料中常称为“对象安全（object safety）”。

常见规则包括：

- trait 不能要求 `Self: Sized`，其 supertrait 也必须 dyn 兼容。
- 不能包含关联常量或泛型关联类型（GAT）。普通关联类型可以使用，但通常需要在对象类型中指定。
- 可动态调用的方法不能有类型泛型参数；生命周期泛型参数可以存在。
- 除接收者外，可动态调用的方法不能使用 `Self` 作为参数或返回值，也不能返回 `impl Trait` 或直接写成 `async fn`。
- 接收者必须支持动态调用，例如 `&self`、`&mut self`、`self: Box<Self>`。
- 不符合动态调用条件的方法可以加 `where Self: Sized`，将其排除在 trait 对象接口之外。

```rust
trait Widget {
    fn draw(&self);

    // 只能用于 Sized 的实现类型，不属于 dyn Widget 的可调用接口。
    fn duplicate(&self) -> Self
    where
        Self: Sized;

    fn configure<T>(&mut self, value: T)
    where
        Self: Sized;
}

fn render(widget: &dyn Widget) {
    widget.draw();
    // widget.duplicate(); // 错误：该方法要求 Self: Sized
}

// 普通关联类型不妨碍形成 trait 对象，但要固定对应类型。
fn sum_values(values: &mut dyn Iterator<Item = i32>) -> i32 {
    let mut total = 0;
    while let Some(value) = values.next() {
        total += value;
    }
    total
}
```

下面是故意不能编译的反例：

```rust
trait Factory {
    fn create(&self) -> Self;
}

fn use_factory(factory: &dyn Factory) {}
// E0038: the trait `Factory` is not dyn compatible
```

**讨论：** 调用方不知道 `dyn Factory` 背后的具体类型，也就不能用统一接口接收返回的 `Self`。可以给方法增加 `where Self: Sized`，或根据需求改成返回 `Box<dyn Factory>`。前者保留具体类型接口，但无法通过 trait 对象调用该方法；后者会改变 API 的所有权与分配方式。

### trait对象的生命周期

trait 对象隐藏类型，却不能隐藏借用是否合法。尤其要区分指针本身的生命周期与对象内部引用的有效期。

```rust
struct Label<'a> {
    text: &'a str,
}

impl Draw for Label<'_> {
    fn draw(&self) {
        println!("{}", self.text);
    }
}

fn boxed_label<'a>(text: &'a str) -> Box<dyn Draw + 'a> {
    Box::new(Label { text })
}

fn main() {
    let text = String::from("借用的标签");
    let label = boxed_label(&text);
    label.draw();
}
```

**讨论：**

- 在结构体字段、函数返回类型等没有其他推导来源的位置，`Box<dyn Draw>` 通常默认是 `Box<dyn Draw + 'static>`。这限制内部不能包含短于 `'static` 的借用，并不是要求这个 `Box` 必须存活到程序结束。
- 由 `String` 等自有数据组成的类型通常可以满足 `'static`；包含局部字符串引用的 `Label<'a>` 则需要显式保留 `'a`。
- `&'a dyn Draw` 通常将对象生命周期约束默认为 `'a`，不能把 `Box` 的默认规则机械套用到所有引用上。
- 原来的 `Screen` 适合存放自有组件；若要保存借用型组件，可改成 `Screen<'a>` 并使用 `Vec<Box<dyn Draw + 'a>>`。

## 面向对象设计模式的实现

状态模式（state pattern）把各状态的行为放入不同的状态对象，宿主将操作委托给当前状态。以博客发布为例：

```text
Draft --request_review--> PendingReview --approve--> Published
```

本节约定：只能编辑草稿，草稿和待审核文章不公开正文，未请求审核不能直接发布。动态状态版本对不适用的操作选择“不改变状态”；真实业务也可以返回 `Result` 明确报告错误。

### 使用传统的面向对象模式

调用方始终操作同一种类型 `Post`，不直接持有状态对象。下面展示目标用法；完整实现见“返回content”一节。

```rust
fn main() {
    let mut post = Post::new();
    post.add_text("我今天学习了 Rust 的状态模式。");
    assert_eq!(post.content(), "");

    post.approve(); // 草稿不能直接发布
    assert_eq!(post.content(), "");

    post.request_review();
    post.add_text("这段文字不会被加入。"); // 待审核时不可编辑
    assert_eq!(post.content(), "");

    post.approve();
    assert_eq!(post.content(), "我今天学习了 Rust 的状态模式。");
}
```

**讨论：** 动态状态模式把转换规则封装在内部，但 `post.approve()` 在草稿阶段仍然可以通过编译。它保证的是运行时规则，不是“所有不合法的方法调用都无法编译”。

#### 定义post并且创建一个实例

`Post` 存储正文与当前状态。以下各小节的片段用于解释实现，不要与后面的完整版本重复粘贴。

```rust
pub struct Post {
    state: Option<Box<dyn State>>,
    content: String,
}

impl Post {
    pub fn new() -> Self {
        Self {
            state: Some(Box::new(Draft)),
            content: String::new(),
        }
    }
}
```

**讨论：** `State` 和具体状态类型保持私有。对于定义模块之外的调用者，私有字段阻止它们任意构造或篡改状态。`Option` 用于转换过程中暂时取走状态，并不表示业务上存在“无状态文章”。

#### 存储文章内容的文本

`add_text` 根据当前状态决定是否接受编辑：

```rust
// impl Post 中的方法
pub fn add_text(&mut self, text: &str) {
    if self.state.as_ref().expect("文章必须有状态").can_edit() {
        self.content.push_str(text);
    }
}
```

**讨论：** 若仅调用 `push_str`，文章在待审核甚至发布之后也能被修改。这里让 `State::can_edit` 默认返回 `false`，仅由 `Draft` 覆盖为 `true`，明确实现“只能编辑草稿”的规则。静默忽略便于演示；生产 API 往往更适合返回编辑失败的原因。

#### 确保草稿文章的内容为空

这里的“内容为空”指**公开读取结果为空**，不是把保存的正文清空。状态 trait 为未发布状态提供默认读取行为：

```rust
// trait State 中的默认方法
fn content<'a>(&self, _post: &'a Post) -> &'a str {
    ""
}
```

**讨论：** `Draft` 和 `PendingReview` 复用该实现，正文仍保留在 `Post` 中。示例只是在演示 API 规则，不等同于完善的权限或数据保密机制。

#### 请求审核并且改变文章状态

转换操作消费旧状态，并返回新状态：

```rust
// impl Post 中的方法
pub fn request_review(&mut self) {
    if let Some(state) = self.state.take() {
        self.state = Some(state.request_review());
    }
}

// impl State for Draft 中的方法
fn request_review(self: Box<Self>) -> Box<dyn State> {
    Box::new(PendingReview)
}
```

**讨论：**

- `self: Box<Self>` 获取状态盒子的所有权，也能通过 `Box<dyn State>` 动态调用。它不同于仅借用状态的 `&self`。
- `Option::take` 取出原值并留下 `None`，使结构体字段始终处于已初始化状态。不能直接从 `&mut self` 背后移走非 `Copy` 字段；此类操作会涉及 E0507：`cannot move out of ... which is behind a mutable reference`。
- `Option` 不是所有状态机的必需品，也可以用 `mem::replace` 放入临时替代值。这里用 `None` 表达短暂的空位最直观。
- 正常返回后恢复 `Some`。如果状态转换 panic 并被调用方捕获，字段可能停留在 `None`；此简化实现不提供 panic 后继续使用的恢复保证。

#### 添加approve来改变content的行为

`Post::approve` 与 `request_review` 使用相同的“取出—转换—放回”步骤。只有 `PendingReview` 的审批会产生 `Published`。

```rust
// impl State for PendingReview 中的方法
fn approve(self: Box<Self>) -> Box<dyn State> {
    Box::new(Published)
}

// impl State for Draft 中的方法
fn approve(self: Box<Self>) -> Box<dyn State> {
    self // 尚未请求审核，保持草稿
}
```

**讨论：** 重复请求审核、重复批准等操作也必须定义清楚。这里选择保持原状态；如果业务需要禁止这些操作，应让接口显式返回错误，而不是依赖调用者猜测。

#### 返回content

下面是完整动态状态实现，可与本节开头的 `main` 一起使用：

```rust
mod blog {
    pub struct Post {
        state: Option<Box<dyn State>>,
        content: String,
    }

    impl Post {
        pub fn new() -> Self {
            Self {
                state: Some(Box::new(Draft)),
                content: String::new(),
            }
        }

        pub fn add_text(&mut self, text: &str) {
            if self.state.as_ref().expect("文章必须有状态").can_edit() {
                self.content.push_str(text);
            }
        }

        pub fn content(&self) -> &str {
            self.state.as_ref().expect("文章必须有状态").content(self)
        }

        pub fn request_review(&mut self) {
            if let Some(state) = self.state.take() {
                self.state = Some(state.request_review());
            }
        }

        pub fn approve(&mut self) {
            if let Some(state) = self.state.take() {
                self.state = Some(state.approve());
            }
        }
    }

    trait State {
        fn request_review(self: Box<Self>) -> Box<dyn State>;
        fn approve(self: Box<Self>) -> Box<dyn State>;

        fn can_edit(&self) -> bool {
            false
        }

        fn content<'a>(&self, _post: &'a Post) -> &'a str {
            ""
        }
    }

    struct Draft;
    struct PendingReview;
    struct Published;

    impl State for Draft {
        fn request_review(self: Box<Self>) -> Box<dyn State> {
            Box::new(PendingReview)
        }

        fn approve(self: Box<Self>) -> Box<dyn State> {
            self
        }

        fn can_edit(&self) -> bool {
            true
        }
    }

    impl State for PendingReview {
        fn request_review(self: Box<Self>) -> Box<dyn State> {
            self
        }

        fn approve(self: Box<Self>) -> Box<dyn State> {
            Box::new(Published)
        }
    }

    impl State for Published {
        fn request_review(self: Box<Self>) -> Box<dyn State> {
            self
        }

        fn approve(self: Box<Self>) -> Box<dyn State> {
            self
        }

        fn content<'a>(&self, post: &'a Post) -> &'a str {
            &post.content
        }
    }
}

use blog::Post;
```

**讨论：**

- `Post::content` 委托给当前状态。只有 `Published` 返回正文，另外两个状态使用返回空字符串的默认实现。
- `State::content` 的返回引用来自 `post`，因此显式用 `'a` 将返回值与 `post` 绑定。若直接省略，带 `&self` 的方法会默认把输出引用绑定到接收者，这不是此处要表达的关系。
- 方法上的生命周期参数不妨碍 dyn 兼容性；`State` 可以作为 trait 对象使用。
- `Draft` 等是零大小类型；虽然使用了 `Box`，这些状态本身不一定发生实际堆分配。若状态携带非零大小数据，分配成本就需要考虑。

#### 评估状态模式

| 优点 | 代价 |
| --- | --- |
| 同一状态的行为集中在一个实现中 | 状态转换仍依赖目标状态类型 |
| 调用方始终使用 `Post` | 不适用的操作仍可通过编译 |
| 宿主不需要针对状态编写 `match` | 转发方法和“保持自身”实现可能重复 |
| 可以为各状态增加独立数据 | 存在动态分发及可能的分配成本 |

增加状态通常要新增类型及其实现，也需要修改指向它的转换。增加一个没有默认实现的 trait 方法，则要修改所有状态实现，不能简单认为该模式能消除所有修改成本。

下面的默认方法是一个常见陷阱，**不能直接编译**：

```rust
trait State {
    fn approve(self: Box<Self>) -> Box<dyn State> {
        self
    }
}
// E0277: the size for values of type `Self` cannot be known at compilation time
```

**讨论：**

- 返回类型已经是已知大小的 `Box<dyn State>`；问题不是“不知道返回类型”，而是默认实现中的 `Self` 不保证 `Sized`，无法一般性地完成 `Box<Self>` 到 `Box<dyn State>` 的转换。
- 加上 `Self: Sized` 可以解决这一大小约束，但使该方法不能经由 trait 对象调用；同时仍需满足返回对象的生命周期约束。这无法替代示例所需的动态转换接口。
- 具体状态实现中的 `Self` 是已知的具体类型，因此 `self` 可以转换为 `Box<dyn State>`。
- 当状态集合固定而且很小，枚举加 `match` 通常比拆出大量状态对象更直接；编译器的穷尽检查也是优势，而不只是“必须修改分支”的负担。

#### 将状态和行为编码为类型

另一种设计叫类型状态（typestate）：让不同状态对应不同类型，只给各类型提供合法操作。

下面是与动态状态版本**相互独立**的完整实现：

```rust
mod typed_blog {
    pub struct DraftPost {
        content: String,
    }

    pub struct PendingReviewPost {
        content: String,
    }

    pub struct Post {
        content: String,
    }

    impl Post {
        // 为了与前面的入口一致，new 返回草稿类型。
        pub fn new() -> DraftPost {
            DraftPost {
                content: String::new(),
            }
        }

        pub fn content(&self) -> &str {
            &self.content
        }
    }

    impl DraftPost {
        pub fn add_text(&mut self, text: &str) {
            self.content.push_str(text);
        }

        pub fn request_review(self) -> PendingReviewPost {
            PendingReviewPost {
                content: self.content,
            }
        }
    }

    impl PendingReviewPost {
        pub fn approve(self) -> Post {
            Post {
                content: self.content,
            }
        }

        pub fn reject(self) -> DraftPost {
            DraftPost {
                content: self.content,
            }
        }
    }
}
```

**讨论：**

- `DraftPost` 没有 `content` 和 `approve`；`PendingReviewPost` 没有 `content` 和 `add_text`；只有 `Post` 暴露公开正文。
- 不再需要 `state` 字段：当前类型本身表达了状态，也不需要 vtable 或为状态对象装箱。正文 `String` 仍然使用自己的存储。
- 字段保持私有，模块外调用者必须走公开的转换路径。模块内部的实现仍然需要正确维护规则。
- `Post::new() -> DraftPost` 是合法 Rust，但不符合 `new` 通常返回 `Self` 的习惯，Clippy 的 `new_ret_no_self` lint 可能提示它。实际项目可改为 `DraftPost::new() -> Self`。

#### 实现状态转移为不同类型的转换

转换方法消费旧值并返回新值，调用方用变量遮蔽记录状态变化：

```rust
use typed_blog::Post;

fn main() {
    let mut post = Post::new(); // DraftPost
    post.add_text("类型状态让非法操作更难表达。");

    let post = post.request_review(); // PendingReviewPost
    let mut post = post.reject();     // DraftPost，退回修改
    post.add_text(" 补充了一段说明。");

    let post = post.request_review(); // PendingReviewPost
    let post = post.approve();        // Post
    assert_eq!(post.content(), "类型状态让非法操作更难表达。 补充了一段说明。");
}
```

下面的错误示例彼此独立：

```rust
let draft = Post::new();
// draft.content();
// E0599: no method named `content` found for struct `DraftPost` in the current scope

// draft.approve();
// E0599: no method named `approve` found for struct `DraftPost` in the current scope

let mut draft = Post::new();
let pending = draft.request_review();
// draft.add_text("再写一点");
// E0382: borrow of moved value: `draft`
```

**讨论：**

- `request_review(self)` 消费草稿，因此不能再用同一个旧值修改内容。正文的 `String` 被移动到新结构体，不需要克隆文本。
- `let post = ...` 创建新绑定，允许类型改变；普通赋值 `post = post.request_review()` 不能改变已有变量的类型，会得到 E0308：`mismatched types`。
- 编译期保证只覆盖编码进去的规则。这里保证先请求审核再调用批准，但没有保证批准者具有审核权限，也没有自动验证文章内容。
- 类型状态把工作流暴露给调用者，换来了更严格的编译期约束。如果需要在一个集合中保存任意状态的文章，或从数据库加载运行时状态，通常还要使用枚举或其他统一表示。

### 用枚举表示封闭的状态集合

如果状态种类固定，可以用枚举保留统一类型，并通过 `Result` 报告非法转换。下面只演示请求审核这一个转换：

```rust
#[derive(Debug, PartialEq, Eq)]
enum Status {
    Draft,
    PendingReview,
    Published,
}

struct Post {
    status: Status,
    content: String,
}

impl Post {
    fn request_review(&mut self) -> Result<(), &'static str> {
        match self.status {
            Status::Draft => {
                self.status = Status::PendingReview;
                Ok(())
            }
            Status::PendingReview => Err("文章已经在审核中"),
            Status::Published => Err("已发布文章不能再次提交审核"),
        }
    }
}
```

**讨论：**

- 枚举提供封闭的状态集合，可以携带各状态专属数据，并由 `match` 检查分支是否穷尽。这个示例只按变体匹配、没有移出字段，因此不要求 `Status: Copy`。
- 枚举版仍是在运行时检查转换是否合法，但无需 trait 对象；代价是增加状态时通常要修改相关匹配逻辑。
- 三种设计没有绝对优劣，选择应围绕需求，而不是以“越像传统 OOP 越好”为目标。

| 设计 | 适合场景 | 非法操作的主要处理方式 |
| --- | --- | --- |
| trait 对象状态模式 | 希望按状态组织行为，通过统一接口委托 | 运行时拒绝、忽略或返回错误 |
| 类型状态 | 流程清晰，希望调用顺序由类型系统约束 | 编译期缺少方法或所有权错误 |
| 枚举状态机 | 状态集合固定，需要统一存储、检查或持久化 | `match` 配合运行时错误处理 |

Rust 可以实现传统面向对象模式，也提供组合、泛型、枚举和所有权等工具。理解这些机制各自保证什么，比套用某一种模式更重要。
