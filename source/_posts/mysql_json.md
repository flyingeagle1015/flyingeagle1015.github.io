---
title: MySQL JSON类型
date: 2019-03-29 16:58:34
tags: [mysql]
categories: [application]
---

MySQL version 5.7.8 引入了一个JSON数据类型，允许您实现它。
在本教程中，您将学习：
1. 如何使用JSON字段设计数据库表。
2. 使用 MYSQL 中各种基于 JSON 的可用的方法对 json 数据进行增、删、改、查。
3. 虚拟列的使用

<!-- more -->

## 创建一个JSON字段的表
首先先创建一个表，这个表包含一个json格式的字段：
``` sql
CREATE TABLE tbl_test(
  id INT NOT NULL AUTO_INCREMENT, 
  json_col JSON,
  PRIMARY KEY(id)
);
```
上面的语句，主要注意json_col这个字段，指定的数据类型是JSON。

## 插入JSON数据
```
INSERT INTO tbl_test(json_col) VALUES
  ('{"City": "Galle", "Description": "Best damn city in the world"}');

INSERT INTO tbl_test(json_col) VALUES
  ('{"opening":"Sicilian","variations":["pelikan","dragon","najdorf"]}');
```
由于json格式的数据里，需要有双引号来标识字符串，所以，VALUES后面的内容需要用单引号包裹。

也可以使用内置的 JSON_OBJECT 函数，而不是自己处理 JSON 对象。
JSON_OBJECT函数接键/值对列表 JSON_OBJECT(key1, value1, key2, value2, ... key(n), value(n)) 并返回一个 JSON 对象。
```
INSERT INTO tbl_test(json_col) VALUES
  (JSON_OBJECT("colors",  JSON_ARRAY("red" , "yellow" , "white" , "black"), "os", "linux"));
```

请注意JSON_ARRAY函数，它在传递一组值时返回一个 JSON 数组。
如果你多次指定一个键，那么只保留第一个键/值对。这就是 MySQL 术语中的 JSON 规范化。另外，作为标准化的一部分，对象键被排序，键/值对之间的额外空白被删除。
我们可以用来创建 JSON 对象的另一个函数是JSON_MERGE函数。
JSON_MERGE函数接受多个JSON对象并产生单个聚合对象。
```
INSERT INTO tbl_test(json_col) VALUES
  (JSON_MERGE(
    JSON_OBJECT("os", "linux"),
    JSON_OBJECT("colors",  JSON_ARRAY("red" , "yellow" , "white" , "black"))));

INSERT INTO tbl_test(json_col) VALUES
  (JSON_MERGE(
    '{"os": "linux"}',
    '{"office": "wps"}'
  ));
```

我们只是将对象传递给JSON_MERGE函数。其中一些是使用我们前面看到的JSON_OBJECT函数构造的，而另一些则作为有效的 JSON 字符串传递。
在JSON_MERGE函数下，如果一个键重复多次，那么它的值就会被保留为数组输出。
```
INSERT INTO tbl_test(json_col) VALUES
  (JSON_MERGE(
    '{"os": "linux"}',
    '{"os": "windows"}',
    '{"office": "wps"}'
  ));
```

## 查询JSON数据
对于不属于 JSON 类型的典型 MySQL 值，where子句是非常直接的。只要指定列、运算符和您需要处理的值即可。
从设计上来说，当使用 JSON 列时，这就行不通了。
```
mysql> select * from tbl_test;
+----+----------------------------------------------------------------------------------------------------------------+
| id | json_col                                                                                                       |
+----+----------------------------------------------------------------------------------------------------------------+
|  1 | {"City": "Galle", "Description": "Best damn city in the world"}                                                |
|  2 | {"opening": "Sicilian", "variations": ["pelikan", "dragon", "najdorf"]}                                        |
|  3 | {"os": "linux", "colors": ["red", "yellow", "white", "black"]}                                                 |
|  4 | {"os": ["linux", "windows"], "colors": ["red", "yellow", "white", "black", "red", "yellow", "white", "black"]} |
|  5 | {"os": "linux", "office": "wps"}                                                                               |
|  6 | {"os": ["linux", "windows"], "office": "wps"}                                                                  |
+----+----------------------------------------------------------------------------------------------------------------+
6 rows in set (0.00 sec)

mysql> select * from tbl_test where json_col='{"os": "linux", "office": "wps"}';
Empty set (0.00 sec)

```

JSON数据使用->操作符，其表达式为：该`json列->'$.键'`与`JSON_EXTRACT(json列 , '$.键')`等效使用。如果传入的不是一个有效的键，则返回Empty set。
该表达式可以用于SELECT查询列表 ，WHERE/HAVING , ORDER/GROUP BY中，但它不能用于设置值。
表达式 ： `json列->'$.键'`
```
mysql>  SELECT * FROM tbl_test WHERE json_col->'$.os' = 'linux';
+----+----------------------------------------------------------------+
| id | json_col                                                       |
+----+----------------------------------------------------------------+
|  3 | {"os": "linux", "colors": ["red", "yellow", "white", "black"]} |
|  5 | {"os": "linux", "office": "wps"}                               |
+----+----------------------------------------------------------------+
2 rows in set (0.00 sec)
```

