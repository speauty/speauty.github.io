# Redis源码之数据类型解析-SDS {docsify-ignore}

当前分析Redis版本为6.2，需要注意。

#### 基础结构

SDS（Simple Dynamic Strings），简单动态字符串，主要用于存储字符串和整型数据（二进制安全）。在其头文件中是这样定义`typedef char *sds`，SDS是字符指针类型（或者字符数组），有五种类型，应该根据字符数组长度进行分类的：

```c
// 该类型似乎没用，在创建新sds中，检测到类型为sdshdr5,直接设置为sdshdr8
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; // 低三位存储类型，高五位存储长度，比较特殊
    char buf[];
}; // 2 bytes
// T: 8 16 32 64
// 由于这几种除了相应类型有变化，结构基本一致，这里就用T代替
// unint8(1 byte) unint16(2 bytes) unint32(4 bytes) unint64(8 bytes)
struct __attribute__ ((__packed__)) sdshdrT {
    uintT_t len; // 记录字符串长度
    uintT_t alloc; // 排除头字段和空指针后的可分配空间
    unsigned char flags; // 1 byte 字符串类型
    char buf[]; // 1 byte
};
```

其中`__attribute__((__packed__))`表示当前结构体不做对齐优化处理，实际占多少就是多少字节的紧凑型结构体，比如这里的sdshdr5在没有`__packed__`指定属性的情况下应该占4个字节，但实际上这里只用了2个字节，看起来只有两个字节的差距，但对于sdshdr8及以后的类型就不止如此了。结构体中的flags参数还需要说明一下，该字节的低三位存储的是指定类型，高五位除了sdshdr5中存储了当前字符串长度之外的其他sdshdr均为使用。

比较细节的就是将`char buf[]`放在结构体尾部，这种不定长的字符数组，放在结构体中间会对长度有极大影响，难以扩展，比如在增加字符串时，需要将后面进行备份再拓展，极其不方便。像现在这样，放在结构体尾部，我们只需要根据字符指针往前偏移固定位置即可找到相应参数，实际上也是这么实现的。

#### 宏定义

接下来，定义了五种类型的标识和一些其他参数函数：

```c
// SDS类型 flags&SDS_TYPE_MASK
#define SDS_TYPE_5  0 // 0000 0000
#define SDS_TYPE_8  1 // 0000 0001
#define SDS_TYPE_16 2 // 0000 0010
#define SDS_TYPE_32 3 // 0000 0011
#define SDS_TYPE_64 4 // 0000 0100
#define SDS_TYPE_MASK 7 // SDS类型掩码 0000 0111
#define SDS_TYPE_BITS 3 // SDS类型位长度
// sdshdr结构体指针变量（临时？）
#define SDS_HDR_VAR(T,s) struct sdshdr##T *sh = (void*)((s)-(sizeof(struct sdshdr##T)));
// 获取sdshdr结构体: T是结构体后缀数字(8/16/32/64), s是字符数组指针
// 做指针偏移，字符数组指针减去结构体长度即结构体指针，再进行类型指定
#define SDS_HDR(T,s) ((struct sdshdr##T *)((s)-(sizeof(struct sdshdr##T))))
// 获取sdshdr5的长度: 对其flags参数右移SDS类型位长度，移除类型标志
#define SDS_TYPE_5_LEN(f) ((f)>>SDS_TYPE_BITS)
```

#### 私有函数

##### `sdslen`

获取sds长度，主要通过记录的长度，除去sds5比较特殊之外，其余大体相似。

```c
static inline size_t sdslen(const sds s) {
    unsigned char flags = s[-1]; // 获取flags字段，就在字符串（sds）的前一字节
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            return SDS_TYPE_5_LEN(flags);
        case SDS_TYPE_T: // T 8 16 32 64
            // 获取到对应sdshdr结构体，读取len字段，记录的是字符串长度
            return SDS_HDR(T,s)->len;
        //...
    }
    return 0;
}
```

##### `sdsavail`

获取sds可用长度。

