# Binder 概述
## IPC 机制

![[Binder IPC.png]]

在安卓系统中，由于所有进程用户空间彼此隔离，但共享一个内核空间，所以我们可以通过 `icotl`，使得内核和共享的用户空间可以相互交换数据，进而实现进程间的通信。

## Binder 原理

Binder 也是一种 IPC 机制，其采用 C/S 架构，如下图所示：
![[Binder Client-Server.png]]

这里 Service Manager 是 Native 层（C++）的，并非 Framework 层的机制。Service Manager 是整个 Binder 通信机制的大管家，是 Android 进程间通信机制Binder的守护进程。当 Service Manager 启动之后，Client 端和 Server 端通信时都需要先获取 Service Manager 接口，才能开始通信服务。

图中 Client/Server/ServiceManage 之间的相互通信都是基于 Binder 机制。既然基于 Binder 机制通信，那么同样也是 C/S 架构，则图中的 3 大步骤都有相应的 Client 端与 Server 端。

|      | Client 端 | Server 端        |
| ---- | -------- | --------------- |
| 注册服务 | Server   | Service Manager |
| 获取服务 | Client   | Service Manager |
| 使用服务 | Client   | Server          |

所有的交互方式都是通过 Binder 驱动进行交互的，其中 Service Manager 和 Binder 驱动都是安卓平台的基础架构。

## C/S 模式

BpBinder（客户端）和 BBinder（服务端）都是 Android 中 Binder 通信相关的代表，它们都从 IBinder 类中派生而来，关系图如下：
![[Binder Classgraph.png]]
- Client 端：`BpBinder.transact()` 来发送事务请求；
- Server 端：`BBinder.onTransact()` 会接收到相应事务。
## Binder 驱动概述

Binder 驱动是 Android 专用的，但底层的驱动架构与 Linux 驱动一样。Binder 驱动在以 misc 设备进行注册，作为虚拟字符设备，没有直接操作硬件，只是对设备内存的处理。主要是驱动设备的初始化 `binder_init`，打开 `binder_open`，映射 `binder_mmap`，数据操作 `binder_ioctl`。

![[Binder Driver.png]]

其中 `open` 等函数均为通过系统调用的形式来进入内核态。
# 驱动核心方法
## binder_init

首先，我们需要分配一个 shrinker，用于未来的 binder 所占用内存的回收。所有的 binder 的状态会放到一个名为 `binder_freelist` 的链表中，该链表与 shrinker 绑定，以便随时释放 binder 用不到的内存。

```C
ret = binder_alloc_shrinker_init();
if (ret)
    return ret;
```

然后就是为 binder 创建相应调试文件，并且在 binder 这个目录下创建 proc 这个子目录。

```C
binder_debugfs_dir_entry_root = debugfs_create_dir("binder", NULL);

binder_for_each_debugfs_entry(db_entry)
    debugfs_create_file(db_entry->name,
                db_entry->mode,
                binder_debugfs_dir_entry_root,
                db_entry->data,
                db_entry->fops);

binder_debugfs_dir_entry_proc = debugfs_create_dir("proc",
                    binder_debugfs_dir_entry_root);
```

这里如果未启用 binder 文件系统，并且 `binder_devices_param` 中设置了相关参数，那么就依次使用 `init_binder_device()` 为这些设备创建相应的设备节点。

```C
if (!IS_ENABLED(CONFIG_ANDROID_BINDERFS) &&
strcmp(binder_devices_param, "") != 0) {
    /*
    * Copy the module_parameter string, because we don't want to
    * tokenize it in-place.
     */
    device_names = kstrdup(binder_devices_param, GFP_KERNEL);
    if (!device_names) {
        ret = -ENOMEM;
        goto err_alloc_device_names_failed;
    }
    
    device_tmp = device_names;
    while ((device_name = strsep(&device_tmp, ","))) {
        ret = init_binder_device(device_name);
        if (ret)
            goto err_init_binder_device_failed;
    }
}
```

在 `init_binder_device()` 中，主要就是将默认的参数和名称填入 `binder_device` 结构体，然后将其中的 `miscdev` 成员传入 `misc_register()`。而 `misc_register()` 主要作用就是初始化如下结构体。

