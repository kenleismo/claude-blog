---
layout: ../../layouts/BlogPost.astro
title: "WIP 子查询去关联化实现笔记"
description: "介绍数据库的子查询去关联化规则以及实现细节"
publishDate: "2025-09-10"
tags: ["database", "kundb", "optimizer"]
---

在为 KunDB 实现子查询去关联化算法时主要拜读了 Eric Fu 大佬的[文章](https://ericfu.me/subquery-optimization/)。

## 规则解读

### 规则1

当 E 不包含关联列时，可以直接将 apply 转为对应类型的 join

$$
R \underset{\bowtie}{\mathbf{A}} E = R \bowtie_{\text{true}} E
$$

示例：

```sql
select a, b from t1 where c
    > (select max(c) from t2);
```

可以变为

```sql
select a, b from t1
    left join (select max(c) as maxc from t2) as t2 on true
    where c > maxc;
```
使用 left join 的写法更明确地表达了"获取一次最大值然后与每行比较"的意图，特别是在 t2 为空表时。


### 规则2

过滤条件可以直接作为 join 条件完成去关联化

$$
R \underset{\bowtie}{\mathbf{A}} (\sigma_p E) = R \bowtie_p E
$$

示例：

```sql
select a, b from t1 where
    exists (select 1 from t2 where c=t1.c)
```

可以变为：

```sql
select a, b from t1 semi join t2 on t1.c = t2.c
```


### 规则3


$$
R \underset{\times}{\mathbf{A}} (\sigma_p E) = \sigma_p(R \underset{\times}{\mathbf{A}} E)
$$

对于 cross apply，可以直接将过滤条件提到 cross apply 之后（也即是计划树更上层）

Q：规则3和规则2有何不同？为什么要列成2条规则？

A：规则3比规则2的适用范围更广，当 E 包含关联列时，我们无法直接将 apply 转换为 join，这个时候可以先将 filter 提升上来，在接着应用其他规则继续去关联化。

规则3其实说明了 cross apply 和 filter 满足交换率。这条规则也很容易理解，如下所示：

```
   CrossApply
   /   \
  R   filter(R.id=5 and S.id=10)
       |
       S
```

```
   filter(R.id=5 and S.id=10)
      |
   CrossApply
    /   \
   R     S
```

首先看上面的执行计划：

- 当 R 的 id 不为 5，带到右子树计算的结果会被过滤掉，右子树输出空，由于是 CrossApply，当右子树为空时，R的相应的行也不会输出
- 当 R 的 id 为 5 时，filter 的结果取决与 S.id 是否为 10
    - S.id 为 10， R 能够输出，并且 S.id 为 10
    - S.id 不为 10， R 不能输出

所以最终的结果就是输出 R.id 为 5， S.id 为 10 的结果集合。

在对比下面的执行计划，虽然 CrossApply 会生成很多额外的数据，但是最上面的 filter 也能保证只有
R.id 为 5，S.id 为 10 的结果才会输出。所以这两个计划是等价的。


### 规则4
$$
R \underset{\times}{\mathbf{A}} (\pi_v E) = \pi_{v \cup \text{columns}(R)}(R \underset{\times}{\mathbf{A}} E)
$$

### 规则8
$$
R \underset{\times}{\mathbf{A}} (\mathbf{G}_{A,F} E) = \mathbf{G}{A \cup \text{columns}(R),F}(R \underset{\times}{\mathbf{A}} E)
$$


### 规则9
$$
R \underset{\times}{\mathbf{A}} (\mathbf{G}^1_F E) = \mathbf{G}_{\text{columns}(R),F'}(R \underset{\text{LOJ}}{\mathbf{A}} E)
$$


规则1和规则2是去关联化的基石，当E不含关联列时，可以直接将 apply 转换为相应类型的 join。