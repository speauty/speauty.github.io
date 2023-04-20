# Redis源码之数据类型解析-DICT {docsify-ignore}

当前分析Redis版本为6.2，需要注意。

`DICT`，字典数据结构，本质是哈希表。相关文件主要有 `dict.h` 和 `dict.c`。

#### 基础结构

`dictEntry` 是哈希节点，键值对，其中值采用了联合体 `v`，从结构体中的 `*next` 可以看出，这是个单向链表，必然头节点是单独保存的，挂在哈希表上。

```c
typedef struct dictEntry {
    void *key; // 键
    union {
        void *val;
        uint64_t u64; // unsigned
        int64_t s64; // signed
        double d;
    } v; // 值
    struct dictEntry *next; // 下一个节点地址
} dictEntry;
```

接着就是 `dictType` 字典类型，对字典进行了区别，不同类型的字典相关处理函数可以不同。

```c
typedef struct dictType {
    // 哈希函数，根据 Key 生成对应的哈希值
    uint64_t (*hashFunction)(const void *key);
    void *(*keyDup)(void *privdata, const void *key); // 键复制函数
    void *(*valDup)(void *privdata, const void *obj); // 值复制函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2); // 键比较函数
    void (*keyDestructor)(void *privdata, void *key); // 销毁键函数
    void (*valDestructor)(void *privdata, void *obj); // 销毁值函数
    int (*expandAllowed)(size_t moreMem, double usedRatio); // 扩容函数？
} dictType;
```

`dictht` 哈希表结构体。

```c
typedef struct dictht {
    dictEntry **table; // 二级指针，存放哈希节点链表的头指针，主要是为了解决哈希冲突
    unsigned long size; // 哈希表大小
    unsigned long sizemask; // 哈希表大小掩码 size-1。计算索引 idx = hash & sizemask
    unsigned long used; // 已有哈希节点数量
} dictht; // 哈希表
```

`dict` 字典。

```c
typedef struct dict {
    dictType *type; // 字典类型
    void *privdata; // 私有数据
    // 哈希表数组，只有两个哈希表，一般情况，只使用第一个哈希表。
    // ht[1]只会在ht[0]的rehash时使用
    dictht ht[2]; 
    // rehash-idx标识 没有rehash时为-1 rehash 过程索引
    long rehashidx;
    // rehash 暂停标识
    // > 0 表示暂停 < 0 表示编码错误
    int16_t pauserehash;
} dict;
```

`dictIterator` 字典迭代器。

```c
typedef struct dictIterator {
    dict *d; // 当前迭代字典
    long index; // 迭代器当前所指向的哈希表索引位置
    // table 当前被迭代的哈希表 0|1
    // safe 标识当前迭代器是否安全
    // safe为1是安全的，在迭代过程中，可以执行 dictAdd、dictFind和其他函数，对字典进行修改
    // safe不为1，只能调用dictNext对字典进行迭代。限制了修改
    int table, safe;
    // entry 当前迭代节点指针
    // nextEntry 当前迭代节点的下一个节点，在安全迭代器运行时，entry节点可能会被修改
    // 需要额外指针保存下一节点，防止指针丢失
    dictEntry *entry, *nextEntry;
    // 用于误用检测的不安全迭代器签名
    long long fingerprint;
} dictIterator;
```

#### 宏定义函数

##### `DICT_NOTUSED`

未使用的参数会生成警告？

##### `dictFreeVal`

释放指定字典的指定哈希节点值，在定义了值析构函数的情况下，如果没有定义的话，将不会处理。

```c
#define dictFreeVal(d, entry) \
    if ((d)->type->valDestructor) \
        (d)->type->valDestructor((d)->privdata, (entry)->v.val)
```

##### `dictSetVal`

设置指定字典的指点哈希节点值，采用 `do/while` 结构，保证代码执行，一次。在定义了 `valDup` 的情况下，使用返回值。反之，直接设置。

```c
#define dictSetVal(d, entry, _val_) do { \
    if ((d)->type->valDup) \
        (entry)->v.val = (d)->type->valDup((d)->privdata, _val_); \
    else \
        (entry)->v.val = (_val_); \
} while(0)
```

##### `dictSetSignedIntegerVal`

设置哈希节点值，有符号整型。直接对 `entry->v.s64` 进行赋值。

##### `dictSetUnsignedIntegerVal`

设置哈希节点值，无符号整型。直接对 `entry->v.u64` 进行赋值。

##### `dictSetDoubleVal`

设置哈希节点值，双精度型。直接对 `entry->v.d` 进行赋值。

##### `dictFreeKey`

释放指定字典的指定哈希节点键，同 `dictFreeVal` 相似。

##### `dictSetKey`

设置指定字典的指定哈希节点键，同 `dictSetVal` 相似。