```c
static inline size_t sdsavail(const sds s) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5: {
            return 0; // sdshdr5不支持扩充
        }
        case SDS_TYPE_T: { // T 8 16 32 64
            SDS_HDR_VAR(T,s); // 赋值临时指针变量sh
            return sh->alloc - sh->len;
        }
        // ...
    }
    return 0;
}
```

##### `sdssetlen`

设置sds长度，应该是初始化时使用，直接设置。

```c
static inline void sdssetlen(sds s, size_t newlen) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            {
                unsigned char *fp = ((unsigned char*)s)-1; // 获取flags指针
                // 左移SDS_TYPE_BITS位（增加类型位），然后和SDS_TYPE_5进行按位或运算
                *fp = SDS_TYPE_5 | (newlen << SDS_TYPE_BITS); 
            }
            break;
        case SDS_TYPE_T: // T 8 16 32 64
            SDS_HDR(T,s)->len = newlen; // 找到sds结构体len字段，直接赋值
            break;
        // ...
    }
}
```

##### `sdsinclen`

增加sds长度，inc应该是increase。

```c
static inline void sdsinclen(sds s, size_t inc) { // 同设置长度相似
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            {
                unsigned char *fp = ((unsigned char*)s)-1;
                unsigned char newlen = SDS_TYPE_5_LEN(flags)+inc;
                *fp = SDS_TYPE_5 | (newlen << SDS_TYPE_BITS);
            }
            break;
        case SDS_TYPE_T: // T 8 16 32 64
            SDS_HDR(8,s)->len += inc;
            break;
    }
}
```

##### `sdsalloc`

获取sds可分配空间，也就是`sdsavail`+`sdslen`。

```c
static inline size_t sdsalloc(const sds s) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            return SDS_TYPE_5_LEN(flags);
        case SDS_TYPE_T: // T 8 16 32 64
            return SDS_HDR(T,s)->alloc;
    }
    return 0;
}
```

##### `sdssetalloc`

设置可分配空间。

```c
static inline void sdssetalloc(sds s, size_t newlen) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            // 该类型没有可分配信息
            break;
        case SDS_TYPE_T: // T 8 16 32 64
            SDS_HDR(T,s)->alloc = newlen;
            break;
    }
}
```

##### `sdsHdrSize`

获取sds的头字段长度，也就是对应结构体的长度。

```c
static inline int sdsHdrSize(char type) {
    switch(type&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            return sizeof(struct sdshdr5);
        case SDS_TYPE_T: // T 8 16 32 64
            return sizeof(struct sdshdrT);
    }
    return 0;
}
```

##### `sdsReqType`

获取sds需要的类型，根据字符串长度。

```c
static inline char sdsReqType(size_t string_size) {
    if (string_size < 1<<5) // 2^5
        return SDS_TYPE_5;
    if (string_size < 1<<8) // 2^8
        return SDS_TYPE_8;
    if (string_size < 1<<16) // 2^16
        return SDS_TYPE_16;
#if (LONG_MAX == LLONG_MAX) // 如果长精度与双长精度的最大值相等
    if (string_size < 1ll<<32) // 2^32
        return SDS_TYPE_32;
    return SDS_TYPE_64;
#else // 不等的情况，否决了64的可能性？直接采用32
    return SDS_TYPE_32;
#endif
}
```

##### `sdsTypeMaxSize`

获取sds类型对应字符串的最大长度。

```c
static inline size_t sdsTypeMaxSize(char type) {
    // 留一个字符结束符
    if (type == SDS_TYPE_5)
        return (1<<5) - 1;
    if (type == SDS_TYPE_8)
        return (1<<8) - 1;
    if (type == SDS_TYPE_16)
        return (1<<16) - 1;
#if (LONG_MAX == LLONG_MAX)
    if (type == SDS_TYPE_32)
        return (1ll<<32) - 1;
#endif
    // 和 SDS_TYPE_64/SDS_TYPE_32最大值相等的
    return -1;
}
```

##### `_sdsnewlen`

创建sds新字符串。

