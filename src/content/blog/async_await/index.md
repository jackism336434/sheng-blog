---
title: 'async await in Rust'
publishDate: 'September 23, 2026'
updatedDate: 'September 23, 2026'
description: 'Courage doesn’t mean the absence of fear, it means moving forward even when your heart is trembling.'
tags:
  - Rust
  - Programming language
  - Async 
language: 'Chinese'
heroImage: { src: 'courage.png', color: '#559ce8' }
---

很多我们要求计算机执行的操作都需要一段时间才能完成。如果在等待这些长时间运行的过程结束时，我们还能做点别的事情，就再好不过了。现代计算机提供了两种同时处理多个操作的技术：并行和并发。然而，我们的程序逻辑通常是以近乎线性的方式编写的。我们希望能够描述程序应执行的操作，以及函数在哪些点可以暂停并让程序的其他部分转而运行，而不需要预先精确指定每段代码究竟该以什么顺序、用什么方式运行。异步编程 正是一种抽象，它让我们能够用“可能暂停的点”和“最终结果”来表达代码，而协调执行的细节则由它来替我们处理。

本章以前一章通过线程实现并行和并发为基础，引入另一种编写代码的方式：Rust 的 futures、streams，以及 async 和 await 语法。它们让我们能够表达“操作可以是异步的”这一点；此外，还有由第三方 crate 提供的异步运行时，用来管理和协调这些异步操作的执行。

来看一个例子。假设你正在导出一个家庭聚会的视频，这个操作可能要花上几分钟甚至几小时。视频导出会尽可能多地占用 CPU 和 GPU 资源。如果你只有一个 CPU 核，而操作系统又不会在导出完成前暂停这个任务，也就是说它会以同步方式执行导出，那么在这个任务运行期间你就什么别的都做不了。这会是相当糟糕的体验。幸运的是，你的操作系统可以，而且确实会，足够频繁地打断导出任务，让你同时完成其他工作。

再假设你正在下载别人分享给你的视频。这个操作也可能耗时很长，但不会占用太多 CPU 时间。此时 CPU 主要是在等待网络数据到达。虽然数据一开始到达时你就可以开始读取，但全部数据到齐仍然可能需要一段时间。即便数据已经全部到达，如果视频文件很大，完整载入它也至少可能需要一两秒。听起来不算长，但对于每秒能执行数十亿次操作的现代处理器来说，这已经是很长的时间了。和前面的例子一样，操作系统也会在等待网络调用完成时悄悄打断程序，让 CPU 去处理其他工作。

视频导出属于 CPU 密集型（CPU-bound）或 计算密集型（compute-bound）操作：它受限于 CPU 或 GPU 处理数据的速度，以及操作能分配到多少计算能力。视频下载则属于 I/O-bound 操作，因为它受限于计算机的 输入输出（input and output）速度；它的速度最多只能和网络传输数据的速度一样快。

在上述两个例子中，操作系统的隐式中断提供了一种形式的并发。不过这种并发仅限于整个程序的级别：操作系统中断一个程序并让其它程序得以执行。在很多场景中，由于我们能比操作系统在更细粒度上理解我们的程序，因此我们可以观察到很多操作系统无法察觉的并发机会。

例如，如果我们正在构建一个管理文件下载的工具，程序就应该能做到：启动一个下载任务不会让 UI 卡死，而且用户还能够同时启动多个下载任务。不过，许多操作系统中与网络交互的 API 都是 blocking 的；也就是说，在它们所处理的数据完全就绪之前，会阻止程序继续向前执行。

我们可以为每个文件单独创建一个线程来避免阻塞主线程。然而，这些线程所消耗的系统资源最终会成为问题。更理想的情况是：这些调用从一开始就不是阻塞的，并且我们只需定义程序想完成的一组任务，然后让运行时自行选择最佳的执行顺序和方式。

这正是 Rust 的 async（asynchronous 的缩写）抽象所提供的能力。本章会介绍以下内容：

- 如何使用 Rust 的 async 和 await 语法，并借助运行时执行异步函数
- 如何用异步模型解决一些我们已经遇到过的挑战
- 多线程和异步如何提供互补的解决方案，以及它们在许多场景下如何组合使用

## 并行与并发

我们大致将并行和并发视为可以互换的概念。但现在我们需要更加精确地区分它们，因为它们的区别将在实际工作中显现出来。

思考一下不同的团队分割方法来开发一个软件项目。我们可以分配给一个个人多个任务，也可以每个团队成员各自负责一个任务，或者可以采用这两种方法的组合。

当多个团队成员在同一时刻分别推进不同任务时，这就是**并行**。当一个人在任何一个任务都还没完成之前，就在多个不同任务之间切换工作，这就是 并发。一种实现并发的方式，很像你在电脑上同时 checkout 了两个不同项目；当你对其中一个项目感到厌倦，或是在上面卡住时，就切到另一个。你只有一个人，所以不可能在完全相同的时刻同时推进两个任务，但你可以通过在它们之间切换来多任务处理，一次推进一个任务。

在这两种工作流中，你都可能需要在不同任务之间做协调。也许你原以为分配给某个人的任务和其他人的工作完全独立，但实际上它必须等另一个人先完成自己的任务。有些工作可以并行完成，但其中一些实际上是 串行 的：它们只能按顺序发生，一个接一个。

同样的基础动态也作用于软件与硬件。在一个单核的机器上，CPU 一次只能执行一个操作，不过它仍然可以并发工作。借助像线程、进程和异步（async）等工具，计算机可以暂停一个活动，并在最终切换回第一个活动之前切换到其它活动。在一个有多个 CPU 核心的机器上，它也可以并行工作。一个核心可以做一件工作的同时另一个核心可以做一些完全不相关的工作，而且这些工作实际上是同时发生的。

在 Rust 中运行 async 代码时，通常是在并发地执行。至于这种并发在底层是否也会利用并行，则取决于硬件、操作系统，以及所使用的异步运行时（稍后我们会进一步介绍异步运行时）。

**讨论：**并发强调多个任务的执行时间段重叠，并行强调同一时刻实际执行多个任务。单线程 async 可以实现并发，却不能让两个 CPU 密集型计算同时执行；多线程运行时则可能让不同任务并行执行。异步主要节省等待 I/O 时占用线程的成本，不会自动加速计算。

## Future与async语法

Rust 异步编程的关键元素是 futures 和 Rust 的 async 与 await 关键字。

future 是一个现在也许还没准备好，但会在将来某个时刻准备好的值。（这个概念在很多语言里都存在，只是有时会用 task 或 promise 之类的名字。）Rust 提供了 Future trait 作为基础构件，让不同的异步操作可以用不同的数据结构来实现，同时又拥有统一的接口。在 Rust 中，future 就是那些实现了 Future trait 的类型。每个 future 都保存了自身的进度信息，以及“就绪”到底意味着什么。

async 关键字可以用于代码块和函数，表示它们可以被中断和恢复。在 async 块或 async 函数中，你可以使用 await 关键字来 await 一个 future，也就是等待它变为就绪。在 async 块或函数里，每个等待 future 的位置，都是这个块或函数可能暂停并随后恢复的点。检查 future、看看它的值是否已经可用，这个过程称为 polling（轮询）。

其他一些语言，例如 C# 和 JavaScript，也用 async 和 await 关键字进行异步编程。如果你熟悉这些语言，可能会注意到 Rust 在语法处理上存在一些明显差异。我们会看到，这样设计是有充分理由的。

编写异步 Rust 时，大多数时候我们直接使用 async 和 await 关键字。Rust 会把它们编译成等价的、基于 Future trait 的代码，就像它把 for 循环编译成基于 Iterator trait 的等价代码一样。不过，既然 Rust 提供了 Future trait，你在需要时也可以为自己的数据类型实现它。本章中我们会见到很多函数，它们都返回拥有各自 Future 实现的类型。我们会在本章结尾回到这个 trait 的定义，进一步深入理解它的工作原理；不过眼下这些细节已经足够让我们继续前进。

