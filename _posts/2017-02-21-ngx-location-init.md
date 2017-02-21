---
layout: post
title:  "nginx的location解析初始化和请求查找"
date:   2017-02-21 14:29:03
categories: update
---

### 简介

nginx的location支持几种不同的配置方式，并且location之间还可以嵌套。
我们通过一个配置文件来看下ngxin是如何解析和初始化location的。
最后当请求到达的时候又是如何匹配这些location的呢。


```
server {
    ......

    location = /test0/ {
    }

    location /test0/ {
        location /test0/hello {
        }
    }

    location = /test1/ {
    }
    
    location /test2 {
    }
    
    location /test22 {
    }
    
    location = /test22 {
    }
    
    location /test222 {
    }
    
    location /zack {
    }
    
}
```


### location的解析和初始化

location块的解析是调用ngx\_http\_core\_location函数解析的。
该函数把解析的结果存入ngx\_http\_core\_loc\_conf\_t结构体clcf中。
其中clcf->name为location和‘{’之间去掉(=、^~、~)的内容，
如上述配置的name为/test0/和/test1/和/test0/hello。= 表示精确匹配，
同时会把对应clcf->exact_match赋值为1。具体看下该结构体相关的变量：

```c
struct ngx_http_core_loc_conf_s {
    ngx_str_t     name;          /* location name */

    unsigned      exact_match:1;
    unsigned      noregex:1;

    unsigned      auto_redirect:1;
    ......
    ngx_http_location_tree_node_t   *static_locations;

    /* pointer to the modules' loc_conf */
    void        **loc_conf;

    ngx_http_handler_pt  handler;
    ......
    ngx_queue_t  *locations;
};
```

每解析完location块以后，ngx会调用ngx\_http\_add\_location把该clcf结构体添加到所属server下ngx\_http\_core\_loc\_conf\_t结构体pclcf的locations双链表队尾。
当解析完的location是嵌套在location内部的，该location对应的clcf结构体会被添加到所属location下的clcf结构体的locations双链表队尾。
注意一下只有前缀匹配可以嵌套location。


该链表的节点类型如下：

```c
typedef struct {
    ngx_queue_t                      queue;
    ngx_http_core_loc_conf_t        *exact;
    ngx_http_core_loc_conf_t        *inclusive;
    ngx_str_t                       *name;
    u_char                          *file_name;
    ngx_uint_t                       line;
    ngx_queue_t                      list;
} ngx_http_location_queue_t;
```

其中，queue为双链表的两个指针。exact或inclusive指向clcf结构体，name为location的name。

当解析完所有http块下所有配置后，ngx框架会循环调用ngx\_http\_init\_locations处理server块下的location配置，
该函数两个入参分别是server级别的ngx\_http\_core\_srv\_conf\_t和ngx\_http\_core\_loc\_conf\_t结构。
该函数会把clcf->locations双链表排序，排序后的结果是：同名的（精确匹配的location、前缀匹配的location）不同名的按字典升序排列、正则匹配的location。
如果location下有嵌套location时，会递归调用该函数对clcf->locations双链表排序。

为了验证，特意修改了ngx\_http\_init\_locations代码，在排序后的遍历循环里，添加了以下代码打印排序后的结果：

```
ngx_log_error(NGX_LOG_ERR, cf->log, 0, "location name: %V exact: %d",
                      lq->name, (int)(lq->exact != NULL));
```

注意一下，下面是排序cmp函数的一段代码，用到一个黑科技，该name还是指向解析配置的地址，
虽然是ngx\_str\_t结构，但是字符串是以0结尾的，
而取了ngx两个字符串长度的最小值再加1就把两个字符串中短的字符串的结束符号0也参与了比较，
0比其他字母数字的ascii码都小，所以相同的字符串前缀的较长的排到后面。

```c
rc = ngx_filename_cmp(first->name.data, second->name.data,
                          ngx_min(first->name.len, second->name.len) + 1);
```

排序处理后，ngx框架会调用ngx\_http\_init\_static\_location\_trees把双链表初始化一颗静态二叉搜索树。
首先会调用ngx\_http\_join\_exact\_locations函数，合并同名的精确匹配和前缀匹配的clcf，
具体实现就是把前缀完全匹配的clcf指针赋给前面精确匹配的ngx\_http\_location\_queue\_t结构的inclusive指针。
接着会调用ngx\_http\_create\_locations\_list函数，把前缀相同的clcf从locations摘下来保存到lq->list上。


例如前面的配置文件中以test2开始的=/test2 /test2 /test22 /test222，其中=/test2和/test2会合并，并且/test22和/test222会挂在test2的list上。
画个图在看下：

![ngx-location-tmp]({{ site.url }}/assets/ngx_location_init_tmp_1_170221.png)


上图只画了test2开始的和zack这几个location。最左边的节点为locations节点也是这个双链表的哨兵节点。
接着test2有两个location，其中一个是精确匹配的另一个是前缀匹配的，所以合并在一个链表节点上。
后面/test22 /test222 和/test2有相同的前缀，被从locations链表删除添加到test2节点的list指针上。

最终才会调用ngx\_http\_create\_locations\_tree函数创建静态二叉搜索树。
先来看下下树的节点定义：

```c
struct ngx_http_location_tree_node_s {
    ngx_http_location_tree_node_t   *left;  // 左子树
    ngx_http_location_tree_node_t   *right; // 右子树
    ngx_http_location_tree_node_t   *tree;  // 二叉搜索树根节点

    ngx_http_core_loc_conf_t        *exact;     // 精确匹配的location
    ngx_http_core_loc_conf_t        *inclusive; // 前缀匹配的location

    u_char                           auto_redirect;
    u_char                           len;     // 节点名称长度
    u_char                           name[1]; // 节点名称
};
```

因为双链表已经是排好序的，取中间节点为根节点递归的构建二叉搜索树。
lq->list在递归弹出栈后开始构建在每个节点上的tree指针上的一颗二叉搜索树。
来看下构建好的二叉搜索树：

![ngx-location-tree]({{ site.url }}/assets/ngx_location_static_tree_170221.png)

上图只画了树形结构，其中clcf就代表ngx\_http\_core\_loc_conf\_t结构的实例，
看过代码的都知道。整个这颗树的树根节点指针保存在这些location
所属的server块下的ngx\_http\_core\_loc\_conf\_t结构的static\_locations字段。
为了说清楚，我们把《nginx http 模块初始化》的一个图拿过来，
下图红色圈内的部分就是我们初始化完成的这棵二叉搜索树。

![ngx-location-tree]({{ site.url }}/assets/ngx_http_location_tree_170221.png)


### location的查找

当一个request到来的时候，ngx框架到底要让哪个location的配置起作用呢？
当解析完request头就开始让http的11个阶段介入处理请求了，
如上图所示11个阶段的第三个阶段是NGX_HTTP\_FIND\_CONFIG\_PHASE，
根据ngx启动的时候阶段初始化可知，该阶段仅可以由框架提供的ngx\_http\_core\_find\_config\_phase函数来处理，
而该函数首先取得所属server下的clcf的静态二叉搜索树根节点，最终调用了ngx\_http\_core\_find\_location
函数查找匹配的location。查找函数从根节点开始先序遍历二叉树，用uri和node->name前缀比较？

等于则判断uri和node->name的长度，

  如果uri的长度大于node->name的长度则判断该node节点是否有前缀匹配的location，

    有则再判断该node的tree树是否为空，不为空继续遍历tree这个二叉搜索树，找到返回。没找到或者tree为空返回最初匹配的location。

    没有前缀匹配的location说明该节点是精确匹配，因长度不能所以不匹配，递归遍历右子树。

  如果uri的长度等于node->name的长度，已经匹配，有精确匹配的location先返回，否则返回前缀匹配的location。

如果是前缀匹配，则还需要调用ngx\_http\_core\_find\_location检测一次是否有嵌套的location，有则覆盖原找到的location作为最终匹配的location。

查找这部分也没法画图，自己对照代码和上述画的那棵树过一遍也许比单看文字描述清晰。


end