```c
// init 初始化字符串指针，可以为空
// initlen 初始化长度
// trymalloc 是否尝试分配空间
sds _sdsnewlen(const void *init, size_t initlen, int trymalloc) {
    void *sh;
    sds s;
    char type = sdsReqType(initlen); // 根据初始化长度确定sds类型
    // 创建空字符串一般都是为了追加，所以采用sdshdr8。而不是sdshdr5。强行更正类型。
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    int hdrlen = sdsHdrSize(type);
    unsigned char *fp; // flags标识指针
    size_t usable; // 可用大小

    assert(initlen + hdrlen + 1 > initlen); // 防止溢出
    // 分配空间
    sh = trymalloc?
        s_trymalloc_usable(hdrlen+initlen+1, &usable) :
        s_malloc_usable(hdrlen+initlen+1, &usable);
    if (sh == NULL) return NULL;
    if (init==SDS_NOINIT) // const char *SDS_NOINIT = "SDS_NOINIT";
        init = NULL; // 重置为空
    else if (!init)
        memset(sh, 0, hdrlen+initlen+1); // 重置空间，防止脏数据干扰
    s = (char*)sh+hdrlen; // sdshdr结构偏移hdrlen，到字符串指针位置
    fp = ((unsigned char*)s)-1; // 获取flags指针位置
    usable = usable-hdrlen-1; // 可分配空间。。。分配的大小除开sds头长度和结束符
    if (usable > sdsTypeMaxSize(type)) // 标准化对齐限制。对应类型最大可用大小，该多少还是多少
        usable = sdsTypeMaxSize(type);
    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS);
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            sh->len = initlen;
            sh->alloc = usable;
            *fp = type;
            break;
        }
        case SDS_TYPE_16: {
            // ...
        }
        case SDS_TYPE_32: {
            // ...
        }
        case SDS_TYPE_64: {
            // ...
        }
    }
    if (initlen && init)
        memcpy(s, init, initlen); // 拷贝字符串 init
    s[initlen] = '\0'; // 补充结束符
    return s; // 返回sds
}
```

##### `sdsnewlen`

创建指定长度sds新字符串，直接调用 `_sdsnewlen` 即可实现，trymalloc为0。

##### `sdstrynewlen`

尝试创建指定长度sds新字符串，和 `sdsnewlen` 基本一样，调用 `_sdsnewlen` 第三个参数为1。

##### `sdsempty`

创建空字符串，调用 `sdsnewlen("",0)` 即可。

#### 接口函数

##### `sdsnew`

从一个以null结束的C字符串创建sds字符串。

```c
sds sdsnew(const char *init) {
    size_t initlen = (init == NULL) ? 0 : strlen(init); // 计算初始化长度
    return sdsnewlen(init, initlen);
}
```

##### `sdsdup`

复制sds字符串，调用 `sdsnewlen(s, sdslen(s))` 实现。

##### `sdsfree`

释放sds字符串，直接调用 `s_free` 对sds对应的sdshdr结构体处理。

##### `sdsupdatelen`

更新sds字符串长度，将len设置为字符串的真实长度。

##### `sdsclear`

将sds字符串置空，直接将len设置为0，并将第一个字符设置为结束符。

##### `sdsMakeRoomFor`

扩充sds长度，不会改变sdshdr->len，只是增加了addlen长度的空间。