##### `dictCompareKeys`

比较指定字典的两个哈希键。如果存在 `d->type->keyCompare`，就调用 `keyCompare` 进行比较，否则直接 `key1 == key2` 比较。

##### `dictHashKey`

生成指定字典指定键的哈希值。

##### `dictGetKey`

获取指定哈希节点键。

##### `dictGetVal`

获取指定哈希节点值。

##### `dictGetSignedIntegerVal`

获取指定哈希节点的有符号整型值。

##### `dictGetUnsignedIntegerVal`

获取指定哈希节点的无符号整型值。

##### `dictGetDoubleVal`

获取指定哈希节点的双精度值。

`dictSlots`

获取指定字典的插槽总量 `ht[0].size + ht[1].size`。

##### `dictSize`

获取指定字典的哈希表已使用节点总量。

##### `dictIsRehashing`

判断指定字典是否正在 rehash。

##### `dictPauseRehashing`

暂停指定字典 rehash 过程。

##### `dictResumeRehashing`

继续指定字典 rehash 过程。

#### 常量和变量

##### `DICT_OK`

宏定义常量。表示字典处理成功。

##### `DICT_ERR`

宏定义常量。表示字典处理失败。

##### `DICT_HT_INITIAL_SIZE`

宏定义常量。哈希表初始化大小。

##### `dictTypeHeapStringCopyKey`

外部变量。字典类型，堆字符串拷贝键？

##### `dictTypeHeapStrings`

外部变量。字典类型，堆字符串？

##### `dictTypeHeapStringCopyKeyValue`

外部变量。字典类型，堆字符串拷贝键值？

##### `dict_can_resize`

静态变量。哈希表大小是否可调整。但该值为0，也不是所有调整都被阻止。

##### `dict_force_resize_ratio`

静态变量。哈希表大小强制调整比例。 `used/size`，如果该比例大于 `dict_force_resize_ratio`，就需要强制调整。

##### `dict_hash_function_seed[16]`

静态变量。哈希函数种子。

#### 哈希函数

##### `dictSetHashFunctionSeed`

设置哈希种子。使用 `memcpy` 实现。

##### `dictGetHashFunctionSeed`

获取哈希种子。直接返回 `dict_hash_function_seed` 即可。

##### `dictGenHashFunction`

哈希函数。套壳 `siphash` 实现，具体实现在 `siphash.c` 文件。

##### `dictGenCaseHashFunction`

也是哈希函数。套壳 `siphash_nocase` 实现，具体实现在 `siphash.c` 文件。

#### 私有函数

##### `_dictExpandIfNeeded`

如果需要，扩展哈希表。调用 `dictExpand` 接口函数实现。

```c
static int _dictExpandIfNeeded(dict *d)
{
    // 正在 rehash 过程，不支持扩展，直接返回即可
    if (dictIsRehashing(d)) return DICT_OK;

    // 如果空哈希表，直接扩展到初始大小 DICT_HT_INITIAL_SIZE
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);

    // 条件一 哈希表已使用大小 >= 哈希表大小，说明已经饱和或溢出
    // 条件二 dict_can_resize 是否支持大小调整 或 已达到强制扩展比例（负载因子）
    // 条件三 该字典类型允许扩展调整
    // 扩展到 used+1 大小。。。这样该函数触发可能比较频繁。只增已有节点+1。
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio) &&
        dictTypeExpandAllowed(d))
    {
        return dictExpand(d, d->ht[0].used + 1);
    }
    return DICT_OK;
}
```

##### `_dictNextPower`

将初始哈希表大小做幂级扩展，直到大于等于给定 `size`。

```c
static unsigned long _dictNextPower(unsigned long size)
{
    unsigned long i = DICT_HT_INITIAL_SIZE; // 初始大小 4

    if (size >= LONG_MAX) return LONG_MAX + 1LU; // 不会溢出？
    while(1) {
        if (i >= size) // i = 2^n
            return i;
        i *= 2;
    }
}
```

##### `dictTypeExpandAllowed`

判断指定字典是否允许哈希表扩展。主要根据字典类型中的 `expandAllowed` 函数。

```c
static int dictTypeExpandAllowed(dict *d) {
    if (d->type->expandAllowed == NULL) return 1;// 没设置，默认可以？
    return d->type->expandAllowed(
                    _dictNextPower(d->ht[0].used + 1) * sizeof(dictEntry*),
                    (double)d->ht[0].used / d->ht[0].size);
}
```

##### `_dictKeyIndex`

返回可用插槽的索引（头节点）。如果在 rehash 过程，总是返回 ht[2] 中的索引。

