# Emiller’s Guide To Nginx Module Development
## 目录（Table of Contents）

0. [先决条件（Prerequisites）](#prerequisites)
1. [Nginx模块委托的高级概述（High-Level Overview of Nginx’s Module Delegation）](#overview)
2. [Nginx模块的组成部分（Components of an Nginx Module）](#components)
   2.1. [模块配置结构体（Module Configuration Struct(s)）](#configuration-structs)
   2.2. [模块指令（Module Directives）](#directives)
   2.3. [模块上下文（The Module Context）](#context)
       2.3.1. [创建loc配置（create_loc_conf）](#create_loc_conf)
       2.3.2. [合并loc配置（merge_loc_conf）](#merge_loc_conf)
   2.4. [模块定义（The Module Definition）](#definition)
   2.5. [模块安装（Module Installation）](#installation)
3. [处理器（Handlers）](#handlers)
   3.1. [处理器结构（非代理）（Anatomy of a Handler (Non-proxying)）](#non-proxying)
       3.1.1. [获取location配置（Getting the location configuration）](#non-proxying-config)
       3.1.2. [生成响应（Generating a response）](#non-proxying-response)
       3.1.3. [发送响应头（Sending the header）](#non-proxying-header)
       3.1.4. [发送响应体（Sending the body）](#non-proxying-body)
   3.2. [上游（即代理）处理器结构（Anatomy of an Upstream (a.k.a. Proxy) Handler）](#proxying)
       3.2.1. [上游回调概述（Summary of upstream callbacks）](#proxying-summary)
       3.2.2. [create_request回调（The create_request callback）](#create_request)
       3.2.3. [process_header回调（The process_header callback）](#process_header)
       3.2.4. [保持状态（Keeping state）](#keeping-state)
   3.3. [处理器安装（Handler Installation）](#handler-installation)
4. [过滤器（Filters）](#filters)
   4.1. [响应头过滤器结构（Anatomy of a Header Filter）](#filters-header)
   4.2. [响应体过滤器结构（Anatomy of a Body Filter）](#filters-body)
   4.3. [过滤器安装（Filter Installation）](#filters-installation)
5. [负载均衡器（Load-Balancers）](#load_balancers)
   5.1. [启用指令（The enabling directive）](#lb-directive)
   5.2. [注册函数（The registration function）](#lb-registration)
   5.3. [上游初始化函数（The upstream initialization function）](#lb-upstream)
   5.4. [peer初始化函数（The peer initialization function）](#lb-peer)
   5.5. [负载均衡函数（The load-balancing function）](#lb-function)
   5.6. [peer释放函数（The peer release function）](#lb-release)
6. [编写与编译新Nginx模块（Writing and Compiling a New Nginx Module）](#compiling)
7. [高级主题（Advanced Topics）](#advanced)
8. [代码参考（Code References）](#code)

## 0. Prerequisites  
## 0. 先决条件

You should be comfortable with C. Not just "C-syntax"; you should know your way around a struct and not be scared off by pointers and function references, and be cognizant of the preprocessor. If you need to brush up, nothing beats [K&R](https://en.wikipedia.org/wiki/The_C_Programming_Language_(book)).  
你需要熟练掌握C语言。不只是了解“C语言语法”；你还需要熟悉结构体，能够轻松应对指针和函数引用，并且了解预处理器。如果需要复习，最好的资料就是[K&R](https://en.wikipedia.org/wiki/The_C_Programming_Language_(book))。

Basic understanding of HTTP is useful. You’ll be working on a web server, after all.  
具备HTTP协议的基本知识会非常有帮助。毕竟你要开发的是Web服务器模块。

You should also be familiar with Nginx’s configuration file. If you’re not, here’s the gist of it: there are four *contexts* (called *main*, *server*, *upstream*, and *location*) which can contain directives with one or more arguments. Directives in the main context apply to everything; directives in the server context apply to a particular host/port; directives in the upstream context refer to a set of backend servers; and directives in a location context apply only to matching web locations (e.g., "/", "/images", etc.) A location context inherits from the surrounding server context, and a server context inherits from the main context. The upstream context neither inherits nor imparts its properties; it has its own special directives that don’t really apply elsewhere. I’ll refer to these four contexts quite a bit, so… don’t forget them.  
你还需要熟悉Nginx的配置文件。如果不太了解，简单来说，配置文件有四种*上下文*（称为*main*、*server*、*upstream*和*location*），每种上下文可以包含一个或多个参数的指令。主（main）上下文中的指令适用于所有内容；服务器（server）上下文的指令适用于特定主机/端口；上游（upstream）上下文的指令用于一组后端服务器；location上下文的指令只在特定路径（如"/"或"/images"等）生效。location上下文会继承所属server上下文，server上下文会继承main上下文。upstream上下文既不会继承其它上下文，也不会被其它上下文继承；它有自己专用的指令，只在upstream中有效。后文会频繁提到这四种上下文，请务必牢记。

Let’s get started.  
让我们开始吧。

## 1. High-Level Overview of Nginx’s Module Delegation  
## 1. Nginx模块委托的高级概述

Nginx modules have three roles we’ll cover:  
Nginx模块主要有三种角色，我们将在本节介绍：

- *handlers* process a request and produce output  
  - **处理器（handler）**：处理请求并产生输出
- *filters* manipulate the output produced by a handler  
  - **过滤器（filter）**：对处理器产生的输出进行加工处理
- *load-balancers* choose a backend server to send a request to, when more than one backend server is eligible  
  - **负载均衡器（load-balancer）**：当有多个后端服务器可用时，选择一个服务器发送请求

Modules do all of the "real work" that you might associate with a web server: whenever Nginx serves a file or proxies a request to another server, there’s a handler module doing the work; when Nginx gzips the output or executes a server-side include, it’s using filter modules.  
模块完成了你所认为Web服务器的所有“实际工作”：每当Nginx提供文件或代理请求到其他服务器时，都是由处理器模块完成的；当Nginx对输出进行gzip压缩或执行服务器端包含时，则使用的是过滤器模块。

The "core" of Nginx simply takes care of all the network and application protocols and sets up the sequence of modules that are eligible to process a request. The de-centralized architecture makes it possible for *you* to make a nice self-contained unit that does something you want.  
Nginx的“核心”只负责所有的网络和应用协议，并设置有资格处理请求的模块序列。这种去中心化的架构让你可以创建一个功能独立、实现特定需求的模块。

Note: Unlike modules in Apache, Nginx modules are *not* dynamically linked. (In other words, they’re compiled right into the Nginx binary.)  
注意：与Apache的模块不同，Nginx的模块**不是**动态链接的。（换句话说，模块会直接被编译进Nginx的可执行文件。）

How does a module get invoked? Typically, at server startup, each handler gets a chance to attach itself to particular locations defined in the configuration; if more than one handler attaches to a particular location, only one will "win" (but a good config writer won’t let a conflict happen). Handlers can return in three ways: all is good, there was an error, or it can decline to process the request and defer to the default handler (typically something that serves static files).  
模块是如何被调用的呢？通常，在服务器启动时，每个处理器都有机会绑定到配置文件中定义的特定location；如果多个处理器绑定到同一个location，最终只有一个会“胜出”（但优秀的配置人员不会让冲突发生）。处理器有三种返回方式：处理成功、发生错误、或者拒绝处理该请求并交由默认处理器（通常是用于提供静态文件的模块）处理。

If the handler happens to be a reverse proxy to some set of backend servers, there is room for another type of module: the load-balancer. A load-balancer takes a request and a set of backend servers and decides which server will get the request. Nginx ships with two load-balancing modules: round-robin, which deals out requests like cards at the start of a poker game, and the "IP hash" method, which ensures that a particular client will hit the same backend server across multiple requests.  
如果处理器是一个反向代理，面对一组后端服务器，那么就需要另一种模块：负载均衡器。负载均衡器负责在收到请求后，从一组后端服务器中选择一个来处理。Nginx内置了两种负载均衡模块：轮询（round-robin），像发扑克一样分配请求；IP哈希（IP hash），保证同一个客户端的请求始终发送到同一个后端服务器。

If the handler does not produce an error, the filters are called. Multiple filters can hook into each location, so that (for example) a response can be compressed and then chunked. The order of their execution is determined at compile-time. Filters have the classic "CHAIN OF RESPONSIBILITY" design pattern: one filter is called, does its work, and then calls the next filter, until the final filter is called, and Nginx finishes up the response.  
如果处理器没有产生错误，就会调用过滤器。每个location可以挂载多个过滤器，例如：响应可以先被压缩、再被分块。过滤器的执行顺序在编译时确定。过滤器采用经典的“责任链”设计模式：一个过滤器被调用、完成其任务后，继续调用下一个过滤器，直到最后一个过滤器完成，Nginx才发送最终响应。

The really cool part about the filter chain is that each filter doesn’t wait for the previous filter to finish; it can process the previous filter’s output as it’s being produced, sort of like the Unix pipeline. Filters operate on *buffers*, which are usually the size of a page (4K), although you can change this in your nginx.conf. This means, for example, a module can start compressing the response from a backend server and stream it to the client before the module has received the entire response from the backend. Nice!  
过滤器链最酷的地方在于，每个过滤器不用等待前一个过滤器完全结束；它可以在前一个过滤器产生输出的同时进行处理，有点像Unix的管道。过滤器操作的是*缓冲区（buffer）*，通常每个缓冲区大小为一页（4K），当然可以在nginx.conf中修改。这意味着，模块可以在还没收到后端服务器完整响应之前，就开始压缩响应并流式发送给客户端。非常高效！

So to wrap up the conceptual overview, the typical processing cycle goes:  
总结一下，典型的处理流程如下：

> Client sends HTTP request → Nginx chooses the appropriate handler based on the location config  
> (if applicable) load-balancer picks a backend server  
> Handler does its thing and passes each output buffer to the first filter  
> First filter passes the output to the second filter  
> second to third  
> third to fourth  
> etc.  
> Final response sent to client  
>  
> 客户端发送HTTP请求 → Nginx根据location配置选择合适的处理器  
> （如有需要）负载均衡器选择后端服务器  
> 处理器处理请求并将输出缓冲区传递给第一个过滤器  
> 第一个过滤器处理后传给第二个过滤器  
> 依次类推  
> 最终将响应发送给客户端

I say "typically" because Nginx’s module invocation is *extremely* customizable. It places a big burden on module writers to define exactly how and when the module should run (I happen to think too big a burden). Invocation is actually performed through a series of callbacks, and there are a lot of them. Namely, you can provide a function to be executed:  
这里说“通常”是因为Nginx的模块调用机制非常灵活，也给模块开发者带来了很多责任，需要精确定义模块的运行时机和方式（我认为这有点过于繁琐）。实际上，模块的调用是通过一系列回调函数完成的，而且回调点非常多。你可以为如下时机提供回调函数：

- Just before the server reads the config file  
  - 在服务器读取配置文件之前
- For every configuration directive for the location and server for which it appears;  
  - 针对每个location和server的配置指令
- When Nginx initializes the main configuration  
  - Nginx初始化主配置时
- When Nginx initializes the server (i.e., host/port) configuration  
  - Nginx初始化服务器（主机/端口）配置时
- When Nginx merges the server configuration with the main configuration  
  - Nginx将服务器配置与主配置合并时
- When Nginx initializes the location configuration  
  - Nginx初始化location配置时
- When Nginx merges the location configuration with its parent server configuration  
  - Nginx将location配置与父服务器配置合并时
- When Nginx’s master process starts  
  - Nginx主进程启动时
- When a new worker process starts  
  - 新的工作进程启动时
- When a worker process exits  
  - 工作进程退出时
- When the master exits  
  - 主进程退出时
- Handling a request  
  - 处理请求时
- Filtering response headers  
  - 过滤响应头时
- Filtering the response body  
  - 过滤响应体时
- Picking a backend server  
  - 选择后端服务器时
- Initiating a request to a backend server  
  - 向后端服务器发起请求时
- *Re*-initiating a request to a backend server  
  - 重新向后端服务器发起请求时
- Processing the response from a backend server  
  - 处理来自后端服务器的响应时
- Finishing an interaction with a backend server  
  - 与后端服务器交互结束时

Holy mackerel! It’s a bit overwhelming. You’ve got a lot of power at your disposal, but you can still do something useful using only a couple of these hooks and a couple of corresponding functions. Time to dive into some modules.  
天哪，实在是太多了！你拥有强大的能力，但只需用到其中少量的钩子和回调函数，就能做很多有用的事情。接下来进入模块的实战开发吧。

## 2. Components of an Nginx Module  
## 2. Nginx模块的组成部分

As I said, you have a *lot* of flexibility when it comes to making an Nginx module. This section will describe the parts that are almost always present. It’s intended as a guide for understanding a module, and a reference for when you think you’re ready to start writing a module.  
如前所述，在开发Nginx模块时，你拥有非常大的灵活性。本节将介绍一个模块几乎总会包含的各个部分。这既是理解模块的指南，也能作为你准备编写模块时的参考。

### 2.1. Module Configuration Struct(s)  
### 2.1. 模块配置结构体

Modules can define up to three configuration structs, one for the main, server, and location contexts. Most modules just need a location configuration. The naming convention for these is `ngx_http_<module name>_(main|srv|loc)_conf_t`. Here’s an example, taken from the dav module:  
模块可以定义最多三个配置结构体，分别对应main（主）、server（服务器）、location（位置）这三种上下文。大多数模块只需要location配置。命名约定通常是`ngx_http_<module name>_(main|srv|loc)_conf_t`。下面是dav模块中的一个例子：

```c
typedef struct {
    ngx_uint_t  methods;
    ngx_flag_t  create_full_put_path;
    ngx_uint_t  access;
} ngx_http_dav_loc_conf_t;
```

Notice that Nginx has special data types (`ngx_uint_t` and `ngx_flag_t`); these are just aliases for the primitive data types you know and love (cf. [core/ngx_config.h](http://lxr.nginx.org/source/src/core/ngx_config.h?v=nginx-1.12.0#0079) if you’re curious).  
请注意，Nginx有一些特殊的数据类型（如`ngx_uint_t`和`ngx_flag_t`）；这些其实只是你熟悉的基本数据类型的别名（如果好奇可以查阅[core/ngx_config.h](http://lxr.nginx.org/source/src/core/ngx_config.h?v=nginx-1.12.0#0079)）。

The elements in the configuration structs are populated by module directives.  
配置结构体中的各项内容是通过模块指令进行赋值的。

### 2.2. Module Directives  
### 2.2. 模块指令

A module’s directives appear in a static array of `ngx_command_t`s. Here’s an example of how they’re declared, taken from a small module I wrote:  
模块的指令会出现在一个静态的`ngx_command_t`数组中。下面是我写的一个小模块的指令声明示例：

```c
static ngx_command_t  ngx_http_circle_gif_commands[] = {
    { ngx_string("circle_gif"),
      NGX_HTTP_LOC_CONF|NGX_CONF_NOARGS,
      ngx_http_circle_gif,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      NULL },

    { ngx_string("circle_gif_min_radius"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_num_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_circle_gif_loc_conf_t, min_radius),
      NULL },
      ...
      ngx_null_command
};
```

And here is the declaration of `ngx_command_t` (the struct we’re declaring), found in [core/ngx_conf_file.h](http://lxr.nginx.org/source/src/core/ngx_conf_file.h?v=nginx-1.12.0#0077):  
下面是`ngx_command_t`结构体的声明，可以在[core/ngx_conf_file.h](http://lxr.nginx.org/source/src/core/ngx_conf_file.h?v=nginx-1.12.0#0077)中找到：

```c
struct ngx_command_t {
    ngx_str_t             name;
    ngx_uint_t            type;
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
    ngx_uint_t            conf;
    ngx_uint_t            offset;
    void                 *post;
};
```

It seems like a bit much, but each element has a purpose.  
看起来元素很多，但每个都有其用途。

The `name` is the directive string, no spaces. The data type is an `ngx_str_t`, which is usually instantiated with just (e.g.) `ngx_str("proxy_pass")`. Note: an `ngx_str_t` is a struct with a `data` element, which is a string, and a `len` element, which is the length of that string. Nginx uses this data structure most places you’d expect a string.  
`name`是指令字符串，不包含空格。其数据类型为`ngx_str_t`，通常用`ngx_str("proxy_pass")`来实例化。注意：`ngx_str_t`是一个结构体，包含`data`（字符串数据）和`len`（字符串长度）两个字段。Nginx在大多数需要字符串的地方都使用这种结构。

`type` is a set of flags that indicate where the directive is legal and how many arguments the directive takes. Applicable flags, which are bitwise-OR’d, are:  
`type`是一组标志，用于指明指令在哪些上下文中合法，以及指令参数的数量。可用的标志（可进行按位或操作）包括：

- `NGX_HTTP_MAIN_CONF`: directive is valid in the main config  
  - `NGX_HTTP_MAIN_CONF`: 指令在主配置中有效
- `NGX_HTTP_SRV_CONF`: directive is valid in the server (host) config  
  - `NGX_HTTP_SRV_CONF`: 指令在服务器（主机）配置中有效
- `NGX_HTTP_LOC_CONF`: directive is valid in a location config  
  - `NGX_HTTP_LOC_CONF`: 指令在location配置中有效
- `NGX_HTTP_UPS_CONF`: directive is valid in an upstream config  
  - `NGX_HTTP_UPS_CONF`: 指令在upstream配置中有效

- `NGX_CONF_NOARGS`: directive can take 0 arguments  
  - `NGX_CONF_NOARGS`: 指令不需要参数
- `NGX_CONF_TAKE1`: directive can take exactly 1 argument  
  - `NGX_CONF_TAKE1`: 指令需要1个参数
- `NGX_CONF_TAKE2`: directive can take exactly 2 arguments  
  - `NGX_CONF_TAKE2`: 指令需要2个参数
- ...  
  - ...
- `NGX_CONF_TAKE7`: directive can take exactly 7 arguments  
  - `NGX_CONF_TAKE7`: 指令需要7个参数

- `NGX_CONF_FLAG`: directive takes a boolean ("on" or "off")  
  - `NGX_CONF_FLAG`: 指令接受布尔值（"on"或"off"）
- `NGX_CONF_1MORE`: directive must be passed at least one argument  
  - `NGX_CONF_1MORE`: 指令至少需要一个参数
- `NGX_CONF_2MORE`: directive must be passed at least two arguments  
  - `NGX_CONF_2MORE`: 指令至少需要两个参数

There are a few other options, too, see [core/ngx_conf_file.h](http://lxr.nginx.org/source/src/core/ngx_conf_file.h?v=nginx-1.12.0).  
还有一些其它选项，可以参见[core/ngx_conf_file.h](http://lxr.nginx.org/source/src/core/ngx_conf_file.h?v=nginx-1.12.0)。

The `set` struct element is a pointer to a function for setting up part of the module’s configuration; typically this function will translate the arguments passed to this directive and save an appropriate value in its configuration struct. This setup function will take three arguments:  
`set`成员是一个函数指针，用于设置模块的某部分配置；通常，这个函数会解析传给指令的参数，并把相应的值保存到配置结构体中。这个设置函数有三个参数：

1. a pointer to an `ngx_conf_t` struct, which contains the arguments passed to the directive  
   1. 一个指向`ngx_conf_t`结构体的指针，存放传给指令的参数
2. a pointer to the current `ngx_command_t` struct  
   2. 一个指向当前`ngx_command_t`结构体的指针
3. a pointer to the module’s custom configuration struct  
   3. 一个指向模块自定义配置结构体的指针

This setup function will be called when the directive is encountered. Nginx provides a number of functions for setting particular types of values in the custom configuration struct. These functions include:  
当遇到指令时会调用这个设置函数。Nginx提供了许多函数用于在自定义配置结构体中设置特定类型的值，这些函数包括：

- `ngx_conf_set_flag_slot`: translates "on" or "off" to 1 or 0  
  - `ngx_conf_set_flag_slot`: 将"on"或"off"转换为1或0
- `ngx_conf_set_str_slot`: saves a string as an `ngx_str_t`  
  - `ngx_conf_set_str_slot`: 以`ngx_str_t`形式保存字符串
- `ngx_conf_set_num_slot`: parses a number and saves it to an `int`  
  - `ngx_conf_set_num_slot`: 解析数字并保存为`int`
- `ngx_conf_set_size_slot`: parses a data size ("8k", "1m", etc.) and saves it to a `size_t`  
  - `ngx_conf_set_size_slot`: 解析数据大小（如"8k"、"1m"等），保存为`size_t`

There are several others, and they’re quite handy (see [core/ngx_conf_file.h](http://lxr.nginx.org/source/src/core/ngx_conf_file.h?v=nginx-1.12.0#0280)). Modules can also put a reference to their own function here, if the built-ins aren’t quite good enough.  
还有其他很多类似的函数，非常方便（详见[core/ngx_conf_file.h](http://lxr.nginx.org/source/src/core/ngx_conf_file.h?v=nginx-1.12.0#0280)）。如果内置函数不满足需求，模块也可以引用自己的自定义函数。

How do these built-in functions know where to save the data? That’s where the next two elements of `ngx_command_t` come in, `conf` and `offset`. `conf` tells Nginx whether this value will get saved to the module’s main configuration, server configuration, or location configuration (with `NGX_HTTP_MAIN_CONF_OFFSET`,  
`NGX_HTTP_SRV_CONF_OFFSET`, or `NGX_HTTP_LOC_CONF_OFFSET`). `offset` then specifies which part of this configuration struct to write to.  
这些内置函数如何知道该把数据保存到哪里？这就需要用到`ngx_command_t`的后两个成员：`conf`和`offset`。`conf`用于告诉Nginx这个值应该保存到主配置、服务器配置还是location配置（分别用`NGX_HTTP_MAIN_CONF_OFFSET`、`NGX_HTTP_SRV_CONF_OFFSET`或`NGX_HTTP_LOC_CONF_OFFSET`表示）。`offset`则指定了要写入配置结构体的哪个成员。

*Finally*, `post` is just a pointer to other crap the module might need while it’s reading the configuration. It’s often `NULL`.  
最后，`post`只是一个指针，用于在读取配置时存放模块可能需要的其它信息。通常情况下是`NULL`。

The commands array is terminated with `ngx_null_command` as the last element.  
命令数组最后一个元素必须是`ngx_null_command`，表示终止。

### 2.3. The Module Context  
### 2.3. 模块上下文

This is a static `ngx_http_module_t` struct, which just has a bunch of function references for creating the three configurations and merging them together. Its name is `ngx_http_<module name>_module_ctx`. In order, the function references are:  
这是一个静态的`ngx_http_module_t`结构体，包含用于创建和合并三种配置（main/server/location）的若干函数指针。命名通常为`ngx_http_<module name>_module_ctx`。依次包含的函数指针有：

- preconfiguration  
  - 预配置
- postconfiguration  
  - 后配置
- creating the main conf (i.e., do a malloc and set defaults)  
  - 创建主配置（如分配内存并设置默认值）
- initializing the main conf (i.e., override the defaults with what’s in nginx.conf)  
  - 初始化主配置（用nginx.conf中的内容覆盖默认值）
- creating the server conf  
  - 创建服务器配置
- merging it with the main conf  
  - 与主配置合并
- creating the location conf  
  - 创建location配置
- merging it with the server conf  
  - 与服务器配置合并

These take different arguments depending on what they’re doing. Here’s the struct definition, taken from [http/ngx_http_config.h](http://lxr.nginx.org/source/src/http/ngx_http_config.h?v=nginx-1.12.0#0024), so you can see the different function signatures of the callbacks:  
这些函数根据用途参数有所不同。下面是结构体定义（见[http/ngx_http_config.h](http://lxr.nginx.org/source/src/http/ngx_http_config.h?v=nginx-1.12.0#0024)），可以看到各回调函数的签名：

```c
typedef struct {
    ngx_int_t   (*preconfiguration)(ngx_conf_t *cf);
    ngx_int_t   (*postconfiguration)(ngx_conf_t *cf);

    void       *(*create_main_conf)(ngx_conf_t *cf);
    char       *(*init_main_conf)(ngx_conf_t *cf, void *conf);

    void       *(*create_srv_conf)(ngx_conf_t *cf);
    char       *(*merge_srv_conf)(ngx_conf_t *cf, void *prev, void *conf);

    void       *(*create_loc_conf)(ngx_conf_t *cf);
    char       *(*merge_loc_conf)(ngx_conf_t *cf, void *prev, void *conf);
} ngx_http_module_t;
```

You can set functions you don’t need to `NULL`, and Nginx will figure it out.  
不需要的函数可以设置为`NULL`，Nginx会自动处理。

Most handlers just use the last two: a function to allocate memory for location-specific configuration (called `ngx_http_<module name>_create_loc_conf`), and a function to set defaults and merge this configuration with any inherited configuration (called `ngx_http_<module name >_merge_loc_conf`). The merge function is also responsible for producing an error if the configuration is invalid; these errors halt server startup.  
大多数处理器只用最后两个：一个用于分配location专用配置内存（一般叫`ngx_http_<module name>_create_loc_conf`），一个用于设置默认值并将配置与继承来的配置合并（一般叫`ngx_http_<module name>_merge_loc_conf`）。合并函数还负责在配置不合法时抛出错误，这些错误会阻止服务器启动。

Here’s an example module context struct:  
下面是一个模块上下文结构体的例子：

```c
static ngx_http_module_t  ngx_http_circle_gif_module_ctx = {
    NULL,                          /* preconfiguration */
    NULL,                          /* postconfiguration */

    NULL,                          /* create main configuration */
    NULL,                          /* init main configuration */

    NULL,                          /* create server configuration */
    NULL,                          /* merge server configuration */

    ngx_http_circle_gif_create_loc_conf,  /* create location configuration */
    ngx_http_circle_gif_merge_loc_conf /* merge location configuration */
};
```

Time to dig in deep a little bit. These configuration callbacks look quite similar across all modules and use the same parts of the Nginx API, so they’re worth knowing about.  
有必要深入了解一下。这些配置回调在所有模块中都非常相似，并且都用到了Nginx的同一套API，因此很值得掌握。

#### 2.3.1. create_loc_conf  
#### 2.3.1. 创建loc配置

Here’s what a bare-bones create_loc_conf function looks like, taken from the circle_gif module I wrote (see the [source](https://github.com/evanmiller/nginx_circle_gif/blob/master/ngx_http_circle_gif_module.c)). It takes a directive struct (`ngx_conf_t`) and returns a newly created module configuration struct (in this case `ngx_http_circle_gif_loc_conf_t`).  
下面是我写的circle_gif模块中的一个最简单的create_loc_conf函数（源码见[这里](https://github.com/evanmiller/nginx_circle_gif/blob/master/ngx_http_circle_gif_module.c)）。它接收一个指令结构体（`ngx_conf_t`），并返回新创建的模块配置结构体（这里是`ngx_http_circle_gif_loc_conf_t`）。

```c
static void *
ngx_http_circle_gif_create_loc_conf(ngx_conf_t *cf)
{
    ngx_http_circle_gif_loc_conf_t  *conf;

    conf = ngx_pcalloc(cf->pool, sizeof(ngx_http_circle_gif_loc_conf_t));
    if (conf == NULL) {
        return NGX_CONF_ERROR;
    }
    conf->min_radius = NGX_CONF_UNSET_UINT;
    conf->max_radius = NGX_CONF_UNSET_UINT;
    return conf;
}
```

First thing to notice is Nginx’s memory allocation; it takes care of the `free`’ing as long as the module uses `ngx_palloc` (a `malloc` wrapper) or `ngx_pcalloc` (a `calloc` wrapper).  
首先需要注意Nginx的内存分配机制；只要模块使用`ngx_palloc`（类似`malloc`的包装器）或`ngx_pcalloc`（类似`calloc`的包装器），Nginx会自动处理释放内存。

The possible UNSET constants are `NGX_CONF_UNSET_UINT`, `NGX_CONF_UNSET_PTR`, `NGX_CONF_UNSET_SIZE`, `NGX_CONF_UNSET_MSEC`, and the catch-all `NGX_CONF_UNSET`. UNSET tell the merging function that the value should be overridden.  
可能用到的UNSET常量包括`NGX_CONF_UNSET_UINT`、`NGX_CONF_UNSET_PTR`、`NGX_CONF_UNSET_SIZE`、`NGX_CONF_UNSET_MSEC`以及万能的`NGX_CONF_UNSET`。UNSET表示该值应由合并函数覆盖。

#### 2.3.2. merge_loc_conf  
#### 2.3.2. 合并loc配置

Here’s the merging function used in the circle_gif module:  
以下是circle_gif模块使用的合并函数：

```c
static char *
ngx_http_circle_gif_merge_loc_conf(ngx_conf_t *cf, void *parent, void *child)
{
    ngx_http_circle_gif_loc_conf_t *prev = parent;
    ngx_http_circle_gif_loc_conf_t *conf = child;

    ngx_conf_merge_uint_value(conf->min_radius, prev->min_radius, 10);
    ngx_conf_merge_uint_value(conf->max_radius, prev->max_radius, 20);

    if (conf->min_radius < 1) {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, 
            "min_radius must be equal or more than 1");
        return NGX_CONF_ERROR;
    }
    if (conf->max_radius < conf->min_radius) {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, 
            "max_radius must be equal or more than min_radius");
        return NGX_CONF_ERROR;
    }

    return NGX_CONF_OK;
}
```

Notice first that Nginx provides nice merging functions for different data types (`ngx_conf_merge_<data type>_value`); the arguments are  
首先请注意，Nginx为不同数据类型提供了非常方便的合并函数（如`ngx_conf_merge_<data type>_value`）；其参数为：

1. *this* location’s value  
   1. 当前location的值
2. the value to inherit if #1 is not set  
   2. 如果#1未设置则继承的值
3. the default if neither #1 nor #2 is set  
   3. 如果#1和#2都未设置则使用的默认值

The result is then stored in the first argument. Available merge functions include `ngx_conf_merge_size_value`, `ngx_conf_merge_msec_value`, and others. See [core/ngx_conf_file.h](http://lxr.nginx.org/source/src/core/ngx_conf_file.h?v=nginx-1.12.0#0205) for a full list.  
结果最终保存在第一个参数里。可用的合并函数还包括`ngx_conf_merge_size_value`、`ngx_conf_merge_msec_value`等，完整列表见[core/ngx_conf_file.h](http://lxr.nginx.org/source/src/core/ngx_conf_file.h?v=nginx-1.12.0#0205)。

> [!NOTE]
>
> Trivia question: How do these functions write to the first argument, since the first argument is passed in by value?  
> 冷知识：这些函数是怎么写入第一个参数的呢？毕竟第一个参数是以值传递的。
>
> Answer: these functions are defined by the preprocessor (so they expand to a few "if" statements and assignments before reaching the compiler).  
> 答案：这些函数其实是由预处理器定义的（在编译前就展开成一系列if语句和赋值操作）。
>
> 

Notice also how errors are produced; the function writes something to the log file, and returns `NGX_CONF_ERROR`. That return code halts server startup. (Since the message is logged at level `NGX_LOG_EMERG`, the message will also go to standard out; FYI, [core/ngx_log.h](http://lxr.nginx.org/source/src/core/ngx_log.h?v=nginx-1.12.0#0016) has a list of log levels.)  
还要注意错误是如何产生的：函数会把信息写入日志文件，并返回`NGX_CONF_ERROR`。该返回值会阻止服务器启动。（由于日志级别为`NGX_LOG_EMERG`，消息也会输出到标准输出；相关日志级别见[core/ngx_log.h](http://lxr.nginx.org/source/src/core/ngx_log.h?v=nginx-1.12.0#0016)。）

### 2.4. The Module Definition  
### 2.4. 模块定义

Next we add one more layer of indirection, the `ngx_module_t` struct. The variable is called `ngx_http_<module name>_module`. This is where references to the context and directives go, as well as the remaining callbacks (exit thread, exit process, etc.). The module definition is sometimes used as a key to look up data associated with a particular module. The module definition usually looks like this:  
接下来我们还需要添加一层间接引用，即`ngx_module_t`结构体。变量名通常叫`ngx_http_<module name>_module`。这里会引用上下文和指令，以及其它回调函数（如退出线程、退出进程等）。模块定义有时也用于查找与特定模块相关的数据。模块定义通常如下所示：

```c
ngx_module_t  ngx_http_<module name>_module = {
    NGX_MODULE_V1,
    &ngx_http_<module name>_module_ctx, /* module context */
    ngx_http_<module name>_commands,   /* module directives */
    NGX_HTTP_MODULE,               /* module type */
    NULL,                          /* init master */
    NULL,                          /* init module */
    NULL,                          /* init process */
    NULL,                          /* init thread */
    NULL,                          /* exit thread */
    NULL,                          /* exit process */
    NULL,                          /* exit master */
    NGX_MODULE_V1_PADDING
};
```

…substituting <module name> appropriately. Modules can add callbacks for process/thread creation and death, but most modules keep things simple. (For the arguments passed to each callback, see [core/ngx_http_config.h](http://lxr.nginx.org/source/src/http/ngx_http_config.h?v=nginx-1.12.0#0024).)  
...将<module name>替换为实际模块名即可。模块可以添加进程/线程创建和销毁的回调，但大多数模块保持简单即可。（各回调的参数见[core/ngx_http_config.h](http://lxr.nginx.org/source/src/http/ngx_http_config.h?v=nginx-1.12.0#0024)。）

### 2.5. Module Installation  
### 2.5. 模块安装

The proper way to install a module depends on whether the module is a handler, filter, or load-balancer; so the details are reserved for those respective sections.  
模块的安装方式依赖于它是处理器、过滤器还是负载均衡器；具体细节将在后续相关章节说明。

## 3. Handlers  
## 3. 处理器

Now we’ll put some trivial modules under the microscope to see how they work.  
现在我们将通过分析一些简单的模块，来了解它们的工作方式。

### 3.1. Anatomy of a Handler (Non-proxying)  
### 3.1. 处理器结构（非代理）

Handlers typically do four things: get the location configuration, generate an appropriate response, send the header, and send the body. A handler has one argument, the request struct. A request struct has a lot of useful information about the client request, such as the request method, URI, and headers. We’ll go over these steps one by one.  
处理器通常完成四个步骤：获取location配置、生成合适的响应、发送响应头、发送响应体。处理器函数有一个参数，即请求结构体。请求结构体包含很多关于客户端请求的有用信息，例如请求方法、URI和请求头。我们将逐步讲解这些步骤。

#### 3.1.1. Getting the location configuration  
#### 3.1.1. 获取location配置

This part’s easy. All you need to do is call `ngx_http_get_module_loc_conf` and pass in the current request struct and the module definition. Here’s the relevant part of my circle gif handler:  
这部分很简单。只需要调用`ngx_http_get_module_loc_conf`，传入当前的请求结构体和模块定义即可。下面是我的circle gif处理器的相关代码：

```c
static ngx_int_t
ngx_http_circle_gif_handler(ngx_http_request_t *r)
{
    ngx_http_circle_gif_loc_conf_t  *circle_gif_config;
    circle_gif_config = ngx_http_get_module_loc_conf(r, ngx_http_circle_gif_module);
    
}
```

Now I’ve got access to all the variables that I set up in my merge function.  
现在你就可以访问在合并函数中设置的所有变量了。

#### 3.1.2. Generating a response  
#### 3.1.2. 生成响应

This is the interesting part where modules actually do work.  
这是模块真正发挥作用的关键部分。

The request struct will be helpful here, particularly these elements:  
请求结构体在这里非常有用，尤其是以下成员：

```c
typedef struct {

/* the memory pool, used in the ngx_palloc functions */
    ngx_pool_t                       *pool; 
    ngx_str_t                         uri;
    ngx_str_t                         args;
    ngx_http_headers_in_t             headers_in;

} ngx_http_request_t;
```

`uri` is the path of the request, e.g. "/query.cgi".  
`uri`表示请求的路径，例如“/query.cgi”。

`args` is the part of the request after the question mark (e.g. "name=john").  
`args`是请求问号后的参数部分（例如“name=john”）。

`headers_in` has a lot of useful stuff, such as cookies and browser information, but many modules don’t need anything from it. See [http/ngx_http_request.h](http://lxr.nginx.org/source/src/http/ngx_http_request.h?v=nginx-1.12.0#0177) if you’re interested.  
`headers_in`包含很多有用的信息，比如cookie和浏览器信息，但许多模块并不需要用到这些。如果感兴趣可以查看[http/ngx_http_request.h](http://lxr.nginx.org/source/src/http/ngx_http_request.h?v=nginx-1.12.0#0177)。

This should be enough information to produce some useful output. The full `ngx_http_request_t` struct can be found in [http/ngx_http_request.h](http://lxr.nginx.org/source/src/http/ngx_http_request.h?v=nginx-1.12.0#0364).  
这些信息足以生成有用的输出。完整的`ngx_http_request_t`结构体定义可以在[http/ngx_http_request.h](http://lxr.nginx.org/source/src/http/ngx_http_request.h?v=nginx-1.12.0#0364)中找到。

#### 3.1.3. Sending the header  
#### 3.1.3. 发送响应头

The response headers live in a struct called `headers_out` referenced by the request struct. The handler sets the ones it wants and then calls `ngx_http_send_header(r)`. Some useful parts of `headers_out` include:  
响应头信息存放在请求结构体引用的`headers_out`结构体中。处理器可以设置需要的响应头，然后调用`ngx_http_send_header(r)`。`headers_out`的一些有用成员包括：

```c
typedef struct {

    ngx_uint_t                        status;
    size_t                            content_type_len;
    ngx_str_t                         content_type;
    ngx_table_elt_t                  *content_encoding;
    off_t                             content_length_n;
    time_t                            date_time;
    time_t                            last_modified_time;

} ngx_http_headers_out_t;
```

So for example, if a module were to set the Content-Type to "image/gif", Content-Length to 100, and return a 200 OK response, this code would do the trick:  
举个例子，如果模块需要将Content-Type设置为“image/gif”，Content-Length设置为100，并返回200 OK响应，可以这样写：

```c
r->headers_out.status = NGX_HTTP_OK;
r->headers_out.content_length_n = 100;
r->headers_out.content_type.len = sizeof("image/gif") - 1;
r->headers_out.content_type.data = (u_char *) "image/gif";
ngx_http_send_header(r);
```

Most legal HTTP headers are available (somewhere) for your setting pleasure. However, some headers are a bit trickier to set than the ones you see above; for example, `content_encoding` has type `(ngx_table_elt_t*)`, so the module must allocate memory for it. This is done with a function called `ngx_list_push`, which takes in an `ngx_list_t` (similar to an array) and returns a reference to a newly created member of the list (of type `ngx_table_elt_t`). The following code sets the Content-Encoding to "deflate" and sends the header:  
大多数合法的HTTP响应头都可以设置。不过，有些头部比上述示例更难设置，比如`content_encoding`的类型是`(ngx_table_elt_t*)`，因此模块必须为其分配内存。这可以通过`ngx_list_push`函数完成，该函数接收一个`ngx_list_t`（类似于数组），返回新创建的成员（类型为`ngx_table_elt_t`）。下面的代码将Content-Encoding设置为“deflate”并发送响应头：

```c
r->headers_out.content_encoding = ngx_list_push(&r->headers_out.headers);
if (r->headers_out.content_encoding == NULL) {
    return NGX_ERROR;
}
r->headers_out.content_encoding->hash = 1;
r->headers_out.content_encoding->key.len = sizeof("Content-Encoding") - 1;
r->headers_out.content_encoding->key.data = (u_char *) "Content-Encoding";
r->headers_out.content_encoding->value.len = sizeof("deflate") - 1;
r->headers_out.content_encoding->value.data = (u_char *) "deflate";
ngx_http_send_header(r);
```

This mechanism is usually used when a header can have multiple values simultaneously; it (theoretically) makes it easier for filter modules to add and delete certain values while preserving others, because they don’t have to resort to string manipulation.  
这种机制通常用于响应头可以同时拥有多个值的场景；理论上，这样可以方便过滤器模块在保留其它值的同时添加或删除某些值，而无需做字符串操作。

#### 3.1.4. Sending the body  
#### 3.1.4. 发送响应体

Now that the module has generated a response and put it in memory, it needs to assign the response to a special buffer, and then assign the buffer to a *chain link*, and *then* call the "send body" function on the chain link.  
当模块生成了响应并存入内存后，需要将响应分配到一个专用的缓冲区，然后将缓冲区关联到一个*链表节点*，最后调用链表相关的“发送响应体”函数。

What are the chain links for? Nginx lets handler modules generate (and filter modules process) responses one buffer at a time; each chain link keeps a pointer to the next link in the chain, or `NULL` if it’s the last one. We’ll keep it simple and assume there is just one buffer.  
为什么要用链表节点？Nginx允许处理器模块逐个缓冲区地生成响应（过滤器模块也可以这样处理）；每个链表节点都指向下一个节点，如果是最后一个节点则为`NULL`。这里我们只考虑单个缓冲区的简单情况。

First, a module will declare the buffer and the chain link:  
首先，模块会声明缓冲区和链表节点：

```c
ngx_buf_t    *b;
ngx_chain_t   out;
```

The next step is to allocate the buffer and point our response data to it:  
接下来要分配缓冲区，并指向我们的响应数据：

```c
b = ngx_pcalloc(r->pool, sizeof(ngx_buf_t));
if (b == NULL) {
    ngx_log_error(NGX_LOG_ERR, r->connection->log, 0, 
        "Failed to allocate response buffer.");
    return NGX_HTTP_INTERNAL_SERVER_ERROR;
}

b->pos = some_bytes; /* first position in memory of the data */
b->last = some_bytes + some_bytes_length; /* last position */

b->memory = 1; /* content is in read-only memory */
/* (i.e., filters should copy it rather than rewrite in place) */

b->last_buf = 1; /* there will be no more buffers in the request */
```

Now the module attaches it to the chain link:  
现在将缓冲区关联到链表节点：

```c
out.buf = b;
out.next = NULL;
```

FINALLY, we send the body, and return the status code of the output filter chain all in one go:  
最后，发送响应体，并一次性返回输出过滤器链的状态码：

```c
return ngx_http_output_filter(r, &out);
```

Buffer chains are a critical part of Nginx’s IO model, so you should be comfortable with how they work.  
缓冲区链是Nginx IO模型中的关键部分，建议熟悉其工作原理。

> [!NOTE]
>
> Trivia question: Why does the buffer have the `last_buf` variable, when we can tell we’re at the end of a chain by checking "next" for `NULL`?  
> 冷知识：为什么缓冲区有`last_buf`变量？我们不是可以通过判断“next”为`NULL`来确定链表结尾吗？
>
> Answer: A chain might be incomplete, i.e., have multiple buffers, but not all the buffers in this request or response. So some buffers are at the end of the chain but not the end of a request. This brings us to…  
> 答案：有时候链表并不完整，可能有多个缓冲区，但并不代表是该请求或响应的全部缓冲区。因此有些缓冲区虽然位于链表末尾，却不是请求的最后一个缓冲区。这就引出了下一个话题……

### 3.2. Anatomy of an Upstream (a.k.a Proxy) Handler  
### 3.2. 上游（即代理）处理器结构

I waved my hands a bit about having your handler generate a response. Sometimes you’ll be able to get that response just with a chunk of C code, but often you’ll want to talk to another server (for example, if you’re writing a module to implement another network protocol). You *could* do all of the network programming yourself, but what happens if you receive a partial response? You don’t want to block the primary event loop with your own event loop while you’re waiting for the rest of the response. You’d kill the Nginx’s performance. Fortunately, Nginx lets you hook right into its own mechanisms for dealing with back-end servers (called "upstreams"), so your module can talk to another server without getting in the way of other requests. This section describes how a module talks to an upstream, such as Memcached, FastCGI, or another HTTP server.  
刚才我们讨论了处理器生成响应的情况。有时你只需用一段C代码就能生成响应，但很多时候，你需要与其他服务器通信（比如实现一种新的网络协议的模块）。你当然可以自己做所有网络编程，但如果只收到部分响应怎么办？你肯定不会希望自己的事件循环阻塞主事件循环，影响Nginx性能。幸运的是，Nginx允许模块直接挂钩到其处理后端服务器（即“上游”）的机制，这样你的模块就能与其他服务器通信而不影响其它请求。本节介绍模块如何与上游（如Memcached、FastCGI或其他HTTP服务器）交互。

#### 3.2.1. Summary of upstream callbacks  
#### 3.2.1. 上游回调概述

Unlike the handler function for other modules, the handler function of an upstream module does little "real work". It does *not* call `ngx_http_output_filter`. It merely sets callbacks that will be invoked when the upstream server is ready to be written to and read from. There are actually 6 available hooks:  
与其它模块的处理器函数不同，上游模块的处理器函数本身几乎不做“实际工作”。它不会调用`ngx_http_output_filter`，而是设置一组回调函数，当与上游服务器的读写准备好时被调用。实际可用的回调点有6个：

- `create_request` crafts a request buffer (or chain of them) to be sent to the upstream  
  - `create_request`：构造要发送给上游的请求缓冲区（或缓冲区链）
- `reinit_request` is called if the connection to the back-end is reset (just before `create_request` is called for the second time)  
  - `reinit_request`：如果与后端的连接被重置（在第二次调用`create_request`之前），会调用此回调
- `process_header` processes the first bit of the upstream’s response, and usually saves a pointer to the upstream’s "payload"  
  - `process_header`：解析上游响应的头部，通常会保存指向上游“负载”的指针
- `abort_request` is called if the client aborts the request  
  - `abort_request`：如果客户端中断请求则调用
- `finalize_request` is called when Nginx is finished reading from the upstream  
  - `finalize_request`：Nginx读取完上游响应后调用
- `input_filter` is a body filter that can be called on the response body (e.g., to remove a trailer)  
  - `input_filter`：对响应体进行处理的过滤器（例如移除尾部）

How do these get attached? An example is in order. Here’s a simplified version of the proxy module’s handler:  
这些回调是如何绑定的？下面是proxy模块处理器的简化版示例：

```c
static ngx_int_t
ngx_http_proxy_handler(ngx_http_request_t *r)
{
    ngx_int_t                   rc;
    ngx_http_upstream_t        *u;
    ngx_http_proxy_loc_conf_t  *plcf;

    plcf = ngx_http_get_module_loc_conf(r, ngx_http_proxy_module);

/* set up our upstream struct */
    u = ngx_pcalloc(r->pool, sizeof(ngx_http_upstream_t));
    if (u == NULL) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }

    u->peer.log = r->connection->log;
    u->peer.log_error = NGX_ERROR_ERR;

    u->output.tag = (ngx_buf_tag_t) &ngx_http_proxy_module;

    u->conf = &plcf->upstream;

/* attach the callback functions */
    u->create_request = ngx_http_proxy_create_request;
    u->reinit_request = ngx_http_proxy_reinit_request;
    u->process_header = ngx_http_proxy_process_status_line;
    u->abort_request = ngx_http_proxy_abort_request;
    u->finalize_request = ngx_http_proxy_finalize_request;

    r->upstream = u;

    rc = ngx_http_read_client_request_body(r, ngx_http_upstream_init);

    if (rc >= NGX_HTTP_SPECIAL_RESPONSE) {
        return rc;
    }

    return NGX_DONE;
}
```

It does a bit of housekeeping, but the important parts are the callbacks. Also notice the bit about `ngx_http_read_client_request_body`. That’s setting another callback for when Nginx has finished reading from the client.  
代码中有一些初始化的操作，但关键是设置这些回调函数。还要注意`ngx_http_read_client_request_body`，它用于设置Nginx从客户端读取数据完毕时的回调。

What will each of these callbacks do? Usually, `reinit_request`, `abort_request`, and `finalize_request` will set or reset some sort of internal state and are only a few lines long. The real workhorses are `create_request` and `process_header`.  
这些回调函数通常做些什么？一般来说，`reinit_request`、`abort_request`和`finalize_request`只用于设置和重置内部状态，代码很短。真正负责处理的主要是`create_request`和`process_header`。

#### 3.2.2. The create_request callback  
#### 3.2.2. create_request回调

For the sake of simplicity, let’s suppose I have an upstream server that reads in one character and prints out two characters. What would my functions look like?  
为了简单起见，假设有一个上游服务器，它读取一个字符并输出两个字符。那么我们的回调函数会是什么样子？

The `create_request` needs to allocate a buffer for the single-character request, allocate a chain link for that buffer, and then point the upstream struct to that chain link. It would look like this:  
`create_request`需要为单字符请求分配缓冲区，然后为该缓冲区分配链表节点，并将上游结构体指向该链表节点。代码如下：

```c
static ngx_int_t
ngx_http_character_server_create_request(ngx_http_request_t *r)
{
/* make a buffer and chain */
    ngx_buf_t *b;
    ngx_chain_t *cl;

    b = ngx_create_temp_buf(r->pool, sizeof("a") - 1);
    if (b == NULL)
        return NGX_ERROR;

    cl = ngx_alloc_chain_link(r->pool);
    if (cl == NULL)
        return NGX_ERROR;

/* hook the buffer to the chain */
    cl->buf = b;
/* chain to the upstream */
    r->upstream->request_bufs = cl;

/* now write to the buffer */
    b->pos = "a";
    b->last = b->pos + sizeof("a") - 1;

    return NGX_OK;
}
```

That wasn’t so bad, was it? Of course, in reality you’ll probably want to use the request URI in some meaningful way. It’s available as an `ngx_str_t` in `r->uri`, and the GET paramaters are in `r->args`, and don’t forget you also have access to the request headers and cookies.  
其实并不复杂。当然，实际应用中你可能会用到请求的URI（`r->uri`，类型为`ngx_str_t`）和GET参数（`r->args`），还有请求头和cookie等信息。

#### 3.2.3. The process_header callback  
#### 3.2.3. process_header回调

Now it’s time for the `process_header`. Just as `create_request` added a pointer to the request body, `process_header` *shifts the response pointer to the part that the client will receive*. It also reads in the header from the upstream and sets the client response headers accordingly.  
接下来是`process_header`。正如`create_request`设置了请求体的指针，`process_header`会*将响应指针移动到客户端应该接收的部分*。它还会解析来自上游的响应头，并相应地设置客户端响应头。

Here’s a bare-minimum example, reading in that two-character response. Let’s suppose the first character is the "status" character. If it’s a question mark, we want to return a 404 File Not Found to the client and disregard the other character. If it’s a space, then we want to return the other character to the client along with a 200 OK response. All right, it’s not the most useful protocol, but it’s a good demonstration. How would we write this `process_header` function?  
这里给出一个最简单的例子，读取两字符响应。假设第一个字符表示“状态”，如果是问号则返回404并忽略第二个字符，如果是空格则返回第二个字符并返回200 OK。虽然协议不太实用，但可以很好地演示如何编写`process_header`函数：

```c
static ngx_int_t
ngx_http_character_server_process_header(ngx_http_request_t *r)
{
    ngx_http_upstream_t       *u;
    u = r->upstream;

    /* read the first character */
    switch(u->buffer.pos[0]) {
        case '?':
            r->header_only; /* suppress this buffer from the client */
            u->headers_in.status_n = 404;
            break;
        case ' ':
            u->buffer.pos++; /* move the buffer to point to the next character */
            u->headers_in.status_n = 200;
            break;
    }

    return NGX_OK;
}
```

That’s it. Manipulate the header, change the pointer, it’s done. Notice that `headers_in` is actually a response header struct like we’ve seen before (cf. [http/ngx_http_request.h](http://lxr.nginx.org/source/src/http/ngx_http_request.h?v=nginx-1.12.0#0177)), but it can be populated with the headers from the upstream. A real proxying module will do a lot more header processing, not to mention error handling, but you get the main idea.  
就是这样。操作响应头，调整指针即可。注意，`headers_in`其实是我们之前提到的响应头结构体（详见[http/ngx_http_request.h](http://lxr.nginx.org/source/src/http/ngx_http_request.h?v=nginx-1.12.0#0177)），可以用上游服务器的响应头来填充。真实的代理模块会处理更多头部和错误，但你已经掌握了基本思路。

But… what if we don’t have the whole header from the upstream in one buffer?  
但是，如果我们在一个缓存中没有收到完整的上游响应头怎么办？

#### 3.2.4. Keeping state  
#### 3.2.4. 保持状态

Well, remember how I said that `abort_request`, `reinit_request`, and `finalize_request` could be used for resetting internal state? That’s because many upstream modules *have* internal state. The module will need to define a *custom context struct* to keep track of what it has read so far from an upstream. This is NOT the same as the "Module Context" referred to above. That’s of a pre-defined type, whereas the custom context can have whatever elements and data you need (it’s your struct). This context struct should be instantiated inside the `create_request` function, perhaps like this:  
还记得之前说过`abort_request`、`reinit_request`和`finalize_request`可以用于重置内部状态吗？这是因为很多上游模块都需要维护内部状态。模块需要自定义一个*上下文结构体*，用来跟踪已从上游读取的数据。这与前面提到的“模块上下文”不同，后者是预定义类型，而自定义上下文结构体可以包含你需要的任何元素和数据（完全由你自己定义）。这个上下文结构体应该在`create_request`函数中实例化，例如：

```c
ngx_http_character_server_ctx_t   *p;   /* my custom context struct */

p = ngx_pcalloc(r->pool, sizeof(ngx_http_character_server_ctx_t));
if (p == NULL) {
    return NGX_HTTP_INTERNAL_SERVER_ERROR;
}

ngx_http_set_ctx(r, p, ngx_http_character_server_module);
```

That last line essentially registers the custom context struct with a particular request and module name for easy retrieval later. Whenever you need this context struct (probably in all the other callbacks), just do:  
最后一行代码实际上是将自定义上下文结构体与特定的请求和模块名关联起来，便于后续检索。以后在其它回调中需要这个上下文结构体时，只需这样做：

```c
ngx_http_proxy_ctx_t  *p;
p = ngx_http_get_module_ctx(r, ngx_http_proxy_module);
```

And `p` will have the current state. Set it, reset it, increment, decrement, shove arbitrary data in there, whatever you want. This is a great way to use a persistent state machine when reading from an upstream that returns data in chunks, again without blocking the primary event loop. Nice!  
这样`p`就拥有当前的上下文状态了。你可以设置、重置、递增、递减或存储任意数据，这为处理分块返回数据的上游提供了很好的持久状态机方案，而且不会阻塞主事件循环。非常棒！

### 3.3. Handler Installation  
### 3.3. 处理器安装

Handlers are installed by adding code to the callback of the directive that enables the module. For example, my circle gif `ngx_command_t` looks like this:  
处理器的安装是通过在指令的回调函数中添加代码实现的。例如，我的circle gif模块的`ngx_command_t`声明如下：

```c
{ ngx_string("circle_gif"),
  NGX_HTTP_LOC_CONF|NGX_CONF_NOARGS,
  ngx_http_circle_gif,
  0,
  0,
  NULL 
}
```

The callback is the third element, in this case `ngx_http_circle_gif`. Recall that the arguments to this callback are the directive struct (`ngx_conf_t`, which holds the user’s arguments), the relevant `ngx_command_t` struct, and a pointer to the module’s custom configuration struct. For my circle gif module, the function looks like:  
回调函数是第三个元素，这里是`ngx_http_circle_gif`。回忆一下，这个回调的参数包括指令结构体（`ngx_conf_t`，保存用户参数）、相关的`ngx_command_t`结构体，以及模块自定义配置结构体的指针。我的circle gif模块中的函数如下：

```c
static char *
ngx_http_circle_gif(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_http_core_loc_conf_t  *clcf;

    clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
    clcf->handler = ngx_http_circle_gif_handler;

    return NGX_CONF_OK;
}
```

There are two steps here: first, get the "core" struct for this location, then assign a handler to it. Pretty simple, eh?  
这里有两步：先获取该location的“核心”结构体，然后为其分配处理器。很简单吧？

I’ve said all I know about handler modules. It’s time to move onto filter modules, the components in the output filter chain.  
关于处理器模块我已讲解完毕。接下来我们将进入过滤器模块的内容，也就是输出过滤链中的组成部分。

## 4. Filters 过滤器

Filters manipulate responses generated by handlers. Header filters manipulate the HTTP headers, and body filters manipulate the response content.  
过滤器用于处理由处理器生成的响应。头部过滤器用于修改HTTP响应头，主体过滤器用于修改响应内容。

### 4.1. Anatomy of a Header Filter 头部过滤器结构

A header filter consists of three basic steps:  
头部过滤器包含三个基本步骤：

1. Decide whether to operate on this response  
   判断是否要对该响应进行处理
2. Operate on the response  
   对响应进行实际操作
3. Call the next filter  
   调用下一个过滤器

To take an example, here’s a simplified version of the "not modified" header filter, which sets the status to 304 Not Modified if the client’s If-Modified-Since header matches the response’s Last-Modified header. Note that header filters take in the `ngx_http_request_t` struct as the only argument, which gets us access to both the client headers and soon-to-be-sent response headers.  
举个例子，这里有一个简化版的"未修改"头部过滤器。如果客户端的 If-Modified-Since 请求头与响应的 Last-Modified 响应头相匹配，则状态被设置为 304 Not Modified。注意，头部过滤器只接收一个参数，即 `ngx_http_request_t` 结构体，可用于访问客户端请求头和即将发送的响应头。

```c
static
ngx_int_t ngx_http_not_modified_header_filter(ngx_http_request_t *r)
{
    time_t  if_modified_since;

    if_modified_since = ngx_http_parse_time(
                              r->headers_in.if_modified_since->value.data,
                              r->headers_in.if_modified_since->value.len);

/* step 1: decide whether to operate */
    if (if_modified_since != NGX_ERROR &&
        if_modified_since == r->headers_out.last_modified_time) {

/* step 2: operate on the header */
        r->headers_out.status = NGX_HTTP_NOT_MODIFIED;
        r->headers_out.content_type.len = 0;
        ngx_http_clear_content_length(r);
        ngx_http_clear_accept_ranges(r);
    }

/* step 3: call the next filter */
    return ngx_http_next_header_filter(r);
}
```

The `headers_out` structure is just the same as we saw in the section about handlers (cf. [http/ngx_http_request.h](http://lxr.nginx.org/source/src/http/ngx_http_request.h?v=nginx-1.12.0#0249)), and can be manipulated to no end.  
`headers_out` 结构体和前面处理器章节介绍的一致（参考 [http/ngx_http_request.h](http://lxr.nginx.org/source/src/http/ngx_http_request.h?v=nginx-1.12.0#0249)），可以灵活操作。

---

### 4.2. Anatomy of a Body Filter 主体过滤器结构

The buffer chain makes it a little tricky to write a body filter, because the body filter can only operate on one buffer (chain link) at a time.  
缓冲区链让编写Body过滤器变得有些棘手，因为主体过滤器一次只能处理一个缓冲区（链节点）。

The module must decide whether to *overwrite* the input buffer, *replace* the buffer with a newly allocated buffer, or *insert* a new buffer before or after the buffer in question.  
模块必须决定是**覆盖**输入缓冲区、用新分配的缓冲区**替换**原缓冲区，还是在相关缓冲区之前或之后**插入**新的缓冲区。

To complicate things, sometimes a module will receive several buffers so that it has an *incomplete buffer chain* that it must operate on.  
更复杂的是，有时模块会收到多个缓冲区，需要处理一个**不完整的缓冲区链**。

Unfortunately, Nginx does not provide a high-level API for manipulating the buffer chain, so body filters can be difficult to understand (and to write).  
不幸的是，Nginx 没有提供高级API用于操作缓冲区链，因此主体过滤器往往难以理解（也难编写）。

But, here are some operations you might see in action.  
不过，下面是你可能会遇到的一些操作场景。

A body filter’s prototype might look like this (example taken from the "chunked" filter in the Nginx source):  
主体过滤器的原型可能如下（示例取自Nginx源码中的chunked过滤器）：

```c
static ngx_int_t ngx_http_chunked_body_filter(ngx_http_request_t *r, ngx_chain_t *in);
```

The first argument is our old friend the request struct. The second argument is a pointer to the head of the current partial chain (which could contain 0, 1, or more buffers).  
第一个参数是我们熟悉的请求结构体。第二个参数是指向当前部分链头部的指针（可能包含0、1或多个缓冲区）。

Let’s take a simple example. Suppose we want to insert the text "<!-- Served by Nginx -->" to the end of every request. First, we need to figure out if the response’s final buffer is included in the buffer chain we were given. Like I said, there’s not a fancy API, so we’ll be rolling our own for loop:  
举个简单例子，假设我们想在每个响应结尾插入文本 "<!-- Served by Nginx -->"。首先，我们需要确定缓冲区链中是否包含了响应的最后一个缓冲区。正如之前所说，没有高级API，所以我们需要自己写循环：

```c
ngx_chain_t *chain_link;
int chain_contains_last_buffer = 0;

chain_link = in;
for ( ; ; ) {
    if (chain_link->buf->last_buf)
        chain_contains_last_buffer = 1;
    if (chain_link->next == NULL)
        break;
    chain_link = chain_link->next;
}
```

Now let’s bail out if we don’t have that last buffer:  
如果没有最后一个缓冲区则直接退出：

```c
if (!chain_contains_last_buffer)
    return ngx_http_next_body_filter(r, in);
```

Super, now the last buffer is stored in chain_link. Now we allocate a new buffer:  
很好，现在最后一个缓冲区存储在 chain_link 中。接下来分配一个新的缓冲区：

```c
ngx_buf_t    *b;
b = ngx_calloc_buf(r->pool);
if (b == NULL) {
    return NGX_ERROR;
}
```

And put some data in it:  
然后将数据写入缓冲区：

```c
b->pos = (u_char *) "<!-- Served by Nginx -->";
b->last = b->pos + sizeof("<!-- Served by Nginx -->") - 1;
```

And hook the buffer into a new chain link:  
再将缓冲区挂载到新的链节点：

```c
ngx_chain_t   *added_link;

added_link = ngx_alloc_chain_link(r->pool);
if (added_link == NULL)
    return NGX_ERROR;

added_link->buf = b;
added_link->next = NULL;
```

Finally, hook the new chain link to the final chain link we found before:  
最后，将新链节点连接到之前找到的最后一个链节点：

```c
chain_link->next = added_link;
```

And reset the "last_buf" variables to reflect reality:  
重置 "last_buf" 变量以反映真实情况：

```c
chain_link->buf->last_buf = 0;
added_link->buf->last_buf = 1;
```

And pass along the modified chain to the next output filter:  
然后将修改过的链传递给下一个输出过滤器：

```c
return ngx_http_next_body_filter(r, in);
```

The resulting function takes much more effort than what you’d do with, say, mod_perl (`$response->body =~ s/$/<!-- Served by mod_perl -->/`), but the buffer chain is a very powerful construct, allowing programmers to process data incrementally so that the client gets something as soon as possible.  
最终实现的函数要比在 mod_perl 里操作麻烦很多（比如 `$response->body =~ s/$/<!-- Served by mod_perl -->/`），但缓冲区链是一个非常强大的结构，允许程序员以增量方式处理数据，使客户端能尽快收到响应。

However, in my opinion, the buffer chain desperately needs a cleaner interface so that programmers can’t leave the chain in an inconsistent state. For now, manipulate it at your own risk.  
不过我认为，缓冲区链确实需要一个更清晰的接口，以避免程序员把链弄成不一致状态。现在只能自己小心操作。

---

### 4.3. Filter Installation 过滤器安装

Filters are installed in the post-configuration step. We install both header filters and body filters in the same place.  
过滤器是在 post-configuration 阶段安装的。头部过滤器和主体过滤器都在此阶段安装。

Let’s take a look at the chunked filter module for a simple example. Its module context looks like this:  
我们以 chunked 过滤器模块为例，模块上下文如下：

```c
static ngx_http_module_t  ngx_http_chunked_filter_module_ctx = {
    NULL,                                  /* preconfiguration */
    ngx_http_chunked_filter_init,          /* postconfiguration */
  ...
};
```

Here’s what happens in `ngx_http_chunked_filter_init`:  
下面是 `ngx_http_chunked_filter_init` 中的内容：

```c
static ngx_int_t
ngx_http_chunked_filter_init(ngx_conf_t *cf)
{
    ngx_http_next_header_filter = ngx_http_top_header_filter;
    ngx_http_top_header_filter = ngx_http_chunked_header_filter;

    ngx_http_next_body_filter = ngx_http_top_body_filter;
    ngx_http_top_body_filter = ngx_http_chunked_body_filter;

    return NGX_OK;
}
```

What’s going on here? Well, if you remember, filters are set up with a CHAIN OF RESPONSIBILITY. When a handler generates a response, it calls two functions: `ngx_http_output_filter`, which calls the global function reference `ngx_http_top_body_filter`; and `ngx_http_send_header`, which calls the global function reference `ngx_http_top_header_filter`.  
这里做了什么？如你所知，过滤器采用了责任链模式。当处理器生成响应时，会调用两个函数：`ngx_http_output_filter`（调用全局函数指针 `ngx_http_top_body_filter`）和 `ngx_http_send_header`（调用全局指针 `ngx_http_top_header_filter`）。

`ngx_http_top_body_filter` and `ngx_http_top_header_filter` are the respective "heads" of the body and header filter chains. Each "link" on the chain keeps a function reference to the next link in the chain (the references are called `ngx_http_next_body_filter` and `ngx_http_next_header_filter`). When a filter is finished executing, it just calls the next filter, until a specially defined "write" filter is called, which wraps up the HTTP response.  
`ngx_http_top_body_filter` 和 `ngx_http_top_header_filter` 分别是主体和头部过滤器链的“头”。链上的每个节点都保存着指向下一个节点的函数指针（分别是 `ngx_http_next_body_filter` 和 `ngx_http_next_header_filter`）。当一个过滤器执行完毕后，会调用下一个过滤器，直到到达一个特殊的“写”过滤器，把 HTTP 响应收尾。

What you see in this filter_init function is the module adding itself to the filter chains; it keeps a reference to the old "top" filters in its own "next" variables and declares *its* functions to be the new "top" filters. (Thus, the last filter to be installed is the first to be executed.)  
你在 filter_init 函数里看到的，就是模块将自身加入到过滤器链：它在自己的“next”变量里保存之前的“top”过滤器的引用，然后把自己的函数声明为新的“top”过滤器。（因此，最后安装的过滤器会最先被执行。）

> [!NOTE]
>
> Side note: how does this work exactly?  
> 附注：这种机制到底是如何工作的？
>
> Each filter either returns an error code or uses this as the return statement:  
> 每个过滤器要么返回错误码，要么像这样返回：
>
> ```c
> return ngx_http_next_body_filter();
> ```
>
> Thus, if the filter chain reaches the (specially-defined) end of the chain, an "OK" response is returned, but if there’s an error along the way, the chain is cut short and Nginx serves up the appropriate error message. It’s a singly-linked list with fast failures implemented solely with function references. Brilliant.  
> 因此，如果过滤器链到达（特殊定义的）末尾，则返回“OK”响应；如果中途有错误，链就会被截断，Nginx 会返回相应的错误信息。这实际上就是一个通过函数指针实现的单链表，并且可以快速失败。很巧妙！

## 5. Load-Balancers 负载均衡器

A load-balancer is just a way to decide which backend server will receive a particular request; implementations exist for distributing requests in round-robin fashion or hashing some information about the request. This section will describe both a load-balancer’s installation and its invocation, using the upstream_hash module ([full source](https://github.com/evanmiller/nginx_upstream_hash/blob/master/ngx_http_upstream_hash_module.c)) as an example. upstream_hash chooses a backend by hashing a variable specified in nginx.conf.
负载均衡器就是一种决定哪个后端服务器会接收特定请求的方法；实现方式可以是轮询分发请求，也可以是根据请求的某些信息进行哈希。本节将描述负载均衡器的安装和调用，以 upstream_hash 模块（[完整源码](https://github.com/evanmiller/nginx_upstream_hash/blob/master/ngx_http_upstream_hash_module.c)）为例。upstream_hash 通过对 nginx.conf 中指定的变量进行哈希选择后端服务器。

A load-balancing module has six pieces:
一个负载均衡模块有六个部分：

1. The enabling configuration directive (e.g, `hash;`) will call a *registration function*
启用配置指令（例如 `hash;`）会调用一个注册函数
2. The registration function will define the legal `server` options (e.g., `weight=`) and register an *upstream initialization function*
注册函数会定义合法的 `server` 选项（例如 `weight=`），并注册一个upstream 初始化函数
3. The upstream initialization function is called just after the configuration is validated, and it:
   upstream 初始化函数在配置验证后被调用，它：
   
    - resolves the `server` names to particular IP addresses
      解析 `server` 名称为具体的 IP 地址
    - allocates space for sockets
      分配套接字空间
    - sets a callback to the *peer initialization function*
      设置一个回调到peer 初始化函数
   
4. the peer initialization function, called once per request, sets up data structures that the *load-balancing function* will access and manipulate;
peer初始化函数，每个请求调用一次，用于设置负载均衡函数会访问和操作的数据结构；
5. the load-balancing function decides where to route requests; it is called at least once per client request (more, if a backend request fails). This is where the interesting stuff happens.
负载均衡函数决定请求路由的位置；它在每个客户端请求至少被调用一次（如果后端请求失败则会调用更多次）。这里是主要的逻辑发生地。
6. and finally, the *peer release function* can update statistics after communication with a particular backend server has finished (whether successfully or not)
最后，peer 释放函数可在与特定后端服务器通信结束后（无论成功与否）更新统计信息

It’s a lot, but I’ll break it down into pieces.
内容较多，下面逐项分解说明。

### 5.1. The enabling directive 启用指令

Directive declarations, recall, specify both where they’re valid and a function to call when they’re encountered. A directive for a load-balancer should have the `NGX_HTTP_UPS_CONF` flag set, so that Nginx knows this directive is only valid inside an `upstream` block. It should provide a pointer to a *registration function*. Here’s the directive declaration from the upstream_hash module:
指令声明既指定了指令的有效范围，也指定了被遇到时调用的函数。负载均衡器的指令应设置 `NGX_HTTP_UPS_CONF` 标志，这样 Nginx 就知道该指令只在 `upstream` 块内有效。它还应提供一个指向注册函数的指针。下面是 upstream_hash 模块中的指令声明：

```c
{ ngx_string("hash"),
      NGX_HTTP_UPS_CONF|NGX_CONF_NOARGS,
      ngx_http_upstream_hash,
      0,
      0,
      NULL },
```

Nothing new there.
没什么新内容。

### 5.2. The registration function 注册函数

The callback `ngx_http_upstream_hash` above is the registration function, so named (by me) because it registers an *upstream initialization function* with the surrounding `upstream` configuration. In addition, the registration function defines which options to the `server` directive are legal inside this particular `upstream` block (e.g., `weight=`, `fail_timeout=`). Here’s the registration function of the upstream_hash module:
上面的回调 `ngx_http_upstream_hash` 就是注册函数，这个名字是我自己取的，因为它会将一个upstream 初始化函数注册到所在的 `upstream` 配置块。此外，注册函数还会定义在该 `upstream` 块内 `server` 指令可以使用哪些选项（比如 `weight=`、`fail_timeout=`）。下面是 upstream_hash 模块的注册函数：

```c
ngx_http_upstream_hash(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
 {
    ngx_http_upstream_srv_conf_t  *uscf;
    ngx_http_script_compile_t      sc;
    ngx_str_t                     *value;
    ngx_array_t                   *vars_lengths, *vars_values;

    value = cf->args->elts;

    /* the following is necessary to evaluate the argument to "hash" as a $variable */
    ngx_memzero(&sc, sizeof(ngx_http_script_compile_t));

    vars_lengths = NULL;
    vars_values = NULL;

    sc.cf = cf;
    sc.source = &value[1];
    sc.lengths = &vars_lengths;
    sc.values = &vars_values;
    sc.complete_lengths = 1;
    sc.complete_values = 1;

    if (ngx_http_script_compile(&sc) != NGX_OK) {
        return NGX_CONF_ERROR;
    }
    /* end of $variable stuff */

    uscf = ngx_http_conf_get_module_srv_conf(cf, ngx_http_upstream_module);

    /* the upstream initialization function */
    uscf->peer.init_upstream = ngx_http_upstream_init_hash;

    uscf->flags = NGX_HTTP_UPSTREAM_CREATE;

    /* OK, more $variable stuff */
    uscf->values = vars_values->elts;
    uscf->lengths = vars_lengths->elts;

    /* set a default value for "hash_method" */
    if (uscf->hash_function == NULL) {
        uscf->hash_function = ngx_hash_key;
    }

    return NGX_CONF_OK;
 }
```

Aside from jumping through hoops so we can evaluation `$variable` later, it’s pretty straightforward; assign a callback, set some flags. What flags are available?
除了为后续的 `$variable` 处理做一些特殊操作之外，流程很直接：分配回调，设置一些标志。都有哪些标志可用呢？

- `NGX_HTTP_UPSTREAM_CREATE`: let there be `server` directives in this upstream block. I can’t think of a situation where you wouldn’t use this.
    `NGX_HTTP_UPSTREAM_CREATE`：允许在该 upstream 块内有 `server` 指令。几乎总是会用到这个。
- `NGX_HTTP_UPSTREAM_WEIGHT`: let the `server` directives take a `weight=` option
    NGX_HTTP_UPSTREAM_WEIGHT`：允许 `server` 指令带有 `weight=` 选项
- `NGX_HTTP_UPSTREAM_MAX_FAILS`: allow the `max_fails=` option
    `NGX_HTTP_UPSTREAM_MAX_FAILS`：允许 `max_fails=` 选项
- `NGX_HTTP_UPSTREAM_FAIL_TIMEOUT`: allow the `fail_timeout=` option
    `NGX_HTTP_UPSTREAM_FAIL_TIMEOUT`：允许 `fail_timeout=` 选项
- `NGX_HTTP_UPSTREAM_DOWN`: allow the `down` option
    `NGX_HTTP_UPSTREAM_DOWN`：允许 `down` 选项
- `NGX_HTTP_UPSTREAM_BACKUP`: allow the `backup` option
    `NGX_HTTP_UPSTREAM_BACKUP`：允许 `backup` 选项

Each module will have access to these configuration values. *It’s up to the module to decide what to do with them.* That is, `max_fails` will not be automatically enforced; all the failure logic is up to the module author. More on that later. For now, we still haven’t finished followed the trail of callbacks. Next up, we have the upstream initialization function (the `init_upstream` callback in the previous function).
每个模块都能访问这些配置值。*具体怎么处理由模块作者决定*。比如 `max_fails` 并不会自动生效，所有的故障处理逻辑都需要模块作者自己实现。稍后会讲更多细节。现在还没完全介绍完回调链，接下来是 upstream 初始化函数（即前面函数中的 `init_upstream` 回调）。

### 5.3. The upstream initialization function upstream 初始化函数

The purpose of the upstream initialization function is to resolve the host names, allocate space for sockets, and assign (yet another) callback. Here’s how upstream_hash does it:
upstream 初始化函数的目的是解析主机名、分配套接字空间，并再指定一个回调。下面是 upstream_hash 的实现方式：

```c
ngx_int_t
ngx_http_upstream_init_hash(ngx_conf_t *cf, ngx_http_upstream_srv_conf_t *us)
{
    ngx_uint_t                       i, j, n;
    ngx_http_upstream_server_t      *server;
    ngx_http_upstream_hash_peers_t  *peers;

    /* set the callback */
    us->peer.init = ngx_http_upstream_init_upstream_hash_peer;

    if (!us->servers) {
        return NGX_ERROR;
    }

    server = us->servers->elts;

    /* figure out how many IP addresses are in this upstream block. */
    /* remember a domain name can resolve to multiple IP addresses. */
    for (n = 0, i = 0; i < us->servers->nelts; i++) {
        n += server[i].naddrs;
    }

    /* allocate space for sockets, etc */
    peers = ngx_pcalloc(cf->pool, sizeof(ngx_http_upstream_hash_peers_t)
            + sizeof(ngx_peer_addr_t) * (n - 1));

    if (peers == NULL) {
        return NGX_ERROR;
    }

    peers->number = n;

    /* one port/IP address per peer */
    for (n = 0, i = 0; i < us->servers->nelts; i++) {
        for (j = 0; j < server[i].naddrs; j++, n++) {
            peers->peer[n].sockaddr = server[i].addrs[j].sockaddr;
            peers->peer[n].socklen = server[i].addrs[j].socklen;
            peers->peer[n].name = server[i].addrs[j].name;
        }
    }

    /* save a pointer to our peers for later */
    us->peer.data = peers;

    return NGX_OK;
}
```

This function is a bit more involved than one might hope. Most of the work seems like it should be abstracted, but it’s not, so that’s what we live with. One strategy for simplifying things is to call the upstream initialization function of another module, have it do all the dirty work (peer allocation, etc), and then override the `us->peer.init` callback afterwards. For an example, see [ngx_http_upstream_ip_hash_module.c](http://lxr.nginx.org/source/src/http/modules/ngx_http_upstream_ip_hash_module.c?v=nginx-1.12.0#0090).
该函数比你想象的要繁琐一些。很多工作应该被抽象出来，但实际上并没有，所以只能按照现有方式处理。简化步骤的一种方法是调用其他模块的 upstream 初始化函数，让其完成所有繁琐工作（peer 分配等），然后再覆盖 `us->peer.init` 回调。可以参考 [ngx_http_upstream_ip_hash_module.c](http://lxr.nginx.org/source/src/http/modules/ngx_http_upstream_ip_hash_module.c?v=nginx-1.12.0#0090) 作为示例。

The important bit from our point of view is setting a pointer to the *peer initialization function*, in this case `ngx_http_upstream_init_upstream_hash_peer`.
我们关心的主要是设置一个指向peer 初始化函数的指针，这里是 `ngx_http_upstream_init_upstream_hash_peer`。

### 5.4. The peer initialization function peer 初始化函数

The peer initialization function is called once per request. It sets up a data structure that the module will use as it tries to find an appropriate backend server to service that request; this structure is persistent across backend re-tries, so it’s a convenient place to keep track of the number of connection failures, or a computed hash value. By convention, this struct is called `ngx_http_upstream_<module name>_peer_data_t`.
peer 初始化函数每次请求调用一次。它会设置一个数据结构，模块会在查找合适的后端服务器时使用该结构；该结构在后端重试时可以保持状态，是跟踪连接失败次数或计算哈希值的好地方。通常该结构命名为 `ngx_http_upstream_<模块名>_peer_data_t`。

In addition, the peer initalization function sets up two callbacks:
此外，peer 初始化函数还要设置两个回调：

- `get`: the load-balancing function
    `get`：负载均衡函数
- `free`: the peer release function (usually just updates some statistics when a connection finishes)
    `free`：peer 释放函数（通常在连接结束时更新统计信息）

As if that weren’t enough, it also initalizes a variable called `tries`. As long as `tries` is positive, nginx will keep retrying this load-balancer. When `tries` is zero, nginx will give up. It’s up to the `get` and `free` functions to set `tries` appropriately.
除此之外，还要初始化一个名为 `tries` 的变量。只要 `tries` 为正，nginx 会不断重试该负载均衡器。`tries` 为零时，nginx 会放弃。`get` 和 `free` 函数负责合理设置 `tries`。

Here’s a peer initialization function from the upstream_hash module:
下面是 upstream_hash 模块的 peer 初始化函数示例：

```c
static ngx_int_t
ngx_http_upstream_init_hash_peer(ngx_http_request_t *r,
    ngx_http_upstream_srv_conf_t *us)
{
    ngx_http_upstream_hash_peer_data_t     *uhpd;
    
    ngx_str_t val;

    /* evaluate the argument to "hash" */
    if (ngx_http_script_run(r, &val, us->lengths, 0, us->values) == NULL) {
        return NGX_ERROR;
    }

    /* data persistent through the request */
    uhpd = ngx_pcalloc(r->pool, sizeof(ngx_http_upstream_hash_peer_data_t)
	    + sizeof(uintptr_t) 
	      * ((ngx_http_upstream_hash_peers_t *)us->peer.data)->number 
                  / (8 * sizeof(uintptr_t)));
    if (uhpd == NULL) {
        return NGX_ERROR;
    }

    /* save our struct for later */
    r->upstream->peer.data = uhpd;

    uhpd->peers = us->peer.data;

    /* set the callbacks and initialize "tries" to "hash_again" + 1*/
    r->upstream->peer.free = ngx_http_upstream_free_hash_peer;
    r->upstream->peer.get = ngx_http_upstream_get_hash_peer;
    r->upstream->peer.tries = us->retries + 1;

    /* do the hash and save the result */
    uhpd->hash = us->hash_function(val.data, val.len);

    return NGX_OK;
}
```

That wasn’t so bad. Now we’re ready to pick an upstream server.
其实并不复杂。现在已经准备好选择 upstream 服务器了。

### 5.5. The load-balancing function 负载均衡函数

It’s time for the main course. The real meat and potatoes. This is where the module picks an upstream. The load-balancing function’s prototype looks like:
终于到核心部分了。模块将在这里选择 upstream。负载均衡函数原型如下：

```c
static ngx_int_t 
ngx_http_upstream_get_<module_name>_peer(ngx_peer_connection_t *pc, void *data);
```

`data` is our struct of useful information concerning this client connection. `pc` will have information about the server we’re going to connect to. The job of the load-balancing function is to fill in values for `pc->sockaddr`, `pc->socklen`, and `pc->name`. If you know some network programming, then those variable names might be familiar; but they’re actually not very important to the task at hand. We don’t care what they stand for; we just want to know where to find appropriate values to fill them.
`data` 是本次客户端连接的相关信息结构。`pc` 包含我们要连接的服务器信息。负载均衡函数的工作就是为 `pc->sockaddr`、`pc->socklen` 和 `pc->name` 填充值。如果你了解网络编程，这些变量名可能很熟悉；但实际任务中并不重要，我们只关心去哪里找合适的值填充进去。

This function must find a list of available servers, choose one, and assign its values to `pc`. Let’s look at how upstream_hash does it.
该函数需要找到可用服务器列表，选中一个，然后将其值赋给 `pc`。下面看 upstream_hash 的做法。

upstream_hash previously stashed the server list into the `ngx_http_upstream_hash_peer_data_t` struct back in the call to `ngx_http_upstream_init_hash` (above). This struct is now available as `data`:
upstream_hash 在调用 `ngx_http_upstream_init_hash` 时已经把服务器列表存入了 `ngx_http_upstream_hash_peer_data_t` 结构，现在可以通过 `data` 访问：

```c
ngx_http_upstream_hash_peer_data_t *uhpd = data;
```

The list of peers is now stored in `uhpd->peers->peer`. Let’s pick a peer from this array by dividing the computed hash value by the number of servers:
peers 列表现在存储在 `uhpd->peers->peer`。通过将计算出的哈希值对服务器数量取模来选择一个：

```c
ngx_peer_addr_t *peer = &uhpd->peers->peer[uhpd->hash % uhpd->peers->number];
```

Now for the grand finale:
最后一步：

```c
pc->sockaddr = peer->sockaddr;
pc->socklen  = peer->socklen;
pc->name     = &peer->name;

return NGX_OK;
```

That’s all! If the load-balancer returns `NGX_OK`, it means, "go ahead and try this server". If it returns `NGX_BUSY`, it means all the backend hosts are unavailable, and Nginx should try again.
就这些！负载均衡函数返回 `NGX_OK` 表示“可以尝试这个服务器”，返回 `NGX_BUSY` 表示所有后端主机都不可用，Nginx 应该再试一次。

But… how do we keep track of what’s unavailable? And what if we don’t want it to try again?
但是……我们如何记录哪些服务器不可用？如果不希望继续重试怎么办？

### 5.6. The peer release function peer 释放函数

The peer release function operates after an upstream connection takes place; its purpose is to track failures. Here is its function prototype:
peer 释放函数在 upstream 连接完成后执行；其作用是跟踪失败。函数原型如下：

```c
void 
ngx_http_upstream_free_<module name>_peer(ngx_peer_connection_t *pc, void *data, 
    ngx_uint_t state);
```

The first two parameters are just the same as we saw in the load-balancer function. The third parameter is a `state` variable, which indicates whether the connection was successful. It may contain two values bitwise OR’d together: `NGX_PEER_FAILED` (the connection failed) and `NGX_PEER_NEXT` (either the connection failed, or it succeeded but the application returned an error). Zero means the connection succeeded.
前两个参数与负载均衡函数一样。第三个参数 `state` 表示连接是否成功。它可能包含两个值的按位或：`NGX_PEER_FAILED`（连接失败）和 `NGX_PEER_NEXT`（连接失败或应用返回错误）。为零表示连接成功。

It’s up to the module author to decide what to do about these failure events. If they are to be used at all, the results should be stored in `data`, a pointer to the custom per-request data struct.
如何处理这些失败事件由模块作者决定。如果要利用这些信息，结果应存储在 `data`（即自定义的每请求数据结构指针）中。

But the crucial purpose of the peer release function is to set `pc->tries` to zero if you don’t want Nginx to keep trying this load-balancer during this request. The simplest peer release function would look like this:
peer 释放函数最关键的作用是在你不希望 Nginx 继续重试该负载均衡器时将 `pc->tries` 设为零。最简单的 peer 释放函数如下：

```c
pc->tries = 0;
```

That would ensure that if there’s ever an error reaching a backend server, a 502 Bad Proxy error will be returned to the client.
这样可以确保如果访问后端服务器出错，客户端会收到 502 Bad Proxy 错误。

Here’s a more complicated example, taken from the upstream_hash module. If a backend connection fails, it marks it as failed in a bit-vector (called `tried`, an array of type `uintptr_t`), then keeps choosing a new backend until it finds one that has not failed.
下面是 upstream_hash 模块中的一个更复杂示例。如果某个后端连接失败，会在位向量（名为 `tried`，类型为 `uintptr_t` 数组）中标记为失败，然后不断选择新的后端，直到找到一个未失败的。

```c
#define ngx_bitvector_index(index) index / (8 * sizeof(uintptr_t))
#define ngx_bitvector_bit(index) (uintptr_t) 1 << index % (8 * sizeof(uintptr_t))

static void
ngx_http_upstream_free_hash_peer(ngx_peer_connection_t *pc, void *data,
    ngx_uint_t state)
{
    ngx_http_upstream_hash_peer_data_t  *uhpd = data;
    ngx_uint_t                           current;

    if (state & NGX_PEER_FAILED
            && --pc->tries)
    {
        /* the backend that failed */
        current = uhpd->hash % uhpd->peers->number;

       /* mark it in the bit-vector */
        uhpd->tried[ngx_bitvector_index(current)] |= ngx_bitvector_bit(current);

        do { /* rehash until we're out of retries or we find one that's untried */
            uhpd->hash = ngx_hash_key((u_char *)&uhpd->hash, sizeof(ngx_uint_t));
            current = uhpd->hash % uhpd->peers->number;
        } while ((uhpd->tried[ngx_bitvector_index(current)] & ngx_bitvector_bit(current)) && --pc->tries);
    }
}
```

This works because the load-balancing function will just look at the new value of `uhpd->hash`.
之所以有效，是因为负载均衡函数只会查看新的 `uhpd->hash` 值。

Many applications won’t need retry or high-availability logic, but it’s possible to provide it with just a few lines of code like you see here.
许多应用并不需要重试或高可用逻辑，但只需几行代码就能实现，如上所示。

## 6. Writing and Compiling a New Nginx Module 
## 编写和编译新的 Nginx 模块

So by now, you should be prepared to look at an Nginx module and try to understand what’s going on (and you’ll know where to look for help). Take a look in [src/http/modules/](http://lxr.nginx.org/source/src/http/modules/?v=nginx-1.12.0) to see the available modules. Pick a module that’s similar to what you’re trying to accomplish and look through it. Stuff look familiar? It should. Refer between this guide and the module source to get an understanding about what’s going on.
到目前为止，你应该已经准备好查看一个 Nginx 模块并尝试理解其工作原理（并且知道该去哪里获取帮助）。可以在 [src/http/modules/](http://lxr.nginx.org/source/src/http/modules/?v=nginx-1.12.0) 查看可用模块。挑选一个与你要实现的功能类似的模块，仔细阅读代码。内容是否很熟悉？应该是的。可以结合本指南和模块源码来理解其实现过程。

But Emiller didn’t write a *Balls-In Guide to Reading Nginx Modules*. Hell no. This is a *Balls-Out Guide*. We’re not reading. We're writing. Creating. Sharing with the world.
但 Emiller 并没有写一份“深入阅读 Nginx 模块指南”。绝不是。这是“一往无前的写作指南”。我们不是在读，我们是在写，在创造，在与世界分享。

First thing, you’re going to need a place to work on your module. Make a folder for your module anywhere on your hard drive, but separate from the Nginx source (and make sure you have the latest copy from [nginx.net](http://nginx.net)). Your new folder should contain two files to start with:
首先，你需要为你的模块准备一个工作目录。在你的硬盘上创建一个文件夹，用于存放你的模块代码，但要与 Nginx 源码分开（并确保你拥有最新的 Nginx 源码）。你的新文件夹一开始应该包含两个文件：

- "config"
- "ngx_http_<your module>_module.c"

The "config" file will be included by `./configure`, and its contents will depend on the type of module.
“config” 文件将被 `./configure` 命令包含，其内容取决于你的模块类型。

**"config" for filter modules:**
**过滤器模块的 “config” 文件格式：**

```bash
ngx_addon_name=ngx_http_<your module>_module
HTTP_AUX_FILTER_MODULES="$HTTP_AUX_FILTER_MODULES ngx_http_<your module>_module"
NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_<your module>_module.c"
```

**"config" for other modules:**
**其他类型模块的 “config” 文件格式：**

```bash
ngx_addon_name=ngx_http_<your module>_module
HTTP_MODULES="$HTTP_MODULES ngx_http_<your module>_module"
NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_<your module>_module.c"
```

For more information about the "config" file format and its various options, see [wiki page](https://www.nginx.com/resources/wiki/extending/old_config/) about it.
关于 “config” 文件格式及其各种选项的更多信息，请参见 [wiki 页面](https://www.nginx.com/resources/wiki/extending/old_config/)。

Now for your C file. I recommend copying an existing module that does something similar to what you want, but rename it "ngx_http_<your module>_module.c". Let this be your model as you change the behavior to suit your needs, and refer to this guide as you understand and refashion the different pieces.
接下来是你的 C 文件。我建议你先复制一个现有的、功能类似的模块，并将其重命名为 “ngx_http_<你的模块>_module.c”。以此为模板，根据需要修改其行为，并结合本指南理解和重塑各个部分。

When you’re ready to compile, just go into the Nginx directory and type
当你准备好编译时，只需进入 Nginx 目录并输入以下命令：

```bash
./configure --add-module=path/to/your/new/module/directory
```

and then `make` and `make install` like you normally would. If all goes well, your module will be compiled right in. Nice, huh? No need to muck with the Nginx source, and adding your module to new versions of Nginx is a snap, just use that same `./configure` command. By the way, if your module needs any dynamically linked libraries, you can add this to your "config" file:
然后像平常一样执行 `make` 和 `make install`。如果一切顺利，你的模块会被直接编译进去。很棒吧？无需改动 Nginx 源码，未来升级 Nginx 时只需使用相同的 `./configure` 命令即可添加你的模块。顺便说一下，如果你的模块需要动态链接库，可以在 “config” 文件中添加如下内容：

```bash
CORE_LIBS="$CORE_LIBS -lfoo"
```

Where `foo` is the library you need. If you make a cool or useful module, be sure to send a note to the [Nginx mailing list](http://wiki.codemongers.com/MailinglistSubscribe) and share your work.
其中 `foo` 是你需要的库名。如果你开发出了很酷或有用的模块，记得通知 [Nginx 邮件列表](http://wiki.codemongers.com/MailinglistSubscribe)，与大家分享你的成果。

---

## 7. Advanced Topics 高级主题

This guide covers the basics of Nginx module development. For tips on writing more sophisticated modules, be sure to check out *[Emiller’s Advanced Topics In Nginx Module Development](nginx-modules-guide-advanced.html)*.
本指南涵盖了 Nginx 模块开发的基础内容。如果你想了解编写更高级模块的技巧，记得查阅 *[Emiller 的 Nginx 模块开发高级主题](nginx-modules-guide-advanced.html)*。



## Appendix A: Code References

- [Nginx source tree (cross-referenced)](http://lxr.nginx.org/source/src/?v=nginx-1.12.0)
- [Nginx module directory (cross-referenced)](http://lxr.nginx.org/source/src/http/modules/?v=nginx-1.12.0)
- [Example addon: circle_gif](https://github.com/evanmiller/nginx_circle_gif)
- [Example addon: upstream_hash](https://github.com/evanmiller/nginx_upstream_hash)
- [Example addon: upstream_fair](http://github.com/gnosek/nginx-upstream-fair/tree/master)

## Appendix B: Changelog

- December 10, 2018: Added link documenting the ["config" file format](https://www.nginx.com/resources/wiki/extending/old_config/).
- August 11, 2017: Updated links to Nginx and module source code.
- January 16, 2013: Corrected code sample in 5.5.
- December 20, 2011: Corrected code sample in 4.2 (one more time).
- March 14, 2011: Corrected code sample in 4.2 (again).
- November 11, 2009: Corrected code sample in 4.2.
- August 13, 2009: Reorganized, and moved *[Advanced Topics](https://www.evanmiller.org/nginx-modules-guide-advanced.html)* to a separate article.
- July 23, 2009: Corrected code sample in 3.5.3.
- December 24, 2008: Corrected code sample in 3.4.
- July 14, 2008: Added information about subrequests; slight reorganization
- July 12, 2008: Added [Grzegorz Nosek](http://localdomain.pl/)’s guide to shared memory
- July 2, 2008: Corrected "config" file for filter modules; rewrote introduction; added TODO section
- May 28, 2007: Changed the load-balancing example to the simpler upstream_hash module
- May 19, 2007: Corrected bug in body filter example
- May 4, 2007: Added information about load-balancers
- April 28, 2007: Initial draft