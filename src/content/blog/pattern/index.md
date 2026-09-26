---
title: '模式与模式匹配'
publishDate: 'September 26, 2026'
updatedDate: 'September 26, 2026'
description: '失去兽性，失去一切'
tags:
  - Rust
  - Programming language
  - Pattern
language: 'Chinese'
heroImage: { src: 'treemirror.png', color: '#559ce8' }
---

模式（pattern）是 Rust 中用来描述和匹配数据结构的语法。把模式与 `match` 等结构结合使用，可以选择控制流，也可以从值中提取数据。

模式可以由以下内容组合而成：

- 字面量，如 `3`、`'a'`、`true`。
- 变量绑定，如 `x`。
- 数组、切片、元组、结构体或枚举的解构，如 `(x, y)`、`Some(value)`。
- 通配符 `_` 和剩余模式 `..`。
- 范围、或模式与 `@` 绑定，如 `1..=5`、`Some(_) | None`、`n @ 1..=5`。

**讨论：** 模式不是一般的表达式。表达式通常用于计算或构造值，模式则用于检查值的形状并绑定其中的部分。例如，表达式 `Some(x)` 构造一个值，模式 `Some(x)` 则检查是否为 `Some`，并把内部值绑定到 `x`。下面各代码块相互独立；标注“无法编译”的示例用于说明限制。

## 所有可能用到模式的地方

模式不仅出现在 `match` 中，也出现在变量绑定、条件、循环和参数列表等位置。

### match分支

```rust
fn main() {
    let value = Some(3);

    let number = match value {
        None => 0,
        Some(i) => i,
    };
    println!("{number}"); // 3

    match number {
        0 => println!("零"),
        other => println!("其他值：{other}"),
    }
}
```

**讨论：** 箭头左边的 `None`、`Some(i)`、`0` 和 `other` 都是模式。`match` 必须穷尽被匹配类型的所有可能情况；遗漏情况会产生 E0004（`non-exhaustive patterns`）。变量模式 `other` 能绑定所有剩余值；如果不需要使用值，则可以改成 `_`。分支按顺序检查，只执行第一个模式匹配且守卫成立的分支，因此兜底分支通常放在最后。`match` 本身是表达式，各分支的结果必须能统一为兼容的类型。

### let语句

普通 `let` 的形式是 `let PATTERN = EXPRESSION;`，左边并不局限于一个变量名。

```rust
fn main() {
    let x = 5;
    let (a, b, c) = (1, 2, 3);
    let (first, ..) = (10, 20, 30);

    println!("{x}, {a}, {b}, {c}, {first}");
}
```

下面的元组解构无法编译：

```rust
fn main() {
    let (x, y) = (1, 2, 3);
    // error[E0308]: mismatched types
    // 右侧是三个元素的元组，左侧模式要求两个元素。
}
```

**讨论：** `let` 是绑定语句，不是对已有变量赋值。元组模式的形状必须与类型一致；不需要某个位置时使用 `_`，不需要一段连续位置时使用 `..`。普通 `let` 还要求模式不可反驳，即不能留下匹配失败的可能性。

### 条件if let 表达式

```rust
fn main() {
    let favorite_color: Option<&str> = None;
    let is_tuesday = false;
    let age: Result<u8, _> = "34".parse();

    if let Some(color) = favorite_color {
        println!("使用喜欢的颜色：{color}");
    } else if is_tuesday {
        println!("使用绿色");
    } else if let Ok(age) = age {
        if age > 30 {
            println!("使用紫色");
        } else {
            println!("使用橙色");
        }
    } else {
        println!("使用蓝色");
    }
}
```

**讨论：** `if let` 适合只关心某一种形状、不需要列出全部变体的情况。这里的不同条件可以检查不同变量。`Ok(age)` 创建了一个新的 `u8` 绑定，遮蔽外层的 `Result`；这个绑定只在对应的成功分支中可用。与 `match` 不同，`if let` 不要求穷尽，因此新增枚举变体时，它不会提醒你补充处理逻辑。

