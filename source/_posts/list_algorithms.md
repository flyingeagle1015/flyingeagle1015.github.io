---
title: 链表常见算法
date: 2019-03-21 14:47:09
tags: [c, list]
categories: [develop]
---

# 1.引言
单链表的操作算法是笔试面试中较为常见的题目。本文将着重介绍平时面试中常见的关于链表的应用题目。

# 2.输出单链表倒数第 K 个节点
## 2.1 问题描述
题目：输入一个单链表，输出此链表中的倒数第 K 个节点。（去除头结点，节点计数从 1 开始）。

## 2.2 两次遍历法
### 2.2.1 解题思想

（1）遍历单链表，遍历同时得出链表长度 N 。
（2）再次从头遍历，访问至第 N - K 个节点为所求节点。

<!-- more -->

### 2.2.2 代码实现
```
const ListNode* searchNodeK(const ListNode* pHead, int k)
{
    int len = 0;
    const ListNode *temp = pHead;
 
    while (temp) {
      ┊ len++;
      ┊ temp = temp->next;
    }
 
    if (len < k) {
      ┊ return NULL;
    }                                                                                                                                                                                      
 
    temp = pHead;
    len = len - k;
    while (len--) {
        temp = temp->next;
    }

    return temp;
}
```
采用这种遍历方式需要两次遍历链表，时间复杂度为O(n※2)。可见这种方式最为简单,也较好理解，但是效率低下。

## 2.3 递归法
### 2.3.1 解题思想

（1）定义num = k
（2）使用递归方式遍历至链表末尾。
（3）由末尾开始返回，每返回一次 num 减 1
（4）当 num 为 0 时，即可找到目标节点

### 2.3.2 代码实现
```
const ListNode* findKthTail_in(const ListNode* pHead, int k, int *num)
{
    const ListNode *temp = NULL;
    if (!pHead) {
      ┊ return NULL;
    }

    temp = searchNodeK_in(pHead->next, k, num);
    if (temp) {
      ┊ return temp;
    } else {
      ┊ *num = (*num) - 1;
      ┊ if (*num == 0) {
        ┊ ┊ return pHead;
      ┊ } else {
        ┊ ┊ return NULL;
      ┊ }
    }
}

const ListNode* findKthTail(const ListNode* pHead, int k)
{
    int num = k;
    if (!pHead) {
      ┊ return NULL;
    }

    return searchNodeK_in(pHead, k, &num);
}
```
采用这种方式只需要一次遍历链表，时间复杂度为O(n), 但链表节点数量巨大的时候占用太多栈空间。

## 2.4 双指针法
### 2.4.1 解题思想

（1）定义两个指针 p1 和 p2 分别指向链表头节点。
（2）p1 前进 K 个节点，则 p1 与 p2 相距 K 个节点。
（3）p1，p2 同时前进，每次前进 1 个节点。
（4）当 p1 指向到达链表末尾，由于 p1 与 p2 相距 K 个节点，则 p2 指向目标节点。

### 2.4.2 代码实现
```
const ListNode* findKthTail(const ListNode* pHead, int k)
{
    int i;
    const ListNode *p1 = pHead;
    const ListNode *p2 = pHead;
    
    for (i = 0; i < k && p2; i++) {
      ┊ p2 = p2->next;
    }
    
    if (!p2) {
        return NULL;
    }
    
    while (p2) {
        p1 = p1->next;
        p2 = p2->next;
    }
    
    return p1;
}
```

# 3. 链表中存在环问题
## 3.1 判断链表是否有环
### 3.1.1 哈希缓存法
#### 3.1.1.1 解题思想
（1）首先创建一个以节点 ID 为键的 HashSe t集合，用来存储曾经遍历过的节点。
（2）从头节点开始，依次遍历单链表的每一个节点。
（3）每遍历到一个新节点，就用新节点和 HashSet 集合当中存储的节点作比较，如果发现 HashSet 当中存在相同节点 ID，则说明链表有环，如果 HashSet 当中不存在相同的节点 ID，就把这个新节点 ID 存入 HashSet ，之后进入下一节点，继续重复刚才的操作。
假设从链表头节点到入环点的距离是 a ，链表的环长是 r 。而每一次 HashSet 查找元素的时间复杂度是 O(1), 所以总体的时间复杂度是 1 * ( a + r ) = a + r，可以简单理解为 O(n) 。而算法的空间复杂度还是 a + r - 1，可以简单地理解成 O(n) 。

