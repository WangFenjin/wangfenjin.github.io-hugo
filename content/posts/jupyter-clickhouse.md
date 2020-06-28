---
title: "xeus-clickhouse: Jupyter 的 ClickHouse 内核"
date: "2020-06-28"
author: "Wang Fenjin"
tags: ["clickhouse", "jupyter", "jupyter-kernal"]
keywords: ["clickhouse", "jupyter", "jupyter-kernal"]
description: "支持直接在 jupyter 中用 clickhouse sql 查询数据"
---

在科学计算领域，Jupyter 是一个使用非常广泛的集成开发环境，它支持多种主流的编程语言比如 Python, C++, R 或者 Julia。同时，数据科学最重要的还是数据，而 SQL 是操作数据最直观的语言。前段时间看到[一篇文章](https://blog.jupyter.org/a-jupyter-kernel-for-sqlite-9549c5dcf551)，有人给 sqlite 做了一个 jupyter 的内核，感觉很有意思。所以我尝试给 ClickHouse 做了一个 jupyter 的内核，目前已经有了一个可以试用的版本，下面做一个简单介绍。

## 现状

新内核允许用户用 ClickHouse SQL 的语法直接操作远程 CH 数据库，通过一些扩展操作比如 `%CONNECT` 支持与 ch cli 一样的连接参数，后续也有计划使用 jupyter magics 支持更多的数据可视化操作。

项目参考了 jupyter sqlite 内核的实现方式，是基于 [xeus](https://github.com/jupyter-xeus/xeus) 框架来实现的。xeus 是一个 c++ 的 lib 库，它对 jupyter 的内核做了很好的封装，我们只需要专注于内核相关的功能就可以了。目前对于 ch 的操作基于 clickhouse-cpp 来实现，它是 ch 的 cpp 客户端。

![ch-sql](/img/ch-sql.gif)

目前实现处于早期阶段，但是基础功能已经可用。它支持了几乎 CH 所有 SQL 语法，具体例子可以参考 [clickhouse.ipynb](https://github.com/wangfenjin/xeus-clickhouse/blob/master/examples/clickhouse.ipynb)。xeus-clickhouse 在 jupyter notebook 和 jupyter lab 中以 HTML 表格的形式展示数据；在 jupyter console 中，我们使用 tabulate 库只做纯文本的表格。

## 未来

对于 xeus-clickhouse 未来的规划是，先打磨好稳定性，目前已知的还有一个非法字符导致内核崩溃的问题，已经提交 issue 给 xeus 仓库；另外clickhouse-cpp 不支持 ssl 连接。除了基础功能的打磨，还计划通过支持更多的 jupyter magic 来实现数据的可视化渲染，提供更方便的数据可视化能力。

## 使用

我制作了一个 Docker 镜像发布在 [docker-hub](https://hub.docker.com/r/wangfenjin/xeus-clickhouse)，不需要安装任何环境就可以试用：

```bash
# start jupyter with clickhouse kernal
docker run -p 8888:8888 wangfenjin/xeus-clickhouse:v0.1.0

# start a local clickhouse for testing
docker run -d --name jupyter-clickhouse-server -p 8123:8123 --ulimit nofile=262144:262144 yandex/clickhouse-server

# open the example/clickhouse.ipynb and connect to local server by 
# %CONNECT --host host.docker.internal --port 8123
```

在 docker 里面连接另外一个 docker 中的 ch 可能会有问题，感觉是目前 clickhouse-cpp 对于网络的处理不太完善。感兴趣的同学也可以下载代码自己编译，具体的编译流程见 [github](https://github.com/wangfenjin/xeus-clickhouse) 仓库。欢迎大家试用！

