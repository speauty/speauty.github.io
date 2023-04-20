# Redis源码之内存模型解析-ZMalloc {docsify-ignore}

当前分析Redis版本为6.2，需要注意。

由于上一次分析了SDS，发现有关内存管理都是使用的zmalloc，虽然中间取了次别名，与此区别。如果接着看其他数据结构，估计还是zmalloc。藉此，将Redis的内存模型分析直接提前，就是为了后面看其他数据结构，更加清晰。为此，先看一下SDS有关内存，`sdsalloc.h`。

```c
#include "zmalloc.h"
#define s_malloc zmalloc
#define s_realloc zrealloc
#define s_trymalloc ztrymalloc
#define s_tryrealloc ztryrealloc
#define s_free zfree
#define s_malloc_usable zmalloc_usable
#define s_realloc_usable zrealloc_usable
#define s_trymalloc_usable ztrymalloc_usable
#define s_tryrealloc_usable ztryrealloc_usable
#define s_free_usable zfree_usable
```

无一例外，全是基于zmalloc的。接下来，切入正题，直接看头文件（`src/zmalloc.h`）的相关常量或原型之类的定义吧。

#### 函数原型

* `zmalloc` `void *zmalloc(size_t size)` 基于 `malloc` 实现的动态内存分配函数；
* `zcalloc` `void *zcalloc(size_t size)` 基于 `calloc` 实现的动态内存分配函数，和 `malloc` 的区别是，该可对分配内存进行初始化工作；
* `zrealloc` `void *zrealloc(void *ptr, size_t size)` 重新分配指定内存，并初始化；
* `ztrymalloc` `void *ztrymalloc(size_t size)` 尝试分配内存；
* `ztrycalloc` `void *ztrycalloc(size_t size)` 尝试分配内存并初始化；
* `ztryrealloc` `void *ztryrealloc(void *ptr, size_t size)` 尝试重新分配指定内存；
* `zfree` `void zfree(void *ptr)` 释放指定内存；
* `zmalloc_usable` `void *zmalloc_usable(size_t size, size_t *usable)` 动态内存分配并返回可用内存大小；
* `zcalloc_usable` `void *zcalloc_usable(size_t size, size_t *usable)` ；
* `zrealloc_usable` `void *zrealloc_usable(void *ptr, size_t size, size_t *usable)` ；
* `ztrymalloc_usable` `void *ztrymalloc_usable(size_t size, size_t *usable)` ；
* `ztrycalloc_usable` `void *ztrycalloc_usable(size_t size, size_t *usable)` ；
* `ztryrealloc_usable` `void *ztryrealloc_usable(void *ptr, size_t size, size_t *usable)` ；
* `zfree_usable` `void zfree_usable(void *ptr, size_t *usable)` ；
* `zstrdup` `char *zstrdup(const char *s)` 拷贝指定字符串；
* `zmalloc_used_memory` `size_t zmalloc_used_memory(void);` 获取已使用内存大小；
* `zmalloc_set_oom_handler` `void zmalloc_set_oom_handler(void (*oom_handler)(size_t))` 设置内存溢出句柄；
* `zmalloc_get_rss` `size_t zmalloc_get_rss(void)` 获取实际使用内存大小，RSS（Resident Set Size）实际使用物理内存（包含共享库占用的内存）；
* `zmalloc_get_allocator_info` `int zmalloc_get_allocator_info(size_t *allocated, size_t *active, size_t *resident)` 获取指定内存状态等信息；
* `set_jemalloc_bg_thread` `void set_jemalloc_bg_thread(int enable)` 设置后台进程；
* `jemalloc_purge` `int jemalloc_purge()` 释放未使用的内存；
* `zmalloc_get_private_dirty` `size_t zmalloc_get_private_dirty(long pid)` 获取指定进程被标记为 私有脏页大小；
* `zmalloc_get_smap_bytes_by_field` `size_t zmalloc_get_smap_bytes_by_field(char *field, long pid)` 从 `libproc` 接口获取指定字段的统计；
* `zmalloc_get_memory_size` `size_t zmalloc_get_memory_size(void)` 获取物理内存的大小；
* `zlibc_free` `void zlibc_free(void *ptr)` 释放内存；

还有一些零零散散的函数，是根据不同环境定义，遇到再具体分析。

