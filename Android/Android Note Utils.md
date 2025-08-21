# 内核内存分配

| 函数                                            | 名称含义                           | 功能                                                             |
| --------------------------------------------- | ------------------------------ | -------------------------------------------------------------- |
| `kmalloc(size_t size, gfp_t flags)`           | Kernel Memory Allocate         | 分配一个变量的空间。（对应的数组分配版本 `kmalloc_array`）                          |
| `kvmalloc(size_t size, gfp_t flags)`          | Kernel Virtual Memory Allocate | 尝试分配连续的物理地址段。如果失败，则改为分配连续的虚拟地址段。(也有对应的数组分配版本 `kvmalloc_array`) |
| `kcalloc(size_t n, size_t size, gfp_t flags)` | Kernel Contiguous Allocate     | 分配一个数组空间并初始化为 0（是 `kmalloc_array` 的带初始化版本）。                    |
| `kzalloc(size_t size, gfp_t flags)`           | Kernel Zero Allocate           | 分配一个变量的空间并初始化为 0（是 `kmalloc` 的带初始化版本）。                         |

`kmalloc` 流程：根据传入的 `size` 值是否为常量判断走的分支，上方是为常量走的分支，下方是不为常量走的分支。但这两个流程唯一的不同就是选择缓存的逻辑不一样，前者是直接根据 `size` 大小递增选择缓存序号，后者在 `size` 较小的时候是根据预先打好的表来计算的。

还有一个区别是，下方的流程在 `__do_kmalloc_node` 中，有设置 NUMA 节点编号的机会。

![[__do_kmalloc_node.png]]

如果是分配（超过 `KMALLOC_MAX_CACHE_SIZE`，此值在该内核中被设置为 8 KB）大内存，需要另一套逻辑。此时我们会尽量将大页面对齐至 2 的幂次。

![[__do_kmalloc_node large alloc.png]]

# 「参数默认值」

```C
/*
 * These macros allow declaring a kmem_buckets * parameter alongside size, which
 * can be compiled out with CONFIG_SLAB_BUCKETS=n so that a large number of call
 * sites don't have to pass NULL.
 */
#ifdef CONFIG_SLAB_BUCKETS
#define DECL_BUCKET_PARAMS(_size, _b)   size_t (_size), kmem_buckets *(_b)
#define PASS_BUCKET_PARAMS(_size, _b)   (_size), (_b)
#define PASS_BUCKET_PARAM(_b)       (_b)

// Here is an example:
void *__kvmalloc_node_noprof(DECL_BUCKET_PARAMS(size, b), gfp_t flags, int node) __alloc_size(1);
#define kvmalloc_node_noprof(size, flags, node) \
    __kvmalloc_node_noprof(PASS_BUCKET_PARAMS(size, NULL), flags, node)
```
# task_struct 和 binder_proc 相关
* `task_struct` 负责用于进程调度，`binder_proc` 用于记录拥有 binder 的进程的信息，便于 binder 的处理。
* `binder_alloc` 负责分配 binder 的缓存。

每个 `task_struct` 都有一个 vma 表，可以通过链表和红黑树两种方式进行访问。binder_mmap 就是将 binder 的 buffer 对应到进程的一个 vma 中。