```C
struct miscdevice {
    int minor; // 次设备号
    const char *name; // 设备名
    const struct file_operations *fops; // 指向文件操作函数表的指针。
    struct list_head list; // 内核的杂项设备链表
    struct device *parent; // 指向父设备
    struct device *this_device; // 指向本设备的设备结构体
    const struct attribute_group **groups; // 指向设备属性组数组的指针
    const char *nodename; // 设备在 devtmpfs 中的节点名称
    umode_t mode; // 设备节点的访问权限模式
};
```
## binder_open

首先进行进程结构 `binder_proc` 初始化，包括分配空间，自旋锁初始化，保存文件操作凭证，建立各种链表和等待队列，设置优先级等等工作。

```C
proc = kzalloc(sizeof(*proc), GFP_KERNEL); // 分配空间
if (proc == NULL)
    return -ENOMEM;
    
dbitmap_init(&proc->dmap); // 初始化动态位图
spin_lock_init(&proc->inner_lock); // 自旋锁初始化
spin_lock_init(&proc->outer_lock);
get_task_struct(current->group_leader); // 保存文件操作凭证
proc->tsk = current->group_leader; // 获取当前进程的主线程
proc->cred = get_cred(filp->f_cred);
INIT_LIST_HEAD(&proc->todo); // 建立链表
init_waitqueue_head(&proc->freeze_wait); // 建立等待队列
if (binder_supported_policy(current->policy)) { // 设置优先级
    proc->default_priority.sched_policy = current->policy;
    proc->default_priority.prio = current->normal_prio;
} else {
    proc->default_priority.sched_policy = SCHED_NORMAL;
    proc->default_priority.prio = NICE_TO_PRIO(0);
}
```

然后将设备和进程进行关联：

```C
if (is_binderfs_device(nodp)) { // 如果是基于文件系统的实现，就从 inode 中进行读取
    binder_dev = nodp->i_private;
    info = nodp->i_sb->s_fs_info;
    binder_binderfs_dir_entry_proc = info->proc_log_dir;
} else { // 如果是传统设备，使用 container_of 提取设备结构
    binder_dev = container_of(filp->private_data,
                  struct binder_device, miscdev);
}
```

进行状态记录和全局注册：

```C
refcount_inc(&binder_dev->ref); // 增加设备引用计数
proc->context = &binder_dev->context;
binder_alloc_init(&proc->alloc); // 初始化内存分配器

binder_stats_created(BINDER_STAT_PROC);
proc->pid = current->group_leader->pid; // 设置 pid
INIT_LIST_HEAD(&proc->delivered_death);
INIT_LIST_HEAD(&proc->delivered_freeze);
INIT_LIST_HEAD(&proc->waiting_threads);
filp->private_data = proc;
```

根据是否该进程是首次打开 binder 来判断相关调试设施。

```C
mutex_lock(&binder_procs_lock);
hlist_for_each_entry(itr, &binder_procs, proc_node) {
    if (itr->pid == proc->pid) {
        existing_pid = true; // 检查是否存在同一 pid 的进程，看该进程是否是首次打开 binder 设备
        break;
    }
}
hlist_add_head(&proc->proc_node, &binder_procs);
mutex_unlock(&binder_procs_lock);

// 提供内核空间的调试信息
if (binder_debugfs_dir_entry_proc && !existing_pid) {
    char strbuf[11];

    snprintf(strbuf, sizeof(strbuf), "%u", proc->pid);
    /*
     * proc debug entries are shared between contexts.
     * Only create for the first PID to avoid debugfs log spamming
     * The printing code will anyway print all contexts for a given
     * PID so this is not a problem.
     */
    proc->debugfs_entry = debugfs_create_file(strbuf, 0444,
        binder_debugfs_dir_entry_proc,
        (void *)(unsigned long)proc->pid,
        &proc_fops);
}

// binderfs log
if (binder_binderfs_dir_entry_proc && !existing_pid) {
    char strbuf[11];
    struct dentry *binderfs_entry;

    snprintf(strbuf, sizeof(strbuf), "%u", proc->pid);
    /*
     * Similar to debugfs, the process specific log file is shared
     * between contexts. Only create for the first PID.
     * This is ok since same as debugfs, the log file will contain
     * information on all contexts of a given PID.
     */
    binderfs_entry = binderfs_create_file(binder_binderfs_dir_entry_proc,
        strbuf, &proc_fops, (void *)(unsigned long)proc->pid);
    if (!IS_ERR(binderfs_entry)) {
        proc->binderfs_entry = binderfs_entry;
    } else {
        int error;

        error = PTR_ERR(binderfs_entry);
        pr_warn("Unable to create file %s in binderfs (error %d)\n",
            strbuf, error);
    }
}
```
## binder_mmap

