---

title: 'Rust的自动化测试'

publishDate: August 24, 2026

updatedDate: 'August 24, 2026'

description: '程序的正确性，指的是代码在多大程度上实现了我们的预期。为了确保程序的正确性，Rust 提供了编写自动化软件测试的支持。'

tags:

- Rust

- Test

- Automation

language: 'Chinese'

heroImage: { src: 'train.png', color: '#9698C1' }

---



## 编写测试

Rust 中的测试函数用来验证非测试代码是否按照期望的方式运行。一个测试通常包含三个步骤：

- 设置所需的数据或状态；
- 运行需要测试的代码；
- 断言结果是否符合预期。

Rust 的测试功能主要由 `#[test]` 属性、断言宏以及 `#[should_panic]` 属性组成。测试代码一般放在 `src/lib.rs` 或 `src/main.rs` 的 `tests` 模块中，并通过 `cargo test` 运行。

**讨论：** 自动化测试应覆盖正常路径、边界条件和预期错误，并保持彼此独立，这样才能在代码演进时持续提供可靠反馈。

### 精心组织测试函数

最简单的测试是一个带有 `#[test]` 属性的函数。属性（attribute）是附加在 Rust 代码上的元数据；结构体上的 `derive` 也是一种属性。`#[test]` 告诉测试执行器：在测试模式下构建测试程序时，需要调用这个函数。

下面的例子测试一个将两个数相加的函数：

```rust
pub fn add_two(a: i32) -> i32 {
    a + 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() {
        let result = add_two(2);
        assert_eq!(result, 4);
    }
}
```

`#[cfg(test)]` 表示这个模块只在测试配置下编译。测试模块中也可以定义辅助函数，但只有标记了 `#[test]` 的函数才会被测试执行器当作测试运行。

执行 `cargo test` 后，测试输出通常包含测试数量和每个测试的结果：

```text
running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

`ok` 表示测试通过，`FAILED` 表示测试函数发生了 panic。`ignored` 是被忽略的测试数量，`filtered out` 是因为名称过滤而没有运行的测试数量；`measured` 主要用于性能测试。

测试失败最直接的方式是调用 `panic!`，或者让断言失败：

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn another() {
        panic!("这个测试故意失败");
    }
}
```

每个测试通常在线程中并行运行。测试失败时，Cargo 会列出失败原因和失败测试名称，可以使用该名称重新运行单个测试以便调试。

**讨论：** 测试函数应该只验证一个清晰的行为，并使用有意义的名称描述被验证的场景。测试失败并不代表测试框架出错，而是说明断言没有成立，或者被测代码发生了 panic。把准备数据、调用代码和断言分开，能够让失败位置更容易定位。

### 使用 `assert!` 宏来检查结果

`assert!` 用于确认一个表达式的值为 `true`。表达式为 `true` 时测试继续执行；表达式为 `false` 时，宏会调用 `panic!`，从而使测试失败。

```rust
fn contains_name(name: &str) -> bool {
    name.contains("Ferris")
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn name_contains_ferris() {
        assert!(contains_name("Hello, Ferris!"));
    }
}
```

**讨论：** `assert!` 适合验证布尔条件，例如集合是否包含某个元素、选项是否为 `Some`。如果需要比较两个具体值，优先使用 `assert_eq!`，因为它能在失败时同时显示左右两边的值。

### 使用 `assert_eq!` 和 `assert_ne!` 宏测试相等

测试功能时经常需要比较被测试代码的结果和期望值。`assert_eq!` 在两个值相等时通过，在不相等时失败；`assert_ne!` 则在两个值不相等时通过，在相等时失败。

```rust
fn add_two(a: i32) -> i32 {
    a + 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn add_two_works() {
        assert_eq!(add_two(2), 4);
    }

    #[test]
    fn add_two_changes_input() {
        let input = 2;
        assert_ne!(add_two(input), input);
    }
}
```

这两个宏底层分别使用 `==` 和 `!=`。断言失败时，宏会以调试格式打印参数，因此被比较的类型需要实现 `PartialEq` 和 `Debug`。基本类型和大多数标准库类型已经实现了这些 trait；自定义类型通常可以派生：

```rust
#[derive(Debug, PartialEq)]
struct User {
    name: String,
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn user_is_equal() {
        let left = User {
            name: String::from("Rust"),
        };
        let right = User {
            name: String::from("Rust"),
        };
        assert_eq!(left, right);
    }
}
```