这些内容可能仍然有些抽象，所以我们来写第一个异步程序：一个小型网页抓取器。我们会从命令行传入两个 URL，并发地抓取它们，然后返回那个最先完成的结果。这个例子会带来不少新语法，不过不用担心，我们会一路把需要知道的内容都解释清楚。

### 第一个异步程序

为了让本章专注于学习 async，而不是在生态系统的各种组件之间来回切换，我们准备了一个 trpl crate（trpl 是 “The Rust Programming Language” 的缩写）。它重新导出了本章需要的所有类型、trait 和函数，主要来自 futures 和 tokio crate。futures crate 是 Rust 异步代码实验的官方阵地，Future trait 最初就是在那里设计出来的。Tokio 则是目前 Rust 中使用最广泛的异步运行时（async runtime），尤其常见于 Web 应用。生态中也还有其他很优秀的运行时，而且它们可能更适合你的实际用途。我们在 trpl 的底层使用 tokio，是因为它经过了充分测试，也足够常用。

本笔记沿用提供 `block_on`、`select` 等接口的 `trpl` 版本。先执行 `cargo add trpl` 添加依赖，项目使用 Rust 2021 或 2024 edition。不同版本的教学 crate 可能调整 API 名称，应以所选版本的文档为准。后文代码块是各自独立的示例；若复用前面的函数，会显式说明。

**讨论：**`async` / `.await` 是语言语法，`Future` 在标准库中，而网络请求、定时器、任务调度来自库。不要把这些层次混为一谈，也不要假定不同运行时的 I/O 类型可以直接混用。

#### 定义page——title函数

让我们开始编写一个函数，它获取一个网页 URL 作为参数，请求该 URL 并返回标题元素的文本（见示例 17-1）。

```rust
use trpl::Html;

async fn page_title(url: &str) -> Option<String> {
    let response = trpl::get(url).await;
    let response_text = response.text().await;
    Html::parse(&response_text)
        .select_first("title")
        .map(|title| title.inner_html())
}

```

我们定义了一个名为 page_title 的函数，并用 async 关键字标记它。然后使用 trpl::get 函数抓取传入的 URL，再用 await 关键字等待响应。为了得到 response 的文本，我们调用它的 text 方法，并再次使用 await 进行等待。这两个步骤都是异步的。对于 get 函数来说，我们必须等待服务器先把响应的第一部分发回来，其中包括 HTTP headers、cookies 等，这些内容可以和响应体分开发送。尤其当响应体很大时，全部数据到达可能要花上一些时间。由于我们必须等待响应完整到达，text 方法自然也是 async 的。

调用 `async fn` 只会创建 future，不会立即执行函数体；它必须被 `poll` 才能推进。`.await` 是组合并推进 future 的常用方式，但不是唯一方式：把 future 交给 `spawn_task` 后，运行时也会轮询它，不必先等待任务句柄。普通函数即使返回 future，也可能在返回前执行同步工作；外部 I/O 或已启动的任务也可能独立推进，因此不能把惰性理解成“所有 future 对应的工作都只在 await 后发生”。

这和迭代器的惰性类似：`map` 本身通常只构造适配器，真正驱动迭代的是 `next`、`for`、`collect` 等消费操作。直接忽略一个 future 通常会触发 `unused_must_use` 警告，而不会帮你执行其中的异步工作。

有了 response_text 之后，我们就可以用 Html::parse 把它解析成 Html 类型的实例。这样一来，我们得到的就不再是原始字符串，而是一个可以把 HTML 当作更丰富数据结构来操作的类型。特别是，我们可以用 select_first 方法找到给定 CSS selector 的第一个匹配项。传入字符串 "title" 后，我们就能拿到文档中的第一个 <title> 元素，如果它存在的话。因为也可能根本没有匹配项，所以 select_first 返回的是 Option<ElementRef>。最后，我们使用 Option::map 方法：如果 Option 中有值，它就会对其中的值进行处理；如果没有，就什么都不做。（这里当然也可以使用 match 表达式，不过 map 更符合惯用写法。）在我们传给 map 的闭包里，会对 title 调用 inner_html 来获取其中的内容，它是一个 String。到这里，我们最终得到的就是一个 Option<String>。

注意，Rust 的 await 关键字放在要等待的表达式后面，而不是前面。也就是说，它是一个 postfix keyword（后缀关键字）。如果你在其他语言里用过 async，这一点可能和你的习惯不同；但在 Rust 中，这种设计会让链式方法调用更易读。因此，我们可以把 page_title 的函数体改写成在 trpl::get 和 text 调用之间插入 await 的链式写法，如示例 17-2 所示：

```rust
use trpl::Html;

async fn page_title(url: &str) -> Option<String> {
    let text = trpl::get(url).await.text().await;
    Html::parse(&text)
        .select_first("title")
        .map(|title| title.inner_html())
}
```

**讨论：**这里两次 `.await` 是有依赖的顺序操作：先取得响应，再读取响应体。`Option<String>` 仅表示是否找到了标题，并没有完整表达网络错误；教学库简化了错误处理，生产代码通常应返回 `Result<Option<String>, E>`。另外，`inner_html()` 返回元素内部的 HTML，并不保证是去掉标签后的纯文本。

当 Rust 遇到一个 async 关键字标记的代码块时，会将其编译为一个实现了 Future trait 的唯一的、匿名的数据类型。当 Rust 遇到一个被标记为 async 的函数时，会将其编译成一个函数体是异步代码块的非异步函数。异步函数的返回值类型是编译器为异步代码块所创建的匿名数据类型。

#### 使用运行时执行异步函数

首先，我们只获取单个页面的标题，如示例 17-3 所示。不幸的是，这段代码还不能编译。

下面的错误示例复用前面的 `page_title`：

```rust
fn main() {
    let title = page_title("https://www.rust-lang.org").await;
    println!("{title:?}");
}
```

这里会出现 **E0728**：`await is only allowed inside async functions and blocks`。如果直接把入口改成 `async fn main()`，则会出现 **E0752**：`main function is not allowed to be async`。

我们使用标准库的 `std::env` 模块来获取命令行参数，而不是额外的 env crate。然后把 URL 参数传给 page_title，再等待它的结果。由于 future 产出的值是 Option<String>，我们使用 match 表达式来根据页面是否含有 <title> 打印不同的信息。

唯一能使用 await 关键字的地方，是 async 函数或 async 代码块中，而 Rust 又不允许我们把特殊的 main 函数标记为 async。

main 不能标记为 async 的原因是异步代码需要一个 运行时：即一个管理执行异步代码细节的 Rust crate。一个程序的 main 函数可以 初始化 一个运行时，但是其 自身 并不是一个运行时。（稍后我们会进一步解释原因。）每一个执行异步代码的 Rust 程序必须至少有一个设置运行时并执行 futures 的地方。

大多数支持 async 的语言都会自带运行时，但 Rust 不会。相反，Rust 有很多不同的异步运行时可供选择，每一种都针对自己的目标用例做了不同权衡。比如，一个拥有许多 CPU 核心和大量 RAM 的高吞吐 Web 服务器，和一个单核、RAM 很小、甚至不能进行堆分配的微控制器，需求就截然不同。提供这些运行时的 crate 往往也会一并提供文件或网络 I/O 等常见功能的异步版本。

在这里，我们会使用 trpl crate 提供的 block_on 函数。它接受一个 future 作为参数，并阻塞当前线程，直到这个 future 运行完成为止。在内部，调用 block_on 会借助 tokio crate 设置一个运行时，用来执行传入的 future（trpl 的 block_on 和其他运行时 crate 提供的同名函数行为类似）。一旦 future 完成，block_on 就会返回 future 产生的值。

