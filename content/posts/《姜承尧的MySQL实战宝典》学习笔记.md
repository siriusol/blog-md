---
title: "《姜承尧的MySQL实战宝典》学习笔记"
subtitle: ""
date: 2022-08-07T18:18:27+08:00
draft: false
author: ""
authorLink: ""
authorEmail: ""
description: ""
keywords: ""
license: ""
comment: false
weight: 0

tags:
- MySQL
categories:
- 学习笔记

hiddenFromHomePage: false
hiddenFromSearch: false

summary: ""
resources:
- name: featured-image
  src: featured-image.jpg
- name: featured-image-preview
  src: featured-image-preview.jpg

toc:
  enable: true
math:
  enable: false
lightgallery: false
seo:
  images: []

# See details front matter: /theme-documentation-content/#front-matter
---

## 1. 表结构设计篇-数字类型

### 整型类型

MySQL 数据库支持 SQL 标准支持的整型类型：INT、SMALLINT。此外，MySQL 数据库也支持诸如 TINYINT、MEDIUMINT 和 BIGINT 整型类型（表 1 显示了各种整型所占用的存储空间及取值范围）：

| 类型      | 占用空间（字节） | 最小值~最大值 [signed]                      | 最小值~最大值 [unsigned]  |
| --------- | ---------------- | ------------------------------------------- | ------------------------- |
| TINYINT   | 1                | [-128, 127]                                 | [0, 255]                  |
| SMALLINT  | 2                | [-32768, 32767]                             | [0, 65535]                |
| MEDIUMINT | 3                | [-8388608, 8388607]                         | [0, 16777215]             |
| INT       | 4                | [-2147483648, 2147483647]                   | [0, 4294967295]           |
| BIGINT    | 8                | [-9223372036854775808, 9223372036854775807] | [0, 18446744073709551615] |

<center>表 1：各 INT 类型的取值范围</center>

在整型类型中，有 signed 和 unsigned 属性，其表示的是整型的取值范围，默认为 signed。在设计时不建议刻意去用 unsigned 属性，因为在做一些数据分析时，SQL 可能返回的结果并不是想要得到的结果。

### 整型类型与自增设计

整型类型最常见的就是在业务中用来表示某件物品的数量。例如上述表的销售数量，或电商中的库存数量、购买次数等。另一个常见且重要的用法是作为表的主键，即用来唯一标识一行数据。

整型结合属性 auto_increment，可以实现自增功能，但在表结构设计时用自增做主键，要注意以下两点：

* 用 BIGINT 做主键，而不是 INT；

* 自增值并不持久化，可能会有回溯现象（MySQL 8.0 版本前）。

从表 1 可以发现，INT 的范围最大在 42 亿的级别，在真实的互联网业务场景的应用中，很容易达到最大值。例如一些流水表、日志表，每天 1000W 数据量，420 天后，INT 类型的上限即可达到。

因此，（敲黑板 1）用自增整型做主键，一律使用 BIGINT，而不是 INT。不要为了节省 4 个字节使用 INT，当达到上限时，再进行表结构的变更，将是巨大的负担与痛苦。

当达到 INT 上限后，再次进行自增插入时，会报重复错误。

其实，在海量互联网架构设计过程中，为了之后更好的分布式架构扩展性，不建议使用整型类型做主键，更为推荐的是字符串类型（这部分内容将在 05 节中详细介绍）。

### 浮点类型和高精度型

MySQL 之前的版本中存在浮点类型 Float 和 Double，但这些类型因为不是高精度，也不是 SQL 标准的类型，所以在真实的生产环境中不推荐使用，否则在计算时，由于精度类型问题，会导致最终的计算结果出错。

更重要的是，从 MySQL 8.0.17 版本开始，当创建表用到类型 Float 或 Double 时，会抛出下面的警告：MySQL 提醒用户不该用上述浮点类型，甚至提醒将在之后版本中废弃浮点类型。

```mysql
Specifying number of digits for floating point data types is deprecated and will be removed in a future release
```