### while let条件循环

```rust
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel();
    tx.send(1).unwrap();
    tx.send(2).unwrap();
    tx.send(3).unwrap();
    drop(tx);

    while let Ok(value) = rx.recv() {
        println!("{value}");
    }
}
```

**讨论：** 每次迭代都会重新求值 `rx.recv()`，只要结果匹配 `Ok(value)` 就执行循环体。代码依次打印 `1`、`2`、`3`；所有发送端被丢弃且缓冲中的消息被取完后，`recv()` 返回 `Err`，循环结束。如果发送端仍存在而当前没有消息，`recv()` 会阻塞，并非立即返回 `Err`。也可以用 `while let Some(value) = stack.pop()` 不断弹出栈顶元素，但打印顺序会是后进先出。

### for循环

```rust
fn main() {
    let letters = vec!['a', 'b', 'c'];

    for (index, letter) in letters.iter().enumerate() {
        println!("{index}: {letter}");
    }
}
```

**讨论：** `enumerate()` 产生 `(usize, &char)`，模式 `(index, letter)` 解构每一个迭代项，`letter` 的类型是 `&char`。`for` 的模式必须不可反驳，不会自动跳过匹配失败的元素。遍历 `Option` 集合时，可以在循环体中使用 `if let`，或者使用 `.flatten()` 跳过 `None`。

### 函数参数

```rust
fn foo(x: i32) {
    println!("{x}");
}

fn print_coordinates(&(x, y): &(i32, i32)) {
    println!("Current location: ({x}, {y})");
}

fn main() {
    foo(1);
    print_coordinates(&(3, 5));

    let sum = |(x, y): (i32, i32)| x + y;
    println!("{}", sum((2, 3))); // 5
}
```

**讨论：** 参数名也是模式，函数和闭包参数都可以直接解构。`&(x, y)` 中的 `&` 匹配一层引用，内部的两个 `i32` 因为实现了 `Copy` 而被复制出来；它不是创建引用的表达式。参数模式必须不可反驳，不能直接写 `Some(x): Option<i32>`；应接收整个 `Option`，再在函数内部处理不同变体。

### let...else语句

```rust
fn print_number(input: Option<&str>) {
    let Some(text) = input else {
        println!("缺少输入");
        return;
    };

    let Ok(number) = text.parse::<i32>() else {
        println!("不是合法整数");
        return;
    };

    println!("整数：{number}");
}

fn main() {
    print_number(Some("42"));
}
```

**讨论：** `let...else` 适合“失败就提前退出，成功则继续处理”的场景，可减少嵌套。匹配成功后，`text`、`number` 在后续作用域中可用。`else` 必须发散，也就是不能正常执行到这条语句之后；常见写法有 `return`、循环中的 `break` / `continue`，以及 `panic!()`。仅在 `else` 中打印日志而不退出，会产生 E0308，诊断包含 `` `else` clause of `let...else` does not diverge ``。它不是在失败时提供默认值；需要默认值可使用 `match` 或 `unwrap_or`。

## 可反驳性：模式是否可能匹配失败

不可反驳模式（irrefutable）能匹配被检查类型的任意可能值；可反驳模式（refutable）则可能匹配失败。

| 使用位置 | 对模式的要求 |
| --- | --- |
| 普通 `let`、函数/闭包参数、`for` | 必须不可反驳 |
| `if let`、`while let`、`let...else` | 通常使用可反驳模式；使用不可反驳模式会收到警告 |
| `match` 分支 | 单个分支可以可反驳，但整体必须穷尽 |

```rust
fn main() {
    let some_option_value: Option<i32> = None;
    let Some(x) = some_option_value; // 无法编译
    // error[E0005]: refutable pattern in local binding
}
```

改用 `let...else` 处理失败路径：

