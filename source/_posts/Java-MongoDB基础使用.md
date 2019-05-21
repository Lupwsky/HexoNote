---
title: Java-MongoDB基础使用
date: 2019-05-21 10:51:02
categories: Java
---

# 创建集合

## 语法格式

```text
db.createCollection(name, options);
```

## 参数 options

* autoIndexId = 创建 ID 时自动创建索引
* capped = 是否创建固定大小集合, 为  true 时必须指定 size 属性, 固定大小集合在文档数量或者空间大小达到最大值时会覆盖最早的文档
* size = 固定大小集合的空间最大值, 单位为 KB
* max = 固定大小集合的文档数量的最大值

<!-- more -->

## 使用示例

```text
db.createCollection("order_info", {autoIndexId: true});
```

输出结果如下:

```text
// 注: 当前使用的版本提示了 autoIndexId 属性将会在将来的版本中移除
{ 
    "note" : "the autoIndexId option is deprecated and will be removed in a future release", 
    "ok" : 1.0
}
```

# 删除集合

## 语法格式

```text
db.collectionName.drop();
```

## 使用示例

```text
// 或者 db.userinfo.frop();
db.getCollection('userinfo').drop();
```

输出结果如下:

```text
// 成功返回 true, 失败返回 fasle
true
```

# 更新集合

无更新集合接口, 只能删除后再创建, 再将备份的数据导入

# 查询集合

## 语法格式

```text
show collections; 
```

## 使用示例

```text
show collections; 
```

输出结果如下:

```text
order_info
startup_log
```

# 插入文档

## 语法格式

```text
// 插入数据, 该接口支持以数组的形式一次插入多条数据
db.collectionName.insert();

// 3.2 支持的新接口
// 插入一条数据
db.collectionName.insertOne();

// 插入多条数据
db.collectionName.insertMany();
```

## 使用示例

```text
db.getCollection("user_info").insert({"name":"lpw", "password":"123", "sex":"男", "phone":"15988884646"});
```

输出结果如下:

```text
WriteResult({ "nInserted" : 1 })
```

插入多条数据:

```text
db.getCollection("user_info").insertMany([
    {"name":"lpw1", "password":"123", "sex":"男", "phone":"15988884646"},
    {"name":"lpw2", "password":"123", "sex":"男", "phone":"15988884646"}
]);
```

输出结果如下:

```text
{ 
    "acknowledged" : true, 
    "insertedIds" : [
        ObjectId("5ccfa662ed4df77310773203"), 
        ObjectId("5ccfa662ed4df77310773204")
    ]
}
```

# 查询文档

## 语法格式

```text

```

# 更新文档

## 语法格式

```text
db.collectionName.update(query, update, options?);
db.collectionName.updateOne(filter, update, options?);
db.collectionName.updateMany(filter, update, options?);
```

* query/filter: 查询条件
* update: 更新操作符的更新操作, 如 $inc, $set, $unset
* options: 包含 upsert, multi 和 writeConcern 三个配置

optiosns 配置项含义:

```json
{
    // 如果没有找到符合条件的记录, 是否新增一条记录, 默认 false
    "upsert": false,

    // 是否只更新找到的第一条记录, 默认 false
    "multi": false,

    // 可选, 抛出异常的级别
    "writeConcern" : ""
}
```