#### 函数实现

以函数出现的先后顺序分析，如果涉及到某些常量再做具体说明。

##### `zlibc_free`

这是在 `zmalloc.c` 源文件引入 `zmalloc.h` 头文件之前定义的使用libc的free接口释放内存的包装函数，在前面定义，主要避免影响到 `Jemalloc` 或其他非标准动态内存分配模型的 `free` 的接口。其实现是直接调用 `free` 。

##### `zmalloc_default_oom`

默认内存溢出处理句柄。

```c
static void zmalloc_default_oom(size_t size) {
    fprintf(stderr, "zmalloc: Out of memory trying to allocate %zu bytes\n",
        size); // 直接向标准错误输出
    fflush(stderr); // 刷新标准错误输出缓冲区
    abort(); // 然后 直接中断进程
}
```

##### `ztrymalloc_usable`

尝试动态内存分配，如果分配失败就返回null，参数usable表示可用内存大小。采用 `malloc` 实现的。

```c
void *ztrymalloc_usable(size_t size, size_t *usable) {
    // 断言size是否溢出 
    // #define ASSERT_NO_SIZE_OVERFLOW(sz) assert((sz) + PREFIX_SIZE > (sz))
    // 由于在HAVE_MALLOC_SIZE常量（系统存在malloc_size函数）定义的情况下，PREFIX_SIZE常量为0
    // 所以，直接就是 #define ASSERT_NO_SIZE_OVERFLOW(sz)
    ASSERT_NO_SIZE_OVERFLOW(size);
    // MALLOC_MIN_SIZE 也是个宏定义 
    // #define MALLOC_MIN_SIZE(x) ((x) > 0 ? (x) : sizeof(long))
    // 在使用 libc 动态内存分配时，使用一个最小可分配大小去适配 jemalloc 内存模型
    // 分配时，需要带上前缀大小，也就是 PREFIX_SIZE
    void *ptr = malloc(MALLOC_MIN_SIZE(size)+PREFIX_SIZE);

    if (!ptr) return NULL;
#ifdef HAVE_MALLOC_SIZE // 系统存在 malloc_size 函数的情况
    size = zmalloc_size(ptr); // 获取指针指向的内存大小
    // #define update_zmalloc_stat_alloc(__n) atomicIncr(used_memory,(__n))
    // 更新原子类静态变量 used_memory（内存总使用大小）的值，互斥操作
    update_zmalloc_stat_alloc(size);
    if (usable) *usable = size; // 更新可用内存
    return ptr;
#else
    // 给内存的指定头部填写值，值为内存的长度
    // ptr指向由类型为size_t的一个或多个对象组成的数组元素 或ptr的前size_t个字节空间
    *((size_t*)ptr) = size;
    update_zmalloc_stat_alloc(size+PREFIX_SIZE);
    if (usable) *usable = size;
    return (char*)ptr+PREFIX_SIZE;
#endif
}
```

##### `zmalloc`

直接调用 `ztrymalloc_usable` 分配内存，成功返回对应指针，失败就调用 `zmalloc_oom_handler` 报错并中断进程，比较暴力。

##### `ztrymalloc`

`zmalloc` 的友好实现，失败后将处理交由函数调用方，也就是返回null的情况。很 `try` 的味道。

#####  `zmalloc_usable`

`zmalloc` 的变形，多了 `usable` 参数，记录可用内存大小。

#####  `zmalloc_no_tcache`

绕过线程缓存，直接在 `Jemalloc` 的 `Arena` `bin` 分配。仅在 `HAVE_DEFRAG` 定义时定义，这是碎片整理的宏定义，在使用 `jemalloc` 内存模型并存在 `JEMALLOC_FRAG_HINT` 定义时定义。

```c
void *zmalloc_no_tcache(size_t size) {
    ASSERT_NO_SIZE_OVERFLOW(size);
    // Jemalloc 的分配函数？
    void *ptr = mallocx(size+PREFIX_SIZE, MALLOCX_TCACHE_NONE);
    if (!ptr) zmalloc_oom_handler(size);
    update_zmalloc_stat_alloc(zmalloc_size(ptr));
    return ptr;
}
```

##### `zfree_no_tcache`

同 `zmalloc_no_tcache` 对应的释放函数。

