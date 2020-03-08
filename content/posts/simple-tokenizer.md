---
title: "Simple: 一个支持中文和拼音搜索的 sqlite fts5插件"
date: "2020-03-08"
author: "Wang Fenjin"
tags: ["sqlite3", "fst5", "search", "cpp"]
keywords: ["sqlite3", "fst5", "search", "cpp"]
description: "实现一个支持中文和拼音的全文搜索插件"
---

之前的工作关系，需要在手机上支持中文和拼音搜索。由于手机上存储数据一般都是用 sqlite，所以是基于 sqlite3 fts5 来实现。这段时间再次入门 c++，所以想用 c++ 实现一下，一来用于练手，二来当时做的时候发现网络上这方面开源的实现不多，也造福下其他人。

## 背景

搜索现在几乎是每个 APP 必备的功能，用户已经习惯了搜索框搜一下，避免到处去找。搜索也是帮助用户查找旧信息，发现新功能的一个重要手段。平常我们用微信的时候经常会搜索联系人和聊天记录，发现微信这一块做的还是非常好的。关于微信的全文搜索，可以看看这两篇文章：[微信全文搜索优化之路](https://juejin.im/entry/59e6cd266fb9a0451968ab02) 和 [微信移动端的全文检索多音字问题解决方案](https://cloud.tencent.com/developer/article/1198371) 。

第一篇文章主要是问题和原理的概述，第二篇文章是核心分词器的实现。我写的这个项目主要是实现了 simple 分词器，并提供一些辅助函数帮助使用。

## Simple 分词器

搜索的核心是建倒排索引，建索引的核心是分词器。 跟名字一下，Simple 分词器的规则非常简单：

1. 空白符跳过
2. 连续的数字作为整体是一个索引
3. 连续的英文字母作为整体并转换成小写索引
4. 中文字单独建索引，并且把中文字转成拼音后也建搜索，这样就能同时支持中文和拼音检索。另外把拼音首字母也建索引，这样搜索 zjl 就能命中 "周杰伦"。
5. 其他字符统一单独建索引，这样搜索 😊 也能搜到

上面的 5 条都比较好理解，关于中文为什么这么做（而不是连续的中文一起建索引），是由于客户端搜索的需求决定的。具体可以参考上面微信的两篇文章。

有了上面的规则，代码写起来就很简单了，[核心逻辑](https://github.com/wangfenjin/simple/blob/a9234eb7169d98522ff07f42e0e9f9aa603bbebd/src/simple_tokenizer.cc#L104) 30 行就解决了。这块代码运行效率也比较高，一遍扫描 O(n) 的复杂度就完成了分词操作。

## query 拆分

索引建好之后，query 需要根据分词规则来写才能查询到数据。比如根据上面的逻辑：

1. 如果查数字，我们要把搜索词当作前缀来用，比如用户搜索 123， query 就需要换成 123*，这样如果索引里面有 12345 也能被搜索出来
2. 对于英文，除了要当作前缀，还需要把搜索词转成小写，比如用护搜索 Hello，query 就需要换成 hello*, 这样如果索引里面有 HelloWorld 也能被命中
3. 对于中文和其他字符，都要拆成单个的才能命中索引
4. 最后对于拼音（其实我们没办法区分英文和拼音，统一当作拼音处理就行），需要把拼音按照规则拆分，因为我们的拼音索引是单字建立的。这样如果用户搜索 "zhangliangy"，拼音就可以被拆成 'zhang AND liang AND y*'，从而命中"张靓颖"。具体规则微信的文章中也有详述。

可以看到 query 词重构的逻辑也比较多，在之前的项目中没有好的办法，所以是自己在应用层代码里面组装好了 query 再给 sqlite 去搜的，这样其实不太方便。在这个项目中，我实现了一个 simple_query 的字符串函数，输入一个 string，它会给转换成组装好的搜索词，用法跟使用 sqlite 内置函数一样，这样就方便很多了，下面是一个例子：

```sql
-- 完整例子：https://github.com/wangfenjin/simple/blob/master/test.sql

-- load so file
.load libsimple.so

-- set tokenize to simple
CREATE VIRTUAL TABLE t1 USING fts5(x, tokenize = "simple");

-- add some values into the table
insert into t1 values ("周杰伦 Jay Chou:最美的不是下雨天，是曾与你躲过雨的屋檐"),

-- query result: [周杰伦] Jay Chou:最美的不是下雨天，是曾与你躲过雨的屋檐
select simple_highlight(t1, 0, '[', ']') from t1 where x match simple_query('zhoujiel');
```

可以看到， match 后面用 simple_query 这个函数，传入用户输入的搜索词就可以用了。

另外 sql 中还有一个 simple_highlight 函数，它的作用和内置的 highlight 函数一样，只是它会把连续命中的词一起高亮。比如对于文档"周杰伦"，如果搜索词是 'zhou AND jie'，那么 highlight 函数会返回 "[周][杰]伦"，simple_highlight 会返回 "[周杰]伦"。

## 总结

最后说几句关于 sqlite fts5 的使用的问题。个人建议通过 trigger 的方式来维护索引的这张表，具体使用的方式可以在官方文章中搜索 trigger 找到例子。这样使用的好处是没有复杂的逻辑去保证文档数据和索引数据一致，微信的文章中很大一部分复杂度在描述怎么保证数据一致的问题。他们可能有自己的业务复杂性，但是对于一般的场景来说， trigger 是最好的方式。

从这个项目我们能学到：

1. 怎么给 sqlite3 做一个支持中文和拼音的 fts5 拓展
2. 怎么给 sqlite3 添加用户自定义的函数
3. 在一个项目中同时使用 c 和 c++ ，并合理处理边界问题

大家可以下载使用，也可以根据自己的需求去改进，定制更多的函数和策略。

## Reference

- Simple 分词器: https://github.com/wangfenjin/simple
- sqlite 官方文档：https://www.sqlite.org/fts5.html
- 微信全文搜索优化之路：https://juejin.im/entry/59e6cd266fb9a0451968ab02
- 微信移动端的全文检索多音字问题解决方案：https://cloud.tencent.com/developer/article/1198371
