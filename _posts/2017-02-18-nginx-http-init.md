---
layout: post
title:  "nginx http 模块初始化"
date:   2017-02-17 12:29:12
categories: update
---

### 简介
nginx 是个web服务器，更多的情况被用作反向代理服务器。Netcraft公布的全球web占比，nginx市场占有率在逐年增加。

+ nginx优点：
 - 轻量级
 - 高并发
 - 配置简单
 - 扩展性好

### nginx 模块

nginx高度的模块化使得nginx扩展性非常好，模块之间的耦合非常低，模块开发相对简单。
nginx所有的模块开始于`ngx_module_t`结构体。该结构体设计的非常巧妙，通过`ctx`这个`void*`兼容了nginx五大模块(NGX_CORE_MODULE、NGX_EVENT_MODULE、NGX_HTTP_MODULE、NGX_STREAM_MODULE、NGX_MAIL_MODULE)的实现。如此分层的设计，使得nginx每层的代码只关心本层的逻辑，降低了开发代码的复杂度。

```c
struct ngx_module_s {
    ngx_uint_t            ctx_index;
    ngx_uint_t            index;

    char                 *name;

    ngx_uint_t            spare0;
    ngx_uint_t            spare1;

    ngx_uint_t            version;
    const char           *signature;

    void                 *ctx;
    ngx_command_t        *commands;
    ngx_uint_t            type;

    ngx_int_t           (*init_master)(ngx_log_t *log);

    ngx_int_t           (*init_module)(ngx_cycle_t *cycle);

    ngx_int_t           (*init_process)(ngx_cycle_t *cycle);
    ngx_int_t           (*init_thread)(ngx_cycle_t *cycle);
    void                (*exit_thread)(ngx_cycle_t *cycle);
    void                (*exit_process)(ngx_cycle_t *cycle);

    void                (*exit_master)(ngx_cycle_t *cycle);

    NGX_MODULE_V1_PADDING;
};
```

![nginx-module]({{ site.url }}/assets/nginx_module_170218.png)

如上图所示，不同的模块type取值不同，`*ctx`的类型不同。`*cmd` 是`ngx_command_t`类型的数组。



### 代码层级

nginx高度的模块化同样离不开代码的层级设计。nginx的框架核心代码只管理`NGX_CORE_MODULE`类型的模块，核心模块`ngx_events_module`再管理`NGX_EVENT_MODULE`类模块，核心模块`ngx_http_module`再管理`NGX_HTTP_MODULE`类型的模块。



### 代码解读

nginx的代码从`ngx_cycle_t`类型结构体开始，通过`ngx_init_cycle`函数解析配置文件并初始化该结构体类型的cycle变量中。

```c
struct ngx_cycle_s {
    void                  ****conf_ctx;

    ngx_connection_t        **files;
    ngx_connection_t         *free_connections;
    ngx_uint_t                free_connection_n;

    ngx_module_t            **modules;
    ngx_uint_t                modules_n;
    ngx_uint_t                modules_used;    /* unsigned  modules_used:1; */

    ngx_queue_t               reusable_connections_queue;

    ngx_array_t               listening;
    
    ......

    ngx_connection_t         *connections;
    ngx_event_t              *read_events;
    ngx_event_t              *write_events;

    ngx_str_t                 conf_file;
    ngx_str_t                 conf_param;

    ......
};

```


上述列出了部分变量，conf_ctx是一个指针数组，保存了所有模块的配置文件结构体指针。module也是个指针数组，保存了所有模块指针。listening存储了监听的端口。


上面说了代码是分层级的，nginx框架核心代码只关心核心模块。

核心模块有如下几个，我们重点只关心http模块，别的模块也涉及到只会简单介绍。

+ NGX\_CORE\_MODULE：
 - ngx_core_module
 - ngx_http_module
 - ngx_events_module
 - ngx_errlog_module

所有的模块基本都是先创建保存配置的结构体，然后解析配置文件，最后初始化配置。只不过有的模块简单有的复杂。而http模块是目前为止最复杂的模块。
核心模块也不例外，首先为所有核心模块创建保存配置的结构体并把指针保存到cycle->conf_ctx数组中，由上图可知，核心模块的`ctx`是 `ngx_core_module_t` 类型，该类型有一个 `ngx_core_module_create_conf` 函数指针，该函数就是用来创建对应模块配置结构体的。

创建好结构体有保存的地方了，接着会解析配置文件，框架核心代码只关心核心模块的配置解析。主要函数是 `ngx_conf_handler` ，该函数会遍历所有模块的commands指针数组(cycle->modules[i]->commands[j])根据配置key寻找处理函数。下面会分模块说明。


#### ngx_core_module

核心相关的配置，对应nginx.c文件 `ngx_core_commands` 指针数组。

#### ngx_events_module