```c
void zfree_no_tcache(void *ptr) {
    if (ptr == NULL) return;
    update_zmalloc_stat_free(zmalloc_size(ptr));
    dallocx(ptr, MALLOCX_TCACHE_NONE);
}
```

##### `ztrycalloc_usable`

尝试动态内存分配并初始化，基于 `calloc` 实现。失败返回 `null`。

```c
void *ztrycalloc_usable(size_t size, size_t *usable) {
    ASSERT_NO_SIZE_OVERFLOW(size);
    // calloc 分配内存，并将分配的内存设置为0
    // 第一个参数 被分配的元素个数，第二个参数 元素的大小
    void *ptr = calloc(1, MALLOC_MIN_SIZE(size)+PREFIX_SIZE);
    if (ptr == NULL) return NULL;

    // 后面同 ztrymalloc_usable 一致，根据 HAVE_MALLOC_SIZE 区分处理
    // ...
}
```

##### `zcalloc`

分配并初始化内存，失败直接调用 `zmalloc_oom_handler` 处理。

##### `ztrycalloc`

同 `ztrymalloc` 大体相似，不过时调用 `ztrycalloc_usable` ，多了初始化的处理。

##### `zcalloc_usable`

`zcalloc` 变形，多了 `usable` 参数，记录可用内存大小。

##### `ztryrealloc_usable`

尝试重新分配内存，失败返回null。

```c
void *ztryrealloc_usable(void *ptr, size_t size, size_t *usable) {
    ASSERT_NO_SIZE_OVERFLOW(size);
#ifndef HAVE_MALLOC_SIZE
    void *realptr;
#endif
    size_t oldsize;
    void *newptr;

    // 如果已分配内存未使用，直接释放并返回null
    if (size == 0 && ptr != NULL) {
        zfree(ptr);
        if (usable) *usable = 0;
        return NULL;
    }
    // 如果空指针，直接调用 ztrymalloc_usable 进行内存分配并返回
    if (ptr == NULL)
        return ztrymalloc_usable(size, usable);

#ifdef HAVE_MALLOC_SIZE // 定义 malloc_size 的情况
    oldsize = zmalloc_size(ptr); // 获取原指针关联内存大小
    newptr = realloc(ptr,size); // 重新分配内存
    if (newptr == NULL) { // 分配失败
        if (usable) *usable = 0;
        return NULL;
    }

    update_zmalloc_stat_free(oldsize); // 从总使用内存中减去原内存大小
    size = zmalloc_size(newptr); 
    update_zmalloc_stat_alloc(size); // 加上重新分配内存大小
    if (usable) *usable = size;
    return newptr;
#else
    realptr = (char*)ptr-PREFIX_SIZE;
    oldsize = *((size_t*)realptr);
    newptr = realloc(realptr,size+PREFIX_SIZE);
    if (newptr == NULL) {
        if (usable) *usable = 0;
        return NULL;
    }

    *((size_t*)newptr) = size;
    update_zmalloc_stat_free(oldsize);
    update_zmalloc_stat_alloc(size);
    if (usable) *usable = size;
    return (char*)newptr+PREFIX_SIZE;
#endif
}
```

##### `zrealloc`

调用 `ztryrealloc_usable` 重新分配内存，失败直接调用 `zmalloc_oom_handler` 处理。

##### `ztryrealloc`

调用 `ztryrealloc_usable` 尝试重新分配内存，失败返回null。

##### `zrealloc_usable`

调用 `ztryrealloc_usable` 重新分配内存，失败直接调用 `zmalloc_oom_handler` 处理。相比 `zrealloc`， 多了 `usable` 参数，记录可用内存大小。

##### `zmalloc_size`

获取已分配内存总大小，一个 `malloc` 本身不提供的功能，就是为了在每个分配内存的第一部分字节存储一个信息头（当前分配内存总大小）。照调用来看，应该是已分配内存可使用大小。仅在 `HAVE_MALLOC_SIZE` 未定义时定义。

```c
size_t zmalloc_size(void *ptr) {
    void *realptr = (char*)ptr-PREFIX_SIZE; // 偏移去掉前缀，指向实际指针
    size_t size = *((size_t*)realptr); // 获取实际指针关联内存大小
    return size+PREFIX_SIZE; // ptr实际分配内存=实际关联内存大小+前缀大小
}
```

