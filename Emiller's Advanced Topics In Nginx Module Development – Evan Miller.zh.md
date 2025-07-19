# Emiller's Advanced Topics In Nginx Module Development
# Emiller 的 Nginx 模块开发高级主题

*Note: This is a bilingual document with English paragraphs followed by Chinese translations. Code examples are not translated.*

*注意：这是一个双语文档，英文段落后跟中文翻译。代码示例不进行翻译。*

---

By Evan Miller (with Grzegorz Nosek)

作者：Evan Miller（与 Grzegorz Nosek 合作）

DRAFT: August 13, 2009

草稿：2009年8月13日

Whereas Emiller's Guide To Nginx Module Development describes the bread-and-butter issues of writing a simple handler, filter, or load-balancer for Nginx, this document covers three advanced topics for the ambitious Nginx developer: shared memory, subrequests, and parsing. Because these are subjects on the boundaries of the Nginx universe, the code here may be sparse. The examples may be out of date. But hopefully, you will make it out not only alive, but with a few extra tools in your belt.

虽然《Emiller 的 Nginx 模块开发指南》描述了为 Nginx 编写简单处理器、过滤器或负载均衡器的基本问题，但本文档涵盖了雄心勃勃的 Nginx 开发者的三个高级主题：共享内存、子请求和解析。由于这些主题处于 Nginx 世界的边界，这里的代码可能比较稀少。示例可能已经过时。但希望你不仅能够生存下来，还能在工具包中多一些额外的工具。

## Table of Contents
## 目录