我们当然可以把 page_title 返回的 future 直接传给 block_on，并在它完成后对得到的 Option<String> 进行匹配，就像我们在示例 17-3 中本来打算做的那样。不过，本章的大部分例子里（以及现实中的大多数 async 代码里），我们都不止会进行一次异步函数调用，因此我们改为传入一个 async 块，并在其中显式等待 page_title 的结果，如示例 17-4 所示。

下面是示例 17-4 的完整代码；运行时通过 `cargo run -- https://www.rust-lang.org` 传入 URL：

```rust
use std::env;
use trpl::Html;

async fn page_title(url: &str) -> Option<String> {
    let text = trpl::get(url).await.text().await;
    Html::parse(&text)
        .select_first("title")
        .map(|title| title.inner_html())
}

fn main() {
    let Some(url) = env::args().nth(1) else {
        eprintln!("用法：cargo run -- <URL>");
        return;
    };

    trpl::block_on(async {
        match page_title(&url).await {
            Some(title) => println!("{url} 的标题：{title}"),
            None => println!("{url} 没有标题"),
        }
    });
}
```

每一个 await point 都是**可能**暂停的位置，而不是必然切换任务的位置。如果子 future 返回 `Poll::Ready`，代码可以立即继续；如果返回 `Poll::Pending`，外层 future 通常也会返回 `Pending`，将控制权交还执行器。编译器会保存暂停后仍需使用的局部变量和子 future，构造类似下面的状态机：

```rust
// 仅示意阶段；真实状态还需要保存 URL、子 future 等数据。
enum PageTitleState {
    Start,
    WaitingForResponse,
    WaitingForBody,
    Done,
}
```

**讨论：**`block_on` 在同步入口阻塞当前线程，直到顶层 future 完成；内部异步 I/O 的等待不等于用同步调用阻塞执行线程。`#[tokio::main]` 看起来允许异步入口，实际上由宏生成同步 `main` 并设置运行时。不要在异步任务里随意嵌套 `block_on`，这可能导致 panic 或死锁。

编写代码来手动控制不同状态之间的转换是非常乏味且容易出错的，特别是之后增加了更多功能和状态的时候。相反，Rust 编译器自动创建并管理异步代码的状态机数据结构。如果你感兴趣的话：是的，正常的借用和所有权也全部适用于这些数据结构。幸运的是，编译器也会为我们处理这些检查，并提供友好的错误信息。

最终，总得有某个组件来执行这个状态机，而那个组件就是运行时。（这也是为什么在了解运行时时，你可能会看到 executor 这个词：executor 是运行时中负责执行异步代码的那一部分。）

现在你就能理解，为什么编译器会在示例 17-3 中阻止我们把 main 本身写成异步函数了。如果 main 是 async 函数，那么就必须有别的东西来管理 main 返回的 future 对应的状态机；可 main 本身就是程序的入口点！因此，我们改为在 main 中调用 trpl::block_on，让它设置好运行时，并运行 async 块返回的 future，直到执行完成。

#### 让两个url并发竞争

在示例 17-5 中，我们会对从命令行传入的两个不同 URL 分别调用 page_title，并选出最先完成的那个 future。

我们首先分别对用户提供的两个 URL 调用 page_title。随后把得到的 future 保存到 title_fut_1 和 title_fut_2 中。记住，它们此时还什么都没做，因为 future 是惰性的，而我们也还没有等待它们。接着把这些 future 传给 `trpl::select`，并等待它返回的 future，得到一个表明哪个输入先完成的值。

任意一个 future 都有可能“获胜”，因此这里返回 Result 并不合理。相反，trpl::select 返回的是一个我们之前还没见过的类型：trpl::Either。Either 在某种程度上有点像 Result，也有两个分支；但不同的是，它并没有内建“成功”或“失败”的语义，而是用 Left 和 Right 来表示“这个或那个”。

```rust
enum Either<A, B> {
    Left(A),
    Right(B),
}
```

如果第一个参数先完成，select 就返回 Left，其中包含该 future 的输出；如果第二个 future 先完成，则返回 Right，其中包含第二个 future 的输出。这正好对应函数调用时参数的顺序：第一个参数位于第二个参数的左边。

我们还更新了 page_title，让它把传入的 URL 一并返回。这样一来，即使最先返回的页面无法解析出 <title>，我们仍然可以打印出一条有意义的信息。有了这些数据之后，我们最后再调整 println! 的输出，让它既能显示哪个 URL 最先完成，也能在页面存在 <title> 时打印出标题内容。

示例 17-5 的完整实现如下，运行时传入两个 URL：

```rust
use std::env;
use trpl::{Either, Html};

async fn page_title(url: &str) -> (&str, Option<String>) {
    let text = trpl::get(url).await.text().await;
    let title = Html::parse(&text)
        .select_first("title")
        .map(|title| title.inner_html());
    (url, title)
}

fn main() {
    let urls: Vec<String> = env::args().skip(1).collect();
    if urls.len() != 2 {
        eprintln!("用法：cargo run -- <URL1> <URL2>");
        return;
    }

    trpl::block_on(async {
        let title_fut_1 = page_title(&urls[0]);
        let title_fut_2 = page_title(&urls[1]);
        let (url, title) = match trpl::select(title_fut_1, title_fut_2).await {
            Either::Left(result) => result,
            Either::Right(result) => result,
        };
        println!("先完成：{url}，标题：{title:?}");
    });
}
```

**讨论：**“先完成”是指被组合器首先轮询到 `Ready`，不一定严格对应网络响应到达的物理时间。这里两个 future 被直接交给 `trpl::select`；获胜后另一个被丢弃，不会继续作为独立任务运行。丢弃 future 不会回滚已经发送的请求或远端副作用。不要把本例的 API 与 `futures::future::select` 混淆：后者会返回未完成的另一个 future。

## 并发与async

在这一部分，我们将使用异步来应对一些与第十六章中通过线程解决的相同的并发问题。因为之前我们已经讨论了很多关键理念了，这一部分我们会专注于线程与 future 的区别。

在很多情况下，使用异步处理并发的 API 与使用线程的非常相似。在其它的一些情况，它们则非常不同。即便线程与异步的 API 看起来 很类似，通常它们有着不同的行为，同时它们几乎总是有着不同的性能特点。

### 使用spawn——task创建异步任务

第十六章中我们应付的第一个任务是在两个不同的线程中计数。让我们用异步来完成相同的任务。trpl crate 提供了一个 spawn_task 函数，它看起来非常像 thread::spawn API，和一个 sleep 函数，这是 thread::sleep API 的异步版本。我们可以将它们结合使用，实现与线程示例相同的计数功能，如示例 17-6 所示。

```rust
use std::time::Duration;

fn main() {
    trpl::block_on(async {
        trpl::spawn_task(async {
            for i in 1..10 {
                println!("hi number {i} from the first task");
                trpl::sleep(Duration::from_millis(500)).await;
            }
        });
        for i in 1..5 {
            println!("hi number {i} from the second task");
            trpl::sleep(Duration::from_millis(500)).await;
        }
    });
}
```

这个版本会在主 async 块中的 for 循环一结束就停止，因为当 main 函数结束时，由 spawn_task 生成的任务也会被关闭。如果你想让它一直运行到任务自身完成，就需要使用 join handle 来等待第一个任务结束。在线程的版本中，我们使用 join 方法“阻塞”等待线程运行结束。在示例 17-7 中，我们可以使用 await 做同样的事，因为任务句柄本身就是一个 future。它的 Output 类型是 Result，所以在等待之后还要再 unwrap 一次。

示例 17-7 保存并等待任务句柄：