### 3.1.2 快慢指针法
#### 3.1.2.1 解题思想

（1）定义两个指针分别为 slow，fast，并且将指针均指向链表头节点。
（2）规定，slow 指针每次前进 1 个节点，fast 指针每次前进两个节点。
（3）当 slow 与 fast 相等，且二者均不为空，则链表存在环。

#### 3.1.2.2 代码实现
```
int isExistLoop(const ListNode* pHead)
{
    const ListNode *fastPt = pHead;
    const ListNode *slowPt = pHead;

    // fastPt在前, fastPt不为空则slowPt不可能为空
    while (fastPt && fastPt->next) {
        slowPt = slowPt->next;
        fastPt = fastPt->next->next;
      
        if (slowPt == fastPt){
            return 1;
        }
    }

    return 0;
}
```

## 3.2 定位环入口
### 3.2.1 解题思想
```
链表头l         环入口
|               |
---------------------------
                |         |
                |         | - 相遇点p
                |         |
                -----------
假定链表头l到环入口点相距a个节点（即移动a次到达），相遇点距环入口为b个节点，相遇点再移动c次再次到达环入口,则：
2*(a+b) = a+b + n * (b + c) 即慢指针的移动数的两倍 = 快指针移动数(第一次到达相遇点 +  N圈环长度)
n * (b + c) = a + b
a = (n - 1)(b + c) + c  可以得出a = (n - 1)圈  + c
指针p1从链表头到环入口的换，同样速度的指针p2从相遇点到出发移动n-1圈后会与p1在环入口相遇。
```

### 3.2.2 代码实现
```
const ListNode* getEntryNodeOfLoop(const ListNode* pHead)
{
    const ListNode *fastPt = pHead;
    const ListNode *slowPt = pHead;

    // 取得相遇点
    while (fastPt && fastPt->next) {
        slowPt = slowPt->next;
        fastPt = fastPt->next->next;
      
        if (slowPt == fastPt){
            break;
        }
    }

    // assert(fastPt); // 确定有环
  
    slowPt = pHead;
    // 再次相遇
    while (slowPt != fastPt) {
        slowPt = slowPt->next;
        fastPt = fastPt->next;
    }

    return slowPt;
}
```

## 3.3 计算环长度
### 3.3.1 解题思想
```
链表头l         环入口
|               |
---------------------------
                |         |
                |         | - 相遇点p
                |         |
                -----------
从相遇点出发，按慢指针移动一次快指针移动两次的规则，假定满指针转一圈快指针刚好转两圈，即再次相遇。
慢指针的移动次数即为环长度。
```

### 3.3.2 代码实现
```
int getLoopLength(const ListNode* pHead)
{
    int length = 0;
    const ListNode *fastPt = pHead;
    const ListNode *slowPt = pHead;

    // 取得相遇点
    while (fastPt && fastPt->next) {
        slowPt = slowPt->next;
        fastPt = fastPt->next->next;
      
        if (slowPt == fastPt){
            break;
        }
    }

    // assert(fastPt); // 确定有环
  
    // 再次相遇
    while (slowPt != fastPt) {
        slowPt = slowPt->next;
        fastPt = fastPt->next->next;
        length++;
    }

    return length;
}
```

# 4. 使用链表实现大数加法
## 4.1 问题描述
两个用链表代表的整数，其中每个节点包含一个数字。数字存储按照在原来整数中相反的顺序，使得第一个数字位于链表的开头。写出一个函数将两个整数相加，用链表形式返回和。
例如：
输入：
3->1->5->null 
5->9->2->null，
输出：
8->0->8->null