```c
static long _dictKeyIndex(dict *d, const void *key, uint64_t hash, dictEntry **existing)
{ // hash 由 key 生成。那这里为什么不调用哈希函数生成，反而传参？
    unsigned long idx, table;
    dictEntry *he; // 哈希节点
    if (existing) *existing = NULL; // 已存在哈希节点。强行重置。
    if (_dictExpandIfNeeded(d) == DICT_ERR) return -1; // 扩展哈希表如果有需要
    
    for (table = 0; table <= 1; table++) { // 在ht[0]和ht[1]中检索，可能在rehash过程。需要循环
        idx = hash & d->ht[table].sizemask; // 索引 哈希值和哈希表大小掩码
        he = d->ht[table].table[idx]; // 获取相应哈希头节点，开启循环检索 key 毕竟单向链表
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key)) { // 存在 返回 -1
                if (existing) *existing = he;
                return -1;
            }
            he = he->next;
        }
        if (!dictIsRehashing(d)) break; // 如果不在 rehash 过程。直接 break。ht[1]=null
    }
    return idx; // 如果不存在，返回值是？可能要深入哈希函数。
}
```

##### `_dictInit`

字典初始化。

```c
int _dictInit(dict *d, dictType *type,
        void *privDataPtr)
{
    _dictReset(&d->ht[0]); // 重置哈希表
    _dictReset(&d->ht[1]);
    d->type = type;
    d->privdata = privDataPtr;
    d->rehashidx = -1; // 默认 无rehash过程
    d->pauserehash = 0;
    return DICT_OK;
}
// 这里存在问题，函数实现与申明不匹配。 dict.c@66
static int _dictInit(dict *ht, dictType *type, void *privDataPtr);
```

#### 接口函数

##### `_dictReset`

重置已使用 `ht_init` 初始化（暂未找到该函数实现）的哈希表。该函数只在 `ht_destroy` 中调用，但是就现在看到的来说，在 `_dictInit` 中也有调用。将哈希表中相关属性重置 `NULL(table)/0(size|sizemask|used)`。

##### `dictCreate`

接口。创建字典，主要是哈希表。

```c
dict *dictCreate(dictType *type,
        void *privDataPtr)
{
    dict *d = zmalloc(sizeof(*d)); // 分配空间

    _dictInit(d,type,privDataPtr);
    return d;
}
```

##### `dictResize`

调整哈希大小，包含节点的最小值。

```c
int dictResize(dict *d)
{
    unsigned long minimal; // 最小值
    // 如果不支持调整哈希表大小或正在rehash过程，直接返回
    if (!dict_can_resize || dictIsRehashing(d)) return DICT_ERR;
    minimal = d->ht[0].used; // 已使用插槽数量
    if (minimal < DICT_HT_INITIAL_SIZE) // 小于初始化大小。替换
        minimal = DICT_HT_INITIAL_SIZE;
    return dictExpand(d, minimal);
}
```

##### `_dictExpand`

扩展或创建哈希表。当 `malloc_failed` 为 `non-NULL` 时，分配内存失败时不会出现终止程序（在这种情况下，它就是1）。

```c
int _dictExpand(dict *d, unsigned long size, int* malloc_failed)
{
    if (malloc_failed) *malloc_failed = 0; // 重置内存分配标识
    // 在rehash过程或已使用插槽大于哈希表大小（需要扩展），直接返回
    if (dictIsRehashing(d) || d->ht[0].used > size) return DICT_ERR;

    dictht n; // 新的哈希表
    unsigned long realsize = _dictNextPower(size); // 新的幂级扩展大小
    // 如果新的大小和哈希表大小相等，无效。
    if (realsize == d->ht[0].size) return DICT_ERR;

    // 分配新的哈希表，并将所有指针重置为NULL
    n.size = realsize;
    n.sizemask = realsize-1;
    if (malloc_failed) { // 应该不会进入这里吧？上面已经重置为0了
        n.table = ztrycalloc(realsize*sizeof(dictEntry*)); // 尝试
        *malloc_failed = n.table == NULL;
        if (*malloc_failed) // 这里malloc_failed的值不是NULL？
            return DICT_ERR;
    } else
        n.table = zcalloc(realsize*sizeof(dictEntry*)); // 分配并初始化内存

    n.used = 0;

    // 如果第一个哈希表为NULL，那就是第一个哈希表的初始化。直接赋值返回即可。不是rehash过程
    if (d->ht[0].table == NULL) {
        d->ht[0] = n;
        return DICT_OK;
    }

    // 准备第二个哈希表进行增量rehash过程
    d->ht[1] = n;
    d->rehashidx = 0;
    return DICT_OK;
}
```

##### `dictExpand`

创建或扩展哈希表。直接调用 `_dictExpand` 实现，`malloc_failed=NULL`。

##### `dictTryExpand`

尝试创建或扩展哈希表。如果分配失败，返回 `DICT_ERR`。

