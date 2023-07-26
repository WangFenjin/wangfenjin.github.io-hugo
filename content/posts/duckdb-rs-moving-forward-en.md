---
title: "I Am No Longer the Maintainer of duckdb-rs"
date: "2023-07-26"
author: "Wang Fenjin"
tags: ["duckdb", "apache-arrow", "olap", "database", "rust", "ffi"]
keywords: ["duckdb", "apache-arrow", "olap", "database", "rust", "ffi"]
description: "I will step down as the maintainer of duckdb-rs!"
---

## Background

[DuckDB](https://duckdb.org/) is an in-process SQL OLAP database management system implemented in C++. When it was first open-sourced, it was positioned as a columnar database comparable to SQLite, providing the same ease of use. With just a header file and a cpp file, it could be easily embeded in any program, even offering a SQLite-compatible interface, which caught the attention of many people [source](https://news.ycombinator.com/item?id=24531085).

I started paying attention to DuckDB a long time ago and began writing the first line of [duckdb-rs](https://github.com/wangfenjin/duckdb-rs) code on June 7, 2021. About a month later, I wrote a [blog post](https://www.wangfenjin.com/posts/duckdb-rs/) introducing the process of building this library, marking the completion of the initial version. Over the past two years, I have released approximately [19 versions](https://crates.io/crates/duckdb), get more than 200 stars in GitHub.

In the past year, there have been many requirements and ideas for optimization, but I found myself lacking the time, and the number of received issues has been increasing. As a result, I will transfer this library to [DuckDB](https://github.com/duckdb) offical organization, believing that make duckdb-rs an official client will lead to further progress and bigger success. Also I'd like to take this chance to thanks Mark and Hannes for building DuckDB and agree to accept duckdb-rs as the official rust client.

This blog post summarizes the main tasks I have undertaken during my maintenance period and points out areas that I believe can be improved.

## Key Decisions

This library is the Rust client of DuckDB, so the primary audience interested in this library are users who appreciate DuckDB and use the Rust tech stack. Below are some key factors that I consider contributed to the "success" of this library:

1. Initial version based on [rusqlite](https://github.com/rusqlite/rusqlite) development. As a Rust beginner myself, I had previously only worked on one Rust project, and this was my second time using Rust. Based on rusqlite, a mature repository, allowed me to quickly obtain a usable version. Additionally, the code structure and API design had already been validated, reducing the likelihood of taking wrong turns. Moreover, the overall code quality could be reasonably assured.

2. Data exchange based on the [arrow](https://github.com/apache/arrow-rs) format. Arrow is now considered the columnar storage data exchange standard and is used in many open-source projects. DuckDB has good support for arrow as well. Although DuckDB has its native C interface, using the arrow format for data exchange allows relatively stable interactions between Rust and the C API. This approach ensures that we won't need to make frequent changes due to DuckDB iterations, thus reducing maintenance efforts and minimizing the impact of interface changes on users.

3. Robust CI process. I believe that all open-source projects should strive for this. With a CI process, we can ensure that the code merged into the master branch was error-free. The CI process also included memory leak detection, avoiding potential safety issues introduced by the FFI. The release process was automated as well, with crates being automatically published by tagging.

## Notable MRs

I've selected a few MRs that I consider significant and that weren't contributed by me:

1. [Add github workflow](https://github.com/wangfenjin/duckdb-rs/pull/1): This was the first MR to add CI checks, which was very meaningful as I used to push directly to master before CI was in place.

2. [Add r2d2 connection pool](https://github.com/wangfenjin/duckdb-rs/pull/32): This MR added a connection pool, improving the library's performance.

3. [Rework bundled compilation to support included extensions](https://github.com/wangfenjin/duckdb-rs/pull/127): This was the largest MR and allowed the library to support extensions, reworking the logic of bundling DuckDB source code to include various extensions without requiring additional installations.

4. [Feat: Develop query polars](https://github.com/wangfenjin/duckdb-rs/pull/169): This MR added support for converting query results into polars data structures. [polars](https://github.com/pola-rs/polars) is a popular data processing tool written in Rust. This feature bridged the gap between DuckDB and polars.

Apart from the daily maintenance, I didn't contribute to the development of major features significantly. [Support table function](https://github.com/wangfenjin/duckdb-rs/pull/138) can be considered one, and I believe writing extensions in Rust is simpler and safer compared to C/C++.

## Outstanding Issues

Due to limited time and resources, there are still some unresolved issues in this library:

1. Better documentation. Writing documentation has always been a headache for me since English is not my strong suit. While this library inherited some documentation from rusqlite, it lacks ongoing maintenance, especially regarding documentation specific to DuckDB features. Good documentation and blog posts are key to the success of an open-source project.

2. Support for more data types. There are two categories of data types: those mapped to Rust data types for results, which are not a high priority since arrow-rs already provides comprehensive data types for users working with arrow data. The other category is query parameters, where we need to support a wider range of data types for better user convenience. Currently, we only support some basic query data types.

3. Improved support for data insertion. Columnar databases require the ability to insert data in batches, such as using DuckDB's built-in append interface or supporting insertion of arrow data.

4. Compilation process optimization. As DuckDB's features expand, the compilation process for this library has become slower and resource-consuming, resulting in larger build artifacts.

5. Support for specific DuckDB interfaces, such as streaming query or relation API. These have been raised as issues by some users.

To achieve the same level as DuckDB in terms of documentation and interfaces, there is still much work to be done.

## Future Plans

With the publication of this article, it means I am no longer the maintainer of duckdb-rs. However, this does not mean that I will no longer contribute code to duckdb-rs. I will continue to follow DuckDB and duckdb-rs and contribute code in my spare time.

If I have time, I may also work on other projects based on duckdb-rs, such as:

* Creating a Rust extension for DuckDB to become a vector database
* Or building a storage server based on duckdb-rs primarily using the [arrow-flight](https://github.com/apache/arrow-rs/tree/master/arrow-flight) protocol. If I have even more time, I might add support for Raft to enable distribution. I'm not sure how useful these projects would be, but they sound like fun.
* Another possibility is creating a distributed data processing tool, using DuckDB as intermediate data storage or for computation acceleration.


---
> Translated from [CN Version](https://www.wangfenjin.com/posts/duckdb-rs-moving-forward/) using ChatGPT and polished manually.