**讨论：** Rust 中 `assert_eq!` 的两个参数没有 expected 和 actual 的固定顺序，失败信息使用 `left` 和 `right` 表示它们。对于浮点数、包含非确定顺序元素的集合等类型，直接比较相等可能不合适，应根据业务语义比较误差范围或集合内容。

### 自定义失败信息

`assert!`、`assert_eq!` 和 `assert_ne!` 都支持可选的失败信息。必需参数之后的参数会按照 `format!` 的规则格式化：

```rust
fn greeting(name: &str) -> String {
    format!("Hello, {name}!")
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn greeting_contains_name() {
        let name = "Carol";
        let result = greeting(name);
        assert!(
            result.contains(name),
            "问候语没有包含名字 {name}，实际结果是：{result}"
        );
    }
}
```

**讨论：** 自定义信息应该说明断言想验证的业务含义，并尽量包含输入和实际结果。不要在每个简单断言中添加重复信息，否则反而会使测试代码变得嘈杂。

### 使用 `should_panic` 属性来测试 panic

除了检查返回值，也需要检查代码是否按照预期处理错误。假设 `Guess` 类型只允许保存 `1` 到 `100` 之间的值，那么创建越界值时就应该 panic：

```rust
pub struct Guess {
    value: i32,
}

impl Guess {
    pub fn new(value: i32) -> Self {
        if !(1..=100).contains(&value) {
            panic!("Guess value must be between 1 and 100, got {value}");
        }
        Self { value }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic]
    fn greater_than_100_panics() {
        Guess::new(200);
    }
}
```

`#[should_panic]` 位于 `#[test]` 之后。只有测试函数发生 panic 时测试才会通过；如果没有 panic，测试就会失败。

单独使用 `#[should_panic]` 可能不够精确，因为任何原因导致的 panic 都可能使测试通过。可以使用 `expected` 指定 panic 信息中必须包含的文本：

```rust
#[test]
#[should_panic(expected = "between 1 and 100")]
fn greater_than_100_has_the_right_message() {
    Guess::new(200);
}
```

`expected` 不必写出完整信息，只要是 panic 信息中的独特子串即可。如果需要区分多种错误场景，可以为每种场景编写单独的测试，并指定更具体的消息。

**讨论：** `should_panic` 适合验证“不变量被破坏时必须终止”的接口，但它无法表达可恢复错误。对用户输入、文件读写等预计会失败的操作，通常更适合返回 `Result`，而不是使用 panic。使用 `expected` 可以避免错误位置不对却意外通过。

### 在测试中使用 `Result<T, E>`

测试函数也可以返回 `Result<(), E>`。测试成功时返回 `Ok(())`，失败时返回 `Err`，这样可以在测试中使用问号运算符：

```rust
fn read_number() -> Result<i32, String> {
    "42".parse::<i32>().map_err(|error| error.to_string())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn reads_a_number() -> Result<(), String> {
        let number = read_number()?;
        assert_eq!(number, 42);
        Ok(())
    }
}
```

不能对返回 `Result` 的测试使用 `#[should_panic]`。如果要验证某个操作返回 `Err`，不要用 `?` 直接传播它，而应检查 `is_err()`：

```rust
#[test]
fn invalid_number_returns_error() {
    let result = "not a number".parse::<i32>();
    assert!(result.is_err());
}
```

**讨论：** `Result` 测试适合测试会返回错误的代码，`?` 能减少层层匹配带来的样板代码。`assert!`、`assert_eq!` 等断言仍然可以和 `Result` 测试一起使用；只是断言失败会 panic，而操作错误可以通过 `Err` 返回给测试执行器。

## 控制测试运行

`cargo test` 会在测试模式下编译代码，并运行生成的测试程序。默认情况下，测试会并行运行，且测试通过时产生的标准输出会被截获。可以将参数传给 Cargo，也可以将参数传给测试二进制文件。两类参数之间使用 `--` 分隔：

- `cargo test` 前面的参数由 Cargo 处理；
- `--` 后面的参数由测试执行器处理。

可以分别执行 `cargo test --help` 和 `cargo test -- --help` 查看两类参数。

**讨论：** 理解参数分隔符可以避免把测试执行器参数误传给 Cargo。日常开发可先使用名称过滤快速反馈，再在提交前运行完整测试集。

### 并行或者顺序运行测试

Rust 默认使用多个线程并行运行测试，以便更快得到反馈。因此测试不应相互依赖，也不应依赖会被其他测试同时修改的共享状态，例如当前工作目录、环境变量或同一个临时文件。