```c
int dictTryExpand(dict *d, unsigned long size) {
    int malloc_failed; // 未初始化
    _dictExpand(d, size, &malloc_failed);
    return malloc_failed? DICT_ERR : DICT_OK;
}
```

##### `dictRehash`

执行n个步骤的增量rehash过程（渐进式）。如果仍有键要从旧哈希表移动到新哈希表，则返回1，否则返回0。

```c
int dictRehash(dict *d, int n) {
    // 浏览空插槽（头节点）最大量，每次10个？
    int empty_visits = n*10;
    if (!dictIsRehashing(d)) return 0; // 不在rehash过程

    while(n-- && d->ht[0].used != 0) { // n不为0，且第一个哈希表已使用插槽量非0
        dictEntry *de, *nextde;

        // rehashidx 不能溢出，保证了第一个哈希表还有节点，因为 ht[0].used!=0
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        // 空头节点过滤。非密集？往后偏移，直到非空头节点。如果empty_visits为0，表示没有了。
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        de = d->ht[0].table[d->rehashidx]; // 获取 rehashidx 对应的头节点
        // 将这个桶中所有键从旧哈希表转移到新哈希表，也就是 ht[0] => ht[1]
        while(de) {
            uint64_t h;

            nextde = de->next; // 获取下一个哈希节点，循环变量。
            // 获取新的哈希索引，在ht[1]中
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            // 更新头节点
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde; // 更新循环变量
        }
        // 向后偏移。更新 rehashidx
        d->ht[0].table[d->rehashidx] = NULL;
        d->rehashidx++;
    }

    // 检测是否完全转移
    if (d->ht[0].used == 0) {
        zfree(d->ht[0].table); // 释放原哈希表空间
        d->ht[0] = d->ht[1]; // 整体转移 ht[1] => ht[0]
        _dictReset(&d->ht[1]); // 重置 ht[1]
        d->rehashidx = -1; // 标识 rehash 结束
        return 0; // 无
    }

    // 还需rehash
    return 1;
}
```

##### `timeInMilliseconds`

获取毫秒级时间。

```c
long long timeInMilliseconds(void) {
    struct timeval tv;

    gettimeofday(&tv,NULL);
    return (((long long)tv.tv_sec)*1000)+(tv.tv_usec/1000);
}
```

##### `dictRehashMilliseconds`

以毫秒+增量毫秒为单位 rehash。“delta”的值大于0，在大多数情况下小于1。确切的上限取决于 `dictRehash(d,100)` 的运行时间。

```c
int dictRehashMilliseconds(dict *d, int ms) {
    if (d->pauserehash > 0) return 0;

    long long start = timeInMilliseconds();
    int rehashes = 0;

    while(dictRehash(d,100)) {
        rehashes += 100;
        if (timeInMilliseconds()-start > ms) break;
    }
    return rehashes;
}
```

##### `_dictRehashStep`

单节点 rehash。如果 `pauserehash==0`。

```c
static void _dictRehashStep(dict *d) {
    if (d->pauserehash == 0) dictRehash(d,1);
}
```

##### `dictAdd`

向指定字典添加哈希元素。

```c
int dictAdd(dict *d, void *key, void *val)
{
    dictEntry *entry = dictAddRaw(d,key,NULL);

    if (!entry) return DICT_ERR;
    dictSetVal(d, entry, val);
    return DICT_OK;
}
```

##### `dictAddRaw`

更底层的添加或查找哈希节点。

```c
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)
{
    long index;
    dictEntry *entry;
    dictht *ht;

    if (dictIsRehashing(d)) _dictRehashStep(d); // 正在 rehash 过程 单节点操作

    // 获取新节点索引，返回-1表示节点已存在。
    if ((index = _dictKeyIndex(d, key, dictHashKey(d,key), existing)) == -1)
        return NULL;

    // 定位哈希表。rehash过程就为ht[1]，否则就是ht[0]
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    entry = zmalloc(sizeof(*entry)); // 分配节点空间
    entry->next = ht->table[index]; // 头节点
    ht->table[index] = entry; // 填入哈希节点
    ht->used++; // 更新已使用插槽量

    dictSetKey(d, entry, key); // 设置哈希键
    return entry;
}
```

##### `dictReplace`

添加或替换哈希节点。

```c
int dictReplace(dict *d, void *key, void *val)
{
    dictEntry *entry, *existing, auxentry;

    // 如果entry非NULL，就是添加哈希节点
    entry = dictAddRaw(d,key,&existing);
    if (entry) {
        dictSetVal(d, entry, val);
        return 1;
    }

    // 设置新值，然后释放旧值（该顺序很重要，因为val可能和existing值完全一样）
    auxentry = *existing;
    dictSetVal(d, existing, val);
    dictFreeVal(d, &auxentry);
    return 0;
}
```

