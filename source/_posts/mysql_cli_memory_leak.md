---
title: mysql客户端内存泄漏
date: 2019-02-26 14:58:57
tags: [mysql, c]
categories: [develop]
---

最近在用Valgrind检查一个库代码, 发现MYSQL的泄漏. 简化MySQL操作代码并测试得到了同样的泄漏.
码：
```
#include <stdio.h>
#include <stdlib.h>

#include <mysql/mysql.h>

int main()
{
    MYSQL *MYSQLIns;
    MYSQLIns = mysql_init(NULL);
    mysql_real_connect(MYSQLIns, "localhost", "test", "123456", "test", 0, NULL, 0);
    mysql_close(MYSQLIns);

    return EXIT_SUCCESS;
}
```
编译：

gcc -g -L/usr/lib64/mysql -lmysqlclient mysql_mem_test.c -o mysql_mem_test

<!-- more -->

Valgrind输出：
```
valgrind --leak-check=full ./mysql_mem_test
==26126== Memcheck, a memory error detector
==26126== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==26126== Using Valgrind-3.14.0 and LibVEX; rerun with -h for copyright info
==26126== Command: ./mysql_mem_test
==26126== 
==26126== 
==26126== HEAP SUMMARY:
==26126==     in use at exit: 8,408 bytes in 4 blocks
==26126==   total heap usage: 77 allocs, 73 frees, 68,596 bytes allocated
==26126== 
==26126== 56 bytes in 1 blocks are possibly lost in loss record 1 of 4
==26126==    at 0x4C29E83: malloc (vg_replace_malloc.c:299)
==26126==    by 0x4E999E7: my_raw_malloc (my_malloc.c:191)
==26126==    by 0x4E999E7: my_malloc (my_malloc.c:54)
==26126==    by 0x4E98A1C: my_error_register (my_error.c:310)
==26126==    by 0x4E5CC24: mysql_server_init (libmysql.c:116)
==26126==    by 0x4E655B6: mysql_init (client.c:2460)
==26126==    by 0x4006CE: main (a.c:9)
==26126== 
==26126== 176 bytes in 1 blocks are possibly lost in loss record 2 of 4
==26126==    at 0x4C29E83: malloc (vg_replace_malloc.c:299)
==26126==    by 0x4E999E7: my_raw_malloc (my_malloc.c:191)
==26126==    by 0x4E999E7: my_malloc (my_malloc.c:54)
==26126==    by 0x4E97B8A: init_alloc_root (my_alloc.c:77)
==26126==    by 0x4E6F01B: mysql_client_plugin_init (client_plugin.c:342)
==26126==    by 0x4E5CC2B: mysql_server_init (libmysql.c:117)
==26126==    by 0x4E655B6: mysql_init (client.c:2460)
==26126==    by 0x4006CE: main (a.c:9)
==26126== 
==26126== LEAK SUMMARY:
==26126==    definitely lost: 0 bytes in 0 blocks
==26126==    indirectly lost: 0 bytes in 0 blocks
==26126==      possibly lost: 232 bytes in 2 blocks
==26126==    still reachable: 8,176 bytes in 2 blocks
==26126==         suppressed: 0 bytes in 0 blocks
==26126== Reachable blocks (those to which a pointer was found) are not shown.
==26126== To see them, rerun with: --leak-check=full --show-leak-kinds=all
==26126== 
==26126== For counts of detected and suppressed errors, rerun with: -v
==26126== ERROR SUMMARY: 2 errors from 2 contexts (suppressed: 0 from 0)
```
查阅MySQL C API了解到:
在非多线程环境中， mysql_init()根据需要自动调用mysql_library_init()。但是mysql_library_init()不是线程安全的，  
因此在多线程环境应该在mysql_init()且产生任何线程之前调用mysql_library_init()，或者使用互斥锁来保护mysql_library_init()。 
这应该在任何其他客户端库调用之前完成。
mysql_close()仅释放mysql_init()或mysql_connect()创建的连接，而不会释放mysql_library_init()分配的内存，需要调用mysql_library_end()执行一些内存清理. 

