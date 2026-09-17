---
title: 'Rust中的并发编程'
publishDate: 'September 17, 2026'
updatedDate: 'September 17, 2026'
description: '我们从不伟大'
tags:
  - Rust
  - Programming language
  - Concurrency
language: 'Chinese'
heroImage: { src: 'hole.png', color: '#559ce8' }
---

安全且高效地处理并发编程，是 Rust 的主要目标之一。并发（concurrent programming）强调多个任务在同一段时间内交替推进，并行（parallel programming）强调多个任务在同一时刻执行。单核处理器也能通过调度实现并发，多核处理器则可以让任务真正并行。

Rust 不强制使用某一种并发模型，而是通过所有权、类型系统以及标准库提供的线程、信道和同步原语，让我们按需求选择消息传递或共享状态。

**讨论：** Rust 的安全保证主要针对数据竞争和内存安全，而不是所有并发逻辑错误。数据竞争涉及不同线程对同一内存位置的未同步访问，且至少一个访问是写入；更广义的竞态条件还包括结果依赖执行顺序的业务错误。安全 Rust 可以防止数据竞争，但仍可能发生死锁、饥饿或执行顺序不符合预期等问题。

## 使用线程同时运行代码

一个进程中可以包含多个线程，它们共享进程的地址空间，但有各自的执行栈。Rust 标准库采用 1:1 线程模型：每个 Rust 线程对应一个操作系统线程。

**讨论：** 线程可以用于并行计算和阻塞式 I/O，但创建线程、维护栈空间和上下文切换都有成本，并非线程越多越快。大量任务通常应考虑线程池；异步任务也不等于操作系统线程。无论采用哪种方式，都不能假设线程会按创建顺序执行。

### 使用spawn创建新线程

使用 `thread::spawn` 创建线程，将需要在线程中执行的代码放进闭包：

```rust
use std::thread;
use std::time::Duration;

fn main() {
    let _handle = thread::spawn(|| {
        for i in 1..10 {
            println!("子线程：{i}");
            thread::sleep(Duration::from_millis(1));
        }
    });

    for i in 1..5 {
        println!("主线程：{i}");
        thread::sleep(Duration::from_millis(1));
    }
}
```

**讨论：** 两个线程的输出可能交错，具体顺序由调度决定。`sleep` 暂停当前线程，但不保证另一个线程此时一定执行，不能拿它代替同步。`main` 返回后整个进程结束，未完成的子线程不会自动被等待，因此上例不保证打印完子线程的所有内容。

### 等待所有线程结束

`thread::spawn` 返回 `JoinHandle<T>`，其中 `T` 是线程闭包的返回值类型。调用 `join` 会消费句柄，阻塞当前线程直到目标线程结束：

```rust
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        let sum: i32 = (1..=100).sum();
        println!("子线程计算完成");
        sum
    });

    println!("主线程可以先做其他工作");

    match handle.join() {
        Ok(sum) => println!("计算结果：{sum}"),
        Err(_) => eprintln!("子线程发生了 panic"),
    }
}
```

**讨论：** `join` 成功时返回 `Ok(T)`；在 panic 展开模式下，子线程发生未捕获的 panic 时返回 `Err`，不会自动让主线程 panic。调用 `join().unwrap()` 才会在失败时让等待方也 panic；若配置 `panic = "abort"`，panic 会直接终止进程。

`join` 的位置很重要：先创建所有线程再逐一 `join`，可以让它们并发推进；若每创建一个线程就立即 `join`，这些任务便基本串行执行。丢弃 `JoinHandle` 不会取消或等待线程，而是将其分离。成功 `join` 也建立了同步关系：目标线程中的操作先于 `join` 返回后的操作完成。

### 将move闭包与线程一同使用

下面的闭包默认借用 `v`，不能满足 `thread::spawn` 对闭包的生命周期要求：

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];

    // 错误示例：E0373
    let handle = thread::spawn(|| {
        println!("向量：{v:?}");
    });

    handle.join().unwrap();
}
```

编译器报错：`error[E0373]: closure may outlive the current function, but it borrows 'v', which is owned by the current function`。

增加 `move`，将向量的所有权移入闭包：

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];

    let handle = thread::spawn(move || {
        println!("向量：{v:?}");
    });

    // 这里不能再使用 v，因为它已经被移动。
    handle.join().unwrap();
}
```