##### `zmalloc_usable_size`

同 `zmalloc_size` 函数定义情况一致。获取指针可用内存大小。也就是调用 `zmalloc_size` 后得到的总分配大小减去前缀大小。

##### `zfree`

```c
void zfree(void *ptr) {
#ifndef HAVE_MALLOC_SIZE
    void *realptr;
    size_t oldsize;
#endif

    if (ptr == NULL) return; // 空指针直接返回？
#ifdef HAVE_MALLOC_SIZE // 从总使用内存中减去指针分配内存总大小，然后释放
    update_zmalloc_stat_free(zmalloc_size(ptr));
    free(ptr);
#else // 相比之下，多了一步，根据 PREFIX_SIZE 获取实际指针
    realptr = (char*)ptr-PREFIX_SIZE;
    oldsize = *((size_t*)realptr);
    update_zmalloc_stat_free(oldsize+PREFIX_SIZE);
    free(realptr);
#endif
}
```

##### `zfree_usable`

同 `zfree` 相似，多了个 `usable` 参数，记录实际释放可使用内存大小。

##### `zstrdup`

拷贝字符串，不是SDS类型，调用 zmalloc 分配的内存空间。

##### `zmalloc_used_memory`

获取总分配内存大小，也就是 `used_memory` 的值。

```c
size_t zmalloc_used_memory(void) {
    size_t um;
    atomicGet(used_memory,um);
    return um;
}
```

##### `zmalloc_set_oom_handler`

设置 `zmalloc` 内存分配异常处理句柄。

##### `zmalloc_get_rss`

同操作系统的特殊方式获取实际使用物理内存大小。Resident Set Size。

######  `HAVE_PROC_STAT`

```c
size_t zmalloc_get_rss(void) { // __linux__ OS
    // sysconf 在运行时获取系统配置
    // _SC_PAGESIZE 系统页面大小
    int page = sysconf(_SC_PAGESIZE);
    size_t rss;
    char buf[4096];
    char filename[256];
    int fd, count;
    char *p, *x;
	// getpid() 获取当前进程号
    // 拼装对应进程文件 保存到filename
    snprintf(filename,256,"/proc/%ld/stat",(long) getpid());
    // 以只读模式打开文件
    if ((fd = open(filename,O_RDONLY)) == -1) return 0;
    if (read(fd,buf,4096) <= 0) { // 读取失败，就关闭文件，并返回
        close(fd);
        return 0;
    }
    close(fd);

    p = buf;
    // RSS 在 /proc/<pid>/stat 文件的第24个字段，所以，索引23
    count = 23;
    while(p && count--) {
        // char *strchr(const char *str, int c) 
        // 返回在字符串 str 中第一次出现字符 c 的位置，如果未找到该字符则返回 NULL
        p = strchr(p,' '); 
        if (p) p++;
    }
    if (!p) return 0;
    // 继续找下一处，设置结束符，即p断尾
    x = strchr(p,' ');
    if (!x) return 0;
    *x = '\0';

    rss = strtoll(p,NULL,10); // 获取p字符串中的十进制整数值
    rss *= page;
    return rss;
}
```

###### `HAVE_TASKINFO`

```c
size_t zmalloc_get_rss(void) { // __APPLE__ OS
    task_t task = MACH_PORT_NULL;
    struct task_basic_info t_info;
    mach_msg_type_number_t t_info_count = TASK_BASIC_INFO_COUNT;

    if (task_for_pid(current_task(), getpid(), &task) != KERN_SUCCESS)
        return 0;
    task_info(task, TASK_BASIC_INFO, (task_info_t)&t_info, &t_info_count);

    return t_info.resident_size;
}
```

###### `__FreeBSD__` or `__DragonFly__`

```c
size_t zmalloc_get_rss(void) { // __FreeBSD__ OS?
    struct kinfo_proc info;
    size_t infolen = sizeof(info);
    int mib[4];
    mib[0] = CTL_KERN;
    mib[1] = KERN_PROC;
    mib[2] = KERN_PROC_PID;
    mib[3] = getpid();

    if (sysctl(mib, 4, &info, &infolen, NULL, 0) == 0)
#if defined(__FreeBSD__)
        return (size_t)info.ki_rssize * getpagesize();
#else
        return (size_t)info.kp_vm_rssize * getpagesize();
#endif

    return 0L;
}
```