这里 mmap 的作用是建立进程和 Binder 驱动之间共享内存区域，为高效的跨进程通信提供理解基础。

首先我们查看当前运行的进程是否拥有我们正在操作的 binder，并且设置 vma 的 `flags`，`vm_ops`，`vm_private_data` 这三个属性，然后进入 `binder_alloc_mmap_handler` 开始为共享内存区域分配内存。

```C
static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{
    struct binder_proc *proc = filp->private_data;

    if (proc->tsk != current->group_leader)
        return -EINVAL;

    binder_debug(BINDER_DEBUG_OPEN_CLOSE,
             "%s: %d %lx-%lx (%ld K) vma %lx pagep %lx\n",
             __func__, proc->pid, vma->vm_start, vma->vm_end,
             (vma->vm_end - vma->vm_start) / SZ_1K, vma->vm_flags,
             (unsigned long)pgprot_val(vma->vm_page_prot));

    if (vma->vm_flags & FORBIDDEN_MMAP_FLAGS) {
        pr_err("%s: %d %lx-%lx %s failed %d\n", __func__,
               proc->pid, vma->vm_start, vma->vm_end, "bad vm_flags", -EPERM);
        return -EPERM;
    }
    vm_flags_mod(vma, VM_DONTCOPY | VM_MIXEDMAP, VM_MAYWRITE);

    vma->vm_ops = &binder_vm_ops;
    vma->vm_private_data = proc;

    return binder_alloc_mmap_handler(&proc->alloc, vma);
}
```

然后就是真正的内存分配：

```C
int binder_alloc_mmap_handler(struct binder_alloc *alloc,
                  struct vm_area_struct *vma)
{
    struct binder_buffer *buffer;
    const char *failure_string;
    int ret;

    if (unlikely(vma->vm_mm != alloc->mm)) {
        ret = -EINVAL;
        failure_string = "invalid vma->vm_mm";
        goto err_invalid_mm;
    }

    mutex_lock(&binder_alloc_mmap_lock);
    if (alloc->buffer_size) {
        ret = -EBUSY;
        failure_string = "already mapped";
        goto err_already_mapped;
    }
    alloc->buffer_size = min_t(unsigned long, vma->vm_end - vma->vm_start,
                   SZ_4M); // 最多分配 4M 内存
    mutex_unlock(&binder_alloc_mmap_lock);

    alloc->vm_start = vma->vm_start;

    alloc->pages = kvcalloc(alloc->buffer_size / PAGE_SIZE,
                sizeof(alloc->pages[0]),
                GFP_KERNEL); // 分配 pages
    if (!alloc->pages) {
        ret = -ENOMEM;
        failure_string = "alloc page array";
        goto err_alloc_pages_failed;
    }

    buffer = kzalloc(sizeof(*buffer), GFP_KERNEL); // 分配 buffer
    if (!buffer) {
        ret = -ENOMEM;
        failure_string = "alloc buffer struct";
        goto err_alloc_buf_struct_failed;
    }

    buffer->user_data = alloc->vm_start;
    list_add(&buffer->entry, &alloc->buffers);
    buffer->free = 1;
    binder_insert_free_buffer(alloc, buffer); // 将对应的 buffer 插入到 binder_alloc 结构体中
    // 异步可用空间大小为 buffer 大小的一半
    alloc->free_async_space = alloc->buffer_size / 2;

    /* Signal binder_alloc is fully initialized */
    binder_alloc_set_mapped(alloc, true);

    return 0;

err_alloc_buf_struct_failed:
    kvfree(alloc->pages);
    alloc->pages = NULL;
err_alloc_pages_failed:
    alloc->vm_start = 0;
    mutex_lock(&binder_alloc_mmap_lock);
    alloc->buffer_size = 0;
err_already_mapped:
    mutex_unlock(&binder_alloc_mmap_lock);
err_invalid_mm:
    binder_alloc_debug(BINDER_DEBUG_USER_ERROR,
               "%s: %d %lx-%lx %s failed %d\n", __func__,
               alloc->pid, vma->vm_start, vma->vm_end,
               failure_string, ret);
    return ret;
}
```