如果测试必须按顺序运行，或者希望限制线程数量，可以将 `--test-threads` 传给测试执行器：

```text
cargo test -- --test-threads=1
```

例如，多个测试都操作名为 `test-output.txt` 的文件时，一个测试可能在另一个测试读取期间覆盖文件，导致与业务代码无关的失败。更好的方案是为每个测试创建不同的临时文件；只有无法避免共享状态时，才考虑串行运行。

**讨论：** 串行运行可以降低共享资源导致的不稳定，但会使测试变慢，也不能解决测试本身没有清理状态的问题。更可靠的测试应尽量隔离输入、输出和环境，避免依赖测试执行顺序。

### 显示函数输出

测试通过时，Rust 测试库默认捕获 `println!` 输出；测试失败时才会显示相关输出。例如：

```rust
fn print_value(value: i32) -> i32 {
    println!("I got the value {value}");
    10
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn this_test_will_pass() {
        assert_eq!(print_value(4), 10);
    }

    #[test]
    fn this_test_will_fail() {
        assert_eq!(print_value(8), 5);
    }
}
```

如果希望查看通过测试中的输出，可以使用：

```text
cargo test -- --show-output
```

**讨论：** 输出捕获能让正常的测试结果保持简洁，失败时又能保留调试信息。调试完成后，应优先使用断言和结构化错误信息表达测试意图，而不要依赖大量打印内容。

### 通过名称运行部分测试

测试数量较多时，可以按名称过滤测试。将测试名称传给 `cargo test`，只会运行名称中包含该字符串的测试。

假设有如下测试：

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn add_two_and_two() {}

    #[test]
    fn add_three_and_three() {}

    #[test]
    fn one_hundred() {}
}
```

运行单个测试可以使用：

```text
cargo test one_hundred
```

运行名称中包含 `add` 的测试可以使用：

```text
cargo test add
```

测试所在的模块名也是测试全名的一部分，因此也可以通过模块名过滤一组测试。

**讨论：** 名称过滤适合快速定位正在修改的功能。测试名称应体现行为而不是实现细节，例如 `rejects_empty_name` 比 `test_case_3` 更容易搜索和维护。

### 除非指定，否则忽略测试

有些测试非常耗时，或者需要额外环境才能运行。可以使用 `#[ignore]` 标记它们，使普通的 `cargo test` 不运行这些测试：

```rust
#[test]
#[ignore]
fn expensive_test() {
    // 这里只运行耗时的测试逻辑
}
```

只运行被忽略的测试：

```text
cargo test -- --ignored
```

无论测试是否被忽略都运行：

```text
cargo test -- --include-ignored
```

**讨论：** `#[ignore]` 不是删除测试，也不是永久跳过测试。应在测试旁边说明被忽略的原因和运行条件，并在持续集成或发布前专门运行这些测试，避免它们长期失效而无人发现。

## 测试的组织结构

Rust 社区通常将测试分为单元测试（unit tests）和集成测试（integration tests）：

- 单元测试较小、较集中，通常一次测试一个模块，并且可以测试私有接口；
- 集成测试位于库 crate 外部，只能使用公有 API，关注多个模块组合后的行为。

两类测试都很重要：单元测试便于精确定位问题，集成测试则能验证对外提供的完整功能。

**讨论：** 单元测试和集成测试关注的边界不同，不能简单地用一种测试完全替代另一种。合理的测试组织应在执行速度、覆盖范围和维护成本之间取得平衡。

### 单元测试

单元测试通常写在被测试代码所在文件的 `tests` 模块中：

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn public_behavior_works() {
        // 测试当前模块中的函数
    }
}
```

`cfg` 是 configuration（配置）的缩写。`#[cfg(test)]` 告诉 Rust，只有使用测试配置编译时才包含这个模块。Cargo 在执行 `cargo test` 时会启用该配置，因此测试辅助函数也不会进入普通的生产构建。

**讨论：** 单元测试与实现放在一起，修改代码时容易同步维护，且可以访问当前模块的私有项。但测试过度依赖私有实现会增加重构成本，所以应优先通过公有行为验证结果，只有在私有逻辑复杂且值得独立验证时才直接测试私有函数。

#### 私有函数测试

测试代码也是 Rust 模块。子模块可以使用祖先模块中的私有项，因此可以测试未标记为 `pub` 的函数：

```rust
fn internal_adder(left: i32, right: i32) -> i32 {
    left + right
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn internal_adder_works() {
        assert_eq!(internal_adder(2, 2), 4);
    }
}
```