```c
sds sdsMakeRoomFor(sds s, size_t addlen) { // addlen 需要扩充的长度
    void *sh, *newsh;
    size_t avail = sdsavail(s); // 获取sds可用长度
    size_t len, newlen;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;
    size_t usable;

    // 如果可用长度大于等于扩充长度，立即返回sds即可
    if (avail >= addlen) return s;

    len = sdslen(s);
    sh = (char*)s-sdsHdrSize(oldtype);
    newlen = (len+addlen); // 新的sds长度
    assert(newlen > len); // 防止溢出
    // #define SDS_MAX_PREALLOC (1024*1024) sds最大预分配大小，在sds.h中定义
    // 新长度小于SDS_MAX_PREALLOC，将新长度扩展成自身两倍，防止频繁分配。
    // 如果大于SDS_MAX_PREALLOC，直接加上SDS_MAX_PREALLOC。
    if (newlen < SDS_MAX_PREALLOC)
        newlen *= 2;
    else
        newlen += SDS_MAX_PREALLOC;

    type = sdsReqType(newlen); // 根据新计算长度确定sds类型

    // 该操作是扩充字符串，sdshdr5不支持更多初始空间，所以每次追加操作都会调用sdsMakeRoomFor
    // 确实废。所以，和_sdsnewlen一样，直接设置为sdshdr8类型。
    if (type == SDS_TYPE_5) type = SDS_TYPE_8;

    hdrlen = sdsHdrSize(type);
    assert(hdrlen + newlen + 1 > len); // 防止溢出
    if (oldtype==type) { // 如果类型相等，直接重分配
        newsh = s_realloc_usable(sh, hdrlen+newlen+1, &usable);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+hdrlen; // 直接更新s指针
    } else {
        // 一旦sdshdr头发生变化，那么需要移除字符串前面的，不能使用realloc
        newsh = s_malloc_usable(hdrlen+newlen+1, &usable);
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1); //拷贝字符串s
        s_free(sh); // 释放原串
        s = (char*)newsh+hdrlen; // 重订s指针
        s[-1] = type;
        sdssetlen(s, len); // 设置长度
    }
    // 处理可分配空间大小
    usable = usable-hdrlen-1;
    if (usable > sdsTypeMaxSize(type))
        usable = sdsTypeMaxSize(type);
    sdssetalloc(s, usable);
    return s;
}
```

##### `sdsRemoveFreeSpace`

和 `sdsMakeRoomFor` 相反操作，瘦身。

```c
sds sdsRemoveFreeSpace(sds s) {
    void *sh, *newsh;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen, oldhdrlen = sdsHdrSize(oldtype);
    size_t len = sdslen(s);
    size_t avail = sdsavail(s);
    sh = (char*)s-oldhdrlen;

    // 如果没有可用长度，直接返回即可，说明不需要瘦身
    if (avail == 0) return s;

    // 根据长度确定最小sds头，更好适配字符串
    type = sdsReqType(len);
    hdrlen = sdsHdrSize(type);

    if (oldtype==type || type > SDS_TYPE_8) { // 类型不变，或类型大于sdshdr8
        newsh = s_realloc(sh, oldhdrlen+len+1); // 重新分配空间
        if (newsh == NULL) return NULL;
        s = (char*)newsh+oldhdrlen;
    } else {
        newsh = s_malloc(hdrlen+len+1); // 分配新空间
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1); // 拷贝s字符串
        s_free(sh); // 释放原串
        s = (char*)newsh+hdrlen; // 重新定位sds
        s[-1] = type;
        sdssetlen(s, len);
    }
    sdssetalloc(s, len); // 设置可分配空间大小为字符串真实长度，适配即可
    return s;
}
```

##### `sdsAllocSize`

获取总可分配空间，包括头大小，可分配空间大小和一个字符结束符。

##### `sdsAllocPtr`

获取sds实际可分配指针，也就是对应的sdshdr。

##### `sdsIncrLen`

sds增加长度，长度可以为负（表示向右截断）。

```c
void sdsIncrLen(sds s, ssize_t incr) { // incr 可以小于0
    unsigned char flags = s[-1];
    size_t len; // 更新后的长度
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5: {
            unsigned char *fp = ((unsigned char*)s)-1;
            unsigned char oldlen = SDS_TYPE_5_LEN(flags);
            assert((incr > 0 && oldlen+incr < 32) || (incr < 0 && oldlen >= (unsigned int)(-incr)));
            *fp = SDS_TYPE_5 | ((oldlen+incr) << SDS_TYPE_BITS);
            len = oldlen+incr;
            break;
        }
        case SDS_TYPE_T: { // T 8 16 32 64
            SDS_HDR_VAR(T,s);
            assert((incr >= 0 && sh->alloc-sh->len >= incr) || (incr < 0 && sh->len >= (unsigned int)(-incr)));
            len = (sh->len += incr);
            break;
        }
        default: len = 0; // 避免汇编警告
    }
    s[len] = '\0'; // 补充结束符
}
```

##### `sdsgrowzero`

以0扩充字符串，指定长度len。