其中 `binder_insert_free_buffer` 就是将分配的新的 buffer 插入到 `binder_alloc` 结构体对应的红黑树里面。

```C
static void binder_insert_free_buffer(struct binder_alloc *alloc,
                      struct binder_buffer *new_buffer)
{
    struct rb_node **p = &alloc->free_buffers.rb_node;
    struct rb_node *parent = NULL;
    struct binder_buffer *buffer;
    size_t buffer_size;
    size_t new_buffer_size;

    BUG_ON(!new_buffer->free);

    new_buffer_size = binder_alloc_buffer_size(alloc, new_buffer);
    // 计算 new_buffer 的大小

    binder_alloc_debug(BINDER_DEBUG_BUFFER_ALLOC,
             "%d: add free buffer, size %zd, at %pK\n",
              alloc->pid, new_buffer_size, new_buffer);

    while (*p) { // 寻找插入位置并插入
        parent = *p;
        buffer = rb_entry(parent, struct binder_buffer, rb_node);
        BUG_ON(!buffer->free);

        buffer_size = binder_alloc_buffer_size(alloc, buffer);

        if (new_buffer_size < buffer_size)
            p = &parent->rb_left;
        else
            p = &parent->rb_right;
    }
    rb_link_node(&new_buffer->rb_node, parent, p);
    rb_insert_color(&new_buffer->rb_node, &alloc->free_buffers);
}
```
## binder_ioctl

这一部分主要就是根据传入 binder，传入相应的命令和参数。根据 `binder_get_thread` 函数来获取对应的 thread。然后根据不同的指令执行不同的动作。