```rust
fn main() {
    let some_option_value = Some(5);
    let Some(x) = some_option_value else {
        return;
    };
    println!("{x}");
}
```

反过来，下面的条件不会匹配失败：

```rust
fn main() {
    if let x = 5 {
        println!("{x}");
    }
    // warning: irrefutable `if let` pattern
    // 这里直接写 let x = 5; 即可。
}
```

**讨论：** 可反驳性要结合类型判断，而不是只看当前初始化的值。即使一个 `Option<i32>` 刚被初始化为 `Some(5)`，普通 `let Some(x) = ...` 仍不合法，因为其类型还允许 `None`。同样，`[a, b]` 对 `[i32; 2]` 是不可反驳模式，但对长度不固定的切片 `[i32]` 就可能匹配失败。`while let` 使用不可反驳模式时，模式本身永远不会终止循环，需特别注意退出条件。

## 模式语法

### 匹配字面量

```rust
fn main() {
    let x = 2;
    match x {
        1 => println!("一"),
        2 => println!("二"),
        3 => println!("三"),
        _ => println!("其他值"),
    }

    let command = "quit";
    match command {
        "quit" => println!("退出"),
        _ => println!("未知命令"),
    }
}
```

**讨论：** 字面量适合匹配固定值。模式必须与被匹配值的类型兼容；如果 `command` 是 `String`，可以先调用 `command.as_str()` 再匹配字符串字面量。模式中不能直接写一般的计算表达式，如 `1 + 2`；固定计算结果可预先定义为适合模式匹配的常量，运行时条件则使用守卫。

### 匹配命名变量

```rust
fn main() {
    let x = Some(5);
    let y = 10;

    match x {
        Some(50) => println!("得到 50"),
        Some(y) => println!("匹配到 y = {y}"),
        _ => println!("其他情况"),
    }

    println!("外部：x = {x:?}, y = {y}");
}
```

输出为 `匹配到 y = 5`，随后是 `外部：x = Some(5), y = 10`。

**讨论：** `Some(y)` 不是与外部变量 `y` 比较，而是创建新绑定。它会匹配任意 `Some`，并在当前分支遮蔽外层 `y`。如果要比较外部的值，应写 `Some(n) if n == y`。常量路径与普通变量绑定不同，例如 `const LIMIT: i32 = 10;` 中的 `LIMIT` 可以用作固定值模式；建议遵循常量大写的命名习惯，避免混淆。

### 匹配多个模式

```rust
fn main() {
    let x = 2;
    match x {
        1 | 2 | 3 => println!("一、二或三"),
        _ => println!("其他值"),
    }

    let result: Result<i32, i32> = Err(7);
    match result {
        Ok(n) | Err(n) => println!("内部数字：{n}"),
    }
}
```

**讨论：** `|` 表示多个备选模式，不是按位或计算。各备选模式必须绑定相同名称的变量，且绑定类型、绑定方式一致，否则共用的分支体无法安全使用它们。例如 `Some(x) | None` 不合法，会产生 E0408（`` variable `x` is not bound in all patterns ``）；只关心形状时可以写 `Some(_) | None`。

### 通过..=匹配范围

```rust
fn main() {
    let x = 5;
    match x {
        1..=5 => println!("一到五，包含两端"),
        _ => println!("其他数字"),
    }

    let letter = 'c';
    match letter {
        'a'..='z' => println!("ASCII 小写字母"),
        'A'..='Z' => println!("ASCII 大写字母"),
        _ => println!("其他字符"),
    }
}
```

**讨论：** `..=` 包含两个端点，整数和字符范围是最常见的用法。端点必须是编译期可确定的合法边界，不能直接使用普通运行时变量。动态区间可以写成守卫，例如 `n if (low..=high).contains(&n)`。字符范围按字符值判断，`'a'..='z'` 并不表示所有语言的小写字母；需要 Unicode 分类时应使用 `char::is_lowercase()` 等方法。