```rust
use std::time::Duration;

fn main() {
    trpl::block_on(async {
        let handle = trpl::spawn_task(async {
            for i in 1..10 {
                println!("任务一：{i}");
                trpl::sleep(Duration::from_millis(500)).await;
            }
        });
        for i in 1..5 {
            println!("任务二：{i}");
            trpl::sleep(Duration::from_millis(500)).await;
        }
        handle.await.expect("任务被取消或发生 panic");
    });
}
```

**讨论：**`spawn_task` 提交的是独立调度的任务；创建普通 async 块则只是创建 future。这里丢弃句柄并不意味着立即取消任务，但运行时关闭时未完成的任务不能保证继续执行。等待句柄能观察任务失败；实际项目不一定适合直接 `unwrap`。

到目前为止，看起来 async 和线程只是用不同语法实现了相似效果：在 join handle 上使用 await，而不是调用 join；同时对 sleep 调用也使用 await。

更大的不同在于，我们根本不需要再创建另一个操作系统线程来做这件事。实际上，这里甚至连任务都不一定要创建。因为 async 代码块会被编译成匿名 future，我们可以把每个循环都放进一个 async 代码块里，然后让运行时使用 trpl::join 让它们都执行到完成。

在第十六章“等待所有线程完成”一节中，我们展示了如何对 std::thread::spawn 返回的 JoinHandle 调用 join 方法。trpl::join 与之类似，不过它面向的是 future。当你把两个 future 传给它时，它会生成一个新的 future；等到两个传入的 future 都完成时，这个新 future 的输出就是一个包含它们各自输出值的元组。因此，在示例 17-8 中，我们用 trpl::join 来等待 fut1 和 fut2 完成。我们不会分别等待 fut1 和 fut2，而是等待 trpl::join 生成的那个新 future。这里我们忽略它的输出，因为那不过是一个包含两个 unit 值的元组。

示例 17-8 不创建额外任务，直接组合两个 future：

```rust
use std::time::Duration;

fn main() {
    trpl::block_on(async {
        let fut1 = async {
            for i in 1..4 {
                println!("future 一：{i}");
                trpl::sleep(Duration::from_millis(500)).await;
            }
            "一完成"
        };
        let fut2 = async {
            for i in 1..3 {
                println!("future 二：{i}");
                trpl::sleep(Duration::from_millis(500)).await;
            }
            "二完成"
        };
        let (first, second) = trpl::join(fut1, fut2).await;
        println!("{first}，{second}");
    });
}
```

**讨论：**`join` 在同一个任务中轮询多个子 future，不会自动创建线程，也不能让其中两段同步计算并行。即使某个实现强调公平轮询，也不代表输出严格交替或每次顺序完全相同；就绪时间、唤醒和具体实现都会影响顺序。返回元组的顺序对应参数顺序，而不是完成顺序。若写成 `fut1.await; fut2.await;`，这两个尚未启动的普通 future 就会顺序执行。

### 通过消息传递在两个任务之间发送数据

在 future 之间共享数据的方式也会让你感到熟悉：我们再次使用消息传递，只不过这次使用的是异步版本的类型和函数。为了展示基于线程的并发和基于 future 的并发之间的一些关键差别，我们会和第十六章“通过消息传递在线程间传送数据”一节稍微走一条不一样的路线。在示例 17-9 中，我们先只使用一个 async 代码块，而不像之前那样显式地创建一个独立任务。

这里我们使用了 trpl::channel，一个第十六章用于线程的多生产者、单消费者信道 API 的异步版本。异步版本的 API 与基于线程的版本只有一点微小的区别：它使用一个可变的而不是不可变的 rx，并且它的 recv 方法产生一个需要 await 的 future 而不是直接返回值。现在我们可以发送端向接收端发送消息了。注意我们无需产生一个独立的线程或者任务；只需等待（await） rx.recv 调用。

`std::sync::mpsc::channel` 中的同步 `Receiver::recv` 会阻塞当前线程，直到收到消息或信道断开。异步 `recv` 在没有消息且信道仍开放时返回 `Pending`，使任务有机会暂停。`trpl::channel` 是**无界信道**，`send` 不需要等待缓冲区容量，因此不需要 `.await`；这与发送端可以克隆多少个无关。

示例 17-9：

```rust
fn main() {
    trpl::block_on(async {
        let (tx, mut rx) = trpl::channel();
        tx.send(String::from("你好")).unwrap();
        drop(tx);
        assert_eq!(rx.recv().await, Some(String::from("你好")));
        assert_eq!(rx.recv().await, None);
    });
}
```

**讨论：**无界信道不提供容量背压，生产速度长期大于消费速度时可能耗尽内存。有界异步信道通常需要 `send(value).await` 等待容量，能约束积压；但也应避免生产者和消费者彼此等待形成死锁。

除了发送消息之外，我们还需要接收它们。在这个例子中我们可以手动接收，就是调用四次 rx.recv().await，因为我们知道进来了多少条消息。然而，在现实世界中，我们通常会等待 未知 数量的消息。这时我们需要一直等待直到可以确认没有更多消息了为止。

rx.recv 调用产生一个 Future，我们会 await 它。运行时会暂停 Future 直到它就绪。一旦消息到达，future 会解析为 Some(message)，每次消息到达时都会如此。当所有发送端都被丢弃（或接收端主动关闭信道）且缓冲区中的消息已读完时，`recv().await` 才会返回 `None`，表明没有更多值。关闭并不意味着丢弃已经入队的消息。

while let 循环将上述逻辑整合在一起。如果 rx.recv().await 调用的结果是 Some(message)，我们会得到消息并可以在循环体中使用它，就像使用 if let 一样。如果结果是 None，则循环停止。每次循环执行完毕，它会再次触发 await point，如此运行时会再次暂停直到另一条消息到达。

示例 17-10 故意把发送和接收写在一个 async 块中，并让发送端存活到接收循环之后：

```rust
use std::time::Duration;

fn main() {
    trpl::block_on(async {
        let (tx, mut rx) = trpl::channel();
        for value in ["hi", "from", "the", "world"] {
            tx.send(value).unwrap();
            trpl::sleep(Duration::from_millis(500)).await;
        }
        while let Some(value) = rx.recv().await {
            println!("收到：{value}");
        }
        drop(tx); // 逻辑陷阱：要先结束循环才会执行到这里。
    });
}
```

现在代码可以成功发送和接收所有的消息了。不幸的是，这里还有一些问题。首先，消息并不是按照半秒的间隔到达的。它们在程序启动后两秒（2000 毫秒）后立刻一起到达。其次，程序永远也不会退出！相反它会永远等待新消息。你会需要使用 ctrl-c 来关闭它。

#### 一个async代码块中的代码会线性执行

先来看为什么这些消息会在完整延迟之后一起到达，而不是在每次延迟之后逐条到达。在一个给定的 async 代码块里，代码中 await 出现的顺序，也就是程序运行时它们执行的顺序。

示例 17-10 中只有一个 async 代码块，所以里面的一切都按线性顺序执行。这里依然没有并发。所有 tx.send 调用，连同 trpl::sleep 调用及其相应的 await 点，都会先全部依次发生。只有在那之后，while let 循环才有机会开始执行 recv 调用上的那些 await 点。

为了得到我们真正想要的行为，也就是在每条消息之间都出现休眠间隔，我们需要把 tx 和 rx 的操作分别放进各自的 async 代码块中，如示例 17-11 所示。这样运行时就可以像示例 17-8 那样，使用 trpl::join 分别执行它们。我们再次等待的是 trpl::join 调用的结果，而不是分别等待每个 future。要是依次等待它们，我们就又回到了顺序执行的流程，这正是我们不想要的。

示例 17-11 先只修复并发问题，刻意保留发送端的生命周期问题：

