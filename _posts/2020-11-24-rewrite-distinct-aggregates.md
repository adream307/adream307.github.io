---
title:  "将 distinct agg 改写成 非 distinct agg"
author: adream307
date:   2020-11-24 16:10:00 +0800
categories: [spark,agg]
tags: [agg, distinct]
---

如果`Aggregate`操作中同时包含`Distinct`与非`Distinct`操作，优化器可以将该操作改写成两个不包含`Distinct`的`Aggregate`
假设`schema`如下
```sql
create table animal(gkey varchar(128), 
                    cat varchar(128), 
                    dog varchar(128), 
                    price double);
```
`animal`表中的数据如下

| gkey | cat | dog | price |
|:-----|:----|:----|:------|
| a    | ca1 | cb1 | 10    |
| a    | ca1 | cb2 | 5     |
| b    | ca1 | cb1 | 13    |

 测试语句如下
 ```sql
SELECT 
	gkey, SUM(price), 
	COUNT(DISTINCT cat),
	COUNT(DISTINCT dog)
FROM 
	animal
GROUP BY
	gkey
 ```
该测试语句拥有3个`aggregate`，其中两个包含`distinct`，优化策略如下
首先将`animal`表格的每行扩展成`3`行，并添加新的一列`grid`，类型为整形，记新的表为`animal2`

|gkey | cat  | dog  | price | grid |
|:----|:-----|:-----|:------|:-----|
|$gkey|`null`|`null`|$price |0     |
|$gkey|$cat  |`null`|`null` |1     |
|$gkey|`null`|$dog  |`null` |2     |

表`animal2`数据如下

|gkey |   cat  |   dog  | price  | grid |
|:----|:-------|:-------|:-------|:-----|
|a    | `null` | `null` | 10     | 0    |
|a    | ca1    | `null` | `null` | 1    |
|a    | `null` | cb1    | `null` | 2    |
|a    | `null` | `null` | 5      | 0    |
|a    | ca1    | `null` | `null` | 1    |
|a    | `null` | cb2    | `null` | 2    |
|b    | `null` | `null` | 13     | 0    |
|b    | ca1    | `null` | `null` | 1    |
|b    | `null` | cb1 | `null`    | 2    |

在`animal2`上按照`gkey,cat,dog,grid`做`group by`创建视图
```sql
CREATE VIEW v_animal(v_gkey, v_cat, v_dog, v_grid, v_total ) AS
SELECT
	gkey, cat, dog, grid
	SUM(price) as total
FROM
	animal2
GROUP BY
	gkey, cat, dog, grid
```
然后在`v_animal`上再一次做`aggregate`操作，语句如下
```sql
SELECT
	v_gkey AS gkey
	SUM(CASE
			WHEN grid==0 THEN v_total
			ELSE null
		END) AS total
	COUNT(CASE
			WHEN grid==1 THEN v_cat
			ELSE null
		END) AS cat_total
	COUNT(CASE
			WHEN grid==2 THEN v_dog
			ELSE null
		END) AS dog_total
FROM
	v_animal
GROUP BY
	v_gkey
```
参考资料:
[RewriteDistinctAggregates.scala](https://github.com/apache/spark/blob/master/sql/catalyst/src/main/scala/org/apache/spark/sql/catalyst/optimizer/RewriteDistinctAggregates.scala)


