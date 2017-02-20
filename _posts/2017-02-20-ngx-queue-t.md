---
layout: post
title:  "nginx 双链表设计与实现"
date:   2017-02-20 12:33:21
categories: update
---

ngx\_queue\_t 是ngx为双链表设计的数据结构。在ngx代码中很多地方用到了该结构体。

结构体类型：

```c
struct ngx_queue_s {
    ngx_queue_t  *prev;
    ngx_queue_t  *next;
};
```

成员函数：

```c
// 宏，用来初始化一个节点
ngx_queue_init(ngx_queue_t q);
// 宏，判断一个链表是否为空链表
ngx_queue_empty(ngx_queue_t q);
// 宏，在链表头部插入一个节点
ngx_queue_insert_head(ngx_queue_t h, ngx_queue_t x);
// 宏，在链表某节点后插入一个节点
ngx_queue_insert_after(ngx_queue_t h, ngx_queue_t x);
// 宏，在链表尾部插入一个节点
ngx_queue_insert_tail(ngx_queue_t h, ngx_queue_t x);
// 宏，返回链表头节点
ngx_queue_head(ngx_queue_t h);
// 宏，返回链表尾节点
ngx_queue_last(ngx_queue_t h);
// 宏，返回链表哨兵节点
ngx_queue_sentinel(ngx_queue_t h);
// 宏，返回当前节点下一个节点
ngx_queue_next(ngx_queue_t q);
// 宏，返回当前节点上一个节点
ngx_queue_prev(ngx_queue_t q);
// 宏，从x所属链表中移除x节点
ngx_queue_remove(ngx_queue_t q);
// 宏，把h这个链表从q节点分成两个链表，q作为第二个链表的头节点，n作为第二个节点的哨兵节点
ngx_queue_split(ngx_queue_t h, ngx_queue_t q, ngx_queue_t n);
// 宏，拼接两个链表，n链表的头节点接着h链表的尾节点，n作为第二个链表的哨兵节点，被移除
ngx_queue_add(ngx_queue_t h, ngx_queue_t n);
// 宏，取得q节点所属的结构体指针
ngx_queue_data(q, type, link);
// 函数，取得链表中的中间节点
// 实现：定义了两个节点，一个节点步进为1，另一个节点步进为2，
//      当后一个节点到达节点尾部时，前一个节点正好到链表中间节点
ngx_queue_t *ngx_queue_middle(ngx_queue_t *queue);
// 函数，插入排序算法排序列表。cmp是函数指针指向比较函数。
//（如果cmp函数如：return a<=b? -1? 1; 升序排序）
void ngx_queue_sort(ngx_queue_t *queue,
    ngx_int_t (*cmp)(const ngx_queue_t *, const ngx_queue_t *));
```

结构定义非常简单，只定义了两个指针。成员函数还是比较多的很多都是宏定义。

使用的时候直接把该结构体作为成员变量添加到目标结构体就可以了。这样设计灵活方便又节省内存。

nginx中使用的时候常常会设置一个sentinel节点，该节点没有值。


对应着代码实现画了个图帮助理解。如下：

![ngx_queue_t_1]({{ site.url }}/assets/ngx_queue_t_1_170220.png)

![ngx_queue_t_2]({{ site.url }}/assets/ngx_queue_t_2_170220.png)

end