```C
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    int ret;
    struct binder_proc *proc = filp->private_data;
    struct binder_thread *thread;
    void __user *ubuf = (void __user *)arg;

    /*pr_info("binder_ioctl: %d:%d %x %lx\n",
            proc->pid, current->pid, cmd, arg);*/

    binder_selftest_alloc(&proc->alloc);

    trace_binder_ioctl(cmd, arg);

    ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
    if (ret)
        goto err_unlocked;

    // 获取 thread
    thread = binder_get_thread(proc);
    if (thread == NULL) {
        ret = -ENOMEM;
        goto err;
    }

    switch (cmd) {
    case BINDER_WRITE_READ:
        ret = binder_ioctl_write_read(filp, arg, thread);
        if (ret)
            goto err;
        break;
    case BINDER_SET_MAX_THREADS: {
        u32 max_threads;

        if (copy_from_user(&max_threads, ubuf,
                   sizeof(max_threads))) {
            ret = -EINVAL;
            goto err;
        }
        binder_inner_proc_lock(proc);
        proc->max_threads = max_threads;
        binder_inner_proc_unlock(proc);
        break;
    }
    case BINDER_SET_CONTEXT_MGR_EXT: {
        struct flat_binder_object fbo;

        if (copy_from_user(&fbo, ubuf, sizeof(fbo))) {
            ret = -EINVAL;
            goto err;
        }
        ret = binder_ioctl_set_ctx_mgr(filp, &fbo);
        if (ret)
            goto err;
        break;
    }
    case BINDER_SET_CONTEXT_MGR:
        ret = binder_ioctl_set_ctx_mgr(filp, NULL);
        if (ret)
            goto err;
        break;
    case BINDER_THREAD_EXIT:
        binder_debug(BINDER_DEBUG_THREADS, "%d:%d exit\n",
                 proc->pid, thread->pid);
        binder_thread_release(proc, thread);
        thread = NULL;
        break;
    case BINDER_VERSION: {
        struct binder_version __user *ver = ubuf;

        if (put_user(BINDER_CURRENT_PROTOCOL_VERSION,
                 &ver->protocol_version)) {
            ret = -EINVAL;
            goto err;
        }
        break;
    }
    case BINDER_GET_NODE_INFO_FOR_REF: {
        struct binder_node_info_for_ref info;

        if (copy_from_user(&info, ubuf, sizeof(info))) {
            ret = -EFAULT;
            goto err;
        }

        ret = binder_ioctl_get_node_info_for_ref(proc, &info);
        if (ret < 0)
            goto err;

        if (copy_to_user(ubuf, &info, sizeof(info))) {
            ret = -EFAULT;
            goto err;
        }

        break;
    }
    case BINDER_GET_NODE_DEBUG_INFO: {
        struct binder_node_debug_info info;

        if (copy_from_user(&info, ubuf, sizeof(info))) {
            ret = -EFAULT;
            goto err;
        }

        ret = binder_ioctl_get_node_debug_info(proc, &info);
        if (ret < 0)
            goto err;

        if (copy_to_user(ubuf, &info, sizeof(info))) {
            ret = -EFAULT;
            goto err;
        }
        break;
    }
    case BINDER_FREEZE: {
        struct binder_freeze_info info;
        struct binder_proc **target_procs = NULL, *target_proc;
        int target_procs_count = 0, i = 0;

        ret = 0;

        if (copy_from_user(&info, ubuf, sizeof(info))) {
            ret = -EFAULT;
            goto err;
        }

        mutex_lock(&binder_procs_lock);
        hlist_for_each_entry(target_proc, &binder_procs, proc_node) {
            if (target_proc->pid == info.pid)
                target_procs_count++;
        }

        if (target_procs_count == 0) {
            mutex_unlock(&binder_procs_lock);
            ret = -EINVAL;
            goto err;
        }

        target_procs = kcalloc(target_procs_count,
                       sizeof(struct binder_proc *),
                       GFP_KERNEL);

        if (!target_procs) {
            mutex_unlock(&binder_procs_lock);
            ret = -ENOMEM;
            goto err;
        }

        hlist_for_each_entry(target_proc, &binder_procs, proc_node) {
            if (target_proc->pid != info.pid)
                continue;

            binder_inner_proc_lock(target_proc);
            target_proc->tmp_ref++;
            binder_inner_proc_unlock(target_proc);

            target_procs[i++] = target_proc;
        }
        mutex_unlock(&binder_procs_lock);

        for (i = 0; i < target_procs_count; i++) {
            if (ret >= 0)
                ret = binder_ioctl_freeze(&info,
                              target_procs[i]);

            binder_proc_dec_tmpref(target_procs[i]);
        }

        kfree(target_procs);

        if (ret < 0)
            goto err;
        break;
    }
    case BINDER_GET_FROZEN_INFO: {
        struct binder_frozen_status_info info;

        if (copy_from_user(&info, ubuf, sizeof(info))) {
            ret = -EFAULT;
            goto err;
        }

        ret = binder_ioctl_get_freezer_info(&info);
        if (ret < 0)
            goto err;

        if (copy_to_user(ubuf, &info, sizeof(info))) {
            ret = -EFAULT;
            goto err;
        }
        break;
    }
    case BINDER_ENABLE_ONEWAY_SPAM_DETECTION: {
        uint32_t enable;

        if (copy_from_user(&enable, ubuf, sizeof(enable))) {
            ret = -EFAULT;
            goto err;
        }
        binder_inner_proc_lock(proc);
        proc->oneway_spam_detection_enabled = (bool)enable;
        binder_inner_proc_unlock(proc);
        break;
    }
    case BINDER_GET_EXTENDED_ERROR:
        ret = binder_ioctl_get_extended_error(thread, ubuf);
        if (ret < 0)
            goto err;
        break;
    default:
        ret = -EINVAL;
        goto err;
    }
    ret = 0;
err:
    if (thread)
        thread->looper_need_return = false;
    wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
    if (ret && ret != -EINTR)
        pr_info("%d:%d ioctl %x %lx returned %d\n", proc->pid, current->pid, cmd, arg, ret);
err_unlocked:
    trace_binder_ioctl_done(ret);
    return ret;
}
```

`binder_get_thread` 会尝试查找 proc 中是否已经用当前进程的 pid 号，如果没有那么就新建一个线程。

```C
static struct binder_thread *binder_get_thread(struct binder_proc *proc)
{
    struct binder_thread *thread;
    struct binder_thread *new_thread;

    binder_inner_proc_lock(proc);
    thread = binder_get_thread_ilocked(proc, NULL);
    binder_inner_proc_unlock(proc);
    if (!thread) {
        new_thread = kzalloc(sizeof(*thread), GFP_KERNEL);
        if (new_thread == NULL)
            return NULL;
        binder_inner_proc_lock(proc);
        thread = binder_get_thread_ilocked(proc, new_thread);
        binder_inner_proc_unlock(proc);
        if (thread != new_thread)
            kfree(new_thread);
    }
    return thread;
}
```

