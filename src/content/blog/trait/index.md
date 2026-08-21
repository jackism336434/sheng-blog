---

title: 'Rust的泛型'

publishDate: July 24, 2026

updatedDate: 'July 24, 2026'

description: 'Rust通过泛型(generics)实现代码复用，允许使用类型占位符替代具体类型。'

tags:

- Rust

- 泛型

- Programming language

language: 'Chinese'

heroImage: { src: 'ASCII.png', color: '#9698C1' }

---





每种编程语言都有一些工具，用来高效处理概念上的重复。在 Rust 中，这样的工具之一就是泛型（generics）：它是具体类型或其他属性的抽象占位符。我们可以描述泛型的行为，或者它们与其他泛型之间的关系，而不需要在编译和运行代码时就提前知道它们具体会被什么替换。

## 提取函数来减少重复

泛型允许我们使用一个可以代表多种类型的占位符来替换特定类型，以此来减少代码冗余。在深入了解泛型的语法之前，我们首先来看一种没有使用泛型的减少冗余的方法，即提取一个函数。在这个函数中，我们用一个可以代表多种值的占位符来替换具体的值。接着我们使用相同的技术来提取一个泛型函数！通过学习如何识别并提取可以整合进一个函数的重复代码，你也会开始识别出可以使用泛型的重复代码。

接下来，我们会用同样的步骤借助泛型来减少重复。就像函数体可以处理抽象的 list、而不是特定的值一样，泛型也允许代码处理抽象类型。

例如，假设我们有两个函数：一个用来找出 i32 切片中的最大项，另一个用来找出 char 切片中的最大项。我们该如何消除这种重复呢？继续往下看。

先看第一步：如果只针对 `i32` 类型求最大值，代码可能是这样写在 `main` 里的：

```rust
fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let mut largest = &number_list[0];

    for number in &number_list {
        if number > largest {
            largest = number;
        }
    }

    println!("The largest number is {largest}");
}
```

这段代码把“遍历切片、比较大小、记录当前最大项”的逻辑全部内联在 `main` 里。如果我们要对第二个 `i32` 列表也求最大值，就不得不把整个 `for` 循环再复制粘贴一遍。这正是“概念上的重复”。

第二步：提取一个函数，用一个参数 `list` 来充当“任意列表”的占位符：

```rust
fn largest(list: &[i32]) -> &i32 {
    let mut largest = &list[0];

    for item in list {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];
    let result = largest(&number_list);
    println!("The largest number is {result}");

    let number_list = vec![102, 34, 6000, 89, 54, 2, 43, 8];
    let result = largest(&number_list);
    println!("The largest number is {result}");
}
```

这里我们让`list: &[i32]` 不再绑定某个具体的列表，而是接受任意 `i32` 切片。函数体处理的是抽象的 `list`，调用者通过传参来决定它到底处理哪份数据。这就是减少重复的核心思路：**识别重复、用占位符替换具体的部分**。

但 `largest` 仍然只能处理 `i32`。如果我们想对 `char` 切片也做同样的事，就会得到第二个几乎一模一样的函数：

```rust
fn largest_i32(list: &[i32]) -> &i32 {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}

fn largest_char(list: &[char]) -> &char {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];
    let result = largest_i32(&number_list);
    println!("The largest number is {result}");

    let char_list = vec!['y', 'm', 'a', 'q'];
    let result = largest_char(&char_list);
    println!("The largest char is {result}");
}
```

现在两个函数的函数体**逐字相同**，唯一的区别是类型标注：`i32` 还是 `char`。这提示我们：既然刚才可以用参数 `list` 抽象“具体的值”，为什么不能再用一个参数 `T` 抽象“具体的类型”呢？这就是泛型。

## 泛型数据类型

我们使用泛型来为函数签名或结构体之类的项创建定义，这样它们就可以配合多种不同的具体数据类型使用。先来看看如何使用泛型定义函数、结构体、枚举和方法。然后再讨论泛型会如何影响代码性能。

### 在函数定义中使用泛型

当使用泛型定义函数时，本来在函数签名中指定参数和返回值的类型的地方，会改用泛型来表示。采用这种技术，使得代码适应性更强，从而为函数的调用者提供更多的功能，同时也避免了代码的重复。