而数字类型中的高精度 DECIMAL 类型可以使用，当声明该类型列时，可以（并且通常必须要）指定精度和标度，例如：

```mysql
salary DECIMAL(8,2)
```

其中，8 是精度（精度表示保存值的主要位数），2 是标度（标度表示小数点后面保存的位数）。通常在表结构设计中，类型 DECIMAL 可以用来表示用户的工资、账户的余额等精确到小数点后 2 位的业务。

但是金额字段的设计并不推荐使用 DECIMAL 类型，而更推荐使用 INT 整型类型。

### 资金字段设计

在海量互联网业务的设计标准中，并不推荐用 DECIMAL 类型，而是更推荐将 DECIMAL 转化为整型类型。资金类型更推荐使用分作为单位来存储，而不是用元单位存储。如 1 元在数据库中用整型类型 100 存储。

金额字段的取值范围如果用 DECIMAL 表示的，如何定义长度呢？因为类型 DECIMAL 是个变长字段，若要定义金额字段，则定义为 DECIMAL(8,2) 是远远不够的。这样只能表示存储最大值为 999999.99，百万级的资金存储。用户的金额至少要存储百亿的字段，而统计局的 GDP 金额字段则可能达到数十万亿级别。用类型 DECIMAL 定义，不好统一。

另外重要的是，类型 DECIMAL 是通过二进制实现的一种编码方式，计算效率远不如整型来的高效。因此，推荐使用 BIG INT 来存储金额相关的字段。

字段存储时采用分存储，即便这样 BIG INT 也能存储千兆级别的金额。这里，1 兆 = 1 万亿。

这样的好处是，所有金额相关字段都是定长字段，占用 8 个字节，存储高效。另一点，直接通过整型计算，效率更高。

注意，在数据库设计中，我们非常强调定长存储，因为定长存储的性能更好。

账户余额字段，设计是用整型类型，而不是 DECIMAL 类型，这样性能更好，存储更紧凑。

## 2. 字符串类型

MySQL 数据库的字符串类型有 CHAR、VARCHAR、BINARY、BLOB、TEXT、ENUM、SET。不同的类型在业务设计、数据库性能方面的表现完全不同，其中最常使用的是 CHAR、VARCHAR。

### CHAR 和 VARCHAR

CHAR(N) 用来保存固定长度的字符，N 的范围是 0 ~ 255。**注意，N 表示的是字符，而不是字节**。VARCHAR(N) 用来保存变长字符，N 的范围为 0 ~ 65536， N 表示字符。

在超出 65536 个字符的情况下，可以考虑使用更大的字符类型 TEXT 或 BLOB，两者最大存储长度为 4G，其区别是 BLOB 没有字符集属性，纯属二进制存储。

和 Oracle、Microsoft SQL Server 等传统关系型数据库不同的是，MySQL 数据库的 VARCHAR 字符类型最大能够存储 65536 个字符，所以在 MySQL 数据库下，绝大部分场景使用类型 VARCHAR 就足够了。

### 字符集

在表结构设计中，除了将列定义为 CHAR 和 VARCHAR 用以存储字符以外，还需要额外定义字符对应的字符集，因为每种字符在不同字符集编码下，对应着不同的二进制值。常见的字符集有 GBK、UTF8，通常推荐把默认字符集设置为 UTF8。

推荐把 MySQL 的默认字符集设置为 UTF8MB4，否则某些 emoji 表情字符无法在 UTF8 字符集下存储，比如 emoji 笑脸表情，对应的字符编码为 0xF09F988E（4 字节）。

若强行在字符集为 UTF8 的列上插入 emoji 表情字符， MySQL 会抛出如下错误信息：

```mysql
ERROR 1366 (HY000): Incorrect string value: '\xF0\x9F\x98\x8E' for column 'a' at row 1
```

包括 MySQL 8.0 版本在内，字符集默认为 UTF8MB4，8.0 版本之前默认的字符集为 Latin1。因为不同版本默认字符集的不同，你要显式地在配置文件中进行相关参数的配置：

```mysql
[mysqld]
character-set-server = utf8mb4
```

