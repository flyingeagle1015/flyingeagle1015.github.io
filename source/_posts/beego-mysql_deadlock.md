---
title: beego orm使用Mysql事务死锁
date: 2019-04-19 17:52:39
tags: [mysql, beego]
categories: [develop]
---

## 问题现象
在beego项目中添加用户的过程中出现mysql死锁。
示例代码：
``` go
def add_user(userdata User) (id int, err error) {
  o := orm.NewOrm()
  err = o.Begin()
  defer func() {
    if err == nil {
      if err = o.Commit(); err != nil {
        // ......
      }
    }
  }()
  if err != nil {
    return
  }

  // add user information to user table.
  // add user other information in other table.
}
```

<!-- more -->

**查看当前运行的所有事务**
```
mysql> SELECT * FROM information_schema.INNODB_TRX\G
*************************** 1. row ***************************
                    trx_id: 45900
                 trx_state: ROLLING BACK
               trx_started: 2018-04-09 10:24:38
     trx_requested_lock_id: NULL
          trx_wait_started: NULL
                trx_weight: 30687
       trx_mysql_thread_id: 123456
                 trx_query: N
......
--------------------- 
mysql> kill 123456 //强制关闭线程和事务
```

**分析原因**
由于代码中添加用户失败没有回滚操作，导致事务死锁。修改为：
``` go
def add_user(userdata User) (id int, err error) {
  o := orm.NewOrm()
  err = o.Begin()
  defer func() {
    if err == nil {
      if err = o.Commit(); err != nil {
        o.Rollback()
        // ......
      }
    } else {
      o.Rollback()
    }
  }()
  if err != nil {
    return
  }
  
  // add user information to user table.
  // add user other information in other table.
}
```