为了给这个新函数中的类型做参数化，我们需要给类型参数命名，就像给函数的值参数命名一样。任何标识符都可以作为类型参数名。但这里我们使用 T，因为按照惯例，Rust 中的类型参数名都很短，通常只有一个字母，而 Rust 类型名的命名约定是 UpperCamelCase。T 是 type 的缩写，也是大多数 Rust 程序员的默认选择。

把上一节的两个函数合并，先把类型位置换成 `T`：

```rust
fn largest<T>(list: &[T]) -> &T {
    let mut largest = &list[0];

    for item in list {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];
    let result = largest(&number_list);
    println!("The largest number is {result}");

    let char_list = vec!['y', 'm', 'a', 'q'];
    let result = largest(&char_list);
    println!("The largest char is {result}");
}
```

语法上，`fn largest<T>` 在函数名后、参数列表前用尖括号 `<>` 声明了类型参数 `T`，随后 `list: &[T] -> &T` 表示“接受一个 `T` 的切片，返回一个对 `T` 的引用”。注意 `T` 在这里只是一个占位符：它不代表任何具体类型，而是“调用者将来会告诉我的那个类型”。

不过，**这段代码无法编译**。原因在于函数体里用了 `item > largest`。并非所有类型都支持 `>` 比较——例如一个自定义的结构体默认并不知道怎么比大小。`T` 可能是任意类型，编译器不能假设 `T` 一定实现了比较操作。要让它通过编译，就必须给 `T` 加上“能比较”的约束（trait bound）：

```rust
use std::cmp::PartialOrd;

fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];

    for item in list {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];
    let result = largest(&number_list);
    println!("The largest number is {result}");

    let char_list = vec!['y', 'm', 'a', 'q'];
    let result = largest(&char_list);
    println!("The largest char is {result}");
}
```

*`T: PartialOrd` 读作“`T` 必须实现 `PartialOrd` trait”，也就是“`T` 必须支持部分比较”。这样编译器就知道函数体内的 `>` 是合法的。`i32` 和 `char` 都实现了 `PartialOrd`，所以 `main` 里两次调用都能成功。**泛型的核心权衡**就在这里：函数越通用（接受越多类型），函数体内能对 `T` 做的事就越少，除非你用 trait bound 明确告诉编译器 `T` 具备哪些能力。关于 trait 的细节会在后面的章节展开，这里先把它当作一种“类型能力声明”即可。

### 结构体定义中的泛型

我们也可以使用 <> 语法来定义结构体，让一个或多个字段使用泛型类型参数。示例定义了一个 Point<T> 结构体，用来保存任意类型的 x 和 y 坐标值：

```rust
struct Point<T> {
    x: T,
    y: T,
}

fn main() {
    let integer = Point { x: 5, y: 10 };
    let float = Point { x: 1.0, y: 4.0 };
}
```

`struct Point<T>` 中的 `T` 是结构体级别的类型参数，`x` 和 `y` 共用同一个 `T`。所以 `integer` 和 `float` 分别实例化出 `Point<i32>` 和 `Point<f64>`。注意约束：因为 `x`、`y` 都是 `T`，它们**必须是同一种类型**，下面的代码会编译失败：

```rust
struct Point<T> {
    x: T,
    y: T,
}

fn main() {
    // 错误！x 是 i32，y 是 f64，但 Point<T> 只允许一个类型 T
    let wont_work = Point { x: 5, y: 4.0 };
}
```

错误会提示“expected integer, found floating-point number”。这就是“单类型参数”的含义：`Point<T>` 的两个坐标必须同类型。

如果想要定义一个 x 和 y 可以有不同类型且仍然是泛型的 Point 结构体，我们可以使用多个泛型类型参数。在示例中，我们修改 Point 的定义为拥有两个泛型类型 T 和 U。其中字段 x 是 T 类型的，而字段 y 是 U 类型的：

```rust
struct Point<T, U> {
    x: T,
    y: U,
}

fn main() {
    let both_integer = Point { x: 5, y: 10 };
    let both_float = Point { x: 1.0, y: 4.0 };
    let integer_and_float = Point { x: 5, y: 4.0 };
}
```

