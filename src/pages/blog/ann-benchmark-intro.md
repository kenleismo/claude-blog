---
layout: ../../layouts/BlogPost.astro
title: "ann benchmark 介绍"
description: "介绍 ann benchmark"
publishDate: "2025-08-06"
tags: ["ai", "相似度查询", "benchmark"]
---


## ann benchmark 使用

下载 ann benchmark（annb） 代码以后，我们可以使用如下命令运行 ann 测试：

```bash
python run.py
```

run.py 是 ann benchmark 的主脚本， 默认情况下，ann benchmark 会对所有的内置支持的向量库进行测试，这个过程非常长，一般我们可以选取感兴趣的向量库进行测试。

run.py 支持很多命令行参数，可以使用 python run.py --help 进行查看：

| 选项 | 参数类型 | 默认值 | 参数含义 |
|--------|----------|--------|----------|
| --help/-h | 无 | - | 显示帮助信息并退出 |
| --dataset | NAME | glove-100-angular | 指定用于加载训练数据点的数据集名称 |
| --count/-k | COUNT | 10 | 指定要搜索的最近邻数量 |
| --definitions | FOLDER | ann_benchmarks/algorithms | 算法定义的基础目录，算法定义文件位于 'FOLDER/*/config.yml' |
| --algorithm | NAME | None | 仅运行指定名称的算法 |
| --docker-tag | NAME | None | 仅运行特定 Docker 镜像中的算法 |
| --list-algorithms | 无 | False | 打印所有已知算法的名称并退出 |
| --force | 无 | False | 即使算法结果已存在也重新运行算法 |
| --runs | COUNT | 5 | 每个算法实例运行指定次数，仅使用最佳结果 |
| --timeout | TIMEOUT | 7200 | 每个算法运行的超时时间（秒），-1 表示无超时限制 |
| --local | 无 | False | 如果设置，将在本地运行所有内容（同一进程内）而不使用 Docker |
| --batch | 无 | False | 如果设置，算法一次性获得所有查询 |
| --max-n-algorithms | MAX_N_ALGORITHMS | -1 | 要运行的最大算法数量（仅用于测试） |
| --run-disabled | 无 | False | 运行在 algos.yml 中被禁用的算法 |
| --parallelism | PARALLELISM | 1 | 并行运行的 Docker 容器数量 |


在 ann benchmark 中，每一个向量库都被称为一个 algorithm，ann benchmark 支持 50 多种算法，包括 faiss，pgvector，miluvs，elasticsearch，kdtree等等。每个算法都有一个配置文件：config.yml，配置文件说明了如何测试向量库。配置文件中的信息分为两类：
- 用于 ann benchmark 上层，如 run_groups 指定了多少组测试
- 用于算法，如 args 表示算法参数，query_args 表示查询参数

ann benchmark 支持 docker 和 local 两种模式：
- docker模式：启动一个docker 容器，并运行测试，需要算法提供 Dockerfile，并使用 `install.py` 提前构建好镜像
- 本地模式：需要本地配置好算法运行所需要的环境，包括安装好所有的依赖等

### 配置文件说明
每一个算法都会提供一个配置文件 config.yml。这个配置文件非常重要，即使是 annb 的使用者，也需要理解。

```yaml
float:                    # Data type
  any:                   # Distance metric compatibility
  - base_args: ['@metric']  # Constructor arguments
    constructor: ClassName
    disabled: false
    docker_tag: ann-benchmarks-{name}
    module: ann_benchmarks.algorithms.{name}
    name: algorithm-variant
    run_groups:           # Parameter combinations
      group_name:
        arg_groups: [{param1: value1, param2: value2}]
        args: {}
        query_args: [[arg1, arg2, arg3, ...]]
```

在 config.yml 的顶层通常是 float/bit，这表示这一组配置适用于什么类型的数据。在 annb 中每个数据集都有一个 point_type 的属性，默认为 float，在运行 annb 时，只有和数据集兼容的配置才会被加载。举例来说，如一个配置为 float，那么当遇到 point_type 为 bit 的数据集，这个配置就不会激活。

在 config.yml 的次顶层，通常可以看到 any/euclidean/angular 等关键字，这个表示此配置适用于什么距离度量方式。每个数据集都会有数据度量方式属性，只有匹配上的参数才会被激活。

