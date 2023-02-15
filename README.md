# 需求分析

1. 只有增加和查询操作，而没有更新和删除操作。
2. 短时间内获取同一短域名的概率高。比如重新点击，分享，群发等操作。
3. 查询的 QPS 要远大于新增的 QPS。比如新增之后群分享短码链接。所以重点考虑查询的情况。
4. 查询的场景有已存在的短码和不存在的短码 2 种场景。

# 架构设计

1. 做三级缓存，一级缓存是内存，二级缓存是 redis，最后是数据库。
2. 一级缓存（内存）: 缓存已经存在的短连接
3. 二级缓存（redis）: 缓存已经存在的短连接和不存在的短连接
4. 三级缓存（数据库）：持久化生成的短连接

![新增流程](./doc/static/%E6%9E%B6%E6%9E%84%E8%AE%BE%E8%AE%A1%E5%9B%BE.png)

## 新增流程

![新增流程](./doc/static/%E6%B7%BB%E5%8A%A0%E6%97%B6%E5%BA%8F%E5%9B%BE.jpg)

## 查询已存在的短链接

![新增流程](./doc/static/%E6%9F%A5%E8%AF%A2%E5%B7%B2%E5%AD%98%E5%9C%A8.jpg)

## 查询不存在的短链接

![新增流程](./doc/static/%E6%9F%A5%E8%AF%A2%E4%B8%8D%E5%AD%98%E5%9C%A8.jpg)

# 表结构设计

|  字段   |    类型    | 主键 |     索引     | 不是 null |   注释   |
| :-----: | :--------: | :--: | :----------: | :-------: | :------: |
|   id    |    int     |  是  |      -       |    是     | 自增 id  |
|  code   | varchar(8) |  否  | btree unique |    是     |   短码   |
|   url   |    text    |  否  |      -       |    是     |  长链接  |
| created | timestamp  |  否  |      -       |    是     | 创建时间 |

# 短码生成逻辑

```
1. 字符集 BASE58 = '123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz';
2. 总的可能是 58^8 = 128063081718016 ≈ 1280630 亿 ， 在生成 1 亿的数据之后的碰撞概率约为: 1/1280629
3. 随机算法采用 Math.floor(Math.random() * (max - min)) + min;
   该随机算法不含最大值，含最小值，所以取 min = 0 , max = 58
   采用随机算法生成8次，取得8个字符。
```

# 内存分析

```
在 1 亿的数据情况下内存约：1 亿 * 8 字节 / 1024 / 1024 ≈ 762.9 M
```

# 单元测试

```bash
npm run test:cov
```

# 集成测试

```
npm run test:e2e
```

# 接口文档

## 长链接获取短连接

Method: POST

Path: /short/upload

Content-Type:application/json

请求参数：

| 字段 | 字段类型 | 是否为空 | 说明   |
| ---- | -------- | -------- | ------ |
| url  | string   | 否       | 长链接 |

返回值：

| 字段       | 字段类型 | 是否为空 | 说明       |
| ---------- | -------- | -------- | ---------- |
| statusCode | number   | 否       | 状态码     |
| message    | string   | 否       | 信息       |
| data       | {url}    | 否       |            |
| -url       | string   | 否       | 短连接地址 |

## 短链接获长短连接

Method: GET

Path : /short/:code

Content-Type:application/json

请求参数(param)：

| 字段 | 字段类型 | 是否为空 | 说明 |
| ---- | -------- | -------- | ---- |
| code | string   | 否       | 短码 |

返回值：

| 字段       | 字段类型 | 是否为空 | 说明       |
| ---------- | -------- | -------- | ---------- |
| statusCode | number   | 否       | 状态码     |
| message    | string   | 否       | 信息       |
| data       | {url}    | 否       |            |
| -url       | string   | 否       | 长连接地址 |
