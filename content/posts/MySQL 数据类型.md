---
title: "MySQL 数据类型"
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

## 1. 数字类型

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

在整型类型中，有 signed 和 unsigned 属性，其表示的是整型的取值范围，默认为 signed。在设计时不建议刻意去用 unsigned 属性，因为在做一些数据分析时，SQL 可能返回的结果并不是想要得到的结果。例如 MySQL 要求 unsigned 数值相减之后依然为 unsigned，否则就会报错。为了避免这个错误，需要对数据库参数 sql_mode 设置为 NO_UNSIGNED_SUBTRACTION，允许相减的结果为 signed。

### 整型类型与自增设计

整型类型最常见的就是在业务中用来表示某件物品的数量。例如上述表的销售数量，或电商中的库存数量、购买次数等。另一个常见且重要的用法是作为表的主键，即用来唯一标识一行数据。

整型结合属性 auto_increment，可以实现自增功能，但在表结构设计时用自增做主键，要注意以下两点：

* 用 BIGINT 做主键，而不是 INT；

* 自增值并不持久化，可能会有回溯现象（MySQL 8.0 版本前）。

从表 1 可以发现，INT 的范围最大在 42 亿的级别，在真实的互联网业务场景的应用中，很容易达到最大值。例如一些流水表、日志表，每天 1000W 数据量，420 天后，INT 类型的上限即可达到。因此用自增整型做主键，一律使用 BIGINT，而不是 INT。不要为了节省 4 个字节使用 INT，当达到上限时，再进行表结构的变更，将是巨大的负担与痛苦。

当达到 INT 上限后，再次进行自增插入时，会报重复错误。

在 MySQL 8.0 前，例如删除自增为 3 的记录后，下一个自增值为 4（AUTO_INCREMENT=4）。但若这时数据库发生重启，则启动后表 t 的自增起始值将再次变为 3，即自增值发生回溯。8.0 版本将每张表的自增值进行了持久化，不会出现这种问题。

其实，在海量互联网架构设计过程中，为了之后更好的分布式架构扩展性，不建议使用整型类型做主键，更为推荐的是字符串类型。

### 浮点类型和高精度型

MySQL 之前的版本中存在浮点类型 Float 和 Double，但这些类型因为不是高精度，也不是 SQL 标准的类型，所以在真实的生产环境中不推荐使用，否则在计算时，由于精度类型问题，会导致最终的计算结果出错。

更重要的是，从 MySQL 8.0.17 版本开始，当创建表用到类型 Float 或 Double 时，会抛出下面的警告：MySQL 提醒用户不该用上述浮点类型，甚至提醒将在之后版本中废弃浮点类型。

```mysql
Specifying number of digits for floating point data types is deprecated and will be removed in a future release
```

而数字类型中的高精度 DECIMAL 类型可以使用，当声明该类型时，可以（并且通常必须要）指定精度和标度，例如：

```mysql
salary DECIMAL(8,2)
```

其中，8 是精度（精度表示保存值的主要位数），2 是标度（标度表示小数点后面保存的位数）。通常在表结构设计中，类型 DECIMAL 可以用来表示用户的工资、账户的余额等精确到小数点后 2 位的业务字段。

### 资金字段设计

资金字段并不推荐用 DECIMAL 类型，而是更推荐将 DECIMAL 转化为整型。使用分作为单位来存储，而不是用元单位存储。如 1 元在数据库中用整型类型 100 存储。

用 DECIMAL 存储金额有以下的缺点：

金额字段的取值范围如果用 DECIMAL 表示的，如何定义长度呢？因为类型 DECIMAL 是个变长字段，若要定义金额字段，则定义为 DECIMAL(8,2) 是远远不够的。这样只能表示存储最大值为 999999.99，百万级的资金存储。用户的金额至少要存储百亿的字段，而统计局的 GDP 金额字段则可能达到数十万亿级别。用类型 DECIMAL 定义，不好统一。