### 解构并分解值

解构可以提取字段，也可以在字段位置继续嵌套其他模式。

#### 结构体

```rust
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 0, y: 7 };
    let Point { x: a, y: b } = p;
    println!("{a}, {b}");

    let Point { x, y } = p;
    println!("{x}, {y}");

    match p {
        Point { x, y: 0 } => println!("位于 x 轴：{x}"),
        Point { x: 0, y } => println!("位于 y 轴：{y}"),
        Point { x, y } => println!("其他位置：({x}, {y})"),
    }
}
```

**讨论：** `x: a` 左边是字段名，右边是模式；`Point { x, y }` 是 `Point { x: x, y: y }` 的简写。字段位置也可以放字面量，从而同时解构和筛选。原点同时满足前两个分支，但只会执行第一个。本例字段均为 `i32`，提取时发生复制，所以可以继续使用 `p`；若字段为 `String`，按值绑定可能发生移动。

#### 枚举

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}

fn main() {
    let message = Message::Move { x: 3, y: 5 };

    match message {
        Message::Quit => println!("退出"),
        Message::Move { x, y } => println!("移动到 ({x}, {y})"),
        Message::Write(text) => println!("文本：{text}"),
        Message::ChangeColor(r, g, b) => println!("RGB：{r}, {g}, {b}"),
    }
}
```

**讨论：** 模式必须对应变体的定义形式：单元变体直接写路径，元组变体使用圆括号，结构体变体使用花括号。对不同变体分别解构，可以避免“先判断类型再手动取字段”的逻辑。这里没有兜底分支，未来增加变体时，编译器会提醒补齐处理。

#### 嵌套的结构体和枚举

```rust
enum Color {
    Rgb(i32, i32, i32),
    Hsv(i32, i32, i32),
}

enum Message {
    ChangeColor(Color),
    Quit,
}

fn main() {
    let message = Message::ChangeColor(Color::Hsv(180, 80, 90));

    match message {
        Message::ChangeColor(Color::Rgb(r, g, b)) => {
            println!("RGB：{r}, {g}, {b}");
        }
        Message::ChangeColor(Color::Hsv(h, s, v)) => {
            println!("HSV：{h}, {s}, {v}");
        }
        Message::Quit => println!("退出"),
    }

    let ((a, b), [c, d]) = ((1, 2), [3, 4]);
    println!("{a}, {b}, {c}, {d}");
}
```

**讨论：** 模式可以递归嵌套，不局限于一种数据结构。外层变体匹配后，内层模式继续检查数据形状。层次过深时，可以先解构外层，再交给辅助函数处理，避免单个分支承担过多业务逻辑。

#### 数组和切片

```rust
fn describe(values: &[i32]) {
    match values {
        [] => println!("空切片"),
        [only] => println!("只有一个元素：{only}"),
        [first, middle @ .., last] => {
            println!("首：{first}，中间：{middle:?}，尾：{last}");
        }
    }
}

fn main() {
    let [a, b, c] = [1, 2, 3];
    println!("{a}, {b}, {c}");

    let values = vec![10, 20, 30, 40];
    describe(&values);
}
```

**讨论：** 数组的长度属于类型的一部分，固定长度数组可以用对应长度的模式直接解构。切片长度在运行时确定，通常需要用 `match` 或 `if let` 检查。`[first, middle @ .., last]` 至少要求两个元素，中间部分可以为空；这里匹配的是 `&[i32]`，所以 `first`、`last` 为 `&i32`，`middle` 为 `&[i32]`，不会复制整个切片。`Vec` 可通过 `.as_slice()` 或引用转换为切片后匹配。

### 忽略模式中的值

#### 用_忽略值

```rust
fn foo(_: i32, y: i32) {
    println!("只使用第二个参数：{y}");
}