## 4.2 代码实现
```
ListNode* numberAddAsList(const ListNode *l1, const ListNode *l2)
{
  int pos = 0;
  ListNode *ret = NULL;
  ListNode *it = NULL;
  const ListNode *it1 = l1;
  const ListNode *it2 = l2;
  
  while (it1 || it2) {
  ┊ if (it1) {
  ┊ ┊ pos += it1->data;
  ┊ ┊ it1 = it1->next;
  ┊ }
  ┊ if (it2) {
  ┊ ┊ pos += it2->data;
  ┊ ┊ it2 = it2->next;
  ┊ }

  ┊ if (!it) {
  ┊ ┊ it = malloc(sizeof(*it));
  ┊ ┊ it->data = pos % 10;
  ┊ ┊ it->next = NULL;
  ┊ ┊ ret = it;
  ┊ } else {
  ┊ ┊ it->next = malloc(sizeof(*it));
  ┊ ┊ it->next->data = pos % 10;
  ┊ ┊ it->next->next = NULL;
  ┊ ┊ it = it->next;
  ┊ }
  ┊ pos /= 10;
  }

  if (pos) {
  ┊ it->next = malloc(sizeof(*it));
  ┊ it->next->data = pos;
  ┊ it->next->next = NULL;
  }
  
  return ret;
}
```

# 5. 有序链表合并
## 5.1 问题描述
题目：将两个有序链表合并为一个新的有序链表并返回。新链表是通过拼接给定的两个链表的所有节点组成的。
示例：
输入：
1->2->4, 
1->3->4 
输出：
1->1->2->3->4->4

## 5.2 通常解法
### 5.2.1 解题思想
（1）对空链表存在的情况进行处理，假如 pHead1 为空则返回 pHead2 ，pHead2 为空则返回 pHead1。（两个都为空此情况在pHead1为空已经被拦截）
（2）在两个链表无空链表的情况下确定第一个结点，比较链表1和链表2的第一个结点的值，将值小的结点保存下来为合并后的第一个结点。并且把第一个结点为最小的链表向后移动一个元素。
（3）继续在剩下的元素中选择小的值，连接到第一个结点后面，并不断next将值小的结点连接到第一个结点后面，直到某一个链表为空。
（4）当两个链表长度不一致时，也就是比较完成后其中一个链表为空，此时需要把另外一个链表剩下的元素都连接到第一个结点的后面。

### 5.2.2 代码实现
```
ListNode* mergeTwoOrderedLists(ListNode *pHead1, ListNode *pHead2)
{
  ListNode *pNewHead = NULL;
  ListNode *pTail = NULL;

  if (!pHead1 || !pHead2) {
    return (pHead1 ? pHead1 : pHead2);
  }

  if (pHead2->data < pHead1->data) {
    pNewHead = pHead2;
    pHead2 = pHead2->next;
  } else {
    pNewHead = pHead1;
    pHead1 = pHead1->next;
  }

  pTail = pNewHead;
  while (pHead1 && pHead2) {
    if (pHead2->data < pHead1->data) {
      pTail->next = pHead2;
      pHead2 = pHead2->next;
    } else {
      pTail->next = pHead1;
      pHead1 = pHead1->next;
    }

    pTail = pTail->next;
  }

  pTail->next = pHead1 ? pHead1 : pHead2;
  return pNewHead;
}
```

## 5.3 递归解法
### 5.3.1 解题思想
（1）对空链表存在的情况进行处理，假如 pHead1 为空则返回 pHead2 ，pHead2 为空则返回 pHead1。
（2）比较两个链表第一个结点的大小，确定头结点的位置
（3）头结点确定后，继续在剩下的结点中选出下一个结点去链接到第二步选出的结点后面，然后在继续重复（2 ）（3） 步，直到有链表为空。

### 5.3.2 代码实现
```
ListNode* mergeTwoOrderedLists(ListNode *pHead1, ListNode *pHead2)
{
  if (!pHead1 || !pHead2) {
    return (pHead1 ? pHead1 : pHead2);
  }

  if (pHead2->data < pHead1->data) {
    pHead2->next = mergeTwoOrderedLists(pHead1, pHead2->next);
    return pHead2;
  } else {
    pHead1->next = mergeTwoOrderedLists(pHead1->next, pHead2);
    return pHead1;
  }
}
```