该函数持有了 `binder_inner_proc_lock`，主要作用为在 proc 的红黑树里查找当前进程 pid，如果找不到则新建线程，并且将此线程插入到红黑树中。

```C
static struct binder_thread *binder_get_thread_ilocked(
        struct binder_proc *proc, struct binder_thread *new_thread)
{
    struct binder_thread *thread = NULL;
    struct rb_node *parent = NULL;
    struct rb_node **p = &proc->threads.rb_node;

    while (*p) {
        parent = *p;
        thread = rb_entry(parent, struct binder_thread, rb_node);

        if (current->pid < thread->pid)
            p = &(*p)->rb_left;
        else if (current->pid > thread->pid)
            p = &(*p)->rb_right;
        else
            return thread;
    }
    if (!new_thread)
        return NULL;
    thread = new_thread;
    binder_stats_created(BINDER_STAT_THREAD);
    thread->proc = proc;
    thread->pid = current->pid;
    get_task_struct(current);
    thread->task = current;
    atomic_set(&thread->tmp_ref, 0);
    init_waitqueue_head(&thread->wait);
    INIT_LIST_HEAD(&thread->todo);
    rb_link_node(&thread->rb_node, parent, p);
    rb_insert_color(&thread->rb_node, &proc->threads);
    thread->looper_need_return = true;
    thread->return_error.work.type = BINDER_WORK_RETURN_ERROR;
    thread->return_error.cmd = BR_OK;
    thread->reply_error.work.type = BINDER_WORK_RETURN_ERROR;
    thread->reply_error.cmd = BR_OK;
    spin_lock_init(&thread->prio_lock);
    thread->prio_state = BINDER_PRIO_SET;
    thread->ee.command = BR_OK;
    INIT_LIST_HEAD(&new_thread->waiting_thread_node);
    return thread;
}
```

其中读写命令是最常用的，下面是读写命令的分支：

```C
static int binder_ioctl_write_read(struct file *filp, unsigned long arg,
                struct binder_thread *thread)
{
    int ret = 0;
    struct binder_proc *proc = filp->private_data;
    void __user *ubuf = (void __user *)arg;
    struct binder_write_read bwr;

    if (copy_from_user(&bwr, ubuf, sizeof(bwr))) { // 从用户空间拷贝数据
        ret = -EFAULT;
        goto out;
    }
    binder_debug(BINDER_DEBUG_READ_WRITE,
             "%d:%d write %lld at %016llx, read %lld at %016llx\n",
             proc->pid, thread->pid,
             (u64)bwr.write_size, (u64)bwr.write_buffer,
             (u64)bwr.read_size, (u64)bwr.read_buffer);

    if (bwr.write_size > 0) { // 如果写缓存有数据则写
        ret = binder_thread_write(proc, thread,
                      bwr.write_buffer,
                      bwr.write_size,
                      &bwr.write_consumed);
        trace_binder_write_done(ret);
        if (ret < 0) {
            bwr.read_consumed = 0;
            if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                ret = -EFAULT;
            goto out;
        }
    }
    if (bwr.read_size > 0) { // 如果都缓存有数据则读
        ret = binder_thread_read(proc, thread, bwr.read_buffer,
                     bwr.read_size,
                     &bwr.read_consumed,
                     filp->f_flags & O_NONBLOCK);
        trace_binder_read_done(ret);
        binder_inner_proc_lock(proc);
        if (!binder_worklist_empty_ilocked(&proc->todo))
            binder_wakeup_proc_ilocked(proc);
        binder_inner_proc_unlock(proc);
        if (ret < 0) {
            if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                ret = -EFAULT;
            goto out;
        }
    }
    binder_debug(BINDER_DEBUG_READ_WRITE,
             "%d:%d wrote %lld of %lld, read return %lld of %lld\n",
             proc->pid, thread->pid,
             (u64)bwr.write_consumed, (u64)bwr.write_size,
             (u64)bwr.read_consumed, (u64)bwr.read_size);
    if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
        ret = -EFAULT;
        goto out;
    }
out:
    return ret;
}
```

# Binder 通信模型