fn main() {
    foo(3, 4);
}
```

**讨论：** `_` 匹配一个值，但不创建变量绑定，适合占位参数或兜底分支。忽略不等于不求值：`let _ = some_function();` 仍会调用函数并产生副作用；函数参数也仍然按参数类型传递，按值传入 `String` 不会因为参数模式是 `_` 就变成借用。

#### 用嵌套的下划线忽略部分值

```rust
fn main() {
    let mut setting_value = Some(5);
    let new_setting_value = Some(10);

    setting_value = match (setting_value, new_setting_value) {
        (Some(_), Some(_)) => {
            println!("不能覆盖已有设置");
            setting_value
        }
        (_, new_value) => new_value,
    };
    println!("设置：{setting_value:?}"); // Some(5)

    let numbers = (2, 4, 8, 16, 32);
    let (first, _, third, _, fifth) = numbers;
    println!("{first}, {third}, {fifth}");
}
```

**讨论：** `Some(_)` 仍然要求变体为 `Some`，只是忽略内部值；单独的 `_` 则连变体也不限制。设置示例拒绝覆盖已有值，但允许新值为 `None` 时取消设置，也允许从未设置变为已设置。这里的 `Option<i32>` 实现了 `Copy`；换成 `Option<String>` 时，构造待匹配元组会移动原值，应在分支中绑定并返回旧值，例如 `(old @ Some(_), Some(_)) => old`。

#### 通过在变量名开头加_忽略值

```rust
fn main() {
    let _unused = 5; // 有绑定，但不警告未使用。
    let unused = 10; // warning: unused variable: `unused`

    let value = Some(String::from("hello"));
    if let Some(_) = value {
        println!("存在字符串");
    }
    println!("{value:?}"); // 内部字符串未被移动。
}
```

`_name` 与 `_` 不同，下面的代码无法编译：

```rust
fn main() {
    let value = Some(String::from("hello"));
    if let Some(_text) = value {
        // _text 是真正的绑定，String 被移动到这里。
    }
    println!("{value:?}");
    // error[E0382]: borrow of partially moved value: `value`
}
```

**讨论：** 下划线前缀只抑制未使用变量的警告，不改变所有权规则。`_` 不绑定，`_text` 会绑定并可能移动值。需要保留资源直到作用域结束时，这一点尤其重要：锁守卫通常应保存为 `let _guard = ...;`，不能简单换成 `let _ = ...;`，后者不会把临时守卫保留到作用域末尾。

#### 用..忽略值的剩余部分

```rust
struct Point {
    x: i32,
    y: i32,
    z: i32,
}

fn main() {
    let point = Point { x: 1, y: 2, z: 3 };
    let Point { x, .. } = point;
    println!("x = {x}");

    let numbers = (2, 4, 8, 16, 32);
    let (first, .., last) = numbers;
    println!("首尾：{first}, {last}");
}
```

**讨论：** `_` 忽略一个位置，`..` 忽略当前结构中未显式列出的剩余部分，数量可以为零。结构体模式中的 `..` 放在字段列表末尾；元组、数组和切片中可以放在中间。同一层不能写多个 `..`，例如 `(.., second, ..)` 无法确定如何划分位置，编译器会报告 `` `..` can only be used once per tuple pattern ``。不同嵌套层可以分别使用自己的 `..`。

#### 使用匹配守卫添加额外条件

```rust
fn main() {
    let number = Some(4);
    let expected = 10;

    match number {
        Some(x) if x % 2 == 0 => println!("偶数：{x}"),
        Some(x) if x == expected => println!("等于外部变量"),
        Some(x) => println!("其他整数：{x}"),
        None => println!("没有值"),
    }

    let x = 5;
    let enabled = false;
    match x {
        4 | 5 | 6 if enabled => println!("已启用"),
        _ => println!("未匹配"),
    }
}
```

**讨论：** 守卫在模式匹配成功后求值，可以引用模式中的绑定和外部变量。守卫失败会继续检查后续分支，而不是直接结束 `match`。第一段中的 `Some(10)` 会先命中偶数分支，不会进入后面的相等分支。

`4 | 5 | 6 if enabled` 的条件作用于整个或模式，相当于 `(4 | 5 | 6) if enabled`，并非只约束 `6`。带守卫的分支不用于保证穷尽覆盖，即使条件看起来必然成立，也应保留无守卫分支覆盖剩余情况。匹配守卫是 `match` 分支语法；在 `if let` 或 `while let` 中，可在成功匹配后另写 `if` 判断。

#### 使用@绑定

```rust
enum Message {
    Hello { id: i32 },
}