###### `__NetBSD__`

```c
size_t zmalloc_get_rss(void) { // __NetBSD__ OS
    struct kinfo_proc2 info;
    size_t infolen = sizeof(info);
    int mib[6];
    mib[0] = CTL_KERN;
    mib[1] = KERN_PROC;
    mib[2] = KERN_PROC_PID;
    mib[3] = getpid();
    mib[4] = sizeof(info);
    mib[5] = 1;
    if (sysctl(mib, 4, &info, &infolen, NULL, 0) == 0)
        return (size_t)info.p_vm_rssize * getpagesize();

    return 0L;
}
```

###### `HAVE_PSINFO`

```c
size_t zmalloc_get_rss(void) { // __SUN__ OS
    // 和Linux很像
    struct prpsinfo info;
    char filename[256];
    int fd;

    snprintf(filename,256,"/proc/%ld/psinfo",(long) getpid());

    if ((fd = open(filename,O_RDONLY)) == -1) return 0;
    if (ioctl(fd, PIOCPSINFO, &info) == -1) {
        close(fd);
	return 0;
    }

    close(fd);
    return info.pr_rssize;
}
```

###### `Other`

```c
size_t zmalloc_get_rss(void) {
    // 如果不能通过OS特殊方法获取，就采用zmalloc()中估算的总的使用内存
    return zmalloc_used_memory();
}
```

##### `zmalloc_get_allocator_info`

```c
// 定义USE_JEMALLOC时 使用 Jemalloc 内存模型管理
int zmalloc_get_allocator_info(size_t *allocated,
                               size_t *active,
                               size_t *resident) {
    uint64_t epoch = 1;
    size_t sz;
    *allocated = *resident = *active = 0;
    // 更新mallctl缓存中的统计
    sz = sizeof(epoch);
    je_mallctl("epoch", &epoch, &sz, &epoch, sz);
    sz = sizeof(size_t);
    /* Unlike RSS, this does not include RSS from shared libraries and other non
     * heap mappings. */
    je_mallctl("stats.resident", resident, &sz, NULL, 0);
    /* Unlike resident, this doesn't not include the pages jemalloc reserves
     * for re-use (purge will clean that). */
    je_mallctl("stats.active", active, &sz, NULL, 0);
    /* Unlike zmalloc_used_memory, this matches the stats.resident by taking
     * into account all allocations done by this process (not only zmalloc). */
    je_mallctl("stats.allocated", allocated, &sz, NULL, 0);
    return 1;
}

// 未使用 Jemalloc 时
// 就很随意，毕竟不支持
int zmalloc_get_allocator_info(size_t *allocated,
                               size_t *active,
                               size_t *resident) {
    *allocated = *resident = *active = 0;
    return 1;
}
```

##### `set_jemalloc_bg_thread`

```c
// 使用 Jemalloc
void set_jemalloc_bg_thread(int enable) {
    // 让 Jemalloc 异步清洗，在 flushdb后没有活动时
    char val = !!enable;
    je_mallctl("background_thread", NULL, 0, &val, 1);
}
// 未使用 Jemalloc
void set_jemalloc_bg_thread(int enable) {
    ((void)(enable));
}
```

##### `jemalloc_purge`

碎片整理，释放未使用内存给OS。

```c
// Jemalloc
int jemalloc_purge() {
    char tmp[32];
    unsigned narenas = 0;
    size_t sz = sizeof(unsigned);
    if (!je_mallctl("arenas.narenas", &narenas, &sz, NULL, 0)) {
        sprintf(tmp, "arena.%d.purge", narenas);
        if (!je_mallctl(tmp, NULL, 0, NULL, 0))
            return 0;
    }
    return -1;
}
// 其他
int jemalloc_purge() {
    return 0;
}
```

##### `zmalloc_get_smap_bytes_by_field`

