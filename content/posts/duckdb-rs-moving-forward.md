---
title: "duckdb-rs 即将成为 DuckDB 官方 rust 客户端"
date: "2023-07-26"
author: "Wang Fenjin"
tags: ["duckdb", "apache-arrow", "olap", "database", "rust", "ffi"]
keywords: ["duckdb", "apache-arrow", "olap", "database", "rust", "ffi"]
description: "我将把维护了两年多的开源项目交接给 duckdb.org！"
---

## 背景

[DuckDB](https://duckdb.org/) 是一个 C++ 编写的单机版嵌入式分析型数据库。它刚开源的时候是对标 SQLite 的列存数据库，并提供与 SQLite 一样的易用性，编译成一个头文件和一个 cpp 文件就可以在程序中使用，甚至提供与 SQLite 兼容的接口，因此受到了很多人的[关注](https://news.ycombinator.com/item?id=24531085)。

我很久之前就开始关注 DuckDB，并在 2021-06-07 开始写第一行 [duckdb-rs](https://github.com/wangfenjin/duckdb-rs) 的代码，在 一个多月后写了一篇[博客](https://www.wangfenjin.com/posts/duckdb-rs/)介绍了构建这个库的过程，算是实现了第一个版本。到今天差不多2年的时间，前后发布了[19个版本](https://crates.io/crates/duckdb)，收获了 200 多个star。

最近一年其实还有很多需求和想法去做优化，但是发现自己并没有那么多时间，收到的 issue 也越来越多。经过沟通，我会把这个库转给 [DuckDB](https://github.com/duckdb) 官方来维护，相信 duckdb-rs 一定会发展得越来越好。同时也非常感谢 Mark 和 Hannes 愿意接手这个仓库并把它作为官方的 rust 客户端。

这篇博客总结下我维护的这段时间主要做的事，以及我认为可以改善的点，算是对过去的总结和对未来的憧憬。

## 关键决策

这个库是 duckdb 的 rust 客户端，所以关注这个库的群体首先是认可 duckdb 的用户，其次因为他们是 rust 技术栈。下面我列举一些我认为是让这个库“成功”的一些关键点。

1. 初始版本基于 [rusqlite](https://github.com/rusqlite/rusqlite) 开发。因为我也是一个 rust 初学者，之前只拿 rust 做过一个项目，这是第二次使用 rust。基于 rustqlite 这样一个成熟的仓库做改造，能让我很快得到一个可用的版本，快速建立信心；另外整个程序的组织，API 的设计都已经经过了验证，不容易走弯路；整体的代码质量也能有基本保障。
2. 基于 [arrow](https://github.com/apache/arrow-rs) 格式来交换数据。arrow 现在基本上算是列存储的数据交换标准，在很多开源项目中都有使用，duckdb 对 arrow 的支持也比较完善。虽然 duckdb 有自己的原生 C 接口，但是基于 arrow 格式来做数据交换，能让 rust 和 c-api 调用相对稳定，不会因为 duckdb 迭代导致 C 接口的变更，我们也需要一直变更，一定程度上减轻了维护的工作量，也减少了接口变更对用户的影响。
3. 完善的 CI 流程，我认为所有的开源项目都应该要做到这一点。因为继承自 rusqlite，这个库从一开始就有 CI 流程，能保证合并到 master 的代码是没问题的，并且 CI 里面还有关于内存泄漏的检测，避免了 ffi 带了的可能不安全的问题。发布过程也是自动化的，只要打个 tag 就自动发布到 crate。CI 的机制保障了任何感兴趣的人都可以提交 MR 并得到检验，也保证自己如果长时间不维护了不至于都不知道从哪里开始改。

## 几个 MR

下面我挑选几个我认为比较关键的，并且不是我贡献的 MR：

1. [Add github workflow](https://github.com/wangfenjin/duckdb-rs/pull/1)，之前我都是直接 push master，这是第一个 MR 添加 CI 检测，非常有意义！
2. [add r2d2 connection pool](https://github.com/wangfenjin/duckdb-rs/pull/32)，添加连接池。
3. [Rework bundled compilation to support included extensions](https://github.com/wangfenjin/duckdb-rs/pull/127)，收到最大的一个 MR，为了支持 extension，重做了 bundle duckdb 源码的逻辑，让这个仓库也能打包进去各种扩展而不用额外安装。
4. [Feat: Develop query polars](https://github.com/wangfenjin/duckdb-rs/pull/169)，支持把 query 的结果转成 polars 的数据结构，[polars](https://github.com/pola-rs/polars) 是目前 rust 写的一个非常流行的数据处理工具，这个功能打通了 duckdb 和 polars。

我自己除了日常维护之外，实际上大的功能开发比较少，[Support table function](https://github.com/wangfenjin/duckdb-rs/pull/138) 算是一个，并且我认为基于 rust 写扩展远比基于 c/c++ 来写更简单，更安全！

## 遗留问题

因为精力有限，这个库还有一些问题需要解决：

1. 更好的文档。因为我的英语也是半路出家，所以写文档一直是想起来就头疼的问题。这个库因为是基于 rusqlite，所以继承了一部分文档，所以基本质量还在，但是后续缺少维护，特别是针对 duckdb 特性的一些文档资料比较少。好的文档和博客也是开源项目成功的关键。
2. 支持更多数据类型。这里的数据类型分两类，一类是对于结果，映射到 rust 的数据类型，这部分的需求倒是不高优，特别是用户如果是使用的 arrow 数据的话，arrow-rs 本身有完整的数据类型；另一类是查询参数，这部分需要支持更多的数据类型绑定，方便用户使用。目前我们只支持了一些基础的数据类型。
3. 更完善的数据插入支持。列存数据库需要有批量插入数据的能力，比如 duckdb 自带的 append 接口，或者支持插入 arrow 的数据等，目前这一块支持得不太好。
4. 编译过程优化。随着 duckdb 功能丰富，这个库的编译也越来越慢，对资源的消耗也越来越多，编译的产物也越来越大。
5. 一些 duckdb 或者列存特定的接口支持，比如 streaming query 或者 relation api，这些都有人提过 issue。

从文档和接口上，要达到和 duckdb 一样的水准，还有不少工作要做。

## 后续计划

这篇文章发布的时候，意味着我不再是 duckdb-rs 的维护者。但是这不代表着后续我不再给 duckdb-rs 贡献代码，我还是会继续关注 duckdb 和 duckdb-rs，并且在闲暇的时候贡献一些代码。

如果有时间还可以基于 duckdb-rs 做一些其他的项目，比如用 rust 给 duckdb 做一个向量数据库的扩展，或者基于 duckdb-rs 搭建一个存储的 server，主要是基于 [arrow-flight](https://github.com/apache/arrow-rs/tree/master/arrow-flight) 协议，如果再有时间还可以加上 raft 支持分布式。不知道有什么用，但是感觉是个很好玩的项目。也可以考虑做一个分布式数据处理的工具，用 duckdb 做中间数据的存储或者计算加速等。