**讨论：** `spawn` 的核心约束可以概括为闭包满足 `FnOnce() -> T + Send + 'static`，返回值满足 `Send + 'static`。`'static` 不表示线程或捕获值必须存活到程序结束，而是要求其中不能包含生命周期不够长的借用。拥有数据的 `Vec<i32>` 满足这一要求；普通的 `join` 调用不会放宽 `spawn` 的生命周期约束。

`move` 按值捕获变量，但不会延长引用所指对象的生命周期。如果捕获的变量本身是局部引用，移动这个引用仍不能使其变成 `'static`。另外，`Vec<T>` 没有实现 `Display`，因此这里使用 `{v:?}`，而不是 `{v}`，避免把格式化错误与生命周期错误混在一起。

如果确实需要让线程借用局部数据，可以使用作用域线程：

```rust
use std::thread;

fn main() {
    let mut values = vec![1, 2, 3];
    let label = String::from("局部数据");

    thread::scope(|scope| {
        scope.spawn(|| {
            values.push(4);
        });
        scope.spawn(|| {
            println!("正在处理：{label}");
        });
    });

    println!("处理完成：{values:?}");
}
```

**讨论：** `thread::scope` 保证作用域内创建的线程在作用域返回前结束，因此允许借用外部局部变量。它没有绕过借用检查：一个线程持有 `values` 的可变借用期间，其他线程仍不能同时访问这份数据。

## 使用消息传递在线程中传递数据

消息传递让线程通过发送和接收数据进行通信。标准库的 `std::sync::mpsc` 提供多生产者、单消费者信道（multiple producer, single consumer）。`channel` 返回发送端 `Sender<T>` 和接收端 `Receiver<T>`，通常命名为 `tx` 和 `rx`。

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    let handle = thread::spawn(move || {
        tx.send(String::from("你好，主线程")).unwrap();
    });

    let message = rx.recv().unwrap();
    println!("收到：{message}");
    handle.join().unwrap();
}
```

**讨论：** 编译器从 `send` 的参数推断信道类型为 `String`。如果暂时没有发送或接收代码提供类型信息，可以显式写成 `mpsc::channel::<String>()`；仅写 `let (tx, rx) = mpsc::channel();` 而没有其他类型线索，会出现 `E0282: type annotations needed`。

发送与接收的关键语义如下：

- `send(value)` 返回 `Result<(), SendError<T>>`。接收端被丢弃后，发送失败；错误值中保留未发送的数据。发送成功不保证接收端最终一定处理该消息。
- `recv()` 阻塞等待一条消息。当**所有发送端**都被丢弃，且缓冲消息已经取完时，返回 `Err(RecvError)`。
- `try_recv()` 不阻塞。暂时无消息返回 `TryRecvError::Empty`；所有发送端都已丢弃且缓冲已空时返回 `TryRecvError::Disconnected`。
- `recv_timeout(duration)` 可以限制等待时间，应区分超时与断开连接。

下面区分 `try_recv` 的两种错误：

```rust
use std::sync::mpsc::{self, TryRecvError};

fn main() {
    let (tx, rx) = mpsc::channel::<i32>();

    match rx.try_recv() {
        Ok(value) => println!("收到：{value}"),
        Err(TryRecvError::Empty) => println!("暂无消息，但发送端还存在"),
        Err(TryRecvError::Disconnected) => println!("不会再有消息"),
    }

    drop(tx);
    assert_eq!(rx.try_recv(), Err(TryRecvError::Disconnected));
}
```

**讨论：** 不要用无休止的 `try_recv` 空循环等待消息，否则会忙等并消耗 CPU。如果没有其他工作，直接使用 `recv`；需要周期性处理任务时，可以考虑 `recv_timeout`。

`channel` 的缓冲在 API 层面没有固定容量限制，生产速度长期超过消费速度可能导致内存增长。需要背压时，可以使用有界信道：

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::sync_channel(1);

    let handle = thread::spawn(move || {
        tx.send(10).unwrap();
        tx.send(20).unwrap(); // 缓冲已满时，等待接收端取走消息。
    });

    for value in rx {
        println!("收到：{value}");
    }
    handle.join().unwrap();
}
```

**讨论：** `sync_channel(n)` 限制缓冲容量，其发送端是 `SyncSender<T>`。`sync_channel(0)` 没有缓冲，发送方必须与接收方会合。使用有界信道时，如果接收方先 `join` 等待发送线程，而发送线程正在等待空余容量，就可能死锁。

### 通过信道转移所有权