```rust
use std::time::Duration;

fn main() {
    trpl::block_on(async {
        let (tx, mut rx) = trpl::channel();
        let tx_fut = async {
            for value in ["hi", "from", "the", "world"] {
                tx.send(value).unwrap();
                trpl::sleep(Duration::from_millis(500)).await;
            }
        };
        let rx_fut = async {
            while let Some(value) = rx.recv().await {
                println!("收到：{value}");
            }
        };
        trpl::join(tx_fut, rx_fut).await;
        drop(tx); // join 等待接收结束，接收却等待 tx 被丢弃。
    });
}
```

**讨论：**单个 async 块中的语句仍然遵循正常控制流。`.await` 允许其他被调度的工作推进，不会自动把当前代码块后面的语句提前执行。此例通过 `join` 得到了并发，但并发正确不等于退出条件正确。

#### 将所有权移入async代码块

但是程序仍然永远也不会退出，这是由于 while let 循环与 trpl::join 的交互方式所致：

- trpl::join 返回的 future 只会完成一次，即传递的 两个 future 都完成的时候。
- tx_fut future 会在发送完 vals 中最后一条消息后，再完成最后一次休眠之后结束。
- rx_fut future 则要等到 while let 循环结束时才会结束。
- 只有当等待 rx.recv 的结果变成 None 时，while let 循环才会结束。
- 只有在信道关闭且缓冲区已读空后，等待 rx.recv 才会返回 None。
- 接收端主动关闭信道，或所有发送端（包括克隆）都被 drop，才会关闭信道。
- 我们根本没有调用 rx.close，而 tx 也要等到传给 trpl::block_on 的最外层 async 代码块结束后才会被 drop。
- 但那个最外层 async 代码块又必须等 trpl::join 完成才能结束，于是我们就又回到了这个列表的起点。

> 注意：`.await` 不会启动线程。执行器轮询顶层 future，顶层 future 再通过 `.await` 或组合器轮询子 future，形成嵌套的轮询链。


目前，发送消息的那个 async 代码块只是借用了 tx，因为发送消息并不需要取得它的所有权。但如果我们能把 tx move 进那个 async 代码块里，那么一旦该代码块结束，tx 就会被 drop。在线程场景下我们也经常需要把数据 move 进闭包。相同的基本原理也适用于 async 代码块，因此 move 关键字同样可以和 async 代码块一起使用。

```rust
use std::time::Duration;
fn main() {
    trpl::block_on(async {
        let (tx, mut rx) = trpl::channel();
        let tx_fut = async move {
            let vals = vec![
                String::from("hi"),
                String::from("from"),
                String::from("the"),
                String::from("world"),
            ];
            for val in vals {
                tx.send(val).unwrap();
                trpl::sleep(Duration::from_millis(500)).await;
            }
        };
        let rx_fut = async {
            while let Some(value) = rx.recv().await {
                println!("received {value}");
            }
        };
        trpl::join(tx_fut, rx_fut).await;
    });
}


```
**讨论：**`async move` 把捕获值移入 future，而不是延长被引用数据的生命周期。如果捕获的是 `&T`，移动的仍然只是引用。本例移动的是拥有所有权的发送端，发送 future 完成后它会被丢弃；若外面还留有一个发送端克隆，接收循环依然不会结束。

运行这个版本的代码后，它就会在最后一条消息发送并接收完之后正常退出。接下来，我们来看看，如果要从多个 future 发送数据，又需要做哪些变化。

#### 使用Join！宏合并多个future

这个异步信道同样也是多生产者信道，因此如果我们希望从多个 future 发送消息，就可以对 tx 调用 clone，如示例 17-13 所示。

首先，我们克隆 tx，在第一个 async 代码块外创建出 tx1。然后像之前处理 tx 那样，把 tx1 move 进这个代码块里。随后，我们再把原始的 tx move 进一个新的 async 代码块，在那里以稍慢一点的节奏继续发送更多消息。这里我们把这个新 async 代码块放在接收消息的 async 代码块后面，不过放在前面也同样可以。关键在于 future 被等待的顺序，而不是它们被创建的顺序。

两个负责发送消息的 async 代码块都必须写成 async move，这样当代码块结束时，tx 和 tx1 都会被 drop。否则，我们又会回到一开始那个无限循环的问题。

最后，我们从 trpl::join 切换为 trpl::join! 来处理新增的 future。join! 宏可以在 future 数量已知于编译期的情况下，等待任意数量的 future。本章稍后我们还会讨论，如何等待一个数量事先未知的 future 集合。


```rust
use std::time::Duration;

fn main() {
    trpl::block_on(async {
        let (tx, mut rx) = trpl::channel();
        let tx1 = tx.clone();
        let producer1 = async move {
            for value in ["A1", "A2"] {
                tx1.send(value).unwrap();
                trpl::sleep(Duration::from_millis(100)).await;
            }
        };
        let producer2 = async move {
            for value in ["B1", "B2"] {
                tx.send(value).unwrap();
                trpl::sleep(Duration::from_millis(200)).await;
            }
        };
        let consumer = async move {
            while let Some(value) = rx.recv().await {
                println!("{value}");
            }
        };
        trpl::join!(producer1, producer2, consumer);
    });
}
```

**讨论：**`trpl::join!` 宏内部已经执行等待，不要照着 `trpl::join(...).await` 再添加 `.await`。同一发送者依次发送的消息保持发送顺序，不同发送者之间的交错顺序则不能依赖。`join!` 等待所有分支完成，即使某个分支产出 `Err`，也不表示自动取消其他分支。

我们已经探索了如何用消息传递在 future 之间发送数据、一个 async 代码块中的代码如何按顺序执行、如何将所有权 move 进 async 代码块，以及如何合并多个 future。接下来，我们来讨论一下，为什么以及如何告诉运行时：它现在可以切换去执行别的任务了。

### 将控制权交还运行时

在每个 await 点，如果被等待的 future 还没准备好，Rust 就会给运行时一个机会来暂停当前任务并切换到其他任务。反过来也成立：Rust 只会 在 await 点暂停 async 代码块，并把控制权交还给运行时。await 点之间的所有内容都是同步执行的。

这意味着，如果你在一个 async 代码块中做了大量工作，却没有让出执行权，那么同一个任务中的其他子 future 无法推进，单线程运行时中的其他任务也可能饥饿。多线程运行时的其他工作线程可能仍能推进任务，但当前工作线程仍会被占住。有时你会听到人们把这称为一个 future 让其他 future starve（饥饿）。在某些场景下，这也许不是什么大问题；但如果你在做昂贵的初始化、长时间运行的工作，或者你有一个会无限执行某项任务的 future，就需要认真考虑该在何时、何地把控制权交还给运行时。

即使面对计算密集型任务，async 依然可能是有用的，具体取决于你的程序还在做什么，因为它提供了一种很实用的手段，用来组织程序不同部分之间的关系（当然代价是 async 状态机本身也有一定开销）。这是一种 协作式多任务（cooperative multitasking）：每个 future 都可以通过 await 点来决定何时交出控制权，因此每个 future 也都负有避免长时间阻塞的责任。在某些基于 Rust 的嵌入式操作系统中，这甚至是唯一的多任务形式！

当然，在真实代码里，你通常不会在每一行之间都交替插入函数调用和 await 点。像这样主动交出控制权虽然相对便宜，但并不是没有代价。在很多情况下，试图把一个计算密集型任务切得太碎，反而可能显著拖慢它的执行速度，所以有时为了整体性能，让某个操作短暂阻塞一下反而更好。还是那句老话：一定要靠测量来确认代码真正的性能瓶颈。不过，如果你发现原本预期会并发执行的工作，实际上却大量串行发生，那么就要记住这里的底层机制。