event模块相关配置，对应ngx_event.c文件 `ngx_events_commands` 指针数组。
解析event{}，调用 `ngx_events_block()` ，管理 `NGX_EVENT_MODULE` 模块。

#### ngx_http_module

http模块相关配置，对应ngx_http.c文件 `ngx_http_commands` 指针数组。
因为该模块管理http类型( `NGX_HTTP_MODULE` )模块，需要解析http块（配置文件中的http{}），并没有直接的配置需要保存，因此 `ngx_core_module_create_conf` 指针为空。

http模块对应的处理函数为 `ngx_http_block` ，我们重点看该函数。看该函数前先了解一下http的配置，http配置分为三层，最外层的是http{}中间是server{}最里层是location{}即(http{server{location{}}})，并且每一层都可以有相同的配置，例如client_body_buffer_size配置，可以配置在http块下、server块下，location块下，也可以同时配置到这三个块下。因此导致配置结构体也错综复杂。

先创建http{}对应的上下文 `ctx` ，类型为 `ngx_http_conf_ctx_t` 。该结构体为三个指针，分别指向了三个指针数组，数组的长度是 `NGX_HTTP_MODULE` 模块的个数。其中 `main_conf` 对应的数组存存了http块下配置结构体指针，`srv_conf`指向的数组存储了server块下配置结构体指针， `loc_conf` 指向的数组存储了location块下配置结构体指针。并且http模块下每一个配置块都将对应一个 `ngx_http_conf_ctx_t` 结构体，详细见下图。

创建完http{}对应的配置结构体后，开始解析http级别的配置。而server{}也属于http级别的配置，所以会调用server对应的处理函数 `ngx_http_core_server` 。其他配置解析请参照 `ngx_http_core_commands` 数组。

上面说了每一个块都会对应一个 `ngx_http_conf_ctx_t` ，所以也会为server{}分配一个 `ctx `，其中 `main_conf` 指向http块下的配置， `srv_conf` 新分配的指针数组存储了server块下配置结构体指针， `loc_conf` 新分配的指针数组存储了location块下配置结构体指针。创建完server{}层级的结构体后开始解析server下的配置。而location{}也属于server级别的配置，所以会调用location的处理函数 `ngx_http_core_location` 。其他配置解析请参照 `ngx_http_core_commands` 数组。

相应的 `ngx_http_core_location` 函数也会创建一个 `ngx_http_conf_ctx_t` 结构体，其中 `main_conf` 指针指向server下的 `main_conf` 也就是http下的 `main_conf` ， `srv_conf` 指向server下的 `srv_conf` ， `loc_conf` 会新建，新建的指针数组存储了location块下配置结构体指针。

创建完location下的配置结构体后开始解析location级别的配置。location是可以嵌套的，也就是说如果遇到location块，还需要调用 `ngx_http_core_location` 函数处理。

至此，配置文件解析完毕，数据结构如下图所示（部分核心）：


![nginx-http-init-tmp]({{ site.url }}/assets/nginx_http_init_tmp_170218.png)


由图可知 `ngx_http_core_loc_conf_s` 结构有三个实例 `ngx_http_core_srv_conf_t` 有两个实例，正是为解决配置文件中同一个配置可以同时配置在不同的配置块中(http块、server块、location块)。

三个实例都有相同的配置，那么到底那个配置会作用于http请求呢？http模块的 `ctx` (类型： `ngx_http_module_t` )有两个函数 `ngx_http_core_merge_srv_conf` 和 `ngx_http_core_merge_loc_conf` 分别用来合并 `ngx_http_core_srv_conf_t` 的两个实例和 `ngx_http_core_loc_conf_s` 的三个实例，至于如何合并，到底是上一级结构覆盖本级结构的配置还是相加什么的。。。完全由这两个函数自己决定（一般情况都是当前配置为空就用上一级配置覆盖否则不处理），每个http模块都有自己的合并处理函数，nginx框架会统一调用。

合并完配置文件后，会完成以下几个操作：

- 为了提高location查找效率，把location列表初始化为一颗平衡二叉树。根地址保存到所属server对应结构体的 `static_location` ，见下图。
- 向11个执行阶段添加处理函数，并初始化。除 `NGX_HTTP_LOG_PHASE` 阶段外的其余10个阶段所有的处理函数被初始化为一个 `ngx_http_phase_handler_s` 类型的数组，最终保存到所属http对应结构体的 `phase_engine` 上。其中 `checker` 为nginx框架控制调用处理 `handler` 的函数， `handler` 为模块添加的函数， `next` 为下一个阶段数组下标，见下图。
- 处理IP:PORT和虚拟主机的关系，监听结构最终会添加到 `cycle->listening` 数组中。虚拟主机以hash表的形式保存到监听结构上。这部分详细看《nginx 虚拟主机》。

自此nginx初始化基本完成，内存结构如下（和真实情况有省略）：

![nginx-http-init]({{ site.url }}/assets/nginx_http_init_170218.png)

end