##### `dictAddOrFind`

添加或查找哈希节点。直接调用 `dictAddRaw` 实现，返回值非空就是添加，否则就是查找 `existing`。

##### `dictGenericDelete`

查找并移除哈希节点。辅助函数。

```c
static dictEntry *dictGenericDelete(dict *d, const void *key, int nofree) {
    uint64_t h, idx;
    dictEntry *he, *prevHe;
    int table;
	// 空哈希表
    if (d->ht[0].used == 0 && d->ht[1].used == 0) return NULL;

    if (dictIsRehashing(d)) _dictRehashStep(d);
    h = dictHashKey(d, key); // 根据key生成对应哈希值

    for (table = 0; table <= 1; table++) {
        idx = h & d->ht[table].sizemask; // 获取哈希头节点索引
        he = d->ht[table].table[idx]; // 获取对应哈希头节点
        prevHe = NULL;
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key)) { // 键匹配
                // 从链表中移除元素
                if (prevHe) // 更新上个节点的next。如果prevHe存在
                    prevHe->next = he->next;
                else // 否则将当前节点的next节点设置为头节点
                    d->ht[table].table[idx] = he->next;
                if (!nofree) { // nofree 不需要释放。!0，释放
                    dictFreeKey(d, he);
                    dictFreeVal(d, he);
                    zfree(he);
                }
                d->ht[table].used--; // 更新已使用插槽量
                return he; // 返回移除的节点
            }
            prevHe = he; // 记录当前节点为上个节点
            he = he->next; // 更新当前节点
        }
        if (!dictIsRehashing(d)) break; // 非rehash，中断循环
    }
    return NULL; // 未找到对应节点
}
```

##### `dictDelete`

删除哈希节点。直接调用 `dictGenericDelete`，其中 `nofree=0`。

##### `dictUnlink`

接触哈希节点绑定。不需要实际释放对应哈希键值节点。也是直接调用 `dictGenericDelete` 实现，其中 `nofree=1` 表示不需要释放对应空间。

##### `dictFreeUnlinkedEntry`

释放调用 `dictUnlink` 解绑的哈希节点。

```c
void dictFreeUnlinkedEntry(dict *d, dictEntry *he) {
    if (he == NULL) return;
    dictFreeKey(d, he);
    dictFreeVal(d, he);
    zfree(he);
}
```

`_dictClear`

销毁整个字典。

```c
int _dictClear(dict *d, dictht *ht, void(callback)(void *)) {
    unsigned long i;
    // 释放所有哈希节点
    for (i = 0; i < ht->size && ht->used > 0; i++) {
        dictEntry *he, *nextHe;
		// 对字典私有数据的处理，i=0时处理，仅一次
        if (callback && (i & 65535) == 0) callback(d->privdata);
		// 当前头节点为空，直接跳过
        if ((he = ht->table[i]) == NULL) continue;
        while(he) {
            nextHe = he->next;
            // 释放 键 值 节点
            dictFreeKey(d, he);
            dictFreeVal(d, he);
            zfree(he);
            // 更新可使用插槽量，和循环变量he
            ht->used--;
            he = nextHe;
        }
    }
    // 释放哈希表
    zfree(ht->table);
    // 重新初始化哈希表
    _dictReset(ht);
    return DICT_OK;
}
```

##### `dictRelease`

销毁并释放哈希表。

```c
void dictRelease(dict *d)
{
    _dictClear(d,&d->ht[0],NULL);
    _dictClear(d,&d->ht[1],NULL);
    zfree(d); // 释放字典
}
```

##### `dictFind`

查找哈希节点。根据键。

```c
dictEntry *dictFind(dict *d, const void *key)
{
    dictEntry *he;
    uint64_t h, idx, table;

    if (dictSize(d) == 0) return NULL; // 空字典
    if (dictIsRehashing(d)) _dictRehashStep(d);
    h = dictHashKey(d, key);
    for (table = 0; table <= 1; table++) {
        idx = h & d->ht[table].sizemask;
        he = d->ht[table].table[idx];
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key))
                return he;
            he = he->next;
        }
        if (!dictIsRehashing(d)) return NULL;
    }
    return NULL;
}
```

##### `dictFetchValue`

获取对应哈希节点的值。

```c
void *dictFetchValue(dict *d, const void *key) {
    dictEntry *he;

    he = dictFind(d,key);
    return he ? dictGetVal(he) : NULL;
}
```

##### `dictFingerprint`

字典签名。

