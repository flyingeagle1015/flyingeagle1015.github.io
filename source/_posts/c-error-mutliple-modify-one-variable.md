---
title: C代码不在一行多次修改一个变量
date: 2019-03-04 17:52:51
tags: [c]
categories: [develop]
---

## 不要在同一行代码对同一变量做多次修改
源于对大神项目代码学习中遇到得问题，经简化复现方便说明。
示例：
```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
  char *buf;
  int top;
}context_t;

static int faaa(context_t *c , int o)
{
  c->top += o;
  return 6;
}

int main()
{
    context_t c;
    memset(&c, 0, sizeof(c));

    c.top -= 32 - faaa(&c, 32);
    printf("top:%d, should be 6\n", c.top);

    return EXIT_SUCCESS;
}

[test@test tmp]$ gcc -g -o test a.c
[test@test tmp]$ ./test 
top:-26, should be 6
```

<!-- more -->

代码行中`c.top -= 32 - faaa(&c, 32);`对c.top做了两次修改：
第一次为faaa中的修改
第二次为自减操作

为什么会这样呢？马上为您揭晓！
```
(gdb) disassemble main
Dump of assembler code for function main:
   0x00000000004005a2 <+0>:	push   %rbp
   0x00000000004005a3 <+1>:	mov    %rsp,%rbp
   0x00000000004005a6 <+4>:	push   %rbx
   0x00000000004005a7 <+5>:	sub    $0x18,%rsp
   0x00000000004005ab <+9>:	lea    -0x20(%rbp),%rax
   0x00000000004005af <+13>:	mov    $0x10,%edx
   0x00000000004005b4 <+18>:	mov    $0x0,%esi
   0x00000000004005b9 <+23>:	mov    %rax,%rdi
   0x00000000004005bc <+26>:	callq  0x400460 <memset@plt> // 对应memset(&c, 0, sizeof(c));
   0x00000000004005c1 <+31>:	mov    -0x18(%rbp),%ebx      // 将c->top放到寄存器ebx
   0x00000000004005c4 <+34>:	lea    -0x20(%rbp),%rax      // 将c地址放到寄存器rax
   0x00000000004005c8 <+38>:	mov    $0x20,%esi            // 将20放到寄存器esi, 作为faaa的第二个参数
   0x00000000004005cd <+43>:	mov    %rax,%rdi             // rdi作为faaa的第一个参数
   0x00000000004005d0 <+46>:	callq  0x40057d <faaa>       // 调用函数faaa
   0x00000000004005d5 <+51>:	sub    $0x20,%eax            // faaa返回值(返回值放eax寄存器) - 32, 这里编译器做了调整a - (b - c) =>  a + (c - b)
   0x00000000004005d8 <+54>:	add    %ebx,%eax             // 加上“缓存”中的c->top
   0x00000000004005da <+56>:	mov    %eax,-0x18(%rbp)
   0x00000000004005dd <+59>:	mov    -0x18(%rbp),%eax
   0x00000000004005e0 <+62>:	mov    %eax,%esi
   0x00000000004005e2 <+64>:	mov    $0x400690,%edi
   0x00000000004005e7 <+69>:	mov    $0x0,%eax
   0x00000000004005ec <+74>:	callq  0x400450 <printf@plt>
   0x00000000004005f1 <+79>:	mov    $0x0,%eax
   0x00000000004005f6 <+84>:	add    $0x18,%rsp
   0x00000000004005fa <+88>:	pop    %rbx
   0x00000000004005fb <+89>:	pop    %rbp
   0x00000000004005fc <+90>:	retq   
End of assembler dump.
```
重点来了, 加上的是`缓存`的c->top。
解决方案：
  将行`c.top -= 32 - faaa(&c, 32);`拆为两行:
```
int n = faaa(&c, 32);
c.top -= (32 - n);
```