`send` 按值接收消息。发送 `String` 等非 `Copy` 类型后，发送方不能继续使用原变量：

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    let handle = thread::spawn(move || {
        let val = String::from("hello");
        tx.send(val).unwrap();
        println!("发送后再次使用：{val}"); // 错误示例：E0382
    });

    println!("收到：{}", rx.recv().unwrap());
    handle.join().unwrap();
}
```

编译器报错：``error[E0382]: borrow of moved value: `val` ``。

需要保留字符串时，可以显式克隆：

```rust
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel();
    let val = String::from("hello");

    tx.send(val.clone()).unwrap();
    println!("原值仍可用：{val}");
    println!("收到副本：{}", rx.recv().unwrap());
}
```

**讨论：** `clone` 会为 `String` 复制数据并产生分配成本；如果无需保留，直接移动更合适。对 `i32` 等 `Copy` 类型，按值发送会复制，原变量仍可使用。发送 `Arc<T>` 则是传递共享所有权句柄，并不复制底层 `T`。信道负责同步消息传递，但不会自动把任意消息类型变成线程安全类型。

### 发送多个值

可以让生产者连续发送消息，接收端通过迭代处理，直到信道断开并且消息耗尽：

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();

    let handle = thread::spawn(move || {
        for word in ["Rust", "通过", "信道", "通信"] {
            tx.send(String::from(word)).unwrap();
            thread::sleep(Duration::from_millis(100));
        }
        // 闭包结束，tx 被丢弃。
    });

    for message in rx {
        println!("收到：{message}");
    }

    handle.join().unwrap();
}
```

**讨论：** `for message in rx` 是阻塞式迭代，不是只检查当前已有消息。缓冲暂时为空时，只要还有发送端存活，就会继续等待。发送线程结束不必然意味着接收循环结束：主线程或其他线程若还保留发送端克隆，循环仍可能阻塞。信道关闭也不等于生产线程已执行完后续清理，等待线程完全结束仍应使用 `join`。

### 创建多个生产者

克隆发送端，即可让多个线程向同一个接收端发送消息：

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();
    let mut handles = Vec::new();

    for producer in 0..2 {
        let tx = tx.clone();
        handles.push(thread::spawn(move || {
            for sequence in 0..3 {
                tx.send((producer, sequence)).unwrap();
            }
        }));
    }

    drop(tx); // 丢弃主线程保留的发送端，避免接收循环永远等待。

    for (producer, sequence) in rx {
        println!("生产者 {producer}：消息 {sequence}");
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

**讨论：** `tx.clone()` 创建的是同一信道的另一个发送端句柄，不是复制信道中的消息。每个生产者依次发送的消息保持自身的顺序，但不同生产者之间的交错顺序不确定，也不保证每次运行都不同。若业务要求全局顺序，应显式设计编号和排序协议。标准库 `mpsc::Receiver<T>` 不支持 `clone`，不能通过克隆它直接创建多个消费者。

## 共享状态并发

消息传递通常通过移动数据来划分所有权；共享状态则让多个线程访问同一份数据，并通过同步机制协调读写。典型组合是 `Arc<Mutex<T>>`：`Arc` 管理共享所有权，`Mutex` 管理互斥访问。

**讨论：** 两种方式并不互斥，例如可以通过信道发送 `Arc<T>`。任务和结果能清晰划分时，消息传递通常更容易理解；多个线程需要共同维护某份状态时，共享状态可能更直接，但必须注意锁的范围和获取顺序。

### 使用互斥器控制访问

互斥器（mutex，mutual exclusion）在同一时刻只允许一个线程持有锁。访问受保护的数据时，需要先获取锁，使用完成后再释放。

```rust
use std::sync::Mutex;

fn main() {
    let message = Mutex::new(String::from("hello"));

    {
        let mut guard = message.lock().unwrap();
        guard.push_str(", Rust");
    } // guard 被丢弃，自动释放锁。

    println!("{}", message.lock().unwrap());
}
```

**讨论：** Rust 用锁守卫的生命周期管理解锁，减少遗漏解锁的错误，但不保证不会死锁。持锁期间再次获取同一把标准库 `Mutex` 的锁，不保证能够完成，可能死锁或 panic。持锁等待其他线程、执行慢速 I/O，或者以不一致的顺序获取多把锁，也容易造成问题。应让临界区尽量短，并统一多锁获取顺序。

### Mutex<T>的API

`Mutex::new(value)` 创建互斥器；`lock()` 返回 `LockResult<MutexGuard<'_, T>>`。真正实现 `Deref`、`DerefMut` 并在 `Drop` 时解锁的是 `MutexGuard`，而不是 `Mutex` 本身。

```rust
use std::sync::Mutex;

fn main() {
    let m = Mutex::new(5);

    {
        let mut num = m.lock().unwrap();
        *num = 6;
    }

    let num = m.lock().unwrap();
    println!("当前值：{}", *num);
    drop(num); // 也可以显式提前释放锁。
}
```

**讨论：** `lock` 在锁被占用时阻塞当前线程；`try_lock` 则立即返回，锁不可用时返回 `TryLockError::WouldBlock`。守卫需要声明为 `mut` 才能通过它修改内部数据，但外层的 `m` 不必声明为 `mut`，这是内部可变性。

若线程在持锁期间 panic，互斥器通常会进入“中毒”（poisoned）状态。后续 `lock()` 返回包含守卫的 `PoisonError`，提示受保护数据可能处于不一致状态；它不是永久无法获取锁。确认能够恢复不变式时，可以取出守卫、修复数据并清除中毒标记：

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let state = Arc::new(Mutex::new(vec![1, 2, 3]));
    let worker_state = Arc::clone(&state);

    let handle = thread::spawn(move || {
        let mut values = worker_state.lock().unwrap();
        values.clear();
        panic!("模拟更新中断");
    });
    assert!(handle.join().is_err());

    let mut values = match state.lock() {
        Ok(guard) => guard,
        Err(poisoned) => {
            let mut guard = poisoned.into_inner();
            // 本例的恢复策略：将数据重置为一个已知有效的状态。
            *guard = vec![1, 2, 3];
            state.clear_poison();
            guard
        }
    };
    values.push(4);
    println!("恢复后：{values:?}");
}
```

**讨论：** 上例假定使用 panic 展开模式。不能不加判断地忽略中毒：实际数据可能需要回滚、重建或终止服务。中毒只是尽力提供的提示，某些 panic 场景不会触发中毒，因此不能将其当作 `unsafe` 代码的内存安全依据。对于独占持有的 `Mutex`，还可以使用 `get_mut(&mut self)` 或消费它的 `into_inner(self)` 访问数据，而不必通过运行时加锁。

### 给Mutex<T>共享访问

单独使用 `Mutex<T>`，并不能让它的所有权被移动到多个线程：

```rust
use std::sync::Mutex;
use std::thread;