```c
long long dictFingerprint(dict *d) {
    long long integers[6], hash = 0;
    int j;

    integers[0] = (long) d->ht[0].table;
    integers[1] = d->ht[0].size;
    integers[2] = d->ht[0].used;
    integers[3] = (long) d->ht[1].table;
    integers[4] = d->ht[1].size;
    integers[5] = d->ht[1].used;

    for (j = 0; j < 6; j++) {
        hash += integers[j];
        /* For the hashing step we use Tomas Wang's 64 bit integer hash. */
        hash = (~hash) + (hash << 21); // hash = (hash << 21) - hash - 1;
        hash = hash ^ (hash >> 24);
        hash = (hash + (hash << 3)) + (hash << 8); // hash * 265
        hash = hash ^ (hash >> 14);
        hash = (hash + (hash << 2)) + (hash << 4); // hash * 21
        hash = hash ^ (hash >> 28);
        hash = hash + (hash << 31);
    }
    return hash;
}
```

##### `dictGetIterator`

获取字典迭代器。

```c
dictIterator *dictGetIterator(dict *d)
{
    dictIterator *iter = zmalloc(sizeof(*iter));

    iter->d = d; // 迭代字典
    iter->table = 0; // 哈希表
    iter->index = -1; // 迭代索引
    iter->safe = 0;
    iter->entry = NULL; // 当前哈希节点
    iter->nextEntry = NULL; // 下一个哈希节点
    return iter;
}
```

##### `dictGetSafeIterator`

获取字典安全迭代器。调用 `dictGetIterator` 后将 `iter->safe` 设置为1即可。

##### `dictNext`

迭代循环。

```c
dictEntry *dictNext(dictIterator *iter)
{
    while (1) {
        if (iter->entry == NULL) { // 当前节点为NULL，说明首次迭代过程？不能。链切换，也会如此。
            dictht *ht = &iter->d->ht[iter->table]; // 获取对应哈希表
            if (iter->index == -1 && iter->table == 0) { // 这里可证为首次迭代过程
                if (iter->safe) // 如果为安全迭代器，暂停 rehash 过程
                    dictPauseRehashing(iter->d);
                else // 否则，检测签名
                    iter->fingerprint = dictFingerprint(iter->d);
            }
            iter->index++; // 更新迭代索引
            if (iter->index >= (long) ht->size) { // 如果溢出，检测，是否当前哈希表涉及到 rehash 判断后，切换表，和重置索引
                if (dictIsRehashing(iter->d) && iter->table == 0) {
                    iter->table++; // 切换哈希表，和重置索引
                    iter->index = 0;
                    ht = &iter->d->ht[1]; // 更新ht为ht[1]
                } else {
                    break; // 如果不是，结束
                }
            }
            iter->entry = ht->table[iter->index]; // 更新当前节点
        } else {
            iter->entry = iter->nextEntry; // 更新当前节点
        }
        if (iter->entry) {
            // 保存下一个节点，迭代器调用者可能会删除当前返回节点
            iter->nextEntry = iter->entry->next; 
            return iter->entry; // 返回当前迭代节点
        }
    }
    return NULL; // 迭代结束 NULL
}
```

##### `dictReleaseIterator`

释放指定迭代器。

```c
void dictReleaseIterator(dictIterator *iter)
{
    if (!(iter->index == -1 && iter->table == 0)) { // 已经迭代过
        if (iter->safe) // 安全迭代器，需要主动重启 rehash 过程
            dictResumeRehashing(iter->d); // 也就是更新 d->pauserehash--
        else
            assert(iter->fingerprint == dictFingerprint(iter->d)); // 验证迭代器签名
    }
    zfree(iter); // 释放迭代器空间
}
```

##### `dictGetRandomKey`

随机获取一个哈希节点。

```c
dictEntry *dictGetRandomKey(dict *d)
{
    dictEntry *he, *orighe; // 目标节点，和原始节点？
    unsigned long h;
    int listlen, listele;

    if (dictSize(d) == 0) return NULL; //空字典
    if (dictIsRehashing(d)) _dictRehashStep(d); // 还是不大清楚作用
    if (dictIsRehashing(d)) { // 在 rehash 过程
        do {
            // 0->rehashidx-1 没有元素
            h = d->rehashidx + (randomULong() % (dictSlots(d) - d->rehashidx));
            he = (h >= d->ht[0].size) ? d->ht[1].table[h - d->ht[0].size] :
                                      d->ht[0].table[h];
        } while(he == NULL);
    } else {
        do {
            h = randomULong() & d->ht[0].sizemask;
            he = d->ht[0].table[h];
        } while(he == NULL);
    }
	
    // 找到一个非空桶，但是它是一个链表，需要从这个链表中获取一个随机元素
    listlen = 0; // 链表长度
    orighe = he; // 原始节点
    while(he) { // next 直到空 计算链表长度
        he = he->next;
        listlen++;
    }
    listele = random() % listlen; // 随机节点位置
    he = orighe; // 归还
    while(listele--) he = he->next; // 循环到目标节点为止
    return he;
}
```