```rust
fn main() {
    trpl::block_on(async {
        let work = async {
            let mut total = 0_u64;
            for batch in 0_u64..100 {
                for i in 0_u64..1_000 {
                    total = total.wrapping_add(batch * 1_000 + i);
                }
                trpl::yield_now().await;
            }
            total
        };
        let observer = async {
            for _ in 0..3 {
                println!("其他工作也有机会推进");
                trpl::yield_now().await;
            }
        };
        let (total, ()) = trpl::join(work, observer).await;
        println!("合计：{total}");
    });
}
```

**讨论：**`yield_now` 主动提供让出执行权的机会，但不保证接下来一定执行某个指定任务。对始终返回 `Ready` 的 future 反复 `.await` 也可能不让出执行权。`std::thread::sleep`、同步网络调用和长时间计算不会因为写在 async 块中就变成非阻塞操作；应使用异步 I/O，或把阻塞工作交给专用线程 / `spawn_blocking`。

### 构建我们自己的异步抽象

我们还可以把多个 future 组合在一起，创造出新的模式。比如，完全可以用手头已有的 async 构件来写一个 timeout 函数。等我们做完，它本身就会成为另一个可以继续拿来构建更多 async 抽象的基础模块。

让我们来实现它。首先先想想 timeout 的 API：

- 它需要返回一个 future；这里使用 `async fn` 实现。
- 第一个参数是要执行的 future，通过泛型支持不同 future 类型。
- 第二个参数是最大等待时间，使用 `Duration`。
- 返回 `Result<F::Output, Duration>`：按时完成时返回输出，否则返回等待时长。这里的 `Ok` 仅表示未超时，不代表业务操作一定成功。

接下来想想我们需要的行为：我们希望让传入的 future 和这个时长“竞争”。可以用 trpl::sleep 根据这个时长构造一个计时器 future，再用 trpl::select 让计时器和调用者传入的 future 一起运行。

trpl::select 的实现并不是公平的：它总是按参数传入的顺序进行轮询（其他一些 select 实现会随机选择先轮询哪个参数）。因此，我们把 future_to_try 作为第一个参数传给 select，好让它即使在 max_time 很短的情况下，也仍然有机会先完成。如果 future_to_try 先完成，select 会返回 Left，其中包含 future_to_try 的输出；如果 timer 先完成，select 就会返回 Right，其中包含计时器的输出 ()。

如果 future_to_try 成功完成，并且我们得到了 Left(output)，那么就返回 Ok(output)。如果相反是睡眠计时器先结束，我们得到 Right(())，那就用 _ 忽略这个 ()，并返回 Err(max_time)。

这样一来，我们就用另外两个 async 小工具拼出了一个可工作的 timeout：

```rust
use std::{future::Future, time::Duration};
use trpl::Either;

async fn timeout<F: Future>(
    future_to_try: F,
    max_time: Duration,
) -> Result<F::Output, Duration> {
    match trpl::select(future_to_try, trpl::sleep(max_time)).await {
        Either::Left(output) => Ok(output),
        Either::Right(()) => Err(max_time),
    }
}

fn main() {
    trpl::block_on(async {
        let slow = async {
            trpl::sleep(Duration::from_secs(2)).await;
            "完成"
        };
        match timeout(slow, Duration::from_millis(100)).await {
            Ok(value) => println!("{value}"),
            Err(limit) => println!("超过等待时间：{limit:?}"),
        }
    });
}
```

**讨论：**这是协作式超时，不是强制中断。如果业务 future 的一次 `poll` 长时间阻塞，计时器就无法及时被检查。超时分支获胜会丢弃本例中直接传入的业务 future，但不会撤销已产生的副作用；若传入的是已启动任务的句柄，丢弃句柄也未必取消后台任务。正式项目通常优先使用运行时提供的超时 API，并检查操作的取消安全性。


### Stream：按顺序出现的future

异步版的 recv 方法会随着时间推移产出一系列条目。这正是一种更普遍模式的实例，通常称为 流（stream）。很多概念都很自然地适合表示成 stream：队列中逐步变得可用的项、当完整数据集太大而无法一次装入内存时从文件系统中逐块拉取的数据、或者随着时间逐渐从网络到达的数据。由于 stream 本身也和 future 密切相关，我们可以把它和其他类型的 future 一起使用，并以有趣的方式进行组合。比如，我们可以把事件分批处理，以避免触发过多网络调用；可以为一串长时间运行的操作设置超时；也可以对 UI 事件进行节流，避免做无谓的工作。

迭代器和异步信道接收端之间有两个区别。第一个区别是时间：迭代器是同步的，而信道接收端是异步的。第二个区别是 API。直接处理 Iterator 时，我们会调用同步的 next 方法；而对于 trpl::Receiver 这个具体的 stream 来说，我们调用的是异步的 recv 方法。除此之外，这些 API 给人的感觉非常相似，而这种相似并非巧合。stream 就像迭代的一种异步形式。不过，trpl::Receiver 专门用于等待接收消息，而更通用的 stream API 则宽泛得多：它像 Iterator 一样提供“下一个条目”，只不过是以异步方式来做。

Rust 中迭代器和 stream 的这种相似性意味着，我们实际上可以从任意迭代器创建一个 stream。和使用迭代器一样，我们也可以通过调用 stream 的 next 方法，再等待其输出，来处理它，如示例 17-21 所示。不过这段代码暂时还编译不过。

编译错误的原因是：我们需要把正确的 trait 放进作用域，才能使用 next 方法。根据前面的讨论，你很可能会合理地猜测这个 trait 应该是 Stream，但实际上它是 StreamExt。这里的 Ext 是 extension 的缩写；在 Rust 社区里，用一个 trait 去扩展另一个 trait，是非常常见的模式。

Stream trait 定义的是一个底层接口，它实际上把 Iterator 和 Future trait 的特征结合在了一起。StreamExt 则在 Stream 之上提供了一组更高层的 API，其中包括 next 方法，以及其他一些和 Iterator trait 提供的工具方法相似的辅助方法。Stream 和 StreamExt 目前都还不是 Rust 标准库的一部分，不过生态系统中的大多数 crate 都使用相似的定义。

示例 17-21 补上 `StreamExt` 后：

```rust
use trpl::StreamExt;

fn main() {
    trpl::block_on(async {
        let mut values = trpl::stream_from_iter([1, 2, 3, 4]);
        while let Some(value) = values.next().await {
            println!("{value}");
        }
    });
}
```

缺少 trait 导入时，典型错误是 **E0599**：``no method named `next` found``，具体诊断会继续指出接收者类型以及可导入的 trait。

**讨论：**`Stream` 不是“存放 future 的迭代器”，而是可逐次异步产出 `Item` 的接口；本例的 `Item` 就是整数。从普通迭代器构造的流通常立即就绪，不会凭空产生异步等待。某些经过组合的流不是 `Unpin`，调用 `next` 前可用 `std::pin::pin!` 固定。`trpl::Receiver` 的 `recv` 具有流式语义，但不能据此推断接收端一定直接实现了 `Stream`，可能还需要适配器。

## 深入理解async相关的trait

### Future trait

`std::future::Future` 的核心定义如下（省略属性）：