另外重要的是，类型 DECIMAL 是通过二进制实现的一种编码方式，计算效率远不如整型高效。因此推荐使用 BIGINT 来存储金额相关的字段。

字段存储时采用分存储，即便这样 BIGINT 也能存储千万亿级别的金额。

存储为 BIGINT 的另一个好处是所有金额相关字段都是定长字段，占用 8 个字节，存储高效

注意，在数据库设计中，我们非常强调定长存储，因为定长存储的性能更好。

## 2. 字符串类型

MySQL 数据库的字符串类型有 CHAR、VARCHAR、BINARY、BLOB、TEXT、ENUM、SET。不同的类型在业务设计、数据库性能方面的表现完全不同，其中最常使用的是 CHAR、VARCHAR。

### CHAR 和 VARCHAR

CHAR(N) 用来保存固定长度的字符，N 的范围是 0 ~ 255。**注意，N 表示的是字符，而不是字节**。VARCHAR(N) 用来保存变长字符，N 的范围为 0 ~ 65536， N 表示字符。

在超出 65536 个字符的情况下，可以考虑使用更大的字符类型 TEXT 或 BLOB，两者最大存储长度为 4G，其区别是 BLOB 没有字符集属性，使用二进制存储。

和 Oracle、Microsoft SQL Server 等传统关系型数据库不同的是，MySQL 数据库的 VARCHAR 字符类型最大能够存储 65536 个字符，所以在 MySQL 数据库下，绝大部分场景使用类型 VARCHAR 就足够了。

### 字符集

在表结构设计中，除了将列定义为 CHAR 和 VARCHAR 用以存储字符以外，还需要额外定义字符对应的字符集，因为每种字符在不同字符集编码下，对应着不同的二进制值。常见的字符集有 GBK、UTF8 等。

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

此外不同的字符集下 CHAR(N)、VARCHAR(N) 对应最长的字节也不相同。如 GBK 字符集 1 个字符最大存储 2 个字节，UTF8MB4 字符集 1 个字符最大存储 4 个字节。所以从底层存储内核看，在多字节字符集下，CHAR 和 VARCHAR 底层的实现完全相同，都是变长存储。例如 CHAR(1) 既可以存储 1 个 'a' 字节，也可以存储 4 个字节的 emoji 笑脸表情，因此 CHAR 本质也是变长的。

鉴于目前默认字符集推荐设置为 UTF8MB4，所以在表结构设计时，可以把 CHAR 全部用 VARCHAR 替换，底层存储的本质实现一模一样。对于变长字符集（如 GBK、UTF8MB4），其本质是一样的，都是变长，**设计时完全可以用 VARCHAR 替代 CHAR；**

### 排序规则

排序规则（Collation）是比较和排序字符串的一种规则，每个字符集都会有默认的排序规则，你可以用命令 SHOW CHARSET 来查看。

排序规则以 `_ci` 结尾，表示不区分大小写（Case Insentive），`_cs` 表示大小写敏感，`_bin` 表示通过存储字符的二进制进行比较。需要注意的是，比较 MySQL 字符串，默认采用不区分大小的排序规则。**绝大部分业务的表结构设计无须设置排序规则为大小写敏感**，除非业务真正需要。

### 修改字符集

业务在设计时没有考虑到字符集对于业务数据存储的影响，所以后期需要进行字符集转换，但很多同学会发现执行如下操作后，依然无法插入 emoji 这类 UTF8MB4 字符：

```sql
ALTER TABLE emoji_test CHARSET utf8mb4;
```

其实，上述修改只是将表的字符集修改为 UTF8MB4，下次新增列时，若不显式地指定字符集，新列的字符集会变更为 UTF8MB4，**但对于已经存在的列，其默认字符集并不做修改**，正确修改列字符集的命令应该使用 ALTER TABLE ... CONVERT TO...这样才能将之前的列字符集从 UTF8 修改为 UTF8MB4：

```sql
ALTER TABLE emoji_test CONVERT TO CHARSET utf8mb4;
```

## 3. 日期类型