fn main() {
    let message = Message::Hello { id: 5 };

    match message {
        Message::Hello {
            id: id_variable @ 3..=7,
        } => println!("范围内的 id：{id_variable}"),
        Message::Hello { id: 10..=12 } => println!("id 位于 10 到 12"),
        Message::Hello { id } => println!("其他 id：{id}"),
    }
}
```

**讨论：** `name @ subpattern` 一边检查子模式，一边把该位置的值绑定到 `name`。仅写 `3..=7` 可以筛选但没有绑定，仅写 `id_variable` 可以绑定但没有筛选；`@` 将两者结合起来。它也可用于切片的 `rest @ ..`。绑定仍然遵循所有权规则：若同时按值绑定一个非 `Copy` 的整体和它的非 `Copy` 字段，就可能发生移动冲突；需要两者时通常应改为借用匹配。

### 模式、引用与所有权

```rust
struct User {
    name: String,
    age: u8,
}

fn main() {
    let mut user = User {
        name: String::from("小明"),
        age: 18,
    };

    // 匹配引用时，匹配人体工学会使内部绑定采用引用方式。
    let User { name, age } = &user;
    // name: &String，age: &u8
    println!("{name}, {age}");

    let User { name, .. } = &mut user;
    // name: &mut String
    name.push_str("同学");

    // ref 在模式中创建借用绑定，而不是移动 String。
    let User { ref name, .. } = user;
    println!("{name}");
    println!("{}", user.name); // 仍可使用。
}
```

**讨论：** 不要把“模式匹配”理解成总会复制数据。对拥有所有权的值按值绑定时，`Copy` 字段被复制，非 `Copy` 字段通常被移动；使用 `&value` 或 `&mut value` 作为匹配对象，通常可以只借用数据。匹配人体工学（match ergonomics）允许用 `Some(text)` 等简洁模式匹配引用，并自动让内部绑定成为引用。

`ref name` 表示绑定一个共享引用，`ref mut name` 表示绑定一个可变引用；`mut name` 则只是允许重新赋值这个绑定，不等价于可变借用。模式中的 `&` 用于匹配一层引用，表达式中的 `&` 用于创建引用，二者方向不同。Rust 2024 收紧了引用模式和显式绑定修饰符的组合规则：依赖自动引用绑定时，不要再随意叠加 `ref`、`ref mut` 或 `mut`；优先采用上面直接匹配 `&user`、`&mut user` 的写法。

下面展示非 `Copy` 字段被移动后的限制：

```rust
struct User {
    name: String,
    age: u8,
}

fn main() {
    let user = User {
        name: String::from("小明"),
        age: 18,
    };
    let User { name, .. } = user;

    println!("{name}");
    println!("{}", user.age); // 未移动的字段仍可使用。
    // println!("{}", user.name); // E0382：该字段已经移动。
    // drop(user); // E0382：不能再把部分移动后的 user 作为整体使用。
}
```

**讨论：** 这称为部分移动。借用解构可以避免部分移动，但借用期间仍受“共享引用或独占可变引用”的规则约束。此外，实现了 `Drop` 的类型不能这样移出其非 `Copy` 字段，否则会产生 E0509（`cannot move out of type ..., which implements the Drop trait`）；通常应选择借用，或通过 `std::mem::take` 等方式替换字段后取出旧值。