```rust
use std::{
    pin::Pin,
    task::{Context, Poll},
};

pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

这个 trait 定义里包含了不少新类型，也有一些我们之前还没见过的语法，所以我们逐部分来看。

首先，Future 的关联类型 Output 指明了这个 future 最终会解析成什么值。这和 Iterator trait 里的关联类型 Item 是类似的。其次，Future 提供了一个 poll 方法。它接收一个特殊的 Pin 包裹的 self 引用、一个指向 Context 类型的可变引用，并返回 Poll<Self::Output>。稍后我们会再讲 Pin 和 Context。现在，先聚焦到这个方法的返回值 Poll：
```rust
pub enum Poll<T> {
    Ready(T),
    Pending,
}
```
这个 Poll 类型有点像 Option。它也有一个带值的变体 Ready(T)，以及一个不带值的变体 Pending。但 Poll 的语义和 Option 完全不同。Pending 表示这个 future 还有工作没做完，因此调用方稍后还需要再次检查。Ready 则表示这个 Future 已经完成，其结果值 T 现在已经可用。

`.await` 会被编译成推进子 future 的状态机逻辑，而不是循环忙等 `poll`。当返回 `Pending` 时，future 应安排在可以继续推进时通过 `cx.waker()` 唤醒任务；执行器收到通知后再安排轮询，不必持续占用 CPU 检查。

下面手写一个教学 future，第一次轮询主动唤醒并返回 `Pending`，第二次返回结果：

```rust
use std::{
    future::Future,
    pin::Pin,
    task::{Context, Poll},
};

struct YieldOnce {
    yielded: bool,
}

