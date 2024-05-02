当我们需要在数据库中表示枚举时，一般有两种常见的方式：`varchar`或者`tinyint`。两种方式各有各的拥趸，支持`varchar`者认为性能差异不大，`varchar`可读性好得多。支持`tinyint`者则认为两者信息密度差异巨大，那么差异到底有多大呢？

# 测试场景

测试中假设以下场景

* 1000名用户，ID从1到1000
* 100种商品，ID从1到100
* 一个字段表示订单来源：
+ MALL=1 表示订单来自自家商城
+ CHANNEL=2 表示订单来自渠道商
+ ONLINE=3 表示线上订单
+ PHONE=4 表示来自电话推销
+ OTHER=5 表示其它来源

分别创建使用`varchar`的表和使用`tinyint`的表，两表唯一的不同就是source字段不同的表示方式：

```
CREATE TABLE `order_with_enum`
(
    `id`         bigint    NOT NULL AUTO_INCREMENT COMMENT 'ID',
    `user_id`    int       NOT NULL COMMENT '用户ID',
    `goods_id`   int       NOT NULL COMMENT '商品ID',
    `source`     varchar(16) COLLATE utf8mb4_bin DEFAULT NULL COMMENT '订单来源',
    `created_at` timestamp NOT NULL              DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    `updated_at` timestamp NOT NULL              DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (`id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_bin
```

```
CREATE TABLE `order_with_tinyint`
(
    `id`         bigint    NOT NULL AUTO_INCREMENT COMMENT 'ID',
    `user_id`    int       NOT NULL COMMENT '用户ID',
    `goods_id`   int       NOT NULL COMMENT '商品ID',
    `source`     tinyint            DEFAULT NULL COMMENT '订单来源',
    `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    `updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (`id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_bin;
```
# 插入速度

首先测试插入场景，插入的字段都在各自允许的范围内随机。使用300线程，30的爬坡，各自允许10分钟，比较如下：

|  |  |  |
| --- | --- | --- |
|  | 使用varchar | 使用tinyint |
| 写入总条数 | 51042 | 53087 |
| RPS | 84.47 | 87.88 |
| 平均RT | 11 | 10 |
| P99 | 70 | 71 |
| P98 | 25 | 16 |
| P95 | 16 | 15 |

可以看出，插入方面性能差别不是很大，除非在性能很关键的领域不会有明显的差别。

# 读取速度

插入数据到2200W行左右，比较表的大小：

|  |  |  |
| --- | --- | --- |
|  | varchar | tinyint |
| 行数 | 22050842 | 22101042 |
| 体积（MB） | 1294.98 | 1155.98 |

可以看出tinyint在体积上确实比较有优势。

## 通过ID查询

在ID有效的范围内，随机选择一个ID进行查询，查询方式：

```
id = randint(1, 22103381)
self.client.execute_query('select * from order_with_tinyint where id = %s', (id,))
self.client.execute_query('select * from order_with_enum where id = %s', (id,))
```
查询耗时基本都在1毫秒以下，但吞吐存在差异，执行一分钟，完成的总查询如下：

|  |  |  |
| --- | --- | --- |
|  | varchar | tinyint |
| 总查询量 | 929271 | 1167277 |

可知通过ID查询上两者还是存在着比较明显的差异。

## 通过条件进行查找

假设需要通过用户ID和来源进行查找，并通过ID倒序进行排序进行排序，首先为两表增加索引：

```
ALTER TABLE order_with_enum
    ADD INDEX ix_user_source (user_id, source);

ALTER TABLE order_with_tinyint
    ADD INDEX ix_user_source (user_id, source);
```
查询使用的语句：

```
SELECT *
FROM order_with_enum/order_with_tinyint
WHERE user_id = ?
  AND source = ?
ORDER BY id DESC
LIMIT 100,10;
```
执行时间一分钟，查询结果：

|  |  |  |
| --- | --- | --- |
|  | varchar | tinyint |
| 总请求量 | 89341 | 60271 |
| RPS | 1484.81 | 999.52 |

 非常神奇，varchar甚至要快出很多，不清除是否mysql内部做了一些专门的优化。

## 查找并聚合

查找提交与之前类似，但是查找完成之后通过商品进行一次聚合：

执行一分钟，结果：

|  |  |  |
| --- | --- | --- |
|  | varchar | tinyint |
| 总请求量 | 3280 | 3281 |
| 平均耗时(ms) | 19 | 19 |
| RPS | 48.24 | 48.69 |

可以看出两者性能几乎完全一致。