fn main() {
    let counter = Mutex::new(0);
    let mut handles = Vec::new();

    for _ in 0..10 {
        // 错误示例：第一次循环已经将 counter 移入闭包。
        handles.push(thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

编译器报错：``error[E0382]: use of moved value: `counter` ``。

**讨论：** 互斥访问和共享所有权是两个问题。`Mutex` 解决的是“谁现在能访问数据”，并不允许一个非 `Copy` 值被多次移动。给每个线程创建新的 `Mutex::new(0)` 虽然不会移动同一个值，却也不再是共享同一份计数器。使用 `spawn` 时通常应为它加上共享所有权容器；使用作用域线程时，也可以让多个线程借用同一个 `Mutex`。

### 多线程和多所有权

`Rc<T>` 能在单线程中提供共享所有权，但不能代替跨线程的共享容器：

```rust
use std::rc::Rc;
use std::sync::Mutex;
use std::thread;

fn main() {
    let counter = Rc::new(Mutex::new(0));
    let shared = Rc::clone(&counter);

    // 错误示例：E0277
    let handle = thread::spawn(move || {
        *shared.lock().unwrap() += 1;
    });

    handle.join().unwrap();
}
```

编译器报错：``error[E0277]: `Rc<Mutex<i32>>` cannot be sent between threads safely``，原因是该类型没有实现 `Send`。

**讨论：** `Rc` 的引用计数更新不是线程安全的。内部加一操作虽然受 `Mutex` 保护，但外层 `Rc` 的克隆和销毁并不受这把锁保护，因此 `Rc<Mutex<T>>` 仍不能跨线程传递。需要将外层引用计数改为线程安全的 `Arc`。

### 原子引用计数 Arc<T>

`Arc<T>` 是原子引用计数（atomically reference counted）智能指针。用它包裹 `Mutex<T>`，即可让多个线程共同更新计数器：

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = Vec::new();

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("结果：{}", *counter.lock().unwrap()); // 10
}
```

**讨论：** `Arc::clone` 增加引用计数，不会复制内部的 `Mutex` 或计数值。各线程和主线程持有的句柄指向同一份数据；最后一个强引用被丢弃时，内部值才会被销毁。先等待所有工作线程，再读取最终结果，才能保证所有加一操作都已经完成。

`Arc` 只保证共享所有权管理的线程安全，不会自动赋予内部数据线程安全的可变访问能力。共享不可变数据可以使用 `Arc<T>`；共享可变数据通常还需要 `Mutex`、`RwLock` 或原子类型。`Arc` 的原子引用计数存在成本，单线程场景不必一律替换 `Rc`。

简单计数也可以使用原子整数：

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;
use std::thread;

fn main() {
    let counter = Arc::new(AtomicUsize::new(0));
    let mut handles = Vec::new();

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            counter.fetch_add(1, Ordering::Relaxed);
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }
    println!("结果：{}", counter.load(Ordering::Relaxed)); // 10
}
```

**讨论：** `fetch_add` 把读、修改、写入组成一个原子操作，避免丢失更新。这里计数器不承担发布其他数据的职责，并且通过 `join` 等待完成，因此可以使用 `Relaxed`。它不保证其他内存访问之间的同步顺序，不能照搬到“设置标志后通知其他线程读取普通数据”等协议中。多个字段需要一起维持不变式时，使用锁往往更容易保证正确性；将原子 `load` 和 `store` 分开，也不等于原子的整体加一。

### 比较 RefCell<T>/Rc<T> 和 Mutex<T>/Arc<T>

| 类型或组合 | 主要作用 | 使用边界 |
| --- | --- | --- |
| `Rc<T>` | 非原子共享所有权 | 不能在线程间传递或共享 |
| `RefCell<T>` | 运行时检查借用的内部可变性 | 不是 `Sync`；当 `T: Send` 时可以整体移动到另一线程 |
| `Rc<RefCell<T>>` | 单线程共享可变数据 | 借用冲突时 `borrow` / `borrow_mut` 会 panic |
| `Arc<T>` | 原子共享所有权 | 跨线程使用仍要求内部类型满足相应约束 |
| `Mutex<T>` | 互斥访问与内部可变性 | 锁竞争时通常阻塞，需处理死锁和中毒 |
| `Arc<Mutex<T>>` | 跨线程共享可变数据 | 通常要求 `T: Send` |

下面把耗时计算放在锁外，只在提交结果时持锁：

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let results = Arc::new(Mutex::new(Vec::new()));
    let worker_results = Arc::clone(&results);

    let handle = thread::spawn(move || {
        let sum: i32 = (1..=1_000).sum(); // 锁外计算。
        {
            let mut results = worker_results.lock().unwrap();
            results.push(sum); // 仅提交结果时持锁。
        }
        println!("结果已提交");
    });

    handle.join().unwrap();
    println!("{:?}", *results.lock().unwrap());
}
```

**讨论：** `RefCell` 与 `Mutex` 都允许通过共享引用修改内部数据，但借用冲突与锁竞争的处理方式不同。`Arc<RefCell<T>>` 不是合法的跨线程共享可变方案，因为 `RefCell<T>` 不满足 `Sync`。读多写少时可以考虑 `RwLock<T>`，但读写锁也有额外成本和公平性问题，并不必然更快。

缩短临界区不能破坏业务原子性。例如“检查库存是否足够”和“扣减库存”如果拆成两次独立加锁，虽然没有数据竞争，却可能产生业务竞态，应放在同一个临界区中。

## 使用 Send 和 Sync trait 的可扩展并发

线程、信道和锁主要由标准库提供；并发安全还依赖语言与编译器识别的 `std::marker::Send` 和 `Sync`。它们是没有方法的、不安全的自动 trait（unsafe auto trait），用于表达类型是否可以跨线程移动或共享。

**讨论：** 第三方并发库同样通过这些约束与 Rust 类型系统协作。大多数业务类型会根据字段类型自动获得相应实现，不需要开发者自行声明“线程安全”。自动推导仍受类型显式实现或否定实现等规则影响，不能简单认为所有容器都无条件继承内部类型的能力。

### 在线程间转移所有权

`T: Send` 表示类型 `T` 的值可以安全地在线程间转移所有权。可以通过泛型约束表达这个要求：

```rust
use std::thread::{self, JoinHandle};

fn send_to_worker<T: Send + 'static>(value: T) -> JoinHandle<T> {
    thread::spawn(move || value)
}