##### `dictGetSomeKeys`

对字典进行采样，随机返回几个键。

```c
unsigned int dictGetSomeKeys(dict *d, dictEntry **des, unsigned int count) {
    // des 存放随机节点的数组 
    unsigned long j; // 内部哈希表ID 0|1
    unsigned long tables; // 需要抽查的哈希表数量 1|2
    unsigned long stored = 0, maxsizemask;
    unsigned long maxsteps;

    if (dictSize(d) < count) count = dictSize(d); // 检测并重置count（看需要）
    maxsteps = count*10; // 最大步数？每次10个？

    // 尝试做一个与 count 成比例的 rehash 过程批次？
    for (j = 0; j < count; j++) {
        if (dictIsRehashing(d)) // 还是需要判断是否在 rehash 过程
            _dictRehashStep(d);
        else
            break;
    }

    tables = dictIsRehashing(d) ? 2 : 1; // 根据当前 rehash 区分
    maxsizemask = d->ht[0].sizemask; // 最大大小掩码？
    if (tables > 1 && maxsizemask < d->ht[1].sizemask) // 确实最大 sizemask
        maxsizemask = d->ht[1].sizemask;

    // 选取一个随机点（索引？）在较大的哈希表类
    unsigned long i = randomULong() & maxsizemask;
    unsigned long emptylen = 0; // 到目前，连续的空节点长度
    while(stored < count && maxsteps--) { // count 随机节点的指定数量
        for (j = 0; j < tables; j++) { // 0|1
            // 恒定 rehash ：将索引更新到ht[0]在rehash过程已经浏览的（已经迁到ht[1]），期间已经没有bucket
            // 所以可以直接过滤 ht[0] 的 0->idx-1 的buckets
            if (tables == 2 && j == 0 && i < (unsigned long) d->rehashidx) {
                // 然而，如果当前索引超出第二表范围
                // 在两表就没有任何元素在更新到当前rehash索引
                // 从大表切到小表
                if (i >= d->ht[1].size)
                    i = d->rehashidx;
                else
                    continue;
            }
            if (i >= d->ht[j].size) continue; // 已经溢出当前表，切到下一个哈希表 ht[1]
            dictEntry *he = d->ht[j].table[i];

            // 统计连续空桶，跳转到其他定位直到满足count数量（5是最小值？）
            if (he == NULL) {
                emptylen++;
                if (emptylen >= 5 && emptylen > count) {
                    i = randomULong() & maxsizemask; // 重新选取索引
                    emptylen = 0;
                }
            } else {
                emptylen = 0;
                while (he) {
                    // 搜集所有在迭代过程非空的buckets（桶）
                    *des = he;
                    des++; // des 偏移
                    he = he->next;
                    stored++; // 更新已存储节点的数量
                    if (stored == count) return stored; // 达到需要获取的数量
                }
            }
        }
        i = (i+1) & maxsizemask; // 更新索引
    }
    return stored;
}
```

##### `dictGetFairRandomKey`

获取一个更合理的随机节点，相比于 `dictGetRandomKey` 而言，多了从更多桶抽取的可能性。

```c
dictEntry *dictGetFairRandomKey(dict *d) {
    dictEntry *entries[GETFAIR_NUM_ENTRIES]; // GETFAIR_NUM_ENTRIES 15
    unsigned int count = dictGetSomeKeys(d,entries,GETFAIR_NUM_ENTRIES); // 随机键
    // 在哈希表有元素时，dictGetSomeKeys 也可能返回0个元素，比较糟糕的是
    // 所以还是需要调用 dictGetRandomKey 至少返回一个节点。
    if (count == 0) return dictGetRandomKey(d);
    unsigned int idx = rand() % count; // 再次随机
    return entries[idx];
}
```

##### `rev`

反转位，工具函数。

```c
static unsigned long rev(unsigned long v) {
    unsigned long s = CHAR_BIT * sizeof(v); // 位大小，必须是2的幂
    unsigned long mask = ~0UL; // 设置掩码 0000 0000...
    while ((s >>= 1) > 0) { // 位右移 >>
        mask ^= (mask << s); // << ^
        v = ((v >> s) & mask) | ((v << s) & ~mask);
    }
    return v;
}
```

##### `dictScan`

迭代字典的元素。

