---
title: 使用MySQL 5.7虚拟列提高查询效率
date: 2019-03-29 19:27:57
tags: [mysql]
categories: [application]
---

## 问题
这个查询运行了一个多小时，并且使用和撑满了整个 tmp目录（需要用到临时文件完成排序）。
```
SELECT 
CONCAT(verb, ' - ', replace(url,'.xml','')) AS 'API Call', 
  COUNT(*) as 'No. of API Calls', 
  AVG(ExecutionTime) as 'Avg. Execution Time', 
  COUNT(distinct AccountId) as 'No. Of Accounts', 
  COUNT(distinct ParentAccountId) as 'No. Of Parents' 
  FROM ApiLog 
  WHERE ts between '2017-10-01 00:00:00' and '2017-12-31 23:59:59' 
  GROUP BY CONCAT(verb, ' - ', replace(url,'.xml','')) 
  HAVING COUNT(*) >= 1 ;
```

<!-- more -->

表结构如下:
```
CREATE TABLE `ApiLog` (
  `Id` int(11) NOT NULL AUTO_INCREMENT,
  `ts` timestamp DEFAULT CURRENT_TIMESTAMP,
  `ServerName` varchar(50)  NOT NULL default '',
  `ServerIP` varchar(50)  NOT NULL default '',
  `ClientIP` varchar(50)  NOT NULL default '',
  `ExecutionTime` int(11) NOT NULL default 0,
  `URL` varchar(3000)  NOT NULL COLLATE utf8mb4_unicode_ci NOT NULL,
  `Verb` varchar(16)  NOT NULL,
  `AccountId` int(11) NOT NULL,
  `ParentAccountId` int(11) NOT NULL,
  `QueryString` varchar(3000) NOT NULL,
  `Request` text NOT NULL,
  `RequestHeaders` varchar(2000) NOT NULL,
  `Response` text NOT NULL,
  `ResponseHeaders` varchar(2000) NOT NULL,
  `ResponseCode` varchar(4000) NOT NULL,
  ... // other fields removed for simplicity
  PRIMARY KEY (`Id`),
  KEY `index_timestamp` (`ts`),
  ... // other indexes removed for simplicity
) ENGINE=InnoDB;
```

我们发现查询没有使用时间戳字段（“TS”）的索引:
```
mysql> explain SELECT CONCAT(verb, ' - ', replace(url,'.xml','')) AS 'API Call', COUNT(*)  as 'No. of API Calls',  avg(ExecutionTime) as 'Avg. Execution Time', count(distinct AccountId) as 'No. Of Accounts',  count(distinct ParentAccountId) as 'No. Of Parents'  FROM ApiLog  WHERE ts between '2017-10-01 00:00:00' and '2017-12-31 23:59:59'  GROUP BY CONCAT(verb, ' - ', replace(url,'.xml',''))  HAVING COUNT(*)  >= 1G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: ApiLog
   partitions: NULL
         type: ALL
possible_keys: ts
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 22255292
     filtered: 50.00
        Extra: Using where; Using filesort1 row in set, 1 warning (0.00 sec)
```
原因很简单：符合过滤条件的行数太大了，以至于影响一次索引扫描扫描的效率（或者至少优化器是这样认为的）：
```
mysql> select count(*) from ApiLog WHERE ts between '2017-10-01 00:00:00' and '2017-12-31 23:59:59' ;
+----------+
| count(*) |
+----------+
|  7948800 |
+----------+
1 row in set (2.68 sec)
```
总行数：21998514。查询需要扫描的总行数的36%（7948800/21998514）（译者按：当预估扫描行数超过20% ～ 30%时，即便有索引，优化器通常也会强制转成全表扫描）。
在这种情况下，我们有许多处理方法：
创建时间戳列和GROUP BY列的联合索引；
创建一个覆盖索引（包含所有查询字段）；
仅对GROUP BY列创建索引；
创建索引松散索引扫描。
然而，如果我们仔细观察查询中“GROUP BY”部分，我们很快就意识到，这些方案都不能解决问题。以下是我们的GROUP BY部分：
```
GROUP BY CONCAT(verb, ' - ', replace(url,'.xml',''))
```
这里有两个问题：
它是计算列，所以MySQL不能扫描verb + url的索引。它首先需要连接两个字段，然后组成连接字符串。这就意味着用不到索引；
URL被定义为“varchar(3000) COLLATE utf8mb4_unicode_ci NOT NULL”，不能被完全索引（即使在全innodb_large_prefix= 1 参数设置下，这是UTF8启用下的默认参数）。我们能做部分索引，这对GROUP BY的sql优化并没有什么帮助。
在这里，我尝试去对URL列添加一个完整的索引，在innodb_large_prefix=1参数下：
```
mysql> alter table ApiLog add key verb_url(verb, url);
ERROR 1071 (42000): Specified key was too long; max key length is 3072 bytes
```
嗯，通过修改“GROUP BY CONCAT(verb, ‘ – ‘, replace(url,’.xml’,”))”为 “GROUP BY verb, url” 会帮助（假设我们把字段定义从 varchar（3000）调小一些，不管业务上允许或不允许）。然而，这将改变结果，因URL字段不会删除 .xml扩展名了。