fn main() {
    let handle = send_to_worker(String::from("在线程间往返"));
    let value = handle.join().unwrap();
    println!("{value}");
}
```

**讨论：** `String`、`Vec<i32>` 等拥有自身数据的类型可以移动到另一线程；`Rc<T>` 不能。`Send` 与 `'static` 是不同约束：前者说明跨线程转移安全，后者在这里避免线程携带可能提前失效的借用。`RefCell<i32>` 是 `Send`，可以整体转移给另一线程，但这并不表示多个线程可以同时通过共享引用使用它。

### 多线程访问

`T: Sync` 表示多个线程可以安全地共享 `&T`。其核心关系是：**`T: Sync` 当且仅当 `&T: Send`**。

```rust
use std::cell::RefCell;
use std::sync::{Arc, Mutex};

fn assert_send<T: Send>() {}
fn assert_sync<T: Sync>() {}

fn main() {
    assert_send::<String>();
    assert_sync::<String>();
    assert_send::<RefCell<i32>>();
    assert_sync::<Mutex<RefCell<i32>>>();
    assert_send::<Arc<Mutex<i32>>>();
    assert_sync::<Arc<Mutex<i32>>>();

    // 取消注释会报 E0277：RefCell<i32> 不能在线程间安全共享。
    // assert_sync::<RefCell<i32>>();
    // assert_send::<Arc<RefCell<i32>>>();
}
```