```c
// 定义 HAVE_PROC_SMAPS
size_t zmalloc_get_smap_bytes_by_field(char *field, long pid) {
    char line[1024];
    size_t bytes = 0;
    int flen = strlen(field);
    FILE *fp;

    if (pid == -1) {
        fp = fopen("/proc/self/smaps","r");
    } else {
        char filename[128];
        snprintf(filename,sizeof(filename),"/proc/%ld/smaps",pid);
        fp = fopen(filename,"r");
    }

    if (!fp) return 0;
    while(fgets(line,sizeof(line),fp) != NULL) {
        if (strncmp(line,field,flen) == 0) {
            char *p = strchr(line,'k');
            if (p) {
                *p = '\0';
                bytes += strtol(line+flen,NULL,10) * 1024;
            }
        }
    }
    fclose(fp);
    return bytes;
}
// 其他
size_t zmalloc_get_smap_bytes_by_field(char *field, long pid) {
#if defined(__APPLE__)
    struct proc_regioninfo pri;
    if (pid == -1) pid = getpid();
    if (proc_pidinfo(pid, PROC_PIDREGIONINFO, 0, &pri,
                     PROC_PIDREGIONINFO_SIZE) == PROC_PIDREGIONINFO_SIZE)
    {
        int pagesize = getpagesize();
        if (!strcmp(field, "Private_Dirty:")) {
            return (size_t)pri.pri_pages_dirtied * pagesize;
        } else if (!strcmp(field, "Rss:")) {
            return (size_t)pri.pri_pages_resident * pagesize;
        } else if (!strcmp(field, "AnonHugePages:")) {
            return 0;
        }
    }
    return 0;
#endif
    ((void) field);
    ((void) pid);
    return 0;
}
```

##### `zmalloc_get_private_dirty`

获取私有脏页数据。

```c
size_t zmalloc_get_private_dirty(long pid) {
    return zmalloc_get_smap_bytes_by_field("Private_Dirty:",pid);
}
```

##### `zmalloc_get_memory_size`

```c
size_t zmalloc_get_memory_size(void) {
#if defined(__unix__) || defined(__unix) || defined(unix) || \
    (defined(__APPLE__) && defined(__MACH__))
#if defined(CTL_HW) && (defined(HW_MEMSIZE) || defined(HW_PHYSMEM64))
    int mib[2];
    mib[0] = CTL_HW;
#if defined(HW_MEMSIZE)
    mib[1] = HW_MEMSIZE;            /* OSX. --------------------- */
#elif defined(HW_PHYSMEM64)
    mib[1] = HW_PHYSMEM64;          /* NetBSD, OpenBSD. --------- */
#endif
    int64_t size = 0;               /* 64-bit */
    size_t len = sizeof(size);
    if (sysctl( mib, 2, &size, &len, NULL, 0) == 0)
        return (size_t)size;
    return 0L;          /* Failed? */

#elif defined(_SC_PHYS_PAGES) && defined(_SC_PAGESIZE)
    /* FreeBSD, Linux, OpenBSD, and Solaris. -------------------- */
    return (size_t)sysconf(_SC_PHYS_PAGES) * (size_t)sysconf(_SC_PAGESIZE);

#elif defined(CTL_HW) && (defined(HW_PHYSMEM) || defined(HW_REALMEM))
    /* DragonFly BSD, FreeBSD, NetBSD, OpenBSD, and OSX. -------- */
    int mib[2];
    mib[0] = CTL_HW;
#if defined(HW_REALMEM)
    mib[1] = HW_REALMEM;        /* FreeBSD. ----------------- */
#elif defined(HW_PHYSMEM)
    mib[1] = HW_PHYSMEM;        /* Others. ------------------ */
#endif
    unsigned int size = 0;      /* 32-bit */
    size_t len = sizeof(size);
    if (sysctl(mib, 2, &size, &len, NULL, 0) == 0)
        return (size_t)size;
    return 0L;          /* Failed? */
#else
    return 0L;          /* Unknown method to get the data. */
#endif
#else
    return 0L;          /* Unknown OS. */
#endif
}
```



最后几个函数就不细看了，着实打脑壳。毕竟还年轻。基本都是OS特殊方法的。

不过通篇看下来，似乎没看见什么特别的地方，`Redis` 对其他内存分配的接口进行了一番包装使用，主要有这几种：`Tcmalloc` `Jemalloc`，以及 `Apple` 的 `malloc/malloc.h` 和 `libc` 的 `malloc.h`。然后在内存分配时顺便也在指针前 `size_t` 字节中保存了本次分配内存可使用大小。就是各种 `malloc` `calloc` 和 `realloc` 操作。