```c
unsigned long dictScan(dict *d,
                       unsigned long v,
                       dictScanFunction *fn,
                       dictScanBucketFunction* bucketfn,
                       void *privdata)
{
    dictht *t0, *t1;
    const dictEntry *de, *next;
    unsigned long m0, m1;

    if (dictSize(d) == 0) return 0;
    // 暂停 rehash，不判断是否在rehash吗？
    dictPauseRehashing(d);

    if (!dictIsRehashing(d)) {
        t0 = &(d->ht[0]);
        m0 = t0->sizemask;

        // 在当前位置，调用 桶回调处理函数
        if (bucketfn) bucketfn(privdata, &t0->table[v & m0]);
        de = t0->table[v & m0];
        while (de) {
            next = de->next;
            fn(privdata, de); // 节点回调处理函数
            de = next; // 继续
        }

        /* Set unmasked bits so incrementing the reversed cursor
         * operates on the masked bits */
        v |= ~m0;

        /* Increment the reverse cursor */
        v = rev(v);
        v++;
        v = rev(v);

    } else {
        t0 = &d->ht[0];
        t1 = &d->ht[1];

        /* Make sure t0 is the smaller and t1 is the bigger table */
        if (t0->size > t1->size) {
            t0 = &d->ht[1];
            t1 = &d->ht[0];
        }

        m0 = t0->sizemask;
        m1 = t1->sizemask;

        /* Emit entries at cursor */
        if (bucketfn) bucketfn(privdata, &t0->table[v & m0]);
        de = t0->table[v & m0];
        while (de) {
            next = de->next;
            fn(privdata, de);
            de = next;
        }

        /* Iterate over indices in larger table that are the expansion
         * of the index pointed to by the cursor in the smaller table */
        do {
            /* Emit entries at cursor */
            if (bucketfn) bucketfn(privdata, &t1->table[v & m1]);
            de = t1->table[v & m1];
            while (de) {
                next = de->next;
                fn(privdata, de);
                de = next;
            }

            /* Increment the reverse cursor not covered by the smaller mask.*/
            v |= ~m1;
            v = rev(v);
            v++;
            v = rev(v);

            /* Continue while bits covered by mask difference is non-zero */
        } while (v & (m0 ^ m1));
    }

    dictResumeRehashing(d); // 还原 rehash

    return v;
}
```

##### `dictEmpty`

重置字典。

```c
void dictEmpty(dict *d, void(callback)(void*)) {
    _dictClear(d,&d->ht[0],callback);
    _dictClear(d,&d->ht[1],callback);
    d->rehashidx = -1;
    d->pauserehash = 0;
}
```

##### `dictEnableResize`

启用字典大小调整参数，`dict_can_resize = 1`。

##### `dictDisableResize`

禁用字典大小调整参数，`dict_can_resize = 0`。

##### `dictGetHash`

获取键的哈希值，直接调用 `dictHashKey` 实现。

##### `dictFindEntryRefByPtrAndHash`

根据指针和哈希值查找节点的引用？

```c
dictEntry **dictFindEntryRefByPtrAndHash(dict *d, const void *oldptr, uint64_t hash) {
    dictEntry *he, **heref;
    unsigned long idx, table;

    if (dictSize(d) == 0) return NULL;
    for (table = 0; table <= 1; table++) {
        idx = hash & d->ht[table].sizemask;
        heref = &d->ht[table].table[idx];
        he = *heref;
        while(he) {
            if (oldptr==he->key) // 键的指针匹配
                return heref;
            heref = &he->next;
            he = *heref;
        }
        if (!dictIsRehashing(d)) return NULL;
    }
    return NULL;
}
```

#### 本章小结

通过对源码的局部解读，可以看到字典的实现基于哈希表，而C语言没有这类型，`Redis` 自行实现了一套，采用 `dict > ht > table > headptr` 的结构。然而哈希冲突是个很重要的问题（当有两个或以上数量的键被分配到了哈希表数组的同一个索引上面的情况），这里采用了公开链地址法解决该问题，由于每个哈希节点都一个 `next` 指针，所以当多个节点分配到同一个索引上时，可形成单向链表。这也难免在查找采用循环结构。不过这也是添加新节点总是加到链表头部的原因，不可能迭代到尾节点追加，太废了。

在哈希节点增加或减少时，也会触发 `rehash` 过程，对哈希表的大小进行相应的调整，`ht[1]`  就是使用在该情况的。在扩展时，`ht[1]` 的大小总是第一个大于等于 `ht[0].used+1`的2^n。如果收缩，`ht[1]`的大小为 第一个大于等于 `ht[0].used`的2^n。这样可有效避免对哈希表进行频繁的调整，造成不必要的损耗。`rehash` 过程就是，重新计算键的哈希值和索引值，将键值对放置到 `ht[1]` 的相应位置，都从 `ht[0]` 迁移到 `ht[1]` 后，释放 `ht[0]`，将 `ht[1]` 设置为 `ht[0]`，`ht[1]` 会创建新的空白哈希表，为下一次准备。当然迁移过程并不是一次性完成，而是渐进式完成。主要是键值对过多时，该过程对机器性能影响太大。