这里的 `use super::*` 将父模块中的项引入 `tests` 模块作用域。是否测试私有函数取决于测试策略：Rust 的可见性规则允许这样做，但并不强迫开发者这样做。

**讨论：** 测试私有函数可以快速覆盖复杂的内部算法，也能帮助定位失败位置；缺点是内部重构可能导致测试频繁修改。对于简单的私有辅助函数，通常通过公有接口间接覆盖即可。

### 集成测试

集成测试对于被测试的库来说是外部代码，使用方式与其他 crate 使用该库的方式相同。因此它们只能调用公有 API，适合验证多个模块能否协同工作。

**讨论：** 单独的模块测试全部通过，并不代表模块组合后一定正确。集成测试的数量不必覆盖所有内部实现，但应覆盖用户最重要的使用流程和边界场景。

#### test 目录

要编写集成测试，需要在项目根目录创建与 `src` 同级的 `tests` 目录：

```text
my_project/
├── Cargo.toml
├── src/
│   └── lib.rs
└── tests/
    └── integration_test.rs
```

`tests` 目录中的每个 `.rs` 文件都会被 Cargo 当作一个独立的测试 crate。假设库名为 `adder`，可以这样编写集成测试：

```rust
use adder::add_two;

#[test]
fn add_two_works() {
    assert_eq!(add_two(2), 4);
}
```

集成测试文件不需要写 `#[cfg(test)]`，因为 Cargo 只会在运行测试时编译 `tests` 目录。执行 `cargo test` 时，通常会依次看到单元测试、集成测试和文档测试的结果。如果前面的测试阶段失败，后续阶段可能不会继续运行。

可以通过测试名称过滤某个集成测试，也可以指定文件名运行该文件中的全部测试：

```text
cargo test add_two_works
cargo test --test integration_test
```

**讨论：** 集成测试只能访问公有 API，这种限制正好模拟真实用户的使用方式。库的名字必须与 `Cargo.toml` 中的包名对应；如果项目只有 `src/main.rs` 而没有 `src/lib.rs`，就不能像库一样从 `tests` 目录导入其中的函数。

#### 集成测试中的子模块

随着集成测试增多，可以按功能拆分文件。需要注意，`tests` 目录中的每个顶层 Rust 文件都会被 Cargo 当作一个独立的测试 crate。如果把通用辅助函数放在 `tests/common.rs` 中，Cargo 可能会把它也当作一个测试文件，并在输出中显示 `running 0 tests`。

可以使用目录模块避免这种情况：

```text
tests/
├── common/
│   └── mod.rs
├── api_test.rs
└── workflow_test.rs
```

在 `tests/common/mod.rs` 中定义辅助函数：

```rust
pub fn setup() {
    // 为多个集成测试准备公共状态
}
```

然后在测试文件中引入：

```rust
mod common;

#[test]
fn user_workflow_works() {
    common::setup();
    // 测试具体功能
}
```

`tests` 下的子目录不会被当作单独的集成测试 crate，因此不会额外出现在测试结果中。

**讨论：** 将公共逻辑放在 `common/mod.rs` 可以避免复制代码，也能保持每个测试文件独立。辅助函数应尽量只负责准备环境，不要把真正的断言隐藏起来，否则测试失败时不容易看出验证了什么。

#### 二进制 crate 的测试

如果项目只有 `src/main.rs` 而没有 `src/lib.rs`，就无法在 `tests` 目录中通过 `use` 导入 `main.rs` 中的函数。二进制 crate 的目标是生成并运行可执行文件，它不会像库 crate 一样向其他 crate 暴露可调用的 API。

常见的组织方式是将核心逻辑放入 `src/lib.rs`，让 `src/main.rs` 只负责解析参数和调用库：

```rust
// src/lib.rs
pub fn run() -> Result<(), String> {
    // 应用程序的主要逻辑
    Ok(())
}
```

```rust
// src/main.rs
fn main() {
    if let Err(error) = important_project::run() {
        eprintln!("{error}");
        std::process::exit(1);
    }
}
```

这样，集成测试就可以测试 `lib.rs` 中的核心功能，而 `main.rs` 中只剩下少量的启动代码。

**讨论：** 把业务逻辑从二进制入口抽到库中，可以提高可测试性、复用性和代码边界的清晰度。入口函数仍然需要少量测试或手动验证，但大部分行为应由库的单元测试和集成测试覆盖。
