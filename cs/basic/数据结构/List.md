---
title: 链表——单链表、双向链表和循环链表
date: 2019-5-15
tags: 数据结构
categories: 基础知识
toc: true
---

![listque](http://prdj7tprx.bkt.clouddn.com/ListFra.png)
以上为线性表的分类，此次我们来讨论单链表、双链表以及循环链表，难点为指针的使用。  
# 一、链表
## 1. 带头结点的链表
链表由数据域，以及指针域组成。一般是以一个指针指向此链表的头节点，但是由于这样表示的话在写代码的时候进行插入删除等操作的时候要设计的分类，所以一般采用带头结点的链表进行实现，头结点是不存储数据的。
## 2. 注意边界条件处理
1. 如果链表为空，是否能够正常工作？
2. 如果链表只包含一个结点，代码能否正常工作？
3. 如果链表只包含了两个结点，代码能否正常工作？
4. 代码逻辑在处理头结点和尾结点的时候，能否正常工作？

# 二、单链表
## 1. 基本操作
单链表的基本操作就插入、删除、查找,其中插入与删除的时间复杂度都为O(1),查找的时间复杂度为O(n);   
单链表是一种动态结构，建立线性表的链式存储结构的过程就是一个动态生成链表的过程。
```c
List *s;
//增加一个新的结点
s = (List *)malloc(sizeof(List *));
//插入,将结点s插入到链表中，分为头插入和尾插入和中间插入，此处为在位置i处插入
int j=0;
while(L->next&&j< i){
    L=L->next;
    j++;
}
p = L;
s->next = L->next;
L->next=s;

//删除
L->next = L->next->next;

```
## 3.合并两个有序链表
将其中一个链表的头结点作为最后的头节点，对两个列表中的值，依次进行比较，谁小，就把最终结点的下一个指针指向它。最后记得要释放链表b的头结点。
```c
void MergeList(LinkNode *La,LinkNode *Lb,LinNode *Lc){
    LinkNode *pa = La->next;
    LinkNode *pb = Lb->next;
    LinkNode *pc ;
    Lc = La;
    pc = Lc;//pc一开始指向没有数据的头结点

    while(pa&&pb){
        if(pa->data<= pb->data){
            pc->next = pa;
            pc = pa;
            pa = pa->next;
        }else{
            pc->next = pb;
            pc = pb;
            pb = pb->next;
        }
        pa->next?pc->next=pa:pc->next=pb;
        free(Lb);
    }
}
```

## 4.逆序输出
如果用双链表实现逆序输出则会相当容易，按照双链表的思路，则把所有指针反过来即可，但是存在一下几个问题：
1. 如何将当前指针所指向的结点，指向前一个结点，这就说明要一个指针指向前面的结点。
2. 当前指针指向前面的结点的时候，是否会找不到后面的结点，则说明，要先找到当前结点的下一个结点，才能断开指向前一个结点。
3. 是否需要头结点
4. 边界处理
```c
//因为没有头节点，头指针指向的结点为逆序后的最后一个结点，此结点的next要为NULL
List *pre=NULL;
//cur指向头指针
List *cur= L;
while(cur!=NULL){
    //在断开与后面的连接之前，要先用一个指针指向下一个结点
    List *next = cur->next;
    cur->next=pre;
    //全部往后移
    pre=cur;
    cur=next;
}
```

对边界进行检验，当cur指到最后一位的时候，next为null，将cur->next=pre,此时完成了全部的逆序，所以要退出循环，pre指向了最后一位，cur赋值为null，边界检查通过。

## 5.求链表的中间结点
快慢指针找链表中点，将其看作速度，速度为2倍，同样的时间，当一个到了，另一个正好在一半的路程上。所以其中一个指针一次往后走一个，另一个一次往后走两个结点。奇数的时候，如果快指针的next为null的时候结束循环，偶数的时候，快指针为null的时候结束循环。
```c
LinkNode * LinkCentre(LinkNode *L){
    LinkNode *fast,*slow;
    fast = L->next;
    slow = L->next;
    while(fast&&fast->next){
        slow = slow->next;
        fast = fast->next->next;
    }
}
```

## 6.判断回文字符串

# 三、循环链表
循环链表就是单链表的最后一个结点的next不为null，而是指向了头结点。

# 四、双链表
通过牺牲空间来换取时间，通过双链表在许多算法中可以大量的节省时间。
```c
struct DulNode{
    ElemType data;
    DulNode *pre;
    DulNode *next;
};
```
# 五、静态链表

静态链表是将链表用数组进行存放