等价于 ：`JSON_EXTRACT(json列 , '$.键')`
```
mysql>  SELECT * FROM tbl_test WHERE JSON_EXTRACT(json_col,'$.os') = 'linux';
+----+----------------------------------------------------------------+
| id | json_col                                                       |
+----+----------------------------------------------------------------+
|  3 | {"os": "linux", "colors": ["red", "yellow", "white", "black"]} |
|  5 | {"os": "linux", "office": "wps"}                               |
+----+----------------------------------------------------------------+
2 rows in set (0.00 sec)
```

比较JSON值采用两个级别。第一级是基于JSON类型的比较。如果类型不同，则取决于哪种类型具有更高的优先级。如果是相同的JSON类型，则是第二级，使用该类型的规则来比较。
下面的列表显示了JSON类型的比较规则,从最高优先级到最低优先级。显示在一行的类型则是具有相同的优先级。
```
BLOB
BIT
OPAQUE
DATETIME
TIME
DATE
BOOLEAN
ARRAY
OBJECT
STRING
INTEGER, DOUBLE
NULL
```

使用JSON_TYPE()函数返回指定属性对应的类型名称：
```
mysql> SELECT JSON_TYPE(json_col->'$.os') FROM tbl_test;
+-----------------------------+
| JSON_TYPE(json_col->'$.os') |
+-----------------------------+
| NULL                        |
| NULL                        |
| STRING                      |
| ARRAY                       |
| STRING                      |
| ARRAY                       |
+-----------------------------+
6 rows in set (0.00 sec)
```

值得一提的是，可以通过虚拟列对JSON类型的指定属性进行快速查询。
创建虚拟列：
```
mysql> ALTER TABLE tbl_test ADD os VARCHAR(15) GENERATED ALWAYS AS (json_col->'$.os') VIRTUAL;
Query OK, 0 rows affected (0.08 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

使用时和普通类型的列查询是一样的：
```
mysql> SELECT os FROM tbl_test ;
+-----------------+
| os              |
+-----------------+
| NULL            |
| NULL            |
| "linux"         |
| "linux"         |
+-----------------+
4 rows in set (0.00 sec)
```

## 创建索引
MySQL的JSON格式数据不能直接创建索引，但是可以变通一下，把要搜索的数据单独拎出来，单独一个数据列，然后在这个字段上键一个索引。下面是官方的例子：
```
mysql> CREATE TABLE jemp (
    ->     c JSON,
    ->     g INT GENERATED ALWAYS AS (c->"$.id"),
    ->     INDEX i (g)
    -> );
Query OK, 0 rows affected (0.28 sec)
```
就是把JSON字段里的id字段，单独拎出来成字段g，然后在字段g上做索引，查询条件也是在字段g上。

## 字符串转JSON格式
把json格式的字符串转换成MySQL的JSON类型:
```
SELECT CAST('[1,2,3]' as JSON) ;
SELECT CAST('{"opening":"Sicilian","variations":["pelikan","dragon","najdorf"]}' as JSON);
```

## 更新JSON数据
更新JSON值，我们将使用JSON_INSERT、JSON_REPLACE和JSON_SET函数。这些函数还需要一个路径表达式来指定要修改的 JSON 部分。
```
-- 整个json更新的话，和插入时类似
update tbl_test set json_col='{"os":"linux", "office":"wps"}' where id =5;

-- JSON_INSERT() 插入新值，但不会覆盖已经存在的值
update tbl_test set json_col=JSON_INSERT(json_col, '$.test', 'aaa')  where id =5;

-- JSON__SET() 插入新值，并覆盖已经存在的值
update tbl_test set json_col=JSON__SET(json_col, '$.test', 'bbb')  where id =5;

-- JSON__REPLACE() 只替换存在的值
update tbl_test set json_col=JSON__REPLACE(json_col, '$.test', 'bbb')  where id =5;

-- JSON__REMOVE() 删除 JSON 元素
update tbl_test set json_col=JSON_REMOVE(json_col, '$.test')  where id =5;
```


## 所有MYSQL JSON函数
Name  | Description
----  | ----
JSON_APPEND() | append data to json document
JSON_ARRAY()  |  Create JSON array
JSON_ARRAY_APPEND()  | Append data to JSON document
JSON_ARRAY_INSERT() | Insert into JSON array Return value from JSON column after evaluating path; equivalent to JSON_EXTRACT().
JSON_CONTAINS() | Whether JSON document contains specific object at path
JSON_CONTAINS_PATH() | Whether JSON document contains any data at path 
JSON_DEPTH() | Maximum depth of JSON document
JSON_EXTRACT() |  Return data from JSON document Return value from JSON column after evaluating path and unquoting the result; equivalent to JSON_UNQUOTE(JSON_EXTRACT()).
JSON_INSERT() |  Insert data into JSON document
JSON_KEYS() |  Array of keys from JSON document
JSON_LENGTH() |  Number of elements in JSON document
JSON_MERGE() |  Merge JSON documents, preserving duplicate keys. Deprecated synonym for JSON_MERGE_PRESERVE()
JSON_MERGE_PRESERVE() |  Merge JSON documents, preserving duplicate keys
JSON_OBJECT() |  Create JSON object
JSON_QUOTE() |  Quote JSON document
JSON_REMOVE() |  Remove data from JSON document
JSON_REPLACE() |  Replace values in JSON document
JSON_SEARCH() |  Path to value within JSON document
JSON_SET() |  Insert data into JSON document
JSON_TYPE() |  Type of JSON value
JSON_UNQUOTE() | Unquote JSON value
JSON_VALID() |	Whether JSON value is valid