impl Future for YieldOnce {
    type Output = &'static str;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = self.get_mut(); // 此类型自动实现 Unpin。
        if this.yielded {
            Poll::Ready("完成")
        } else {
            this.yielded = true;
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

fn main() {
    trpl::block_on(async {
        println!("{}", YieldOnce { yielded: false }.await);
    });
}
```

**讨论：**`wake` 是“请再次调度”的通知，不会直接递归执行 `poll`，也不保证立刻执行。真实 I/O future 通常注册 waker，由 I/O 就绪事件唤醒；不要每次返回 `Pending` 都无条件自我唤醒，否则会制造忙轮询。`poll` 应尽快返回；future 返回 `Ready` 后不应再被轮询，重复轮询可能 panic。

### Pin类型和Unpin trait

回到示例 17-13，我们使用过 trpl::join! 宏来等待三个 future。不过，更常见的情况是你会有一个集合，比如一个向量，其中包含若干个 future，而这些 future 的个数要到运行时才知道。让我们把示例 17-13 改成示例 17-23 中的代码：把这三个 future 放进一个向量里，再调用 trpl::join_all。

先看没有类型擦除的错误写法：

```rust
let first = async { println!("第一个"); };
let second = async { println!("第二个"); };
let futures = vec![first, second]; // E0308: mismatched types
```

每个 async 块都有独立的匿名类型。解决方式是显式使用 `dyn Future<Output = ()>` 擦除具体类型；仅分别写 `Box::new(first)`、`Box::new(second)`，并不会自动得到相同类型的盒子。

这也许会让人意外。毕竟，这些 async 代码块都没有返回任何值，所以它们每一个产生的都是 Future<Output = ()>。但别忘了：Future 是个 trait，而编译器会为每个 async 代码块生成一个独一无二的 enum，即使它们的输出类型完全相同。就像你不能把两个不同的手写 struct 放进同一个 Vec，你也同样不能把这些编译器生成的不同 enum 混在一起。

如果进一步改成 `Vec<Box<dyn Future<Output = ()>>>` 并交给 `join_all`，仍可能遇到 **E0277**：`dyn Future<Output = ()> cannot be unpinned`。核心问题是：普通 `Box` 虽然把 future 放到堆上了，但它还没有表达固定其地址的保证。而 Future::poll 恰恰要求这种保证。trpl::join_all 返回的是一个名为 JoinAll 的结构体。这个结构体在类型参数 F 上是泛型的，而 F 又被约束必须实现 Future trait。直接通过 await 去等待一个 future 时，Rust 会隐式地把它 pin 住。这也正是为什么我们平常不需要在每个想等待 future 的地方都显式写 pin!。

但这里，我们并不是直接在等待某个 future。相反，我们是通过把一组 future 传给 join_all，构造出了一个新的 future：JoinAll。而 join_all 的签名要求集合中的元素类型都必须实现 Future trait。另一方面，Box<T> 只有在它包裹的 T 本身是 future 且实现了 Unpin trait 时，才会实现 Future。

Pin 是一种针对指针类类型的包装器，比如 &、&mut、Box 和 Rc。（严格来说，Pin 作用于实现了 Deref 或 DerefMut 的类型，但实际效果基本等同于“引用和智能指针”。）Pin 本身并不是指针，也不像 Rc 或 Arc 那样自带引用计数之类的行为；它纯粹是一个让编译器能够对指针使用方式施加约束的工具。

记住，我们在本章前面提过，一个 future 里的多个 await 点会被编译成一个状态机，而编译器会确保这个状态机遵守 Rust 关于安全性的全部常规规则，包括借用和所有权。为了做到这一点，Rust 会分析：在某个 await 点和下一个 await 点之间，或者直到 async 代码块结束之前，哪些数据是需要保留的。然后，它会在编译出来的状态机里生成对应的变体。每个变体都会得到其对应源代码片段所需的数据访问权限，这种访问可能是获得所有权，也可能是获得可变或不可变引用。

到这里为止，一切都很好：如果你在某个 async 代码块里把所有权或引用关系写错了，借用检查器会告诉你。但当我们想要移动这个代码块对应的 future 时，比如把它放进 Vec 然后传给 join_all，事情就开始变复杂了。

当我们移动一个 future 时，无论是把它放进数据结构，以便通过 join_all 这种方式迭代处理，还是从函数里返回它，本质上都是在移动 Rust 为我们生成的那个状态机。与 Rust 中大多数其他类型不同的是，Rust 为 async 代码块生成的 future，可能会在某个状态变体的字段里保存指向它自身其他字段的引用，这类内部自引用需要稳定的存储地址。

在形成依赖稳定地址的自引用之后，直接移动对象可能破坏这些引用，因为引用指向的是原来的内存地址。如果我们移动了这个数据结构本身，那么这些内部引用仍然会指向旧位置。然而那个内存地址现在已经失效了。一方面，你之后对数据结构做的修改不会再反映到那些旧引用上；另一方面，更严重的是，计算机此时已经可以把那块内存拿去做别的用途了。若通过 unsafe 代码破坏了这类约定，后续访问可能形成悬垂引用并触发未定义行为；安全 Rust 不会允许我们随意进行这种移动。

理论上，Rust 编译器也可以尝试在对象被移动时更新所有引用，但这样很可能带来大量性能开销，尤其在需要更新的是一整张引用网络的时候。如果我们反过来，确保这个数据结构根本不在内存中移动，那就完全不需要更新任何引用。这正是 Rust 借用检查器要做的事：在安全代码里，它会阻止你移动任何仍然存在活动引用的值。

`Pin` 正是在这个基础上，进一步表达地址稳定性的约定。对于依赖 pin 保证的 `!Unpin` 值，一旦被正确固定，就不能再从该位置移出，其存储位置必须保持有效直到析构完成。它不是自动检测移动的运行时机制，底层 unsafe 实现也必须维护这一约定。因此，如果你有的是 Pin<Box<SomeType>>，那么真正被 pin 住的是 SomeType 这个值，而不是 Box 指针本身。

实际上，Box 指针本身仍然可以自由移动。请记住：我们真正关心的是最终被引用的数据必须固定不动。如果指针移动了，但它指向的数据仍然留在原地，那么就不会产生问题。关键在于：那个自引用的类型本身不能移动，因为它仍然是被 pin 住的。

不过，大多数类型并不依赖稳定地址，可以实现 `Unpin`，例如整数、布尔值以及 `String`。内部有引用不一定需要 pin，内部没有 Rust 引用也不一定不需要 pin；关键是类型是否有依赖地址稳定性的约定，例如自引用或侵入式链表。

应区分 `Pin<Box<Vec<String>>>` 与 `Pin<Vec<String>>`：前者通过 `Box` 指向一个 `Vec`，后者的 `Vec` 自身作为指针包装器，通过 `Deref` 指向切片，并不是“固定 Vec 对象”的通常写法。这里学习固定 future 时，优先关注 `Pin<Box<F>>` 和 `Pin<&mut F>`。

Unpin 是一个标记 trait（marker trait），就像我们在第十六章见过的 Send 和 Sync 一样，它本身没有任何功能。marker trait 的存在，只是为了告诉编译器：实现了该 trait 的类型，在某种特定上下文里可以被安全使用。`Unpin` 表示类型不依赖 pin 的地址稳定性保证，可以通过安全 API 从 `Pin` 中取得普通可变访问。它是自动 trait，大多数普通类型会自动实现；编译器生成的 async future 通常不能假定实现 `Unpin`。

示例 17-23 使用 `Box::pin` 同时完成堆分配与固定：

```rust
use std::{future::Future, pin::Pin};

fn main() {
    trpl::block_on(async {
        let first = async { println!("第一个"); };
        let second = async { println!("第二个"); };
        let futures: Vec<Pin<Box<dyn Future<Output = ()>>>> = vec![
            Box::pin(first),
            Box::pin(second),
        ];
        trpl::join_all(futures).await;
    });
}
```

如果不需要堆分配，可以把 future 固定在当前作用域，再统一为固定引用：

```rust
use std::{future::Future, pin::{pin, Pin}};

fn main() {
    trpl::block_on(async {
        let mut first = pin!(async { println!("第一个"); });
        let mut second = pin!(async { println!("第二个"); });
        let futures: Vec<Pin<&mut dyn Future<Output = ()>>> = vec![
            first.as_mut(),
            second.as_mut(),
        ];
        trpl::join_all(futures).await;
    });
}
```

**讨论：**固定后仍可以移动 `Pin<Box<F>>` 这个指针包装器，不能随意移出的是它指向的 `!Unpin` future。future 在被固定前可以正常移动，所以调用异步函数后再把返回值交给组合器是安全的。若多个 future 来自同一个异步函数调用，它们的类型相同，通常无需 `dyn Future`；固定数量的异构 future 则优先考虑 `join!`。`join_all` 会保留全部结果并按输入顺序返回，大量任务应另行限制并发数。

### Stream trait

`Stream` 可以理解为反复异步取得下一个元素的接口，其核心定义为：

```rust
use std::{pin::Pin, task::{Context, Poll}};

pub trait Stream {
    type Item;

    fn poll_next(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
    ) -> Poll<Option<Self::Item>>;
}
```

**讨论：**`Pending` 表示暂时没有下一个元素，`Ready(Some(item))` 表示取得一个元素，`Ready(None)` 表示流结束，而不是“稍后重试”。`StreamExt::next` 把一次 `poll_next` 等待封装成 future；通常不应在流结束后继续轮询，除非相应类型明确保证支持。

## future，任务和线程

线程提供了一种实现并发的方式。而在本章里，我们又看到了另一种方式：使用基于 future 和 stream 的 async。如果你想知道应该在什么时候选哪一种，答案是：这取决于具体情况！而且在很多情况下，真正的选择并不是线程或 async，而是线程和 async 一起用。

许多操作系统几十年来一直都提供基于线程的并发模型，因此许多编程语言也都支持它们。但这些模型并非没有代价。在许多操作系统中，每个线程都要占用相当可观的内存。线程也只有在你的操作系统和硬件支持它们时才可用。与主流桌面和移动设备不同，一些嵌入式系统甚至根本没有操作系统，因此也就根本没有线程。

async 模型提供了一组不同的，而且归根结底是互补的权衡。在 async 模型中，并发操作不需要各自独占一个线程。相反，它们可以运行在任务上，就像前面通过 `trpl::spawn_task` 提交工作时那样。任务和线程有点像，但它不是由操作系统管理，而是由库层面的代码，也就是运行时来管理。

线程 API 和任务 API 长得这么像并不是偶然。线程是“一组同步操作”的边界；并发发生在线程之间。任务则是“一组异步操作”的边界；并发既可能发生在任务之间，也可能发生在任务内部，因为任务的函数体里可以在多个 future 之间切换。最后，future 是 Rust 中最细粒度的并发单位，而每个 future 又可能代表一棵由其他 future 组成的树。运行时，准确地说是它的 executor，管理任务；任务再去管理 future。从这个角度说，任务有点像“轻量级的、由运行时管理的线程”，只不过由于它们是由运行时而不是操作系统管理，所以又拥有了一些额外能力。

这并不意味着 async 任务就一定总比线程更好，反过来也一样。在线程基础上的并发，在某些方面其实是比 async 并发更简单的编程模型。这一点既可能是优点，也可能是缺点。线程有点像 “射后不理”（“fire and forget”）：它们没有原生对应 future 的机制，因此通常只是一路运行到结束，除非被操作系统本身打断。

在思考什么时候该用哪种方式时，可以先记住这些经验法则：

- 如果工作是非常适合并行化的，也就是典型的 CPU 密集型任务，比如有一大批数据而且每一部分都能单独处理，那么线程通常是更好的选择。
- 如果工作是高度并发的，也就是典型的 I/O 密集型任务，比如要同时处理来自很多不同来源、且到达间隔和频率都各不相同的消息，那么 async 通常是更好的选择。

下面单独使用 Tokio，演示把阻塞工作移出异步工作线程。先执行 `cargo add tokio --features rt-multi-thread,macros,time`；此例不依赖前面的 `trpl` 代码：

```rust
use std::time::Duration;

#[tokio::main]
async fn main() -> Result<(), tokio::task::JoinError> {
    let blocking = tokio::task::spawn_blocking(|| {
        // 模拟无法改成异步接口的第三方阻塞调用。
        std::thread::sleep(Duration::from_millis(200));
        42
    });
    let observer = async {
        tokio::time::sleep(Duration::from_millis(20)).await;
        println!("异步任务仍能推进");
    };

    let (result, ()) = tokio::join!(blocking, observer);
    println!("阻塞工作结果：{}", result?);
    Ok(())
}
```

**讨论：**`spawn_blocking` 使用阻塞线程池，适合隔离同步 I/O 或有限的阻塞工作，不会让这些操作本身变成异步。大量 CPU 密集型工作应考虑 Rayon 等专用计算线程池，并限制并发度；已经开始运行的阻塞闭包通常不能通过取消句柄强行终止。

使用多线程任务 API 时，还应理解所有权与线程安全边界：

```rust
#[tokio::main]
async fn main() {
    let name = String::from("Rust");
    let handle = tokio::spawn(async move {
        tokio::task::yield_now().await;
        println!("你好，{name}");
    });
    handle.await.expect("任务执行失败");
}
```

**讨论：**`tokio::spawn` 要求 future 及其输出满足 `Send + 'static`。`Send` 允许任务在不同线程间迁移；`'static` 表示不能依赖短命的外部借用，并不表示任务永远存活。`async move` 能转移拥有所有权的数据，却不会把引用变成 `'static`，也不会把 `Rc` 变成 `Send`。若 future 在暂停时保存 `Rc` 或某些锁守卫，可能出现 `future cannot be sent between threads safely`。

共享状态通常使用 `Arc` 配合适当的锁。只做短小同步操作时可在明确的作用域内使用同步互斥锁，并在 `.await` 前释放守卫；确实需要跨等待持锁时可考虑异步锁，但仍要警惕死锁和过长临界区。不是所有 future 都必须满足 `Send + 'static`：在当前任务内使用 `join` 可以借用局部数据，`LocalSet` / `spawn_local` 则支持局部执行的 `!Send` 任务。

可以把本章的关系概括为：**future 描述可暂停的工作，任务是运行时调度的单位，线程是操作系统调度的单位；`.await` 负责组合等待，而不是自动创建任务或线程。**