## 解决方案
好消息是，在MySQL 5.7中我们有虚拟列。所以我们可以在“CONCAT(verb, ‘ – ‘, replace(url,’.xml’,”))”之上创建一个虚拟列。最好的部分：我们不需要执行一组完整的字符串（可能大于3000字节）。我们可以使用MD5哈希（或更长的哈希，例如SHA1 / SHA2）作为GROUP BY的对象。
下面是解决方案:
```
alter table ApiLog add verb_url_hash varbinary(16) GENERATED ALWAYS AS (unhex(md5(CONCAT(verb, ' - ', replace(url,'.xml',''))))) VIRTUAL;
alter table ApiLog add key (verb_url_hash);
```
所以我们在这里做的是：
声明虚拟列，类型为varbinary（16）；
在CONCAT(verb, ‘ – ‘, replace(url,’.xml’,”)上创建虚拟列，并且使用MD5哈希转化后再使用unhex转化32位十六进制为16位二进制；
对上面的虚拟列创建索引。
现在我们可以修改查询语句，GROUP BY verb_url_hash列：
```
mysql> explain SELECT CONCAT(verb, ' - ', replace(url,'.xml',''))
AS 'API Call', COUNT(*)  as 'No. of API Calls',
avg(ExecutionTime) as 'Avg. Execution Time',
count(distinct AccountId) as 'No. Of Accounts',
count(distinct ParentAccountId) as 'No. Of Parents'
FROM ApiLog
WHERE ts between '2017-10-01 00:00:00' and '2017-12-31 23:59:59'
GROUP BY verb_url_hash
HAVING COUNT(*)  >= 1;
ERROR 1055 (42000): Expression #1 of SELECT list is not in
GROUP BY clause and contains nonaggregated column 'ApiLog.ApiLog.Verb'
which is not functionally dependent on columns in GROUP BY clause;
this is incompatible with sql_mode=only_full_group_by
```
MySQL 5.7的严格模式是默认启用的，我们可以只针对这次查询修改一下。

现在解释计划看上去好多了：
```
mysql> select @@sql_mode;
+-------------------------------------------------------------------------------------------------------------------------------------------+
| @@sql_mode                                                                                                                                
|+-------------------------------------------------------------------------------------------------------------------------------------------+
| ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION |
+-------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
mysql> set sql_mode='STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION';
Query OK, 0 rows affected (0.00 sec)
mysql> explain SELECT CONCAT(verb, ' - ', replace(url,'.xml','')) AS 'API Call', COUNT(*)  as 'No. of API Calls',  avg(ExecutionTime) as 'Avg. Execution Time', count(distinct AccountId) as 'No. Of Accounts',  count(distinct ParentAccountId) as 'No. Of Parents'  FROM ApiLog  WHERE ts between '2017-10-01 00:00:00' and '2017-12-31 23:59:59'  GROUP BY verb_url_hash HAVING COUNT(*)  >= 1G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: ApiLog
   partitions: NULL
         type: index
possible_keys: ts,verb_url_hash
          key: verb_url_hash
      key_len: 19
          ref: NULL
         rows: 22008891
     filtered: 50.00
        Extra: Using where1 row in set, 1 warning (0.00 sec)
```
MySQL可以避免排序，速度更快。它将最终还是要扫描所有表的索引的顺序。响应时间明显更好：只需大概38秒而不再是大于一小时。

## 覆盖索引
现在我们可以尝试做一个覆盖索引，这将相当大：
```
mysql> alter table ApiLog add key covered_index (`verb_url_hash`,`ts`,`ExecutionTime`,`AccountId`,`ParentAccountId`, verb, url);
Query OK, 0 rows affected (1 min 29.71 sec)
Records: 0  Duplicates: 0  Warnings: 0
```
我们添加了一个“verb”和“URL”，所以之前我不得不删除表定义的COLLATE utf8mb4_unicode_ci。现在执行计划表明，我们使用了覆盖索引：
```
mysql> explain SELECT  CONCAT(verb, ' - ', replace(url,'.xml','')) AS 'API Call',  COUNT(*) as 'No. of API Calls',  AVG(ExecutionTime) as 'Avg. Execution Time',  COUNT(distinct AccountId) as 'No. Of Accounts',  COUNT(distinct ParentAccountId) as 'No. Of Parents'  FROM ApiLog  WHERE ts between '2017-10-01 00:00:00' and '2017-12-31 23:59:59'  GROUP BY verb_url_hash  HAVING COUNT(*) >= 1G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: ApiLog
   partitions: NULL
         type: index
possible_keys: ts,verb_url_hash,covered_index
          key: covered_index
      key_len: 3057
          ref: NULL
         rows: 22382136
     filtered: 50.00
        Extra: Using where; Using index
1 row in set, 1 warning (0.00 sec)
```
响应时间下降到约12秒！但是，索引的大小明显地比仅verb_url_hash的索引（每个记录16字节）要大得多。

## 结论
MySQL 5.7的生成列提供一个有价值的方法来提高查询性能。如果你有一个有趣的案例，请在评论中分享。