现在 `x: T` 与 `y: U` 相互独立，于是 `both_integer`（`Point<i32, i32>`）、`both_float`（`Point<f64, f64>`）、`integer_and_float`（`Point<i32, f64>`）三种组合都合法。我们需要多少个独立的类型，就声明多少个类型参数。

### 枚举定义中的泛型

和结构体类似，枚举也可以在成员中存放泛型数据类型。我们曾用过标准库提供的 Option<T> 枚举，现在这个定义应该更容易理解了：

```rust
enum Option<T> {
    Some(T),
    None,
}
```

如你所见 Option<T> 是一个拥有泛型 T 的枚举，它有两个成员：Some，它存放了一个类型 T 的值，和不存在任何值的 None。通过 Option<T> 枚举可以表达有一个可能的值的抽象概念，同时因为 Option<T> 是泛型的，无论这个可能的值是什么类型都可以使用这个抽象。

`Option<T>` 的妙处在于，`Some` 里的值可以是任意类型：`Option<i32>`、`Option<String>`、`Option<&str>`……它们共用同一套“可能有值、可能没值”的语义，而不用为每种类型各写一个枚举。这就是标准库能用一个 `Option` 统一表达“可空”的原因。

枚举也可以拥有多个泛型类型。Result 枚举定义就是一个这样的例子：

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

`Result<T, E>` 有两个类型参数：`T` 是成功值的类型，`E` 是错误值的类型。之所以用两个，是因为成功和失败时携带的数据类型往往不同（例如 `Result<String, io::Error>` 表示“成功返回字符串，失败返回 IO 错误”）。如果只有 `T`，就无法表达“错误类型”这个独立维度。

### 方法定义中的泛型

我们可以像第五章那样为结构体和枚举实现方法，并在这些方法定义中使用泛型：

```rust
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}

fn main() {
    let p = Point { x: 5, y: 10 };
    println!("p.x = {}", p.x());
}
```

注意必须在 impl 后面声明 T，这样就可以在 Point<T> 上实现的方法中使用 T 了。通过在 impl 之后声明泛型 T，Rust 就知道 Point 的尖括号中的类型是泛型而不是具体类型。我们可以为泛型参数选择一个与结构体定义中声明的泛型参数所不同的名称，不过依照惯例使用了相同的名称。

`impl<T> Point<T>` 里的第一个 `<T>` 是“声明”：它告诉编译器“`T` 是一个类型参数”。而 `Point<T>` 里的 `T` 是“使用”：它引用刚刚声明的那个类型参数。如果漏掉 `impl` 后的 `<T>`，编译器会报错“cannot find type `T` in this scope”，因为它会把 `T` 当成某个实际存在的具体类型名。你在 `impl` 中编写一个声明了泛型类型的方法，该方法就会定义在 `Point<T>` 的**任何**实例上——无论最终 `T` 被替换成 `i32`、`f64` 还是别的类型，这个 `x` 方法都可用。

定义方法时也可以为泛型指定限制（constraint）。例如，可以选择为 Point<f32> 实例实现方法，而不是为泛型 Point 实例。示例 10-10 展示了一个没有在 impl 之后（的尖括号）声明泛型的例子，这里使用了一个具体类型，f32：

```rust
struct Point<T> {
    x: T,
    y: T,
}

impl Point<f32> {
    fn distance_from_origin(&self) -> f32 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}

fn main() {
    let p = Point { x: 3.0, y: 4.0 };
    println!("Distance from origin: {}", p.distance_from_origin()); // 5
}
```

这里 `impl Point<f32>` 后面**没有** `<T>`，因为 `f32` 是具体类型，不是泛型。结果是：`distance_from_origin` 只存在于 `Point<f32>` 上。`Point<i32>` 实例不能调用它——这也是合理的，因为 `powi` 和 `sqrt` 是浮点运算。这个例子展示了泛型的灵活性：**同一个结构体，可以对“所有类型”实现通用方法，也可以只对“某个具体类型”实现专用方法**，二者可以并存。