**讨论：** `Sync` 不是“多个线程都能随便修改”的许可。`String: Sync` 表示可以共享只读引用，不意味着可以同时调用需要 `&mut String` 的方法。对这里使用默认分配器的类型，重要约束包括：

- `Mutex<T>` 在 `T: Send` 时实现 `Send` 和 `Sync`，不要求 `T: Sync`，因为它将访问串行化。
- `Arc<T>` 的 `Send` 和 `Sync` 实现均要求 `T: Send + Sync`。原子引用计数不能补救内部数据缺少同步的问题。
- `Rc<T>` 既不是 `Send`，也不是 `Sync`。
- `RefCell<T>` 不是 `Sync`，因为其运行时借用检查机制不是线程安全的。
- `MutexGuard<'_, T>` 不是 `Send`，不能把持有的标准库互斥锁守卫移动到另一线程后再解锁。

若取消注释 `assert_sync::<RefCell<i32>>()`，诊断包含 ``error[E0277]: `RefCell<i32>` cannot be shared between threads safely``。这些约束让错误在编译期暴露，而不是等到运行时发生内存破坏。

### 手动实现send和sync是不安全的

组合现有安全类型，通常就能得到正确的自动实现：

```rust
use std::sync::Mutex;

struct SharedLog {
    entries: Mutex<Vec<String>>,
}

fn assert_send_sync<T: Send + Sync>() {}

fn main() {
    // 自动满足约束，无须编写 unsafe impl。
    assert_send_sync::<SharedLog>();

    let log = SharedLog {
        entries: Mutex::new(Vec::new()),
    };
    log.entries.lock().unwrap().push(String::from("启动"));
}
```

**讨论：** 手动实现的语法是 `unsafe impl Send for MyType {}` 或 `unsafe impl Sync for MyType {}`。`unsafe` 不是关闭检查后随意使用的开关，而是要求实现者证明：所有安全公开接口都不会破坏相应的跨线程安全保证。

包含原始指针、封装外部库句柄或实现自定义同步结构时，可能需要手动实现，但必须审查底层资源是否具有线程亲和性、共享访问是否同步、析构能否发生在另一线程等问题。不能为了让编译通过，就给包装 `Rc` 或未同步可变状态的类型强行实现这些 trait。应优先使用 `Arc`、`Mutex` 等已有抽象。

## 总结

- `spawn` 创建线程，`join` 等待结束；`move` 转移捕获值，作用域线程允许受控地借用局部数据。
- 信道通过消息组织通信；所有发送端被丢弃且缓冲耗尽后，接收方才确认不会再有消息。
- `Arc` 解决共享所有权，`Mutex` 解决互斥访问；二者不能相互替代。
- `Send` 约束跨线程转移，`Sync` 约束跨线程共享引用。

**讨论：** 在所依赖的 `unsafe` 实现正确的前提下，安全 Rust 能防止数据竞争和无效引用，但“能够编译”不等于“并发逻辑正确”。设计时仍需检查线程是否被等待、信道能否正常关闭、锁的顺序是否一致，以及业务操作是否作为整体得到同步。示例中的 `unwrap` 用于突出核心机制，实际程序还应明确 panic、断连、中毒和任务退出的处理策略。