接下来定义的是对于给定的数据类型和度量方式，我们要运行的测试具体是什么：
- base_args：基础参数，该组所有的测试共享，@count, @metric, @dimension 为占位符，会在程序运行时替换成真实值，@count 来自用户指定的参数，@metric 和 @dimension 来自测试数据
- constructor：必填项，算法实现的 python 类
- disabled：是否禁用，禁用的测试默认不运行，可以使用 `--run-disabled` 运行禁用的测试
- docker_tag：必填项，docker模式下使用的 docker 的镜像，如果算法不支持 docker 模式，随便填一个即可
- module：必填项，python 模块
- name：测试项的名称，当我们使用 `--algorithm` 指定参数时，指定的就是这些名称
- run_groups：run groups 定义了若干 run group，在 run group 中指定算法的参数和查询参数。每一组 run group 可能会生成多组测试，此处涉及到 annb 的参数组合生成机制，最简单的情况，一组 run group 对应一组测试



### 参数组合生成
在 run group 中，可以定义 args，arg_groups，query_args，query_arg_groups。args_groups 优于 args，query_arg_groups 优于 query_args，也就是说当 run group 中定义了 `*_groups`，那么相应的 args，query_args 就不会生效。

ann benchmark 使用 _generate_combinations 来生成参数组合：

1. 对于列表输入：生成笛卡尔积组合
- `_generate_combinations([1, [2, 3], 4])`
  - 返回: `[[1, 2, 4], [1, 3, 4]]`
- `_generate_combinations([[1, 2], [3, 4]])`
  - 返回: `[[1, 3], [1, 4], [2, 3], [2, 4]]`

2. 对于字典输入：生成所有键值对组合的字典

- `_generate_combinations({'a': [1, 2], 'b': 3})`
  - 返回: `[{'a': 1, 'b': 3}, {'a': 2, 'b': 3}]`
- `_generate_combinations({'x': [1, 2], 'y': [3, 4]})`
  - 返回: `[{'x': 1, 'y': 3}, {'x': 1, 'y': 4}, {'x': 2, 'y': 3}, {'x': 2, 'y': 4}]`


## ann benchmark 的运行流程
1.处理数据集

从 https://ann-benchmarks.com/{dataset_name}.hdf5 下载数据集到 data 目录下，下载的数据集是 hdf5 格式的文件，包含训练和测试两部分数据，数据集中也包含了向量的维度，距离度量方式。第一次会下载数据，时间依赖网速。下载完成后，后续就无需下载。我们也可以手动下载好数据集放到 data 目录下

2.创建 definitions

definition 是测试的最小单元，一个算法可以在 config.yml 中定义若干 definition。程序会读取所有的算法的配置文件（config.yml)，根据数据集的 point_type 和 distance_metric 得到一个最大的 definitions 的集合。程序会根据用户指定的参数，如 `--algorithm`，`--local`，`--force`等，config.yml 是否禁用等信息对这个最大集合进行过滤，最终获得本次运行所要测试的 definition 集合。

3.启动若干 worker （由 parallelism 指定），每一个worker负责一个 definition，具体每个worker做的事情如下：

1. 根据 definition 初始化算法对象

2. 调用算法的 fit 接口，创建索引，并将数据加载到索引中

3. 每个 definition 运行 run_count（-k 参数指定） 次

4. 根据 --batch 参数决定运行 single_query 或 batch_query

无论是 single 还是 batch，annb 都允许算法实现 prepare 模式，正常情况下 single_query 会调用 query 接口， batch_query 会调用  batch_query 接口。如果算法实现了prepare 机制，single_query 会先调用 prepare_query，然后调用 run_prepared_query 接口，batch_query 则会调用 prepare_batch_query 和 run_batch_query。annb 会优先使用 prepare 模式。
  single_query 会将查询一个个发送给算法，batch_query 会将查询矩阵发送给算法。

## 自定义算法

对于开发人员来说，自定义算法就是实现 BaseANN 的子类，实现一些接口，其中最重要的如下所示：
- fit：加载数据，对于 faiss 来说就是 faiss_add
- query: 相似度搜索，一次执行一个查询
- batch_query: 可选，批量执行 query 的方法，如果算法没有实现，默认会使用 ThreadPool 并发调用 query
- set_query_arguments：设置查询参数，将 config.yml 中的 query_args 传递给算法

## annb 的性能评价体系

TODO