```c
sds sdsgrowzero(sds s, size_t len) {
    size_t curlen = sdslen(s); // 获取当前字符串长度

    if (len <= curlen) return s; // 指定长度过小，直接返回即可，不需要填充
    s = sdsMakeRoomFor(s,len-curlen); // 反之，需要扩充，先扩充空间
    if (s == NULL) return NULL;

    // 以0填充。初始化，防止脏数据干扰
    memset(s+curlen,0,(len-curlen+1));
    sdssetlen(s, len);
    return s;
}
```

##### `sdscatlen`

追加指定长度len的二进制安全C字符串t到sds字符串。计算原长度，调用 `sdsMakeRoomFor` 扩展空间，然后拷贝字符串t，设置sds的len，最后补充结束符。

##### `sdscat`

追加C字符串（以null结束的），相当于对 `sdscatlen` 的包装。

##### `sdscatsds`

追加sds字符串，和 `sdscat` 相似。

##### `sdscpylen`

覆盖操作，将指定长度len的二进制安全字符串t覆盖sds字符串。

```c
sds sdscpylen(sds s, const char *t, size_t len) {
    if (sdsalloc(s) < len) { // 空间不够，需要进行扩容，如果
        s = sdsMakeRoomFor(s,len-sdslen(s));
        if (s == NULL) return NULL;
    }
    memcpy(s, t, len); // 直接拷贝
    s[len] = '\0';
    sdssetlen(s, len);
    return s;
}
```

##### `sdscpy`

覆盖拷贝，以null结束的字符串到sds字符串，直接调用 `sdscpylen`。

##### `sdsll2str`

双长整型转字符串，s是已分配好内存的地址。

```c
int sdsll2str(char *s, long long value) {
    char *p, aux; // aux 临时中间交换变量
    unsigned long long v;
    size_t l;

    // 生成字符串表示，这个方法会对字符串反转
    v = (value < 0) ? -value : value; // 转为非负数
    p = s; // 指向同一块空间
    do {
        // 按低位到高位顺序
        // '0' ASCII 48
        *p++ = '0'+(v%10); // *p = val & p++
        v /= 10;
    } while(v);
    if (value < 0) *p++ = '-'; // 判断符号

    l = p-s; // 计算长度
    *p = '\0'; // 补充结束符

    // 所以，再次反转表示
    p--;
    while(s < p) { // s向后 p向前
        aux = *s;
        *s = *p;
        *p = aux;
        s++;
        p--;
    }
    return l;
}
```

##### `sdsull2str`

无符号双长整型转字符串，和 `sdsll2str` 大体相似。

##### `sdsfromlonglong`

从双长整型转为sds字符串。

```c
sds sdsfromlonglong(long long value) {
      char buf[SDS_LLSTR_SIZE]; // #define SDS_LLSTR_SIZE 21
      int len = sdsll2str(buf,value);
      return sdsnewlen(buf,len);
}
```

##### `sdscatvprintf`

暂未分析。

##### `sdscatprintf`

将任意数量字符串追加到指定sds。

##### `sdscatfmt`

同 `sdscatprintf` 相似。

##### `sdstrim`

清除sds两端所有指定字符。

```c
sds sdstrim(sds s, const char *cset) {
    char *start, *end, *sp, *ep;
    size_t len;

    sp = start = s; // 定位到字符串头
    ep = end = s+sdslen(s)-1; // 定位到字符串尾
    // 从头开始遍历，只要当前sp指向的字符，在cset中，指针就继续后移
    while(sp <= end && strchr(cset, *sp)) sp++;
    // 从尾部朝前遍历，只要当前ep指向的字符，在cset中，指针就继续前移
    while(ep > sp && strchr(cset, *ep)) ep--;
    // 到这的时候，sp和ep所指向的字符，在cset中不存在，但是sp和ep之间的不管
    len = (sp > ep) ? 0 : ((ep-sp)+1); // 计算剩余长度
    if (s != sp) memmove(s, sp, len); // 如果有需要，前移字符串内容
    s[len] = '\0'; // 补充结束符
    sdssetlen(s,len); // 更新 sds->len
    return s;
}
```

##### `sdsrange`

截取从start到end索引的子串，闭区间。