- [Shared Memory](#shm)
  - [A (fore)word of caution](#shm-foreword)
  - [Creating and using a shared memory segment](#shm-creating)
  - [Using the slab allocator](#shm-slab)
  - [Spinlocks, atomic memory access](#shm-spinlocks)
  - [Using rbtrees](#shm-rbtrees)
- [Subrequests](#subrequests)
  - [Internal redirect](#subrequests-redirect)
  - [A single subrequest](#subrequests-single)
  - [Sequential subrequests](#subrequests-sequential)
  - [Parallel subrequests](#subrequests-parallel)
- [Parsing With Ragel](#parsing) *NEW*
  - [Installing ragel](#parsing-install)
  - [Calling ragel from nginx](#parsing-call)
  - [Writing a grammar](#parsing-grammar)
  - [Writing some actions](#parsing-actions)
  - [Putting it all together](#parsing-finale)
- [TODO](#todo)

- [共享内存](#shm)
  - [注意事项（前言）](#shm-foreword)
  - [创建和使用共享内存段](#shm-creating)
  - [使用 slab 分配器](#shm-slab)
  - [自旋锁，原子内存访问](#shm-spinlocks)
  - [使用红黑树](#shm-rbtrees)
- [子请求](#subrequests)
  - [内部重定向](#subrequests-redirect)
  - [单个子请求](#subrequests-single)
  - [顺序子请求](#subrequests-sequential)
  - [并行子请求](#subrequests-parallel)
- [使用 Ragel 进行解析](#parsing) *新增*
  - [安装 ragel](#parsing-install)
  - [从 nginx 调用 ragel](#parsing-call)
  - [编写语法](#parsing-grammar)
  - [编写一些动作](#parsing-actions)
  - [将所有内容整合起来](#parsing-finale)
- [待办事项](#todo)

<a name="shm"></a>

## 1. Shared Memory
## 1. 共享内存

*Guest chapter written by Grzegorz Nosek*

*客座章节由 Grzegorz Nosek 编写*

Nginx, while being unthreaded, allows worker processes to share memory between them. However, this is quite different from the standard pool allocator as the shared segment has fixed size and cannot be resized without restarting nginx or destroying its contents in another way.

Nginx 虽然是非线程的，但允许工作进程之间共享内存。然而，这与标准的池分配器截然不同，因为共享段的大小是固定的，不能在不重启 nginx 或以其他方式销毁其内容的情况下调整大小。

<a name="shm-foreword"></a>

### 1.1. A (fore)word of caution
### 1.1. 注意事项（前言）

First of all, caveat hacker. This guide has been written several months after hands-on experience with shared memory in nginx and while I try my best to be accurate (and have spent some time refreshing my memory), in no way is it guaranteed. You've been warned.

首先，黑客请注意。本指南是在 nginx 共享内存实践经验几个月后编写的，虽然我尽最大努力保证准确性（并花了一些时间回忆），但无法保证完全正确。你已经被警告过了。

Also, 100% of this knowledge comes from reading the source and reverse-engineering the core concepts, so there are probably better ways to do most of the stuff described.

此外，这些知识100%来自阅读源代码和逆向工程核心概念，因此可能有更好的方法来完成所描述的大部分工作。

Oh, and this guide is based on 0.6.31, though 0.5.x is 100% compatible AFAIK and 0.7.x also brings no compatibility-breaking changes that I know of.

哦，还有这个指南是基于 0.6.31 版本的，虽然据我所知 0.5.x 是100%兼容的，0.7.x 也没有带来我所知的不兼容变化。

For real-world usage of shared memory in nginx, see my [upstream_fair module](http://github.com/gnosek/nginx-upstream-fair/tree/master).

有关 nginx 中共享内存的实际使用，请参阅我的 [upstream_fair 模块](http://github.com/gnosek/nginx-upstream-fair/tree/master)。

This probably does not work on Windows at all. Core dumps in the rear mirror are closer than they appear.

这在 Windows 上可能根本不起作用。后视镜中的核心转储比它们看起来更近。

<a name="shm-creating"></a>

### 1.2. Creating and using a shared memory segment
### 1.2. 创建和使用共享内存段

To create a shared memory segment in nginx, you need to:

要在 nginx 中创建共享内存段，你需要：

- provide a constructor function to initialise the segment
- call `ngx_shared_memory_add`

- 提供一个构造函数来初始化段
- 调用 `ngx_shared_memory_add`

These two points contain the main gotchas (that I came across), namely:

这两点包含了主要的陷阱（我遇到的），即：

1. Your constructor will be called multiple times and it's up to you to find out whether you're called the first time (and should set something up), or not (and should probably leave everything alone). The prototype for the shared memory constructor looks like:

你的构造函数将被多次调用，你需要找出是否是第一次被调用（应该设置一些东西），还是不是（应该保持一切不变）。共享内存构造函数的原型如下：

```c
static ngx_int_t init(ngx_shm_zone_t *shm_zone, void *data);
```

The data variable will contain the contents of `oshm_zone->data`, where `oshm_zone` is the "old" shm zone descriptor (more about it later). This variable is the only value that can survive a reload, so you must use it if you don't want to lose the contents of your shared memory.

data 变量将包含 `oshm_zone->data` 的内容，其中 `oshm_zone` 是"旧的"shm 区域描述符（稍后详述）。这个变量是唯一能够在重载中幸存的值，所以如果你不想丢失共享内存的内容，必须使用它。

Your constructor function will probably look roughly similar to the one in upstream_fair, i.e.:

你的构造函数可能看起来与 upstream_fair 中的类似，即：

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

2. You must be careful when to access the shm segment.

你必须小心何时访问 shm 段。

The interface for adding a shared memory segment looks like:

添加共享内存段的接口如下：

```c
ngx_shm_zone_t *
ngx_shared_memory_add(ngx_conf_t *cf, ngx_str_t *name, size_t size,
        void *tag);
```

`cf` is the reference to the config file (you'll probably create the segment in response to a config option), name is the name of the segment (as a `ngx_str_t`, i.e. a counted string), size is the size in bytes (which will usually get rounded up to the nearest multiple of the page size, e.g. 4KB on many popular architectures) and tag is a, well, tag for detecting naming conflicts. If you call `ngx_shared_memory_add` multiple times with the same name, tag and size, you'll get only a single segment. If you specify different names, you'll get several distinct segments and if you specify the same name but different size or tag, you'll get an error. A good choice for the tag value could be e.g. the pointer to your module descriptor.

`cf` 是配置文件的引用（你可能会在响应配置选项时创建段），name 是段的名称（作为 `ngx_str_t`，即计数字符串），size 是字节大小（通常会向上舍入到页面大小的最近倍数，例如在许多流行架构上是 4KB），tag 是用于检测命名冲突的标签。如果你用相同的名称、标签和大小多次调用 `ngx_shared_memory_add`，你只会得到一个段。如果你指定不同的名称，你会得到几个不同的段，如果你指定相同的名称但不同的大小或标签，你会得到错误。标签值的一个好选择可能是指向你的模块描述符的指针。

After you call `ngx_shared_memory_add` and receive the new `shm_zone` descriptor, you must set up the constructor in `shm_zone->init`. Wait… after you add the segment? Yes, and that's a major gotcha. This implies that the segment is not created while calling `ngx_shared_memory_add` (because you specify the constructor only later). What really happens looks like this (grossly simplified):

在你调用 `ngx_shared_memory_add` 并接收到新的 `shm_zone` 描述符后，你必须在 `shm_zone->init` 中设置构造函数。等等...在添加段之后？是的，这是一个主要的陷阱。这意味着在调用 `ngx_shared_memory_add` 时段并没有被创建（因为你只是稍后指定构造函数）。真正发生的情况是这样的（极度简化）：

1. parse the whole config file, noting requested shm segments

解析整个配置文件，记录请求的 shm 段

2. afterwards, create/destroy all the segments in one go

之后，一次性创建/销毁所有段

The constructors are called here. Note that every time your ctor is called, it is with another value of `shm_zone`. The reason is that the descriptor lives as long as the cycle (generation in Apache terms) while the segment lives as long as the master and all the workers. To let *some* data survive a reload, you have access to the old descriptor's `->data` field (mentioned above).

构造函数在这里被调用。注意每次调用你的构造函数时，都是用另一个 `shm_zone` 值。原因是描述符的生命周期与周期（Apache 术语中的生成）一样长，而段的生命周期与主进程和所有工作进程一样长。为了让*一些*数据在重载中幸存，你可以访问旧描述符的 `->data` 字段（如上所述）。

3. (re)start workers which begin handling requests

（重新）启动开始处理请求的工作进程

4. upon receipt of SIGHUP, goto 1

收到 SIGHUP 后，转到步骤 1

Also, you really must set the constructor, otherwise nginx will consider your segment unused and won't create it at all.

另外，你真的必须设置构造函数，否则 nginx 会认为你的段未使用，根本不会创建它。

Now that you know it, it's pretty clear that you cannot rely on having access to the shared memory while parsing the config. You can access the whole segment as `shm_zone->shm.addr` (which will be NULL before the segment gets really created). Any access after the first parsing run (e.g. inside request handlers or on subsequent reloads) should be fine.

现在你知道了，很明显，在解析配置时不能依赖访问共享内存。你可以通过 `shm_zone->shm.addr` 访问整个段（在段真正创建之前将为 NULL）。第一次解析运行后的任何访问（例如在请求处理程序中或后续重载时）应该都没问题。

<a name="shm-slab"></a>

### 1.3. Using the slab allocator
### 1.3. 使用 slab 分配器

Now that you have your new and shiny shm segment, how do you use it? The simplest way is to use another memory tool that nginx has at your disposal, namely the slab allocator. Nginx is nice enough to initialise the slab for you in every new shm segment, so you can either use it, or ignore the slab structures and overwrite them with your own data.

现在你有了新的闪亮的 shm 段，如何使用它？最简单的方法是使用 nginx 为你提供的另一个内存工具，即 slab 分配器。Nginx 足够友好，在每个新的 shm 段中为你初始化 slab，所以你可以使用它，或者忽略 slab 结构并用你自己的数据覆盖它们。

The interface consists of two functions:

接口由两个函数组成：

- `void *ngx_slab_alloc(ngx_slab_pool_t *pool, size_t size);`
- `void ngx_slab_free(ngx_slab_pool_t *pool, void *p);`

The first argument is simply `(ngx_slab_pool_t *)shm_zone->shm.addr` and the other one is either the size of the block to allocate, or the pointer to the block to free. (trivia: not once is `ngx_slab_free` called in vanilla nginx code)

第一个参数简单地是 `(ngx_slab_pool_t *)shm_zone->shm.addr`，另一个参数要么是要分配的块的大小，要么是要释放的块的指针。（趣事：在原版 nginx 代码中从未调用过 `ngx_slab_free`）

<a name="shm-spinlocks"></a>

### 1.4. Spinlocks, atomic memory access
### 1.4. 自旋锁，原子内存访问

Remember that shared memory is inherently dangerous because you can have multiple processes accessing it at the same time. The slab allocator has a per-segment lock (`shpool->mutex`) which is used to protect the segment against concurrent modifications.

记住，共享内存本质上是危险的，因为你可能有多个进程同时访问它。slab 分配器有一个每段锁（`shpool->mutex`），用于保护段免受并发修改。

You can also acquire and release the lock yourself, which is useful if you want to implement some more complicated operations on the segment, like searching or walking a tree. The two snippets below are essentially equivalent:

你也可以自己获取和释放锁，这在你想要在段上实现一些更复杂的操作（如搜索或遍历树）时很有用。下面的两个代码片段本质上是等效的：

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

实际上，ngx_slab_alloc 看起来几乎与上面完全一样。

If you perform any operations which depend on no new allocations (or, more to the point, frees), protect them with the slab mutex. However, remember that nginx mutexes are implemented as spinlocks (non-sleeping), so while they are very fast in the uncontended case, they can easily eat 100% CPU when waiting. So don't do any long-running operations while holding the mutex (especially I/O, but you should avoid any system calls at all).

如果你执行任何依赖于没有新分配（或者更确切地说，释放）的操作，请用 slab 互斥锁保护它们。但是，记住 nginx 互斥锁是作为自旋锁（非睡眠）实现的，所以虽然在无竞争情况下它们非常快，但在等待时很容易占用 100% 的 CPU。所以在持有互斥锁时不要做任何长时间运行的操作（特别是 I/O，但你应该完全避免任何系统调用）。

You can also use your own mutexes for more fine-grained locking, via the `ngx_mutex_init()`, `ngx_mutex_lock()` and `ngx_mutex_unlock()` functions.

你也可以通过 `ngx_mutex_init()`、`ngx_mutex_lock()` 和 `ngx_mutex_unlock()` 函数使用你自己的互斥锁进行更细粒度的锁定。

As an alternative for locks, you can use atomic variables which are guaranteed to be read or written in an uninterruptible way (no worker process may see the value halfway as it's being written by another one).

作为锁的替代，你可以使用原子变量，这些变量保证以不可中断的方式读取或写入（没有工作进程可能在另一个进程写入时看到中间值）。

Atomic variables are defined with the type `ngx_atomic_t` or `ngx_atomic_uint_t` (depending on signedness). They should have at least 32 bits. To simply read or unconditionally set an atomic variable, you don't need any special constructs:

原子变量用 `ngx_atomic_t` 或 `ngx_atomic_uint_t`（取决于符号性）类型定义。它们应该至少有 32 位。要简单地读取或无条件设置原子变量，你不需要任何特殊构造：

```c
ngx_atomic_t i = an_atomic_var;
an_atomic_var = i + 5;
```

Note that anything can happen between the two lines; context switches, execution of code on other other CPUs, etc.

注意两行之间可能发生任何事情；上下文切换、其他 CPU 上的代码执行等。

To atomically read and modify a variable, you have two functions (very platform-specific) with their interface declared in `src/os/unix/ngx_atomic.h`:

要原子地读取和修改变量，你有两个函数（非常特定于平台），它们的接口在 `src/os/unix/ngx_atomic.h` 中声明：

- `ngx_atomic_cmp_set(lock, old, new)`

Atomically retrieves old value of `*lock` and stores `new` under the same address. Returns 1 if `*lock` was equal to `old` before overwriting.

原子地检索 `*lock` 的旧值并在相同地址下存储 `new`。如果 `*lock` 在覆盖前等于 `old`，则返回 1。

- `ngx_atomic_fetch_add(value, add)`

Atomically adds `add` to `*value` and returns the old `*value`.

原子地将 `add` 添加到 `*value` 并返回旧的 `*value`。

<a name="shm-rbtrees"></a>

### 1.5. Using rbtrees
### 1.5. 使用红黑树

OK, you have your data neatly allocated, protected with a suitable lock but you'd also like to organise it somehow. Again, nginx has a very nice structure just for this purpose - a red-black tree.

好的，你的数据已经整齐地分配了，用合适的锁保护了，但你也想以某种方式组织它。再次，nginx 正好有一个非常好的结构用于这个目的 - 红黑树。

Highlights (API-wise):

要点（API 方面）：

- requires an insertion callback, which inserts the element in the tree (probably according to some predefined order) and then calls `ngx_rbt_red(the_newly_added_node)` to rebalance the tree
- requires all leaves to be set to a predefined sentinel object (not NULL)

- 需要一个插入回调，将元素插入树中（可能根据某种预定义顺序），然后调用 `ngx_rbt_red(the_newly_added_node)` 来重新平衡树
- 要求所有叶子都设置为预定义的哨兵对象（不是 NULL）

This chapter is about shared memory, not rbtrees so shoo! Go read the source for `ngx_rbtree_insert` in `src/core/ngx_rbtree.c` if you must know more.

这一章是关于共享内存的，不是红黑树，所以走开！如果你必须知道更多，去阅读 `src/core/ngx_rbtree.c` 中 `ngx_rbtree_insert` 的源代码。

<a name="subrequests"></a>

## 2. Subrequests
## 2. 子请求

Subrequests are nginx's way of decomposing one request into several requests handled sequentially, or sometimes concurrently.

子请求是 nginx 将一个请求分解为多个按顺序处理的请求的方式，有时是并发处理的。

For example, SSI (Server Side Includes) is implemented using subrequests. When the SSI filter sees this directive:

例如，SSI（服务器端包含）是使用子请求实现的。当 SSI 过滤器看到这个指令时：

```html
<!--#include virtual="/foo" -->
```

It creates a subrequest for "/foo", and the output of this subrequest is included in the output of the original request.

它为"/foo"创建一个子请求，这个子请求的输出包含在原始请求的输出中。

Or, to take an example that's more like an upstream module: If you wanted to access a file on another server, you might want to create a subrequest which fetches the file using the proxy module. This is similar to what the X-Accel-Redirect header does.

或者，举一个更像上游模块的例子：如果你想访问另一台服务器上的文件，你可能想创建一个使用代理模块获取文件的子请求。这类似于 X-Accel-Redirect 头的作用。

There are several types of subrequests:

有几种类型的子请求：

<a name="subrequests-redirect"></a>

### 2.1. Internal redirects
### 2.1. 内部重定向

This is the simplest: you're done with processing this request, have another request processed in its place. The function is `ngx_http_internal_redirect`. See how it's used in the index module (which processes the directive `index file1.html file2.html` by trying to redirect internally to `file1.html`, then if that fails to `file2.html`, etc.)

这是最简单的：你完成了对这个请求的处理，让另一个请求代替它被处理。函数是 `ngx_http_internal_redirect`。看看它在索引模块中是如何使用的（该模块通过尝试内部重定向到 `file1.html` 来处理指令 `index file1.html file2.html`，如果失败则重定向到 `file2.html`，等等）。

<a name="subrequests-single"></a>

### 2.2. A single subrequest
### 2.2. 单个子请求

You have another request processed, but return the content of the original request. Or: you want to include the output of another request in the output of the original request.

你让另一个请求被处理，但返回原始请求的内容。或者：你想在原始请求的输出中包含另一个请求的输出。

To do this, you need to provide a callback that gets called when the subrequest finishes. The callback must be set up like this:

要做到这一点，你需要提供一个在子请求完成时被调用的回调。回调必须这样设置：

```c
ngx_int_t rc;
ngx_http_request_t *sr; /* subrequest object */

rc = ngx_http_subrequest(r, &uri, &args, &sr, &ps, flags);
if (rc != NGX_OK) { /* error */ }

/* 
 * Here we have our subrequest object, so we can set its callback.
 * sr is the subrequest.
 * mymodule_subrequest_callback is explained below.
 * mymodule_callback_ctx can be used to pass your own data structure to
 * the callback.
 */
sr->post_subrequest = &mymodule_subrequest_post_handler;
mymodule_callback_ctx *callback_ctx = ngx_palloc(r->pool, sizeof(mymodule_callback_ctx));
/* ...set up callback_ctx to refer to your data... */ 
sr->ctx[mymodule.ctx_index] = callback_ctx;
```

...then later, define the callback:

...然后稍后，定义回调：

```c
ngx_int_t mymodule_subrequest_callback(ngx_http_request_t *r, void *data, ngx_int_t rc) {
    ngx_http_request_t *pr = r->parent;

    mymodule_callback_ctx *ctx = ngx_http_get_module_ctx(r, mymodule);

    pr->headers_out.status = r->headers_out.status;

    if (r->headers_out.status == NGX_HTTP_OK) {
        /* parse r->upstream->buffer and do something with it */
    }

    return rc;
}
```

To create the subrequest, you have to call `ngx_http_subrequest`:

要创建子请求，你必须调用 `ngx_http_subrequest`：

```c
ngx_int_t ngx_http_subrequest(ngx_http_request_t *r,
            ngx_str_t *uri, ngx_str_t *args, ngx_http_request_t **psr,
            ngx_http_post_subrequest_t *ps, ngx_uint_t flags)
```

`r` is the original request; `uri` and `args` refer to the URI and query string of the sub-request; `psr` is a reference to a pointer that will point to the new (sub)request structure; `ps` is a callback structure described above; `flags` can be 0, or some combination of:

`r` 是原始请求；`uri` 和 `args` 指的是子请求的 URI 和查询字符串；`psr` 是将指向新（子）请求结构的指针的引用；`ps` 是上面描述的回调结构；`flags` 可以是 0，或者以下的某种组合：

- `NGX_HTTP_SUBREQUEST_IN_MEMORY` tells the upstream modules (like proxy) to place the response in memory rather than a temporary file. Obviously useful.
- `NGX_HTTP_SUBREQUEST_WAITED` tells nginx to set the "done" flag on the subrequest even if it's not... done. Obscure.

- `NGX_HTTP_SUBREQUEST_IN_MEMORY` 告诉上游模块（如代理）将响应放在内存中而不是临时文件中。显然很有用。
- `NGX_HTTP_SUBREQUEST_WAITED` 告诉 nginx 即使子请求没有...完成，也在子请求上设置"完成"标志。晦涩。

<a name="subrequests-sequential"></a>

### 2.3. Sequential subrequests
### 2.3. 顺序子请求

Let's say I want to access /foo, and then /bar, and then /baz. One approach is: in your main handler, issue a subrequest for /foo with a callback. The callback issues a subrequest for /bar with its own callback, which issues a subrequest for /baz with its own callback, which calls finalize.

假设我想访问 /foo，然后 /bar，然后 /baz。一种方法是：在你的主处理程序中，为 /foo 发出一个带回调的子请求。回调为 /bar 发出一个带有自己回调的子请求，该回调为 /baz 发出一个带有自己回调的子请求，然后调用 finalize。

This is not a good approach, though it will work. The problem is that, it's almost impossible to handle errors or cleanly maintain your ctx structures.

这不是一个好方法，尽管它会起作用。问题是，几乎不可能处理错误或干净地维护你的 ctx 结构。

A better approach is to have a single callback, and to have your callback-function issue the next request.

一个更好的方法是有一个单一的回调，让你的回调函数发出下一个请求。

```c
ngx_int_t mymodule_subrequest_callback(ngx_http_request_t *r, void *data, ngx_int_t rc) {
    ngx_http_request_t *pr = r->parent;
    mymodule_callback_ctx *ctx = ngx_http_get_module_ctx(pr, mymodule);

    if (r->headers_out.status != NGX_HTTP_OK) {
        /* handle the error and finalize */
    }

    /* parse r->upstream->buffer and do something with it */

    if (ctx->current_subrequest_index >= ctx->number_of_subrequests) {
        /* we're done! */
        ngx_http_finalize_request(pr, NGX_OK);
        return NGX_OK;
    }

    /* make the next subrequest */
    ngx_http_subrequest(pr, 
        &ctx->subrequest_uris[ctx->current_subrequest_index], 
        &ctx->subrequest_args[ctx->current_subrequest_index], 
        NULL, &mymodule_subrequest_post_handler, 0);

    ctx->current_subrequest_index++;

    return NGX_OK;
}
```

<a name="subrequests-parallel"></a>

### 2.4. Parallel subrequests
### 2.4. 并行子请求

Issuing multiple subrequests at once, waiting for them all to finish. This is the hardest case. You need to set up callbacks for each subrequest, and keep track of which ones have finished. When they have all finished, then finalize.

一次发出多个子请求，等待它们全部完成。这是最困难的情况。你需要为每个子请求设置回调，并跟踪哪些已经完成。当它们全部完成时，然后 finalize。

This example is crude but shows the technique:

这个例子很粗糙，但展示了技术：

```c
/* ctx is the context for the original request */
for (i = 0; i < number_of_subrequests; i++) {
    ngx_http_post_subrequest_t *psr = ngx_palloc(r->pool, sizeof(ngx_http_post_subrequest_t));
    psr->handler = mymodule_subrequest_callback;
    psr->data = ctx;

    ngx_http_subrequest(r, &uris[i], &args[i], NULL, psr, 0);
}

ctx->subrequests_completed = 0;
```

...and then have the callback something like:

...然后让回调像这样：

```c
ngx_int_t mymodule_subrequest_callback(ngx_http_request_t *r, void *data, ngx_int_t rc) {
    ngx_http_request_t *pr = r->parent;
    mymodule_callback_ctx *ctx = (mymodule_callback_ctx*)data;

    ctx->subrequests_completed++;

    if (r->headers_out.status != NGX_HTTP_OK) {
        ctx->error = 1;
    }

    /* save the result from r->upstream->buffer */

    if (ctx->subrequests_completed == ctx->number_of_subrequests) {
        /* we're done! */
        if (ctx->error) {
            ngx_http_finalize_request(pr, NGX_HTTP_INTERNAL_SERVER_ERROR);
        } else {
            /* do something with the results and finalize normally */
            ngx_http_finalize_request(pr, NGX_OK);
        }
    }

    return NGX_OK;
}
```

<a name="parsing"></a>

## 3. Parsing with Ragel
## 3. 使用 Ragel 进行解析

*NEW*

*新增*

<a name="parsing-install"></a>

### 3.1. Installing Ragel
### 3.1. 安装 Ragel

*TODO*

*待办事项*

<a name="parsing-call"></a>

### 3.2. Calling Ragel from Nginx
### 3.2. 从 Nginx 调用 Ragel

*TODO*

*待办事项*

<a name="parsing-grammar"></a>

### 3.3. Writing a grammar
### 3.3. 编写语法

*TODO*

*待办事项*

<a name="parsing-actions"></a>

### 3.4. Writing some actions
### 3.4. 编写一些动作

*TODO*

*待办事项*

<a name="parsing-finale"></a>

### 3.5. Putting it all together
### 3.5. 将所有内容整合起来

*TODO*

*待办事项*

<a name="todo"></a>

## 4. TODO: Advanced Topics Not Yet Covered Here
## 4. 待办事项：此处尚未涵盖的高级主题

- Filters
- Load-balancers
- Non-blocking network I/O
- Timers

- 过滤器
- 负载均衡器
- 非阻塞网络 I/O
- 定时器
