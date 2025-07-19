# Emiller’s Advanced Topics In Nginx Module Development   
---

**By [Evan Miller](/) (with [Grzegorz Nosek](http://nginx.localdomain.pl/))**  
**作者：[Evan Miller](/)（与 [Grzegorz Nosek](http://nginx.localdomain.pl/) 合著）**

**DRAFT: August 13, 2009**  
**草稿：2009年8月13日**

---

Whereas *[Emiller’s Guide To Nginx Module Development](nginx-modules-guide.html)* describes the bread-and-butter issues of writing a simple handler, filter, or load-balancer for Nginx, this document covers three advanced topics for the ambitious Nginx developer: shared memory, subrequests, and parsing. Because these are subjects on the boundaries of the Nginx universe, the code here may be sparse. The examples may be out of date. But hopefully, you will make it out not only alive, but with a few extra tools in your belt.  
*《Emiller的Nginx模块开发指南》主要讲述了编写Nginx简单处理器、过滤器或负载均衡器的基础知识，而本文档则面向有志于深入Nginx开发的开发者，涵盖了三个高级话题：共享内存、子请求和解析。由于这些内容位于Nginx开发的边界地带，相关代码可能较少，示例可能已经过时。但希望你不仅能顺利读完，还能获得更多实用工具。*

---

## Table of Contents  
## 目录

1. [Shared Memory](#shm)  
   1. [A (fore)word of caution](#shm-foreword)  
   2. [Creating and using a shared memory segment](#shm-creating)  
   3. [Using the slab allocator](#shm-slab)  
   4. [Spinlocks, atomic memory access](#shm-spinlocks)  
   5. [Using rbtrees](#shm-rbtrees)
2. [Subrequests](#subrequests)  
   1. [Internal redirect](#subrequests-redirect)  
   2. [A single subrequest](#subrequests-single)  
   3. [Sequential subrequests](#subrequests-sequential)  
   4. [Parallel subrequests](#subrequests-parallel)
3. [Parsing With Ragel](#parsing) *NEW*  
   1. [Installing ragel](#parsing-install)  
   2. [Calling ragel from nginx](#parsing-call)  
   3. [Writing a grammar](#parsing-grammar)  
   4. [Writing some actions](#parsing-actions)  
   5. [Putting it all together](#parsing-finale)
4. [TODO](#todo)  

---

<a name="shm"></a>
## 1. Shared Memory  
## 1. 共享内存

*Guest chapter written by [Grzegorz Nosek](http://localdomain.pl/)*  
*本章节由 [Grzegorz Nosek](http://localdomain.pl/) 撰写*

Nginx, while being unthreaded, allows worker processes to share memory between them. However, this is quite different from the standard pool allocator as the shared segment has fixed size and cannot be resized without restarting nginx or destroying its contents in another way.  
Nginx虽然是无线程的，但允许worker进程间共享内存。不过，这与标准内存池分配器有很大不同，因为共享内存段的大小是固定的，不能在不重启nginx或销毁其内容的情况下调整大小。

### 1.1. A (fore)word of caution  
### 1.1. 前言与警告

First of all, caveat hacker. This guide has been written several months after hands-on experience with shared memory in nginx and while I try my best to be accurate (and have spent some time refreshing my memory), in no way is it guaranteed. You’ve been warned.  
首先，黑客请注意。本文是在实际操作Nginx共享内存几个月后写的，虽然我尽力保证准确性（并抽时间查阅相关内容），但并不保证绝对正确。请谨慎参考。

Also, 100% of this knowledge comes from reading the source and reverse-engineering the core concepts, so there are probably better ways to do most of the stuff described.  
此外，本文所有知识均来自于阅读源码和逆向分析核心概念，因此，大多数描述的做法可能有更好的实现方式。

Oh, and this guide is based on 0.6.31, though 0.5.x is 100% compatible AFAIK and 0.7.x also brings no compatibility-breaking changes that I know of.  
另外，本指南基于0.6.31版本，据我所知0.5.x完全兼容，0.7.x也没有破坏兼容性的变化。

For real-world usage of shared memory in nginx, see my [upstream_fair module](http://github.com/gnosek/nginx-upstream-fair/tree/master).  
关于Nginx共享内存的实际用例，请参考我的 [upstream_fair模块](http://github.com/gnosek/nginx-upstream-fair/tree/master)。

This probably does not work on Windows at all. Core dumps in the rear mirror are closer than they appear.  
这些内容在Windows下可能完全不可用。意外的core dump比你想象的离你更近。

### 1.2. Creating and using a shared memory segment  
### 1.2. 创建和使用共享内存段

To create a shared memory segment in nginx, you need to:  
要在Nginx中创建共享内存段，你需要：

- provide a constructor function to initialise the segment  
  - 提供一个构造函数初始化内存段
- call `ngx_shared_memory_add`  
  - 调用 `ngx_shared_memory_add`

These two points contain the main gotchas (that I came across), namely:  
这两点包含了我遇到的主要陷阱，即：

1. Your constructor will be called multiple times and it’s up to you to find out whether you’re called the first time (and should set something up), or not (and should probably leave everything alone). The prototype for the shared memory constructor looks like:  
   你的构造函数会被多次调用，你需要自己判断是否第一次被调用（需要初始化），还是后续调用（建议保持原状）。共享内存构造函数的原型如下：
   ```c
   static ngx_int_t init(ngx_shm_zone_t *shm_zone, void *data);
   ```
   The data variable will contain the contents of `oshm_zone->data`, where `oshm_zone` is the "old" shm zone descriptor (more about it later). This variable is the only value that can survive a reload, so you must use it if you don’t want to lose the contents of your shared memory.  
   data变量包含了`oshm_zone->data`的内容，`oshm_zone`是“旧的”共享内存区描述符（后面会详述）。这个变量是唯一能在reload后保留下来的值，所以如果你不想丢失共享内存内容，必须用它。

   Your constructor function will probably look roughly similar to the one in upstream_fair, i.e.:  
   你的构造函数大致如下：
   ```c
   static ngx_int_t
   init(ngx_shm_zone_t *shm_zone, void *data)
   {
           if (data) { /* we're being reloaded, propagate the data "cookie" */
                   shm_zone->data = data;
                   return NGX_OK;
           }
   
           /* set up whatever structures you wish to keep in the shm */
   
           /* initialise shm_zone->data so that we know we have
           been called; if nothing interesting comes to your mind, try
           shm_zone->shm.addr or, if you're desperate, (void*) 1, just set
           the value to something non-NULL for future invocations
           */
           shm_zone->data = something_interesting;
   
           return NGX_OK;
   }
   ```
   你需要初始化你想要保存在共享内存中的结构体，并确保`shm_zone->data`被赋值（非NULL），以便后续reload时判断。

2. You must be careful when to access the shm segment.  
   你必须注意何时访问共享内存段。

   The interface for adding a shared memory segment looks like:  
   添加共享内存段的接口如下：
   ```c
   ngx_shm_zone_t *
   ngx_shared_memory_add(ngx_conf_t *cf, ngx_str_t *name, size_t size,
           void *tag);
   ```
   cf 是配置文件引用（通常在响应配置选项时创建内存段），name是段名（ngx_str_t类型，即计数字符串），size是以字节为单位的大小（通常会被向上取整到页面大小，比如很多架构下4KB），tag是用来检测命名冲突的标签。如果你用相同的name、tag和size多次调用`ngx_shared_memory_add`，只会得到一个段。如果指定不同的name，会得到多个不同的段；如果指定相同的name但不同的size或tag，则会报错。一个不错的tag选择可以是你的模块描述符的指针。

   After you call `ngx_shared_memory_add` and receive the new `shm_zone` descriptor, you must set up the constructor in `shm_zone->init`. Wait… after you add the segment? Yes, and that’s a major gotcha. This implies that the segment is not created while calling `ngx_shared_memory_add` (because you specify the constructor only later). What really happens looks like this (grossly simplified):  
   调用`ngx_shared_memory_add`获得新的`shm_zone`描述符后，你必须在`shm_zone->init`中设置构造函数。注意，这一步是在添加段之后才做的，这是一个重要的陷阱。这意味着共享内存段在调用`ngx_shared_memory_add`时并未创建（因为你是之后才指定构造函数）。实际流程大致如下：

   1. parse the whole config file, noting requested shm segments  
      解析整个配置文件，记录需要的共享内存段
   2. afterwards, create/destroy all the segments in one go  
      随后，一次性创建/销毁所有内存段  
      The constructors are called here. Note that every time your ctor is called, it is with another value of `shm_zone`. The reason is that the descriptor lives as long as the cycle (generation in Apache terms) while the segment lives as long as the master and all the workers. To let *some* data survive a reload, you have access to the old descriptor’s `->data` field (mentioned above).  
      此时会调用构造函数。注意，每次调用构造函数时，`shm_zone`的值都不同。原因在于，描述符的生命周期与cycle（类似Apache的generation）一致，而内存段的生命周期与master及所有worker一致。为了让某些数据在reload时保留，你可以访问旧描述符的`->data`字段（前面已提到）。
   3. (re)start workers which begin handling requests  
      （重新）启动worker，开始处理请求
   4. upon receipt of SIGHUP, goto 1  
      收到SIGHUP信号后，回到第1步

   Also, you really must set the constructor, otherwise nginx will consider your segment unused and won’t create it at all.  
   你必须设置构造函数，否则nginx会认为你的段未被使用，根本不会创建。

   Now that you know it, it’s pretty clear that you cannot rely on having access to the shared memory while parsing the config. You can access the whole segment as `shm_zone->shm.addr` (which will be NULL before the segment gets really created). Any access after the first parsing run (e.g. inside request handlers or on subsequent reloads) should be fine.  
   了解这一点后，你会发现不能在解析配置文件时访问共享内存段。你可以通过`shm_zone->shm.addr`访问整个段（段未真正创建前该值为NULL）。第一次解析后（比如在请求处理器里或后续reload时）访问是没问题的。

### 1.3. Using the slab allocator  
### 1.3. 使用slab分配器

Now that you have your new and shiny shm segment, how do you use it? The simplest way is to use another memory tool that nginx has at your disposal, namely the slab allocator. Nginx is nice enough to initialise the slab for you in every new shm segment, so you can either use it, or ignore the slab structures and overwrite them with your own data.  
现在你已经有了新的共享内存段，如何使用？最简单的方法是使用nginx提供的另一个内存工具——slab分配器。Nginx会为每个新共享内存段自动初始化slab，所以你可以直接用它，也可以忽略slab结构，用自己的数据覆盖。

The interface consists of two functions:  
接口包含两个函数：

```c
void *ngx_slab_alloc(ngx_slab_pool_t *pool, size_t size);
void ngx_slab_free(ngx_slab_pool_t *pool, void *p);
```
The first argument is simply `(ngx_slab_pool_t *)shm_zone->shm.addr` and the other one is either the size of the block to allocate, or the pointer to the block to free. (trivia: not once is `ngx_slab_free` called in vanilla nginx code)  
第一个参数就是`(ngx_slab_pool_t *)shm_zone->shm.addr`，另外一个参数要么是要分配的块大小，要么是要释放的块指针。（趣闻：nginx原生代码中没有调用过`ngx_slab_free`）

### 1.4. Spinlocks, atomic memory access  
### 1.4. 自旋锁与原子内存访问

Remember that shared memory is inherently dangerous because you can have multiple processes accessing it at the same time. The slab allocator has a per-segment lock (`shpool->mutex`) which is used to protect the segment against concurrent modifications.  
请记住，共享内存本身是危险的，因为可能有多个进程同时访问。slab分配器为每个段都设置了锁（`shpool->mutex`），用于保护该段免受并发修改。

You can also acquire and release the lock yourself, which is useful if you want to implement some more complicated operations on the segment, like searching or walking a tree. The two snippets below are essentially equivalent:  
你也可以自己获取和释放锁，这在需要对段进行复杂操作（如查找、遍历树结构）时非常有用。下面两个代码片段实际上是等价的：

```c
/*
void *new_block;
ngx_slab_pool_t *shpool = (ngx_slab_pool_t *)shm_zone->shm.addr;
*/

new_block = ngx_slab_alloc(shpool, ngx_pagesize);
```
```c
ngx_shmtx_lock(&shpool->mutex);
new_block = ngx_slab_alloc_locked(shpool, ngx_pagesize);
ngx_shmtx_unlock(&shpool->mutex);
```
In fact, ngx_slab_alloc looks almost exactly like above.  
实际上，`ngx_slab_alloc`内部实现几乎和上面一样。

If you perform any operations which depend on no new allocations (or, more to the point, frees), protect them with the slab mutex. However, remember that nginx mutexes are implemented as spinlocks (non-sleeping), so while they are very fast in the uncontended case, they can easily eat 100% CPU when waiting. So don’t do any long-running operations while holding the mutex (especially I/O, but you should avoid any system calls at all).  
如果你的操作依赖于没有新的分配（或者说没有释放），请用slab的互斥锁保护它们。不过要记住nginx的互斥锁是自旋锁（不会睡眠），所以在无竞争情况下非常快，但如果等待就会消耗100% CPU。因此，持锁时不要做耗时操作（尤其是I/O，最好避免系统调用）。

You can also use your own mutexes for more fine-grained locking, via the `ngx_mutex_init()`, `ngx_mutex_lock()` and `ngx_mutex_unlock()` functions.  
你也可以通过`ngx_mutex_init()`、`ngx_mutex_lock()`、`ngx_mutex_unlock()`等函数实现自己的细粒度锁。

As an alternative for locks, you can use atomic variables which are guaranteed to be read or written in an uninterruptible way (no worker process may see the value halfway as it’s being written by another one).  
除了锁，你还可以用原子变量，保证读写不可中断（不会有worker在另一个worker写一半时读取到中间值）。

Atomic variables are defined with the type `ngx_atomic_t` or `ngx_atomic_uint_t` (depending on signedness). They should have at least 32 bits. To simply read or unconditionally set an atomic variable, you don’t need any special constructs:  
原子变量类型为`ngx_atomic_t`或`ngx_atomic_uint_t`（取决于有无符号），至少32位。简单读写原子变量无需特殊语法：

```c
ngx_atomic_t i = an_atomic_var;
an_atomic_var = i + 5;
```
Note that anything can happen between the two lines; context switches, execution of code on other other CPUs, etc.  
需要注意这两行之间可能发生任何事情，比如上下文切换、其他CPU上的代码执行等。

To atomically read and modify a variable, you have two functions (very platform-specific) with their interface declared in `src/os/unix/ngx_atomic.h`:  
要原子地读写变量，有两个高度依赖平台的函数，接口声明在`src/os/unix/ngx_atomic.h`：

- `ngx_atomic_cmp_set(lock, old, new)`  
  Atomically retrieves old value of `*lock` and stores `new` under the same address. Returns 1 if `*lock` was equal to `old` before overwriting.  
  原子地获取`*lock`的旧值并写入新值，如果写入前`*lock`等于`old`则返回1。
- `ngx_atomic_fetch_add(value, add)`  
  Atomically adds `add` to `*value` and returns the old `*value`.  
  原子地将`add`加到`*value`并返回原值。

### 1.5. Using rbtrees  
### 1.5. 使用红黑树

OK, you have your data neatly allocated, protected with a suitable lock but you’d also like to organise it somehow. Again, nginx has a very nice structure just for this purpose - a red-black tree.  
好了，你已经分配好了数据并用合适的锁保护，现在还需要组织这些数据。nginx为此提供了非常棒的数据结构——红黑树。

Highlights (API-wise):  
API要点如下：

- requires an insertion callback, which inserts the element in the tree (probably according to some predefined order) and then calls `ngx_rbt_red(the_newly_added_node)` to rebalance the tree  
  - 需要插入回调，用于按预定义顺序插入元素到树中，然后调用`ngx_rbt_red(the_newly_added_node)`重新平衡树
- requires all leaves to be set to a predefined sentinel object (not NULL)  
  - 所有叶节点必须设置为预定义的哨兵对象（不能为NULL）

This chapter is about shared memory, not rbtrees so shoo! Go read the source for [upstream_fair](http://github.com/gnosek/nginx-upstream-fair/tree/master) to see creating and walking an rbtree in action.  
本章主要讲共享内存，不深入讲红黑树。想了解如何创建和遍历红黑树，请参阅 [upstream_fair模块源码](http://github.com/gnosek/nginx-upstream-fair/tree/master)。

## 2. Subrequests  
## 2. 子请求

Subrequests are one of the most powerful aspects of Nginx. With subrequests, you can return the results of a *different URL* than what the client originally requested. Some web frameworks call this an "internal redirect." But Nginx goes further: not only can modules perform *multiple subrequests* and combine the outputs into a single response, subrequests can perform their own sub-subrequests, and sub-subrequests can initiate sub-sub-subrequests, and… you get the idea. Subrequests can map to files on the hard disk, other handlers, or upstream servers; it doesn’t matter from the perspective of Nginx. As far as I know, only filters can issue subrequests.  
子请求是Nginx最强大的功能之一。通过子请求，你可以返回与客户端原始请求不同URL的结果。有些web框架把这称为“内部重定向”。但Nginx更进一步：模块不仅可以发起多个子请求并将结果合并为一个响应，子请求还可以继续产生子子请求，子子请求还可以继续产生更多层级的请求……你能想象到的都可以。子请求可以映射到硬盘上的文件、其他处理器或上游服务器；对Nginx来说都没区别。据我所知，只有过滤器能发起子请求。

### 2.1. Internal redirects  
### 2.1. 内部重定向

If all you want to do is return a different URL than what the client originally requested, you will want to use the `ngx_http_internal_redirect` function. Its prototype is:  
如果你只想返回一个与客户端原请求不同的URL，可以使用`ngx_http_internal_redirect`函数。其原型如下：
```c
ngx_int_t
ngx_http_internal_redirect(ngx_http_request_t *r, ngx_str_t *uri, ngx_str_t *args)
```
Where `r` is the request struct, and `uri` and `args` are the new URI. Note that URIs *must* be locations already defined in nginx.conf; you cannot, for instance, redirect to an arbitrary domain. Handlers should return the return value of `ngx_http_internal_redirect`, i.e. redirecting handlers will typically end like  
其中`r`是请求结构体，`uri`和`args`是新的URI。注意URI *必须*是nginx.conf中已经定义的位置，不能重定向到任意域名。处理器应返回`ngx_http_internal_redirect`的返回值，重定向处理器通常这样结束：
```c
return ngx_http_internal_redirect(r, &uri, &args);
```
Internal redirects are used in the "index" module (which maps URLs that end in / to index.html) as well as Nginx’s X-Accel-Redirect feature.  
内部重定向用在“index”模块（将以斜杠结尾的URL映射到index.html）和Nginx的X-Accel-Redirect功能中。

### 2.2. A single subrequest  
### 2.2. 单个子请求

Subrequests are most useful for inserting additional content *based on data from the original response*. For example, the SSI (server-side include) module uses a filter to scan the contents of the returned document, and then replaces "include" directives with the contents of the specified URLs.  
子请求常用于根据原始响应的数据插入额外内容。例如，SSI（服务器端包含）模块会用过滤器扫描返回的文档内容，然后将“include”指令替换为指定URL的内容。

We’ll start with a simpler example. We’ll make a filter that treats the entire contents of a document as a URL to be retrieved, and then appends the new document to the URL itself. Remember that the URL must be a location in nginx.conf.  
我们先来看一个简单例子：编写一个过滤器，把文档的全部内容当作要获取的URL，然后把新文档追加到URL后面。注意URL必须是nginx.conf中定义的位置。
```c
static ngx_int_t
ngx_http_append_uri_body_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    int                 rc; 
    ngx_str_t           uri;
    ngx_http_request_t *sr;

    /* First copy the document buffer into the URI string */
    uri.len = in->buf->last - in->buf->pos;
    uri.data = ngx_palloc(r->pool, uri.len);
    if (uri.data == NULL)
        return NGX_ERROR;
    ngx_memcpy(uri.data, in->buf->pos, uri.len);

    /* Now return the original document (i.e. the URI) to the client */
    rc = ngx_http_next_body_filter(r, in);

    if (rc == NGX_ERROR)
        return rc;

    /* Finally issue the subrequest */
    return ngx_http_subrequest(r, &uri, NULL /* args */, 
        NULL /* callback */, 0 /* flags */);
}
```
The prototype of `ngx_http_subrequest` is:  
`ngx_http_subrequest`的原型如下：
```c
ngx_int_t ngx_http_subrequest(ngx_http_request_t *r,
    ngx_str_t *uri, ngx_str_t *args, ngx_http_request_t **psr, 
        ngx_http_post_subrequest_t *ps, ngx_uint_t flags)
```
Where:  
参数说明：

- `*r` is the original request  
  - `*r`为原始请求
- `*uri` and `*args` refer to the sub-request  
  - `*uri`和`*args`为子请求的参数
- `**psr` is a *reference to a NULL pointer* that will point to the new (sub-)request structure  
  - `**psr`为指向新子请求结构体的NULL指针引用
- `*ps` is a callback for when the subrequest is finished. I’ve never used this, but see [http/ngx_http_request.h](http://lxr.nginx.org/source/src/http/ngx_http_request.h?v=nginx-1.12.0#0334) for details.  
  - `*ps`为子请求完成时的回调。作者未用过，详细见[http/ngx_http_request.h](http://lxr.nginx.org/source/src/http/ngx_http_request.h?v=nginx-1.12.0#0334)
- `flags` can be a bitwise-OR’ed combination of:  
  - `flags`可以按位或组合以下选项：
  - `NGX_HTTP_ZERO_IN_URI`：URI包含ASCII码为0的字符（即'\0'），或包含"%00"
  - `NGX_HTTP_SUBREQUEST_IN_MEMORY`：将子请求结果存储在连续内存块中（通常不需要）

The results of the subrequest will be inserted where you expect. If you want to modify the results of the subrequest, you can use another filter (or the same one!). You can tell whether a filter is operating on the primary request or a subrequest with this test:  
子请求的结果会插入到你预期的位置。如果你想修改子请求结果，可以用另一个过滤器（或同一个）。可以通过以下判断当前过滤器处理的是主请求还是子请求：
```c
if (r == r->main) { 
    /* primary request */
} else {
    /* subrequest */
}
```
The simplest example of a module that issues a single subrequest is the ["addition" module](http://lxr.nginx.org/source/src/http/modules/ngx_http_addition_filter_module.c?v=nginx-1.12.0).  
最简单的发起单个子请求的模块是[addition模块](http://lxr.nginx.org/source/src/http/modules/ngx_http_addition_filter_module.c?v=nginx-1.12.0)。

### 2.3. Sequential subrequests  
### 2.3. 顺序子请求

*Note, 8/13/2009: This section may be out of date due to changes in Nginx’s subrequest processing introduced in Nginx 0.7.25. Follow at your own risk. -EM*  
*注：2009年8月13日：本节可能因Nginx 0.7.25引入的子请求处理变更而过时，请谨慎参考。——EM*

You might think issuing multiple subrequests is as simple as:  
你可能认为发起多个子请求很简单：
```c
int rc1, rc2, rc3;
rc1 = ngx_http_subrequest(r, uri1, ...);
rc2 = ngx_http_subrequest(r, uri2, ...);
rc3 = ngx_http_subrequest(r, uri3, ...);
```
You’d be wrong! Remember that Nginx is single-threaded. Subrequests might need to access the network, and if so, Nginx needs to return to its other work while it waits for a response. So we need to check the return value of `ngx_http_subrequest`, which can be one of:  
你错了！记住Nginx是单线程的。子请求可能需要访问网络，这时Nginx需要在等待响应时继续处理其他工作。因此我们需要检查`ngx_http_subrequest`的返回值，可能有：
- `NGX_OK`：子请求完成，无需访问网络
- `NGX_DONE`：客户端重置了网络连接
- `NGX_ERROR`：发生服务器错误
- `NGX_AGAIN`：子请求需要网络活动

If your subrequest returns `NGX_AGAIN`, your filter should also immediately return `NGX_AGAIN`. When that subrequest finishes, and the results have been sent to the client, Nginx is nice enough to call your filter again, from which you can issue the next subrequest (or do some work in between subrequests). It helps, of course, to keep track of your planned subrequests in a context struct. You should also take care to return errors immediately, too.  
如果子请求返回`NGX_AGAIN`，你的过滤器也应立即返回`NGX_AGAIN`。当该子请求完成并将结果发送给客户端后，Nginx会很贴心地再次调用你的过滤器，这时你可以发起下一个子请求（或在子请求之间做些处理）。当然，你最好用上下文结构体保存待处理的子请求列表。遇到错误也要立即返回。

Let’s make a simple example. Suppose our context struct contains an array of URIs, and the index of the next subrequest:  
来看一个简单例子。假设我们的上下文结构体包含URI数组和下一个子请求的索引：
```c
typedef struct {
    ngx_array_t  uris;
    int          i;
} my_ctx_t;
```
Then a filter that simply concatenates the contents of these URIs together might something look like:  
然后，一个简单串联这些URI内容的过滤器可能像这样：
```c
static ngx_int_t
ngx_http_multiple_uris_body_filter(ngx_http_request_t *r, ngx_chain_t *in)
{
    my_ctx_t  *ctx;
    int rc = NGX_OK;
    ngx_http_request_t *sr;

    if (r != r->main) { /* subrequest */
        return ngx_http_next_body_filter(r, in);
    }

    ctx = ngx_http_get_module_ctx(r, my_module);
    if (ctx == NULL) {
        /* populate ctx and ctx->uris here */
    }
    while (rc == NGX_OK && ctx->i < ctx->uris.nelts) {
        rc = ngx_http_subrequest(r, &((ngx_str_t *)ctx->uris.elts)[ctx->i++],
            NULL /* args */, &sr, NULL /* cb */, 0 /* flags */);
    }

    return rc; /* NGX_OK/NGX_ERROR/NGX_DONE/NGX_AGAIN */
}
```
Let’s think this code through. There might be more going on than you expect.  
分析一下这段代码，实际情况比你预想的要复杂。

First, the filter is called on the original response. Based on this response we populate `ctx` and `ctx->uris`. Then we enter the while loop and call `ngx_http_subrequest` for the first time.  
首先，过滤器在原始响应时被调用。根据响应填充`ctx`和`ctx->uris`，然后进入while循环，第一次调用`ngx_http_subrequest`。

If `ngx_http_subrequest` returns NGX_OK then we move onto the next subrequest immediately. If it returns with NGX_AGAIN, we break out of the while loop and return NGX_AGAIN.  
如果`ngx_http_subrequest`返回`NGX_OK`，则立即进入下一个子请求。如果返回`NGX_AGAIN`，则跳出循环并返回`NGX_AGAIN`。

Suppose we’ve returned an NGX_AGAIN. The subrequest is pending some network activity, and Nginx has moved on to other things. But when that subrequest is finished, Nginx will call our filter at least two more times:  
假设我们返回了`NGX_AGAIN`，此时子请求正在等待网络响应，Nginx转去处理其他事务。但当该子请求完成后，Nginx会至少再调用两次过滤器：

1. once with `r` set to the subrequest, and `in` set to buffers from the subrequest’s response  
   第一次，`r`为子请求，`in`为子请求响应的缓冲区
2. once with `r` set to the original request, and `in` set to NULL  
   第二次，`r`为原始请求，`in`为NULL

To distinguish these two cases, we must test whether `r == r->main`. In this example we call the next filter if we’re filtering the subrequest. But if we’re in the main request, we’ll just pick up the while loop where we last left off. `in` will be set to NULL because there aren’t actually any new buffers to process.  
区分这两种情况时，应判断`r == r->main`。本例中，如果处理的是子请求，则调用下一个过滤器；如果是主请求，则从while循环上次离开的位置继续。此时`in`为NULL，因为没有新缓冲区需要处理。

When the last subrequest finishes and all is well, we return NGX_OK.  
最后一个子请求完成且一切正常时，返回`NGX_OK`。

This example is of course greatly simplified. You’ll have to figure out how to populate `ctx->uris` on your own. But the example shows how simple it is to re-enter the subrequesting loop, and break out as soon as we get an error or `NGX_AGAIN`.  
当然，这只是简化示例，至于如何填充`ctx->uris`需要你自己实现。但本例展示了如何在遇到错误或`NGX_AGAIN`时跳出循环，并在合适时重新进入子请求处理循环。

### 2.4. Parallel subrequests  
### 2.4. 并行子请求

It’s also possible to issue several subrequests at once without waiting for previous subrequests to finish. This technique is, in fact, too advanced even for *Emiller’s Advanced Topics in Nginx Module Development*. See the [SSI module](http://lxr.nginx.org/source/src/http/modules/ngx_http_ssi_filter_module.c?v=nginx-1.12.0) for an example.  
你也可以一次发起多个子请求，而无需等待前面的子请求完成。这种技术甚至超出了《Emiller的Nginx模块开发高级话题》的范围。相关示例请参见[SSI模块](http://lxr.nginx.org/source/src/http/modules/ngx_http_ssi_filter_module.c?v=nginx-1.12.0)。

## 3. Parsing with Ragel  
## 3. 使用Ragel解析

If your module is dealing with any kind of input, be it an incoming HTTP header or a full-blown template language, you will need to write a parser. Parsing is one of those things that seems easy—how hard can it be to convert a string into a struct?—but there is definitely a right way to parse and a wrong way to parse. Unfortunately, Nginx sets a bad example by choosing (what I feel is) the wrong way.  
如果你的模块需要处理任何输入，无论是HTTP头还是复杂模板语言，都需要编写解析器。解析看起来很简单——把字符串转成结构体有多难？——但解析也有正确和错误方式。不幸的是，Nginx选了我认为不太好的方式。

What’s wrong with Nginx’s parsing code?  
Nginx的解析代码有什么问题？

```c
/* Random snippet from Nginx parsing code */

for (p = ctx->pos; p < last; p++) {
    ch = *p;

    switch (state) {
        case ssi_tag_state:
            switch (ch) {
                case '!':
                    /* do stuff */
            ...
```
Nginx does all of its parsing, whether of SSI includes, HTTP headers, or Nginx configuration files, using state machines. A **state machine**, you might recall from your college Theory of Computation class, reads a tape of characters, moves from state to state based on what it reads, and might perform some action based on what character it reads and what state it is in. So for example, if I wanted to parse positive decimal point numbers with a state machine, I might have a "reading stuff left of the period" state, a "just read a period" state, and a "reading stuff right of the period" state, and move among them as I read in each digit.  
Nginx的所有解析，无论是SSI包含、HTTP头还是配置文件，都是用状态机实现的。你可能还记得大学计算理论课上的**状态机**：它读取字符带，根据所读字符从一个状态转到另一个状态，并根据当前字符和状态执行动作。比如，解析正小数点数时，可以有“读小数点左侧”、“刚读到小数点”、“读小数点右侧”等状态，读每个数字都在状态之间切换。

Unfortunately, state machine parsers are usually verbose, complex, hard to understand, and hard to modify. From a software development point of view, a better approach is to use a parser generator. A **parser generator** translates high-level, highly readable parsing rules into a low-level state machine. The *compiled* code from a parser generator is virtually the same as that of handwritten state machine, but *your* code is much easier to work with.  
然而，状态机解析器通常冗长、复杂、难以理解且难以维护。软件开发中更好的方式是使用解析器生成器。**解析器生成器**可以将高级、可读性强的规则转化为低级状态机。解析器生成器编译出的代码和手写状态机几乎一样，但你的代码会易于维护得多。

There are a number of parser generators available, each with their own special syntaxes, but I am going to focus on one parser generator in particular: Ragel. Ragel is a good choice because it was designed to work with buffered inputs. Given Nginx’s buffer-chain architecture, there is a very good chance that you will parsing a buffered input, whether you really want to or not.  
市面上有很多解析器生成器，各自语法不同，但我主要推荐Ragel。Ragel非常适合处理缓冲输入，鉴于Nginx的缓冲链架构，你很可能会处理缓冲输入，无论你是否愿意。

### 3.1. Installing Ragel  
### 3.1. 安装Ragel

Use your system’s package manager or else [download Ragel from here](http://www.complang.org/ragel/).  
可以用系统的包管理工具安装Ragel，或[点击这里下载](http://www.complang.org/ragel/)。

### 3.2. Calling Ragel from Nginx  
### 3.2. Nginx中调用Ragel

It’s a good idea to put your parser functions in a separate file from the rest of the module. You will then need to:  
建议将解析函数与模块其他代码分离，具体步骤如下：

- Create a header (`.h`) file for the parser  
  - 为解析器创建头文件（.h）
- Include the header from your module  
  - 在你的模块中引入该头文件
- Create a Ragel (`.rl`) file  
  - 创建Ragel文件（.rl）
- Generate a C (`.c`) file from the Ragel file  
  - 用Ragel生成C文件（.c）
- Include the C file in your module config  
  - 在模块的config文件中包含C文件

The header file should just have prototype for parser functions, which you can include in your module via the usual `#include "my_module_parser.h"` directive. The real work is writing the Ragel file. We will work through a simple example. The official Ragel User Guide ([PDF available here](http://www.complang.org/ragel/ragel-guide-6.5.pdf)) is fully 56 pages long and gives the programmer tremendous power, but we will just go through the parts of Ragel you really need for a simple parser.  
头文件只需声明解析函数原型，然后用`#include "my_module_parser.h"`引入即可。关键工作在于编写Ragel文件。官方Ragel用户指南（[PDF点此下载](http://www.complang.org/ragel/ragel-guide-6.5.pdf)）长达56页，功能强大，但我们只讲简单解析器需要的部分。

Ragel files are C files interspersed with special Ragel commands and functions. Ragel commands are in blocks of code surrounded by `%%{` and `}%%`. The first two Ragel commands you will want in your parser are:  
Ragel文件是C代码和Ragel命令混合的。Ragel命令用`%%{`和`}%%`包裹。解析器最常用的两个Ragel命令如下：
```c
%%{
    machine my_parser;
    write data;
}%%
```
These two commands should appear after any pre-processor directives but before your parser function. The `machine` command gives a name to the state machine Ragel is about to build for you. The `write` command will create the state definitions that the state machine will use. Don’t worry about these commands too much.  
这两个命令应出现在预处理指令后、解析函数前。`machine`为状态机命名，`write`生成状态机需要的状态定义。无需过多关注这些命令。

Next you can start writing your parser function as regular C. It can take any arguments you want and should return an `ngx_int_t` with `NGX_OK` upon success and `NGX_ERROR` upon failure. You should pass in, if not a pointer to the input you want to parse, then at least some kind of context struct that contains the input data.  
接下来可以像普通C函数一样写解析器，可以有任意参数，成功时返回`NGX_OK`，失败时返回`NGX_ERROR`。建议传入要解析的数据指针，或包含输入数据的上下文结构体。

Ragel will create a number of variables implicitly for you. Other variables you need to define yourself in order to use Ragel. At the top of your function, you need to declare:  
Ragel会隐式生成一些变量。其他变量你需要自己定义，通常函数顶部要声明：

- `u_char *p` - pointer to the beginning of the input  
  - `u_char *p`输入起始指针
- `u_char *pe` - pointer to the end of the input  
  - `u_char *pe`输入结束指针
- `int cs` - an integer which stores the state machine’s state  
  - `int cs`保存状态机当前状态

Ragel will start its parsing wherever `p` points, and finish up as soon as it reaches `pe`. Therefore `p` and `pe` should both be pointers on a contiguous chunk of memory. Note that when Ragel is finished running on a particular input, you can save the value of `cs` (the machine state) and resume parsing on additional input buffers exactly where you left off. In this way Ragel works across multiple input buffers and fits beautifully into Nginx’s event-driven architecture.  
Ragel会从`p`所指位置开始解析，到`pe`结束。`p`和`pe`应指向同一块连续内存。Ragel解析完当前输入后，你可以保存`cs`状态，然后在新缓冲区中恢复解析，这样可以跨多缓冲区解析，非常适合Nginx的事件驱动架构。

### 3.3. Writing a grammar  
### 3.3. 编写语法规则

Next we want to write the Ragel grammar for our parser. A grammar is just a set of rules that specifies which kinds of input are allowed; a Ragel grammar is special because it allows us to perform actions as we scan each character. To take advantage of Ragel, you must learn the Ragel grammar syntax; it is not difficult, but it is not trivial, either.  
接下来我们要为解析器编写Ragel语法规则。语法就是规定允许哪些输入的规则；Ragel语法的特殊之处在于可以在扫描字符时执行动作。要用好Ragel，你必须学习其语法规则，虽然不难但也不简单。

Ragel grammars are defined by sets of rules. A rule has an arbitrary name on the left side of an equals sign and a specification on the right side, followed by a semicolon. The rule specification is a mixture of regular expressions and actions. We will get to actions in a minute.  
Ragel语法由一组规则组成。规则名在等号左侧，规则定义在右侧，末尾加分号。规则定义可包含正则表达式和动作，动作后面会讲。

The most important rule is called "main." All grammars must have a rule for main. The rule for main is special in that 1) the name is not arbitrary and 2) it uses `:=` instead of `=` to separate the name from the specification.  
最重要的规则叫“main”。所有语法都必须有main规则。main规则特殊之处在于：1）名字不可自定义；2）用`:=`而不是`=`分隔规则名和定义。

Let’s start with a simple example: a parser for processing Range requests. This code is adapted from my [mod_zip](http://wiki.nginx.org/NginxNgxZip) module, which also includes a more complicated parser for processing lists of files, if you are interested.  
来看一个简单例子：处理Range请求的解析器。代码改自我的[mod_zip模块](http://wiki.nginx.org/NginxNgxZip)，其中还有更复杂的文件列表解析器，感兴趣可以查看。

The "main" rule for our byte range parser is quite simple:  
我们的字节范围解析器的main规则很简单：
```c
main := "bytes=" byte_range_set;
```
That rule just says "the input should consist of the string `bytes=` followed by input which follows the rule called `byte_range_set`." So we need to define the rule `byte_range_set`:  
该规则表示“输入应为`bytes=`字符串，后跟符合`byte_range_set`规则的内容”。所以我们需要定义`byte_range_set`规则：
```c
byte_range_set = byte_range_specs ( "," byte_range_specs )*;
```
That rule just says "`byte_range_set` consists of a `byte_range_specs` followed by zero or more commas each followed by a `byte_range_specs`." In other words, a `byte_range_set` is a comma-separated list of `byte_range_specs`’s. You might recognize the `*` as a Kleene star or from regular expressions.  
该规则表示“`byte_range_set`由一个`byte_range_specs`，后跟零个或多个逗号和`byte_range_specs`组成”。即，`byte_range_set`是逗号分隔的`byte_range_specs`列表。你可能已经识别出`*`是正则中的Kleene闭包。

Next we need to define the `byte_range_specs` rule:  
接着定义`byte_range_specs`规则：
```c
byte_range_specs = byte_range_spec >new_range;
```
The `>` character is special. It says that `new_range` is not the name of another rule, but the name of an *action*, and the action should be taken at the *beginning* of *this* rule, i.e. the beginning of `byte_range_specs`. The most important special characters are:  
`>`字符很特殊，表示`new_range`不是另一个规则名，而是一个*动作*，该动作应在*本规则开始时*执行，即`byte_range_specs`开始时。主要特殊字符如下：

- `>` - action should be taken at the beginning of this rule  
  - `>`：规则开始时执行动作
- `$` - action should be taken as each character is processed  
  - `$`：每处理一个字符执行动作
- `%` - action should be taken at the end of this rule  
  - `%`：规则结束时执行动作

There are others as well, which you can read about in the Ragel User Guide. These are enough to get you started without being too confusing.  
还有其他符号，详见Ragel用户指南。上述几个已足够入门。

Before we get into actions, let’s finish defining our rules. In the rule for `byte_range_specs` (plural), we referred to a rule called `byte_range_spec` (singular). It is defined as:  
在讲动作前，先定义剩下的规则。`byte_range_specs`用到了`byte_range_spec`（单数），定义如下：
```c
byte_range_spec = [0-9]+ $start_incr
                  "-"
                  [0-9]+ $end_incr;
```
This rule states "read one or more digits, executing the action `start_incr` for each, then read a dash, then read one or more digits, executing the action `end_incr` for each." Notice that no actions are taken at the beginning or end of `byte_range_spec`.  
该规则表示“读取一个或多个数字，每个数字执行`start_incr`动作，然后读一个短横线，再读一个或多个数字，每个数字执行`end_incr`动作”。注意`byte_range_spec`开始和结束时没有动作。

When you are actually writing a grammar, you should write the rules in reverse order of what I have here. Rules should refer only to other rules that have been previously defined. So "main" should always be the *last* rule in your grammar, not the first.  
实际编写语法时，应按规则依赖逆序编写。规则只能引用已定义的其他规则。所以main应始终放在最后而非最前。

Our byte-range grammar is now finished; it’s time to specify the actions.  
字节范围语法已完成，下面定义动作。

### 3.4. Writing some actions  
### 3.4. 编写动作

Actions are chunks of C code which have access to a few special variables. The most important special variables are:  
动作就是一段C代码，可以访问一些特殊变量。最重要的有：

- `fc` - the current character being read  
  - `fc` 当前读取的字符
- `fpc` - a pointer to the current character being read  
  - `fpc` 当前读取字符的指针

`fc` is most useful for `$` actions, i.e. actions performed on each character of a string or regular expression. `fpc` is more useful for `>` and `%` actions, that is, actions taken at the start or end of a rule.  
`fc`用于`$`动作，即处理每个字符。`fpc`用于`>`和`%`动作，即规则开始或结束。

To return to our byte-range example, here is the `new_range` action. It does not use any special variables.  
回到字节范围例子，`new_range`动作不使用特殊变量。
```c
action new_range {
    if ((range = ngx_array_push(&ctx->ranges)) == NULL) {
        return NGX_ERROR;
    }
    range->start = 0; range->end = 0;
}
```
`new_range` is surprisingly dull. It just allocated a new "range" struct on the "ranges" array stored in our context struct. Notice that as long as we include the right header files, Ragel actions have full access to the Nginx API.  
`new_range`动作很简单，只是在上下文结构体中的“ranges”数组分配一个新“range”结构体。只要包含相关头文件，Ragel动作就能完全访问Nginx API。

Next we define the two remaining actions, `start_incr` and `end_incr`. These actions parse positive integers into the appropriate variables. As we read each digit of a number, we want to multiply the stored number by 10 and add the digit. Here we take advantage of the special variable `fc` described above:  
接下来定义剩下两个动作，`start_incr`和`end_incr`。它们用于将数字字符解析为整数。每读到一个数字，都要把当前值乘以10再加该数字。这里用到了`fc`：
```c
action start_incr { range->start = range->start * 10 + (fc - '0'); }

action end_incr { range->end = range->end * 10 + (fc - '0'); }
```
Note the old parsing trick of subtracting '0' to convert a character to an integer.  
注意，将字符减去'0'可转换为整数，这是常用解析技巧。

That’s it for actions. We are almost finished with our parser.  
动作就这些。解析器基本完成。

### 3.5. Putting it all together  
### 3.5. 总结完整流程

Actions and the grammar should go inside a Ragel block inside your parser function, but after the declarations of `p`, `pe`, and `cs`. I.e., something like:  
动作和语法应写在解析函数中的Ragel块里，放在`p`、`pe`、`cs`声明之后。例如：
```c
ngx_int_t my_parser(/* some context, the request struct, etc. */) 
{
    int cs;
    u_char *p = ctx->beginning_of_data;
    u_char *pe = ctx->end_of_data;

    %%{
        /* Actions */
        action some_action { /* C code goes here */ }
        action other_action { /* etc. */ }

        /* Grammar */
        my_rule = [0-9]+ "-" >some_action;
        main := ( my_rule )* >other_action;

        write init;
        write exec;
    }%%

    if (cs < my_parser_first_final) {
        return NGX_ERROR;
    }

    return NGX_OK;
}
```
We’ve added a few extra pieces here. The first are `write init` and `write exec`. These are commands to Ragel to insert the generated parser (written in C) right there.  
这里多了几个内容，最重要的是`write init`和`write exec`，它们指示Ragel在此插入生成的C解析器代码。

The other extra bit is the comparison of `cs` to `my_parser_first_final`. Recall that `cs` stores the parser’s state. This check ensures that the parser is in a valid state after it has finished processing input. If we are parsing across multiple input buffers, then instead of this check we will store `cs` somewhere and retrieve it when we want to continue parsing.  
另一个补充是比较`cs`和`my_parser_first_final`。`cs`保存解析器状态，这一判断确保解析结束后状态有效。如果要跨多缓冲区解析，则应保存`cs`，之后继续解析时恢复。

Finally, we are ready to generate the actual parser. The code we’ve written so far should be in a Ragel (`.rl`) file; when we’re ready to compile, we just run the command:  
最后，我们准备生成实际解析器。所有代码应写在Ragel（.rl）文件里，编译时只需运行命令：
```bash
ragel my_parser.rl
```
This command will produce a file called "my_parser.c". To ensure that it is compiled by Nginx, you then need to add a line to your module’s "config" file, like this:  
此命令会生成一个名为“my_parser.c”的文件。为了确保该文件被Nginx编译，在模块config文件中添加如下内容：
```bash
NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/my_parser.c"
```
Once you get the hang of parsing with Ragel, you will wonder how you ever did without it. You will actually *want* to write parsers in your Nginx modules. Ragel opens up a whole new set of possible modules to the imaginative developer.  
掌握了Ragel解析后，你会觉得离不开它。你会真的*想*在Nginx模块里写解析器。Ragel为有想象力的开发者打开了全新模块开发大门。

## 4. TODO: Advanced Topics Not Yet Covered Here  
## 4. 待补充：尚未覆盖的高级话题

Topics not yet covered in this guide:  
本文尚未覆盖的话题：

- Parallel subrequests  
  - 并行子请求
- Built-in data structures (red-black trees, arrays, hash tables…)  
  - 内建数据结构（红黑树、数组、哈希表……）
- Access control modules  
  - 访问控制模块
