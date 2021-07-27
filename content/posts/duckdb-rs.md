---
title: "基于 apache-arrow 的 duckdb rust 客户端"
date: "2021-07-27"
author: "Wang Fenjin"
tags: ["duckdb", "apache-arrow", "olap", "database", "rust", "ffi"]
keywords: ["duckdb", "apache-arrow", "olap", "database", "rust", "ffi"]
description: "本文介绍了 duckdb-rs 的设计和实现"
---

## 背景

[duckdb](https://duckdb.org/) 是一个 C++ 编写的单机版嵌入式分析型数据库。它刚开源的时候是对标 SQLite 的列存数据库，并提供与 SQLite 一样的易用性，编译成一个头文件和一个 cpp 文件就可以在程序中使用，甚至提供与 SQLite 兼容的接口，因此受到了很多人的[关注](https://news.ycombinator.com/item?id=24531085)。

本文介绍笔者近期开发的 duckdb-rs 库，让大家可以很方便地在 rust 代码库中使用 duckdb 的功能。

## libduckdb-sys

了解过 rust 的同学可能知道，rust 提供了 [ffi](https://doc.rust-lang.org/nomicon/ffi.html) 的方式与其他语言互通。因为 duckdb 本身是 C++ 编写的，想要在 rust 里面使用 duckdb，就需要考虑 ffi 的问题。而基于 ffi 对其他语言程序封装的基础库，一般会被命名为 libxxx-sys，这也就是 libduckdb-sys 的由来。

为了方便大家使用，duckdb 提供了 C++ 原生接口，C 接口，以及与 SQLite3 兼容的 C 接口。我在做 libduckdb-sys 的时候对这三种接口都尝试过，相关的讨论可以参见 [Rust Support](https://github.com/duckdb/duckdb/issues/949)，我这里介绍一下当时的情况。

### 基于 SQLite3 接口

最开始我使用的是 SQLite3 的接口，原因主要有三个：
1. 我对 SQLite 比较熟悉，想必用起来会比较方便；
2. 觉得 SQLite 的接口被广泛使用，接口比较稳定，以后不至于大改；
3. 也许是最重要的一点，市面上已经有 SQLite 的 rust 封装[rusqlite](https://github.com/rusqlite/rusqlite)，基于 SQLite 的接口应该能最大程度复用 rusqlite 的代码。

尝试之后确实发现很快能把程序跑起来，基本的功能也能使用。但是随着进一步的深入以及对 duckdb 更多的了解，发现了一些弊端：
1. 虽说 duckdb 是想最大程度兼容 SQLite，但是毕竟一个是行存一个是列存，有区别在所难免，接口肯定也没办法做到 100% 兼容；
2. 有一个区别需要特别提出来，SQLite 是[动态数据类型](https://www.sqlite.org/datatype3.html)，而 duckdb 是静态类型，也就是说在 SQLite 中你可以认为所有的数据都是存成 Text，在读取的时候根据 schema 来解析数据；而 duckdb 是会根据数据类型来存储数据，并且根据列存的特性做一些存储优化。有了这个区别之后，如果我们使用 SQLite 的接口的话，会做一些不必要的数据格式转换，性能有损，程序也不直观。
3. duckdb 可以被编译成一个 so 使用，如果想使用 SQLite 的接口，需要再编译一个 sqlite3_api_wrapper 出来，两个库合作才能使用 SQLite 的接口，这给程序分发引入了额外的负担；另外目前 duckdb 在 release 的时候没有自带 sqlite3_api_wrapper，需要用户自己去编译，使用上又多了一些不便。
4. 由于上面的封装的问题，数据类型的问题，以及通过 SQLite 接口查询 duckdb 的数据时候，结果集会被复制一遍，资源占用必定上升。

基于上面一些原因，我最终放弃了基于 SQLite 接口来开发，转而尝试使用原生的 C++ 或者 C 接口。

### 基于 C++ 接口

既然为了性能和接口丰富性，使用 C++ 接口当然是首选，毕竟 duckdb 本身主要都是拿 C++ 开发的，duckdb 的 [python 封装](https://github.com/duckdb/duckdb/tree/master/tools/pythonpkg) 也是拿 C++ 接口来做的。

市面上也有方便 rust 与 C++ 交互的一些代码库，比如 [cxx](https://github.com/dtolnay/cxx) 和 [autocxx](https://github.com/google/autocxx)。其中 autocxx 入手门槛低使用上更简单，而 cxx 的可定制性更强，功能更丰富。在尝试了几次之后发现了一些问题，主要还是 rust ffi 只能支持部分的 C++ 语法，大部分情况下可能是够用的，但是对于 duckdb 这样比较大型的数据库代码，还是有很多不支持的地方。除非自己再基于现有的 C++ 接口封装一份支持 cxx 的版本，否则就算这一次编译过了，也很难保证以后 duckdb 的作者以后不会引入其他的特性导致不能兼容。

而 rust 基于 C 语言的 ffi 是原生支持的，所以最终还是下定决心基于 C 接口来开发。

### 基于 C 接口

因为有 rusqlite 作为参考，所以很快实现了基于 C 接口的版本。简单来说，主要是通过 [cbindgen](https://github.com/eqrion/cbindgen)、[build.rs](https://doc.rust-lang.org/cargo/reference/build-scripts.html) 和 rust 的 [features](https://doc.rust-lang.org/cargo/reference/features.html) 功能来实现。其中：
* cbindgen 用于生成基于 C 接口的 rust 代码，方便 rust 其他程序使用
* build.rs 和 features 用于控制整个编译流程，用户可以根据需要是当场编译依赖库，还是使用机器上已经安装好的版本
* build.rs 中还可以选择使用 [cc](https://crates.io/crates/cc) 来实时编译 duckdb 实现，这样其他使用 rust 封装的人不用关心 duckdb 的安装问题

应该说这是一个很通用的提供 C 接口 rust 封装的解决方案，感兴趣的同学可以 [参考](https://github.com/wangfenjin/duckdb-rs/tree/main/libduckdb-sys)。

## duckdb-rs

完成了 libduckdb-sys 之后其实只是第一步，因为这样生成的代码都是 unsafe 代码，具体的使用例子可以参考 [lib.rs](https://github.com/wangfenjin/duckdb-rs/blob/main/libduckdb-sys/src/lib.rs) 中的测试代码。但是我们使用 rust 主要是为了他的安全性，rust 希望我们尽量减少 unsafe 的使用。所以一般的 rust 封装都会基于 libxxx-sys 提供一个内存安全的版本，这就是 duckdb-rs 的部分。

### 小试牛刀

还是因为有 rusqlite 的参考，所以花了一些时间终于实现了最初始的版本，并且我已经把这个版本发布到 [crates.io](https://crates.io/crates/duckdb) 上了。这个版本的目标是基于 rusqlite 做最小的改动，并删掉 SQLite 特有的功能，让整个程序跑起来。完成之后效果不错，下面是文档中给的一个使用范例：

```rust
use duckdb::{params, Connection, Result};

#[derive(Debug)]
struct Person {
    id: i32,
    name: String,
    data: Option<Vec<u8>>,
}

fn main() -> Result<()> {
    let conn = Connection::open_in_memory()?;

    conn.execute_batch(
        r"CREATE SEQUENCE seq;
          CREATE TABLE person (
                  id              INTEGER PRIMARY KEY DEFAULT NEXTVAL('seq'),
                  name            TEXT NOT NULL,
                  data            BLOB
                  );
        ")?;

    let me = Person {
        id: 0,
        name: "Steven".to_string(),
        data: None,
    };
    conn.execute(
        "INSERT INTO person (name, data) VALUES (?, ?)",
        params![me.name, me.data],
    )?;

    let mut stmt = conn.prepare("SELECT id, name, data FROM person")?;
    let person_iter = stmt.query_map([], |row| {
        Ok(Person {
            id: row.get(0)?,
            name: row.get(1)?,
            data: row.get(2)?,
        })
    })?;

    for person in person_iter {
        println!("Found person {:?}", person.unwrap());
    }
    Ok(())
}
```

可以看到，接口设计非常优雅，代码也非常符合 rust 的风格，使用上也非常方便。实现过程中发现有些 duckdb 的 C 接口还不支持的部分，我也通过提 issue 或者 PR 去解决了。这里必须要提一点，duckdb 的维护者非常耐心，不管是回答问题还是 review 代码都非常专业。

剩下的问题有一个是之前提到的，duckdb 是静态类型的数据，所以需要支持很多数据类型，这里面工作量不小。另外，因为我之前也有关注 [Apache Arrow](https://arrow.apache.org/)，做过 OLAP 数据库的同学可能知道，Apache Arrow 是一个通用的列式内存格式，方便在内存中做大数据量的计算或者传输，有很多 OLAP 数据引擎都在用。刚好 duckdb 也支持 arrow 格式，所以就想尝试使用 arrow 格式来查询数据，这样至少有两个好处，一个是这样我们就可以暴露 arrow 格式的数据给用户，在使用的时候就可以用上 arrow 生态的其他功能，有可能会产生一些化学反应；另外 arrow 也是有丰富的数据类型和明确的定义，反正我们是要支持很多数据类型的，现在的 C 接口本身也不完善，用 arrow 格式反而更加清晰。

### 通过 Apache Arrow 查询数据

基于上面的考虑，我把目标又看向了 [arrow-rs](https://github.com/apache/arrow-rs)，并给 duckdb 的 C 接口也加上了 [arrow 的功能](https://github.com/duckdb/duckdb/pull/1978)，最终在 duckdb-rs 中实现了通过 Arrow 格式来查询数据，实现参见 [这里](https://github.com/wangfenjin/duckdb-rs/pull/8)。

实现之后，之前通过行来读取数据的接口完全不变，还能直接查询到 Arrow 格式的数据，下面是一个测试的例子：

```rust
fn test_query_arrow_record_batch_large() -> Result<()> {
    let db = Connection::open_in_memory().unwrap();
    db.execute_batch("BEGIN TRANSACTION")?;
    db.execute_batch("CREATE TABLE test(t INTEGER);")?;
    for _ in 0..300 {
        db.execute_batch("INSERT INTO test VALUES (1); INSERT INTO test VALUES (2); INSERT INTO test VALUES (3); INSERT INTO test VALUES (4); INSERT INTO test VALUES (5);")?;
    }
    db.execute_batch("END TRANSACTION")?;
    let rbs = db.query_arrow("select t from test order by t", [])?;
    assert_eq!(rbs.len(), 2);
    assert_eq!(rbs.iter().map(|rb| rb.num_rows()).sum::<usize>(), 1500);
    assert_eq!(
        rbs.iter()
            .map(|rb| rb
                .column(0)
                .as_any()
                .downcast_ref::<Int32Array>()
                .unwrap()
                .iter()
                .map(|i| i.unwrap())
                .sum::<i32>())
            .sum::<i32>(),
        4500
    );
    Ok(())
}
```

可以看到，我们查询到 Arrow 格式的数据之后，还能通过 arrow-rs 中提供的其他能力做进一步的计算，十分方便。

## 总结

本文主要介绍了 duckdb-rs 的设计和实现，笔者之前有一些开发 OLAP 数据的经验，但是对于 rust 算是新手，之前虽然写过一些但是没有深入学习，做这个项目也有一个目的是为了重新学习一下 rust。好在有 rusqlite 作为参考，所以没有碰到特别多语言层面的问题。

希望这篇文章对于其他对 rust 和数据库感兴趣的同学有一些帮助。同时这个库还有很多没解决的问题，比如支持更多的数据类型，支持连接池，支持更快的数据导入接口等等，我已经建了一些 issues，感兴趣的同学可以回复 [issue](https://github.com/wangfenjin/duckdb-rs/issues) 认领，我也会竭力提供需要的帮助，大家一起讨论和学习。

## 参考

* duckdb 的官网：https://duckdb.org/
* duckdb 的代码库：https://github.com/duckdb/duckdb
* SQLite 的 rust 封装，duckdb-rs 也是基于它改的：https://github.com/rusqlite/rusqlite
* duckdb-rs 的代码库：https://github.com/wangfenjin/duckdb-rs
* Apache Arrow 的 rust 实现：https://github.com/apache/arrow-rs