Client 进程通过 RPC（Remote Procedure Call Protocol）与 Server 通信，可以简单地划分为三层，驱动层、IPC层、应用层。

![[Binder Client RPC.png]]

具体来讲 Client 端调用 Server 端的远程服务，会通过 BC 协议，发送 `BC_TRANSACTION` 命令，并将服务对应 binder 的标识符和函数参数传递给 binder 驱动；binder 驱动寻找服务端的 binder 对象，转发事务；binder 驱动可以通过 BR 协议将调用请求通过 `BR_TRANSACTION` 调用，并调用相应的实现。结果返回过程和上述类似，时序图如下：

![[Binder Driver Seq Diagram.png]]

具体来讲，driver 收到调用请求是 client 间接调用了 `binder_thread_write()`，这个函数会根据请求生成一个 work 加到 binder 的任务队列里，然后 binder 会使用 `binder_thread_read()` 将调用请求返回给 server。如下图所示：

![[Binder Driver Call.png]]

## binder_thread_write

函数原型 `binder_thread_write(struct binder_proc *proc, struct binder_thread *thread, binder_uintptr_t binder_buffer, size_t size, binder_size_t *consumed)`

其中比较重要的是，对于请求码为 `BC_TRANSACTION` 或 `BC_REPLY` 时，会执行 `binder_transaction()`。

该函数主要将 reply 的请求转发给 `target_thread`，而非 reply 的请求转发给 `target_proc`，对特殊的嵌套调用会根据 `transaction_stack` 来决定是转发到目标线程还是目标进程。

## binder_thread_read

该函数是通过 `binder_thread_read()` 方法，该方法根据不同的 `binder_work->type` 以及不同状态，生成相应的响应码。

首先判断是否需要等待客户端请求：如果 transcation 堆栈和线程 todo 链表都是空，且并不是阻塞读取方式，需要等待。

之后有事务需要处理就会进入循环处理过程，并生成相应的响应码。在处理完毕后跳出循环然后判断一下四个条件是否可以满足：

1. `binder_proc` 的 `requested_threads` 线程数为 0；
2. `binder_proc` 的 `ready_threads` 线程数为 0；
3. `binder_proc` 的 `requested_threads_started` 个数小于15（即最大线程个数）；
4. `binder_thread` 的 `looper` 状态为 `BINDER_LOOPER_STATE_REGISTERED` 或 `BINDER_LOOPER_STATE_ENTERED`。

如果均满足说明进程中没有线程等待即将到来的事务。那么就会创建 `BR_SPAWN_LOOPER` 命令，让进程创建一条新的服务线程并注册该线程，继续等待即将到来的事务。
# Binder 通信流程

![[Binder Message.png]]

从上图可以看到，`mRemote.transact()` 是一个 JNI 接口，通过调用该接口进入了 Native 层。

首先，`android_os_BinderProxy_transact()`，将 Java 中的 `Parcel` 对象转化为 C++ 的 `Parcel` 对象；

接着进入 `BpBinder.transact()`，以及 `IPC.transact()`，使用 `writeTransactionData()` 进行数据传输，之后调用 `waitForResponse()` 等待服务端的回应，受到回应后，根据不同的相应码执行相应的操作。

以上两个函数均使用 `talkWithDriver()` 和 driver 进行通信，通过 `ioctl()` 这个系统调用，将对应的参数传入 driver。之后就是上周阅读的 driver 流程了。

在 binder 服务注册时，创建的新线程会运行 `IPC.joinThreadPool()`，通过 `IPC.getAndExecuteCommand()` 轮询查看是否有命令被放入 todo 队列里。如果有就调用 `executeCommand()` 执行命令。

`executeCommand()` 调用 `BBinder.transact()` 回到 Java 层，并且如果不是 oneway 命令，就会调用 `IPC.sendReply()`。

`Binder.execTransact()` 查看是否在 Java 运行是否会报错，如果会就写入 reply。

`IPC.sendReply()` 会向驱动发送 `BC_Reply`，等待 `BR_TRANSACTION_COMPLETE`，驱动给客户端发送 `BR_Reply` 后，给服务端发送 `BINDER_WORK_TRANSACTION_COMPLETE`。

客户端收到 `BR_Reply` 后，调用 `freeBuffer()`，然后给驱动发送 `BC_FREE_BUFFER` 释放掉驱动的 buffer。