```c
void sdsrange(sds s, ssize_t start, ssize_t end) {
    size_t newlen, len = sdslen(s);

    if (len == 0) return; // 空串，不做处理
    if (start < 0) { // 小于0，反向截取
        start = len+start;
        if (start < 0) start = 0; // 如果start还小于0，说明负值过大，默认从0开始
    }
    if (end < 0) { // 结束索引也一样
        end = len+end;
        if (end < 0) end = 0;
    }
    // 计算新长度
    newlen = (start > end) ? 0 : (end-start)+1;
    if (newlen != 0) {
        if (start >= (ssize_t)len) {
            newlen = 0;
        } else if (end >= (ssize_t)len) { // 当end大于字符串长度时，截取到尾部
            end = len-1;
            newlen = (start > end) ? 0 : (end-start)+1;
        }
    }
    if (start && newlen) memmove(s, s+start, newlen);
    s[newlen] = 0; // 补充结束符
    sdssetlen(s,newlen); // 更新长度
}
```

##### `sdstolower`

将sds字符串转为小写，循环调用 `tolower` 即可。

##### `sdstoupper`

将sds字符串转为大写，循环调用 `toupper` 即可。

##### `sdscmp`

比较两个sds字符串，s1大返回正数，s2大返回负数，相等就为0。

##### `sdssplitlen`

使用指定分割字符或字符串对sds字符进行分割，返回分割后的sds字符串数组。

```c
sds *sdssplitlen(const char *s, ssize_t len, const char *sep, int seplen, int *count) { // count 分割后的数组数量
    // elements 返回字符串数组的元素个数
    // slots 用于申请返回字符串数组的容量
    int elements = 0, slots = 5;
    long start = 0, j; // 标记分割位置
    sds *tokens; // 存储分割好的sds字符串数组
	// 分割字符串长度或目标sds字符串长度为0，不进行处理，直接返回null
    if (seplen < 1 || len < 0) return NULL;

    tokens = s_malloc(sizeof(sds)*slots); // 申请slots个sds空间
    if (tokens == NULL) return NULL;

    if (len == 0) {
        *count = 0;
        return tokens;
    }
    for (j = 0; j < (len-(seplen-1)); j++) {
        // 如果数组元素不太够用，直接重新分配两倍
        if (slots < elements+2) {
            sds *newtokens;

            slots *= 2;
            newtokens = s_realloc(tokens,sizeof(sds)*slots);
            if (newtokens == NULL) goto cleanup;
            tokens = newtokens;
        }
        // 分隔符只有一个字符，直接判断
        // 分隔符是多个字符组成，调用memcmp进行判断
        if ((seplen == 1 && *(s+j) == sep[0]) || (memcmp(s+j,sep,seplen) == 0)) {
            tokens[elements] = sdsnewlen(s+start,j-start); // 存放分割好的字符串
            if (tokens[elements] == NULL) goto cleanup;
            elements++;
            start = j+seplen; // 重新确定开始比较位置
            j = j+seplen-1; // 跳过分割符长度
        }
    }
    // 添加最后一个元素
    tokens[elements] = sdsnewlen(s+start,len-start);
    if (tokens[elements] == NULL) goto cleanup;
    elements++;
    *count = elements;
    return tokens;

cleanup:
    {
        int i;
        for (i = 0; i < elements; i++) sdsfree(tokens[i]);
        s_free(tokens);
        *count = 0;
        return NULL;
    }
}
```

##### `sdsfreesplitres`

释放由 `sdssplitlen` 分割产生的数组。

##### `sdscatrepr`

将字符串转义后追加到sds字符串。

##### `is_hex_digit`

判断字符是否为十六进制。

##### `hex_digit_to_int`

将十六进制转化成整型。

##### `sdssplitargs`

从字符串中切割参数？。

##### `sdsmapchars`

替换字符串，从 `from` 到 `to`。

##### `sdsjoin`

将 `C` 字符串数组以分隔符 `sep` 粘贴到一起。

##### `sdsjoinsds`

粘贴sds字符串。

#### 本章小结

终于结束了，SDS类型，但是其中涉及到有关内存分配的函数暂未分析，那个需要独立分析。