# 6. 删除链表中节点，要求时间复杂度为O(1)
## 6.1 问题描述
给定一个单链表中的表头和一个等待被删除的节点。请在 O(1) 时间复杂度删除该链表节点。并在删除该节点后，返回表头。
示例：
给定 1->2->3->4，和节点 3，返回 1->2->4。

## 6.2 解题思想
在之前介绍的单链表删除节点中，最普通的方法就是遍历链表，复杂度为O(n)。 
如果我们把删除节点的下一个结点的值赋值给要删除的结点，然后删除这个结点，这相当于删除了需要删除的那个结点。因为我们很容易获取到删除节点的下一个节点，所以复杂度只需要O(1)。
示例 
单链表：1->2->3->4->NULL 
若要删除节点 3 。第一步将节点3的下一个节点的值4赋值给当前节点。变成 1->2->4->4->NULL，然后将就 4 这个结点删除，就达到目的了。 1->2->4->NULL
如果删除的节点的是头节点，把头结点指向 NULL。 
如果删除的节点的是尾节点，那只能从头遍历到头节点的上一个结点。

## 6.3 代码实现
```
void deleteNode(ListNode **pHead, ListNode *pDelNode)
{
  ListNode *pTmpNode = NULL;
  if (*pHead == NULL || pDelNode == NULL) {
    return;
  }

  // 删除头节点
  if (*pHead == pDelNode) {
    *pHead = pDelNode->next;
    free(pDelNode);
    return;
  }

  if (pDelNode->next) {
    pTmpNode = pDelNode->next;
    pDelNode->data = pTmpNode->data;
    pDelNode->next = pTmpNode->next;
    free(pTmpNode);
  } else { // 删除尾节点
    pTmpNode = *pHead;
    while (pTmpNode->next != pDelNode) {
      pTmpNode = pTmpNode->next;
    }
    
    pTmpNode->next = NULL;
    free(pDelNode);
  }
}
```

# 7. 从尾到头打印链表
## 7.1 问题描述
输入一个链表，按链表值从尾到头的顺序返回一个 ArrayList 。

## 7.2 解法
初看题目意思就是输出的时候链表尾部的元素放在前面，链表头部的元素放在后面。这不就是 先进后出，后进先出 么。
什么数据结构符合这个要求？
`栈`！

代码实现：
```
void printListFromTailToHead(ListNode* head)
{
  // stack<int> stk;
  const ListNode *tmp = head;
  while (tmp) {
    // stk.push(tmp->data)
    tmp = tmp->next;
  }

  // stk.pop
}
```

## 7.3 解法二
第二种方法也比较容易想到，通过链表的构造，如果将末尾的节点存储之后，剩余的链表处理方式还是不变，所以可以使用递归的形式进行处理。

代码实现：
```
void printListFromTailToHead(ListNode* head)
{
  if (!head) {
    return;
  }

  printListFromTailToHead(head->next);
  printf("%d ", head->data);
}
```

# 8. 反转链表
## 8.1 题目描述
反转一个单链表。
示例:
输入: 1->2->3->4->5->NULL
输出: 5->4->3->2->1->NULL

## 8.2 解题思路
设置三个节点pre、cur、next
（1）每次查看cur节点是否为NULL，如果是，则结束循环，获得结果
（2）如果cur节点不是为NULL，则先设置临时变量next为cur的下一个节点
（3）让cur的下一个节点变成指向pre，而后pre移动cur，cur移动到next
（4）重复（1）（2）（3）

## 8.3 代码实现
### 8.3.1 迭代实现
```
ListNode* reverseList(ListNode *head)
{
  ListNode *pre = NULL, *cur = NULL, *next = NULL;

  cur = head;
  while (cur) {
    next = cur->next;
    cur->next = pre;
    pre = cur;
    cur = next;
  }

  return pre;
}
```

### 8.3.2 递归实现
```
ListNode* reverseList(ListNode *head)
{
  ListNode *rhead = NULL;
  if (!head || !head->next) {
    return head;
  }

  rhead = reverseList(head->next);
  head->next->next = head;
  head->next = NULL;
  return rhead;
}
```

