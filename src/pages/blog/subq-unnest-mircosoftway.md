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

规则3和规则2有何不同？为什么要列成2条规则？




规则4:
$$
R \underset{\times}{\mathbf{A}} (\pi_v E) = \pi_{v \cup \text{columns}(R)}(R \underset{\times}{\mathbf{A}} E)
$$

规则8:
$$
R \underset{\times}{\mathbf{A}} (\mathbf{G}_{A,F} E) = \mathbf{G}{A \cup \text{columns}(R),F}(R \underset{\times}{\mathbf{A}} E)
$$


规则9:
$$
R \underset{\times}{\mathbf{A}} (\mathbf{G}^1_F E) = \mathbf{G}_{\text{columns}(R),F'}(R \underset{\text{LOJ}}{\mathbf{A}} E)
$$


规则1和规则2是去关联化的基石，当E不含关联列时，可以直接将 apply 转换为相应类型的 join。