MySQL 数据库中常见的日期类型有 YEAR、DATE、TIME、DATETIME、TIMESTAMEP。因为业务绝大部分场景都需要将日期精确到秒，所以在表结构设计中，常见使用的日期类型为DATETIME 和 TIMESTAMP。

#### DATETIME

类型 DATETIME 最终展现的形式为：yyyy-MM-dd HH:mm:ss，占用 5 个字节。

从 MySQL 5.6 版本开始，DATETIME 类型支持毫秒，DATETIME(N) 中的 **N 表示毫秒的精度**。例如，DATETIME(6) 表示可以存储 6 位的毫秒值，此时占8个字节。同时，一些日期函数也支持精确到毫秒，例如常见的函数 NOW、SYSDATE。

可以将 DATETIME 初始化值设置为当前时间，并设置自动更新当前时间的属性：

```sql
CREATE TABLE User (
    created_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    updated_at DATETIME(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6)
);
```

#### TIMESTAMP

TIMESTAMP 实际存储的内容为 1970-01-01 00:00:00 到现在的毫秒数。在 MySQL 中，由于类型 TIMESTAMP 占用 4 个字节，因此其存储的时间上限只能到 '2038-01-19 03:14:07'。

同类型 DATETIME 一样，从 MySQL 5.6 版本开始，类型 TIMESTAMP 也能支持毫秒。带有毫秒时，类型TIMESTAMP 占用 7 个字节。TIMESTAMP 占用 7 个字节的时候和占用 4 个字节时，上限一样到 2038 。

类型 TIMESTAMP 最大的优点是可以带有时区属性，因为它本质上是从毫秒转化而来。如果你的业务需要对应不同的国家时区，那么类型 TIMESTAMP 是一种不错的选择。比如新闻类的业务，通常用户想知道这篇新闻发布时对应的自己国家时间，那么 TIMESTAMP 是一种选择。另外，有些国家会执行夏令时。根据不同的季节，人为地调快或调慢 1 个小时，带有时区属性的 TIMESTAMP 类型本身就能解决这个问题。

参数 time_zone 指定了当前使用的时区，默认为 SYSTEM 使用操作系统时区，用户可以通过该参数指定所需要的时区：

```sql
SET time_zone = '-08:00';
```

在 MySQL 中可以直接设置时区的名字：

```sql
SET time_zone = 'America/Los_Angeles';
```

可以通过下面的语句将字段类型从 DATETIME(6) 修改为 TIMESTAMP(6)：

```sql
 ALTER TABLE User 
 CHANGE created_at 
 created_at TIMESTAMP(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6);
```

#### TIMESTAMP 的性能问题

TIMESTAMP 的上限值 2038 年很快就会到来，那时业务又将面临一次类似千年虫的问题。另外，TIMESTAMP 还存在潜在的性能问题。

虽然从毫秒数转换到类型 TIMESTAMP 本身需要的 CPU 指令并不多，这并不会带来直接的性能问题。但是如果使用默认的操作系统时区，则每次通过时区计算时间时，要调用操作系统底层系统函数 __tz_convert()，而这个函数需要额外的加锁操作，以确保这时操作系统时区没有修改。所以，当大规模并发访问时，由于热点资源竞争，会产生两个问题。

- **性能不如 DATETIME：** DATETIME 不存在时区转化问题。
- **性能抖动：** 海量并发时，存在性能抖动问题。

为了优化 TIMESTAMP 的使用，强烈建议使用显式的时区，而不是操作系统时区。比如在配置文件中显示地设置时区，而不要使用系统时区：

```mysql
[mysqld]

time_zone = "+08:00"
```

#### DATETIME 与 TIMESTAMP

* MySQL 5.6 版本开始 DATETIME 和 TIMESTAMP 精度支持到毫秒；
* DATETIME 占用 8 个字节，TIMESTAMP 占用 4 个字节，DATETIME(6) 依然占用 8 个字节，TIMESTAMP(6) 占用 7 个字节；