结构体定义中的泛型类型参数并不总是与结构体方法签名中使用的泛型是同一类型。示例 10-11 中为 Point 结构体使用了泛型类型 X1 和 Y1，为 mixup 方法签名使用了 X2 和 Y2 来使得示例更加清楚。这个方法用 self 的 Point 类型的 x 值（类型 X1）和参数的 Point 类型的 y 值（类型 Y2）来创建一个新 Point 类型的实例：

```rust
struct Point<X1, Y1> {
    x: X1,
    y: Y1,
}

impl<X1, Y1> Point<X1, Y1> {
    fn mixup<X2, Y2>(self, other: Point<X2, Y2>) -> Point<X1, Y2> {
        Point {
            x: self.x,
            y: other.y,
        }
    }
}

fn main() {
    let p1 = Point { x: 5, y: 10.4 };
    let p2 = Point { x: "Hello", y: 'c' };

    let p3 = p1.mixup(p2);

    println!("p3.x = {}, p3.y = {}", p3.x, p3.y); // p3.x = 5, p3.y = c
}
```

这个例子的目的是展示一些泛型通过 impl 声明而另一些通过方法定义声明的情况。这里泛型参数 X1 和 Y1 声明于 impl 之后，因为它们与结构体定义相对应。而泛型参数 X2 和 Y2 声明于 fn mixup 之后，因为它们只是相对于方法本身的。

注意两个层次的泛型是独立的：
- `X1`、`Y1` 属于 `self` 所在的结构体（`Point<X1, Y1>`），在 `impl<X1, Y1>` 处声明；
- `X2`、`Y2` 属于参数 `other`（`Point<X2, Y2>`），只在 `fn mixup<X2, Y2>` 处声明。

`mixup` 的返回类型 `Point<X1, Y2>` 很有意思：它从 `self` 拿 `x`（类型 `X1`），从 `other` 拿 `y`（类型 `Y2`），于是返回值的坐标类型是“混合”出来的。在 `main` 里，`p1` 是 `Point<i32, f64>`，`p2` 是 `Point<&str, char>`，所以 `p3` 是 `Point<i32, char>`。这个例子说明：**方法可以引入结构体泛型之外的新类型参数**，让单个方法的签名拥有自己的“局部泛型”。

### 泛型代码的性能

你可能会好奇，使用泛型类型参数是否会带来运行时开销。好消息是：使用泛型不会让程序比使用具体类型运行得更慢。

Rust 通过在编译时对泛型代码进行单态化（monomorphization）来实现这一点。单态化就是把泛型代码转换成具体代码的过程，方法是用编译时实际用到的具体类型去填充泛型代码。

先回顾一下，泛型代码本身在编译前是“抽象”的。假设我们写了：

```rust
let integer = Some(5);       // Option<i32>
let float = Some(5.0);       // Option<f64>
```

在这个过程中，编译器所做的工作正好与示例 10-5 中我们创建泛型函数的步骤相反。编译器寻找所有泛型代码被调用的位置并使用泛型代码针对具体类型生成代码。当 Rust 编译这些代码的时候，它会进行单态化。比如编译器会读取传递给 Option<T> 的值并发现有两种 Option<T>：一个对应 i32 另一个对应 f64。为此，它会将泛型定义 Option<T> 展开为两个针对 i32 和 f64 的定义，接着将泛型定义替换为这两个具体的定义。

单态化在概念上等价于编译器“自动展开”出如下代码：

```rust
// 概念示意：编译器生成的等价代码，而非你手写的代码
enum Option_i32 {
    Some(i32),
    None,
}

enum Option_f64 {
    Some(f64),
    None,
}

fn main() {
    let integer = Option_i32::Some(5);
    let float = Option_f64::Some(5.0);
}
```

因为编译器在编译期就把 `Option<T>` 替换成了 `Option_i32` 和 `Option_f64` 两个具体版本，所以运行时没有任何“查表判断类型”或“动态分发”的额外开销。泛型版本的代码和手写具体类型版本的代码**运行速度完全相同**。代价是：每个被实际使用的具体类型组合都会生成一份独立的机器码，可能使最终二进制体积略有增大——这就是“零运行时开销，但可能增加编译产物体积”的权衡。