- TIMESTAMP 日期存储的上限为 2038-01-19 03:14:07，业务用 TIMESTAMP 存在风险；
- 使用 TIMESTAMP 必须显式地设置时区，不要使用默认系统时区，否则存在性能问题，推荐在配置文件中设置参数 time_zone = '+08:00'；
- 推荐日期类型使用 DATETIME，而不是 TIMESTAMP 和 INT 类型；

##  4. 非结构存储：JSON

JSON（JavaScript Object Notation）主要用于互联网应用服务之间的数据交换。MySQL 支持 [RFC 7159 ](https://tools.ietf.org/html/rfc7159?fileGuid=xxQTRXtVcqtHK6j8)定义的 JSON 规范，主要有 **JSON 对象**和 **JSON 数组**两种类型。

JSON 类型从表面上来看像是字段串类型，但本质上 JSON 是一种新的类型，有自己的存储格式，还能在每个对应的字段上创建索引，做特定的优化，这是传统字段串无法实现的。JSON 类型的另一个好处是**无须预定义字段**，字段可以无限扩展。而传统关系型数据库的列都需预先定义，想要扩展需要执行 ALTER TABLE ... ADD COLUMN ... 这样比较重的操作。

需要注意是，JSON 类型是从 MySQL 5.7 版本开始支持的功能，而 8.0 版本解决了更新 JSON 的日志性能瓶颈。如果要在生产环境中使用 JSON 数据类型，强烈推荐使用 MySQL 8.0 版本。

在数据库中，**JSON 类型比较适合存储一些修改较少、相对静态的数据**。

JSON_EXTRACT，它用来从 JSON 数据中提取所需要的字段内容，如下面的这条 SQL 语句就查询用户的手机和微信信息：

```sql
SELECT
    userId,
    JSON_UNQUOTE(JSON_EXTRACT(loginInfo,"$.cellphone")) cellphone,
    JSON_UNQUOTE(JSON_EXTRACT(loginInfo,"$.wxchat")) wxchat
FROM UserLogin;
```

`JSON_UNQUOTE` 用于将原 json 串的引号去掉转成字符串类型。

每次写 JSON_EXTRACT、JSON_UNQUOTE 非常麻烦，MySQL 还提供了 ->> 表达式，和上述 SQL 效果完全一样：

```sql
SELECT
    userId,
    loginInfo->>"$.cellphone" cellphone,
    loginInfo->>"$.wxchat" wxchat
FROM UserLogin;
```

当 JSON 数据量非常大，用户希望对 JSON 数据进行有效检索时，可以利用 MySQL 的**函数索引**功能对 JSON 中的某个字段进行索引。

比如在上面的用户登录示例中，假设用户必须绑定唯一手机号，且希望未来能用手机号码进行用户检索时，可以创建下面的索引：

```sql
ALTER TABLE UserLogin ADD COLUMN cellphone VARCHAR(255) AS (loginInfo->>"$.cellphone");
ALTER TABLE UserLogin ADD UNIQUE INDEX idx_cellphone(cellphone);
```

上述 SQL 首先创建了一个虚拟列 cellphone，这个列是由函数 loginInfo->>"$.cellphone" 计算得到的。然后在这个虚拟列上创建一个唯一索引 idx_cellphone。这时再通过虚拟列 cellphone 进行查询，就可以看到优化器会使用到新创建的 idx_cellphone 索引：

当然也可以在一开始创建表的时候，就完成虚拟列及函数索引的创建。如下表创建的列 cellphone 对应的就是 JSON 中的内容，是个虚拟列；uk_idx_cellphone 就是在虚拟列 cellphone 上所创建的索引。

```sql
CREATE TABLE User (
    userId BIGINT,
    loginInfo JSON,
    cellphone VARCHAR(255) AS (loginInfo->>"$.cellphone"),
    PRIMARY KEY(userId),
    UNIQUE KEY uk_idx_cellphone(cellphone)
);
```

MySQL 8.0.17 版本开始支持 Multi-Valued Indexes，用于在 JSON 数组上创建索引，并通过函数 member of、json_contains、json_overlaps 来快速检索索引数据。