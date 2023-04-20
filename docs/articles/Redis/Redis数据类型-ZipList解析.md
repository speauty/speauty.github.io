# Redis源码之数据类型解析-ZipList {docsify-ignore}

当前分析 `Redis` 版本为6.2，需要注意。

`ZipList`，压缩列表，可以任意包含多个节点。

#### 基础结构

##### `ZipList`

压缩列表。其整体布局，`<zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>`。

* `uint32_t zlbytes`，压缩列表占有的内存字节大小，包括 `zlbytes` 字段本身的四个字节。需要存储该值，以便能够调整整个结构的大小，而不是先遍历它。
* `uint32_t zltail`，压缩列表的最后一个节点的偏移。允许在压缩列表远端弹出节点操作，而不需要整个遍历。
* `uint16_t zllen`，节点的数量。当这个字段的值小于 `UINT16_MAX(2^16-1)` 时，该字段的值就是节点的数量。当这个值等于 `UINT16_MAX` 时，节点的真实数量就需要遍历整个列表计算。
* `uint8_t zlend`，标记压缩列表尾端的特殊值 `0xFF`。

##### 压缩列表节点

在压缩列表中的节点都是以包含两部分信息的元数据开始的，然后才是 `entry-data`。第一个是前一个节点的长度。第二个是当前节点编码，整型或字符串。大致结构 `<prevlen><encoding><entry-data>`，不过有时编码（`encoding`）包含节点数据本身，也就是 `<prevlen><encoding>`这种简单结构。

* `prevlen`，前一个节点的长度（字节）（该值长度可选值1|5）。如果前一个节点长度小于254字节，那么该属性的长度为1字节：前一个节点的长度就保存在这个字节里面。如果前一个节点长度大于等于254字节，那么该属性长度为5个字节，第一个字节会被设置为 `254(0xFE)`，剩余的四个字节保存前一个节点的长度。
* `encoding`，编码，依赖于节点的内容。当节点是字符串时，该属性第一个字节前两位会保存用于存储字符串长度的编码类型，剩余是字符串的实际长度。当节点是整型时，头两位均被设置为1，接下来的两位被用于表示在该头后存储的整型类型。第一个字节经常足够判断节点类型（整型|字符串）。
  * 字符串 `00` 编码一字节长，保存字符串长度6位，字符串最大长度为 `2^6-1` 字节。
  * 字符串 `01` 编码两字节长，保存字符串长度14位（大端），字符串最大长度为 `2^14-1` 字节。
  * 字符串 `10` 编码五字节长，保存字符串长度32位（大端），字符串最大长度为 `2^32-1` 字节。第一个字节的低6位并没有使用，0。
  * 整型 `11000000` ，`int16_t` 类型的整数（两字节）。
  * 整型 `11010000` ，`int32_t` 类型的整数（四个字节）。
  * 整型 `11100000`，`int64_t` 类型的整数（八个字节）。
  * 整型 `11110000`，24位有符号整数。
  * 整型 `11111110`，8位有符号整数。
  * 整型 `1111xxxx`，`xxxx` 介于 `0001` 和 `1101` 之间。从0到12的无符号整数。这个编码值实际是1到13，因为 `0000` 和 `1111` 不能使用，所以需要从这四位值中减1才是正确值。
  * `11111111`，压缩列表特殊尾节点。

##### `ziplistEntry`

压缩节点值标准化模板。

```c
// 在压缩列表中每个节点不是字符串就是整数
typedef struct {
    // 如果是字符串，长度就是slen
    unsigned char *sval;
    unsigned int slen;
    // 如果是整数，sval是NULL，lval保存整数
    long long lval;
} ziplistEntry;
```

##### `zlentry`

获取有关压缩列表节点信息的模板结构。这并不是节点实际编码，只是为了填充起来方便操作。

```c
typedef struct zlentry {
    unsigned int prevrawlensize; // 用于编码前一个节点长度的字节，数？
    unsigned int prevrawlen; // 前一个节点的长度
    unsigned int lensize; // 用于编码该节点类型或长度的字节。比如，字符串有1|2|5字节的头，整型通常只有一个字节。
    unsigned int len; // 实际节点字节数，对于字符串就是字符串长度。对于整型就要取决于其范围      
    unsigned int headersize; // 头大小=prevrawlensize+lensize
    unsigned char encoding; // 节点编码方式
    unsigned char *p; // 节点的起始指针，也就是指向上一个节点长度属性。
} zlentry;
```



#### 宏常量

##### `ZIP_END`

`#define ZIP_END 255`，压缩列表特殊的尾节点。

##### `ZIP_BIG_PREVLEN`

`#define ZIP_BIG_PREVLEN 254`，对于每个节点前仅代表一个字节的 `prevlen` 属性来说，`ZIP_BIG_PREVLEN-1`是其最大的字节数。否则，它就是形如 `FE AA BB CC DD` 的四个字节的无符号整数表示前一个节点的长度。

##### `ZIP_STR_MASK`

`#define ZIP_STR_MASK 0xc0`，字符串掩码（`1100 0000`）。

##### `ZIP_INT_MASK`

`#define ZIP_INT_MASK 0x30`，整型掩码（`0011 0000`）。

##### `ZIP_STR_06B`

`#define ZIP_STR_06B (0 << 6)`，6位存储字符串长度编码的字符串（`0000 0000`）。

##### `ZIP_STR_14B`

`#define ZIP_STR_14B (1 << 6)`，14位存储字符串长度编码的字符串（`0100 0000`）。

##### `ZIP_STR_32B`

`#define ZIP_STR_32B (2 << 6)`，32位存储字符串长度编码的字符串（`1000 0000`）。

##### `ZIP_INT_16B`

`#define ZIP_INT_16B (0xc0 | 0<<4)`，16位有符号整数（`int16_t`）（`1100 0000`）。

##### `ZIP_INT_32B`

`#define ZIP_INT_32B (0xc0 | 1<<4)`，32位有符号整数（`int32_t`）（`1101 0000`）。

##### `ZIP_INT_64B`

`#define ZIP_INT_64B (0xc0 | 2<<4)`，64位有符号整数（`int64_t`）（`1110 0000`）。

##### `ZIP_INT_24B`

`#define ZIP_INT_24B (0xc0 | 3<<4)`，24位有符号整数（`1111 0000`）。

##### `ZIP_INT_8B`

`#define ZIP_INT_8B 0xfe`，8位有符号整数（`1111 1110`）。

##### `ZIP_INT_IMM_MASK`

`#define ZIP_INT_IMM_MASK 0x0f`，4位无符号整数掩码。

##### `ZIP_INT_IMM_MIN`

`#define ZIP_INT_IMM_MIN 0xf1`，4位无符号整数最小值 0。

##### `ZIP_INT_IMM_MAX`

`#define ZIP_INT_IMM_MAX 0xfd`，4位无符号整数最大值 12。

##### `INT24_MAX`

`#define INT24_MAX 0x7fffff`，24位有符号整数最大值。

##### `INT24_MIN`

`#define INT24_MIN (-INT24_MAX - 1)`，24位有符号整数最小值。

##### `ZIP_ENCODING_SIZE_INVALID`

`#define ZIP_ENCODING_SIZE_INVALID 0xff`，编码大小无效值。

#### 宏函数

##### `ZIP_IS_STR`

判断指定编码 `enc` 是否表示字符串。字符串节点不会以 `11` 作为首字节的最高有效位。

```c
#define ZIP_IS_STR(enc) (((enc) & ZIP_STR_MASK) < ZIP_STR_MASK)
```

##### `ZIPLIST_BYTES`

返回压缩列表包含的总字节数指针。

```c
#define ZIPLIST_BYTES(zl)       (*((uint32_t*)(zl)))
// ziplist 结构
// zlbytes zltail zllen entry...entry zlend
```

##### `ZIPLIST_TAIL_OFFSET`

返回压缩列表最后一个节点的偏移指针

```c
#define ZIPLIST_TAIL_OFFSET(zl) (*((uint32_t*)((zl)+sizeof(uint32_t))))
// zl 偏移32位 zlbytes->zltail 
```

##### `ZIPLIST_LENGTH`

返回压缩列表节点数量指针，如果等于 `UINT16_MAX`，就需要遍历整个列表计算节点数量。

```c
#define ZIPLIST_LENGTH(zl)      (*((uint16_t*)((zl)+sizeof(uint32_t)*2)))
```

##### `ZIPLIST_HEADER_SIZE`

压缩列表头大小：两个32位整型保存总字节数和最后一个节点偏移，16位整型为了节点数量。

```c
#define ZIPLIST_HEADER_SIZE     (sizeof(uint32_t)*2+sizeof(uint16_t))
```

##### `ZIPLIST_END_SIZE`

压缩列表结束节点大小。只有一个字节。

```c
#define ZIPLIST_END_SIZE        (sizeof(uint8_t))
```

##### `ZIPLIST_ENTRY_HEAD`

返回压缩列表第一个节点指针。也就是 `ziplist+headerSize`。

```c
#define ZIPLIST_ENTRY_HEAD(zl)  ((zl)+ZIPLIST_HEADER_SIZE)
```

##### `ZIPLIST_ENTRY_TAIL`

返回压缩列表中最后一个节点的指针。

```c
#define ZIPLIST_ENTRY_TAIL(zl)  ((zl)+intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl)))
```

##### `ZIPLIST_ENTRY_END`

返回压缩列表的最后一字节指针，也就是结束节点 `FF`。

```c
#define ZIPLIST_ENTRY_END(zl)   ((zl)+intrev32ifbe(ZIPLIST_BYTES(zl))-1)
```

##### `ZIPLIST_INCR_LENGTH`

增加压缩列表头中的长度 `zllen`。

```c
#define ZIPLIST_INCR_LENGTH(zl,incr) { \
    if (ZIPLIST_LENGTH(zl) < UINT16_MAX) \
        ZIPLIST_LENGTH(zl) = intrev16ifbe(intrev16ifbe(ZIPLIST_LENGTH(zl))+incr); \
}
```

##### `ZIPLIST_ENTRY_ZERO`

初始化压缩列表节点模板结构。

```c
#define ZIPLIST_ENTRY_ZERO(zle) { \
    (zle)->prevrawlensize = (zle)->prevrawlen = 0; \
    (zle)->lensize = (zle)->len = (zle)->headersize = 0; \
    (zle)->encoding = 0; \
    (zle)->p = NULL; \
}
```

##### `ZIP_ENTRY_ENCODING`

从 `ptr` 指针字节中获取编码方式，并设置到 `zlentry` 结构中的 `encoding` 属性。

```c
#define ZIP_ENTRY_ENCODING(ptr, encoding) do {  \
    (encoding) = ((ptr)[0]); \
    if ((encoding) < ZIP_STR_MASK) (encoding) &= ZIP_STR_MASK; \
} while(0)
```

##### `ZIP_ASSERT_ENCODING`

检测编码是否无效。

```c
#define ZIP_ASSERT_ENCODING(encoding) do {                                     \
    assert(zipEncodingLenSize(encoding) != ZIP_ENCODING_SIZE_INVALID);         \
} while (0)
```

##### `ZIP_DECODE_LENGTH`

解码在 `ptr` 中编码的节点类型和数据长度（字符串长度，整型字节数）。`lensize` 对节点编码的字节数。`len` 节点长度。和 `zipStoreEntryEncoding` 部分类似。

```c
#define ZIP_DECODE_LENGTH(ptr, encoding, lensize, len) do {                    \
    if ((encoding) < ZIP_STR_MASK) {                                           \
        // 字符串编码长度 1|2|5
        if ((encoding) == ZIP_STR_06B) {                                       \
            (lensize) = 1;                                                     \
            (len) = (ptr)[0] & 0x3f;                                           \
        } else if ((encoding) == ZIP_STR_14B) {                                \
            (lensize) = 2;                                                     \
            (len) = (((ptr)[0] & 0x3f) << 8) | (ptr)[1];                       \
        } else if ((encoding) == ZIP_STR_32B) {                                \
            (lensize) = 5;                                                     \
            (len) = ((ptr)[1] << 24) |                                         \
                    ((ptr)[2] << 16) |                                         \
                    ((ptr)[3] <<  8) |                                         \
                    ((ptr)[4]);                                                \
        } else {                                                               \
            // 异常编码
            (lensize) = 0;  \
            (len) = 0;    \
        }                                                                      \
    } else {                                                                   \
        // 整型节点 编码的长度都是1 数据字节数，需要根据encoding判定
        (lensize) = 1;                                                         \
        if ((encoding) == ZIP_INT_8B)  (len) = 1;                              \
        else if ((encoding) == ZIP_INT_16B) (len) = 2;                         \
        else if ((encoding) == ZIP_INT_24B) (len) = 3;                         \
        else if ((encoding) == ZIP_INT_32B) (len) = 4;                         \
        else if ((encoding) == ZIP_INT_64B) (len) = 8;                         \
        else if (encoding >= ZIP_INT_IMM_MIN && encoding <= ZIP_INT_IMM_MAX)   \
            (len) = 0; /* 4 bit immediate */                                   \
        else                                                                   \
            (lensize) = (len) = 0; // 异常编码                        \
    }                                                                          \
} while(0)
```

##### `ZIP_DECODE_PREVLENSIZE`

返回编码前一个节点长度使用的字节数。通过设置 `prelensize`。

```c
#define ZIP_DECODE_PREVLENSIZE(ptr, prevlensize) do {                          \
    if ((ptr)[0] < ZIP_BIG_PREVLEN) {                                          \
        (prevlensize) = 1;                                                     \
    } else {                                                                   \
        (prevlensize) = 5;                                                     \
    }                                                                          \
} while(0)
```

##### `ZIP_DECODE_PREVLEN`

返回前一个节点的长度 `prevlen`，和编码这种长度的字节数 `prevlensize`。

```c
#define ZIP_DECODE_PREVLEN(ptr, prevlensize, prevlen) do {                     \
    ZIP_DECODE_PREVLENSIZE(ptr, prevlensize);                                  \
    if ((prevlensize) == 1) {                                                  \
        (prevlen) = (ptr)[0];                                                  \
    } else { /* prevlensize == 5 */                                            \
        (prevlen) = ((ptr)[4] << 24) |                                         \
                    ((ptr)[3] << 16) |                                         \
                    ((ptr)[2] <<  8) |                                         \
                    ((ptr)[1]);                                                \
    }                                                                          \
} while(0)
```



#### 内部函数

##### `zipEncodingLenSize`

返回节点类型和长度编码的字节数，错误就返回 `ZIP_ENCODING_SIZE_INVALID`。

```c
static inline unsigned int zipEncodingLenSize(unsigned char encoding) {
    if (encoding == ZIP_INT_16B || encoding == ZIP_INT_32B ||
        encoding == ZIP_INT_24B || encoding == ZIP_INT_64B ||
        encoding == ZIP_INT_8B)
        return 1; // 整型就只有一个字节
    if (encoding >= ZIP_INT_IMM_MIN && encoding <= ZIP_INT_IMM_MAX) //特殊整型 4位
        return 1;
    if (encoding == ZIP_STR_06B)
        return 1;
    if (encoding == ZIP_STR_14B)
        return 2;
    if (encoding == ZIP_STR_32B)
        return 5;
    return ZIP_ENCODING_SIZE_INVALID;
}
```

##### `zipIntSize`

返回存储由 `encoding` 编码的整数所需的字节。

```c
static inline unsigned int zipIntSize(unsigned char encoding) {
    switch(encoding) {
    case ZIP_INT_8B:  return 1;
    case ZIP_INT_16B: return 2;
    case ZIP_INT_24B: return 3;
    case ZIP_INT_32B: return 4;
    case ZIP_INT_64B: return 8;
    }
    if (encoding >= ZIP_INT_IMM_MIN && encoding <= ZIP_INT_IMM_MAX)
        return 0; /* 4位直接在encoding中 */
    redis_unreachable(); // abort
    return 0;
}
```

##### `zipStoreEntryEncoding`

将节点的编码头写入 `p` 中，如果 `p` 是空，直接返回编码这种长度所需的字节数。

```c
unsigned int zipStoreEntryEncoding(unsigned char *p, unsigned char encoding, unsigned int rawlen) {
    unsigned char len = 1, buf[5]; // 最长为5，所以是buf[5]

    if (ZIP_IS_STR(encoding)) { // 字符串
    	// 由于给出的encoding可能不是为了字符串，所以还是需要rawlen进行判断
        if (rawlen <= 0x3f) { // 1  63 2^6-1 00
            if (!p) return len;
            buf[0] = ZIP_STR_06B | rawlen; // 类型和长度
            // 00 00 0000
            // 2bits 符号  6bits len
        } else if (rawlen <= 0x3fff) { // 2 10383 2^14-1 01
            len += 1;
            if (!p) return len;
            // ZIP_STR_14B 0100 0000 
            // 0x3f 0011 1111
            buf[0] = ZIP_STR_14B | ((rawlen >> 8) & 0x3f); // 两位+六位
            buf[1] = rawlen & 0xff; // 后八位
            // 0100 0000 0000 0000
            // 2bits 符号  14bits len
        } else { // 5 10
            len += 4;
            if (!p) return len;
            buf[0] = ZIP_STR_32B; // 符号 独立一字节
            // 剩余四字节保存长度
            buf[1] = (rawlen >> 24) & 0xff;
            buf[2] = (rawlen >> 16) & 0xff;
            buf[3] = (rawlen >> 8) & 0xff;
            buf[4] = rawlen & 0xff;
        }
    } else { // 整型
        if (!p) return len;
        buf[0] = encoding;
    }
    memcpy(p,buf,len); // 存储
    return len;
}
```

##### `zipStorePrevEntryLengthLarge`

对前一个节点长度进行编码并写入 `p`。仅用于更大的编码（`__ziplistCascadeUpdate`）。

```c
int zipStorePrevEntryLengthLarge(unsigned char *p, unsigned int len) {
    uint32_t u32;
    if (p != NULL) { // 分段？
        p[0] = ZIP_BIG_PREVLEN; // 254
        u32 = len;
        memcpy(p+1,&u32,sizeof(u32));
        memrev32ifbe(p+1);
    }
    return 1 + sizeof(uint32_t);
}
```

##### `zipStorePrevEntryLength`

对前一个节点长度进行编码并写入 `p`。如果 `p` 为空，返回需要编码这种长度的字节数。

```c
unsigned int zipStorePrevEntryLength(unsigned char *p, unsigned int len) {
    if (p == NULL) {
        return (len < ZIP_BIG_PREVLEN) ? 1 : sizeof(uint32_t) + 1;
    } else {
        if (len < ZIP_BIG_PREVLEN) { // 大头区分
            p[0] = len;
            return 1;
        } else {
            return zipStorePrevEntryLengthLarge(p,len);
        }
    }
}
```

##### `zipPrevLenByteDiff`

返回编码前一个节点长度（`prevlen`）的长度差，当前一个节点大小发生变化时。

```c
int zipPrevLenByteDiff(unsigned char *p, unsigned int len) {
    unsigned int prevlensize; // prevlen 的长度
    ZIP_DECODE_PREVLENSIZE(p, prevlensize); // 获取旧的prevlensize
    return zipStorePrevEntryLength(NULL, len) - prevlensize;
}
```

##### `zipTryEncoding`

检测字符串节点是否能转成整型。

```c
int zipTryEncoding(unsigned char *entry, unsigned int entrylen, long long *v, unsigned char *encoding) { // **v 整型值 *encoding 对应的编码
    long long value;

    if (entrylen >= 32 || entrylen == 0) return 0;
    if (string2ll((char*)entry,entrylen,&value)) { // string2ll 字符串转换成长整型
        // 判断整型范围 确定其编码类型
        if (value >= 0 && value <= 12) {
            *encoding = ZIP_INT_IMM_MIN+value; // 编码 1111 xxxx
        } else if (value >= INT8_MIN && value <= INT8_MAX) {
            *encoding = ZIP_INT_8B;
        } else if (value >= INT16_MIN && value <= INT16_MAX) {
            *encoding = ZIP_INT_16B;
        } else if (value >= INT24_MIN && value <= INT24_MAX) {
            *encoding = ZIP_INT_24B;
        } else if (value >= INT32_MIN && value <= INT32_MAX) {
            *encoding = ZIP_INT_32B;
        } else {
            *encoding = ZIP_INT_64B;
        }
        *v = value; // 赋值 整型
        return 1;
    }
    return 0;
}
```

##### `zipSaveInteger`

保存整型 `value` 到 `p`。

```c
void zipSaveInteger(unsigned char *p, int64_t value, unsigned char encoding) {
    int16_t i16;
    int32_t i32;
    int64_t i64;
    if (encoding == ZIP_INT_8B) {
        ((int8_t*)p)[0] = (int8_t)value;
    } else if (encoding == ZIP_INT_16B) {
        i16 = value;
        memcpy(p,&i16,sizeof(i16));
        memrev16ifbe(p);
    } else if (encoding == ZIP_INT_24B) { // 24bits
        i32 = value<<8;
        memrev32ifbe(&i32);
        memcpy(p,((uint8_t*)&i32)+1,sizeof(i32)-sizeof(uint8_t));
    } else if (encoding == ZIP_INT_32B) {
        i32 = value;
        memcpy(p,&i32,sizeof(i32));
        memrev32ifbe(p);
    } else if (encoding == ZIP_INT_64B) {
        i64 = value;
        memcpy(p,&i64,sizeof(i64));
        memrev64ifbe(p);
    } else if (encoding >= ZIP_INT_IMM_MIN && encoding <= ZIP_INT_IMM_MAX) {
        // 值就在 encoding 中
    } else {
        assert(NULL);
    }
}
```

##### `zipLoadInteger`

从 `p` 中读取整型值。`zipSaveInteger` 的逆向操作。

```c
int64_t zipLoadInteger(unsigned char *p, unsigned char encoding) {
    int16_t i16;
    int32_t i32;
    int64_t i64, ret = 0;
    if (encoding == ZIP_INT_8B) {
        ret = ((int8_t*)p)[0];
    } else if (encoding == ZIP_INT_16B) {
        memcpy(&i16,p,sizeof(i16));
        memrev16ifbe(&i16);
        ret = i16;
    } else if (encoding == ZIP_INT_32B) {
        memcpy(&i32,p,sizeof(i32));
        memrev32ifbe(&i32);
        ret = i32;
    } else if (encoding == ZIP_INT_24B) {
        i32 = 0;
        memcpy(((uint8_t*)&i32)+1,p,sizeof(i32)-sizeof(uint8_t));
        memrev32ifbe(&i32);
        ret = i32>>8;
    } else if (encoding == ZIP_INT_64B) {
        memcpy(&i64,p,sizeof(i64));
        memrev64ifbe(&i64);
        ret = i64;
    } else if (encoding >= ZIP_INT_IMM_MIN && encoding <= ZIP_INT_IMM_MAX) {
        ret = (encoding & ZIP_INT_IMM_MASK)-1; // 需要-1
    } else {
        assert(NULL);
    }
    return ret;
}
```

##### `zipEntry`

使用节点 `p` 的信息填充结构体  `e`。

```c
static inline void zipEntry(unsigned char *p, zlentry *e) {
    ZIP_DECODE_PREVLEN(p, e->prevrawlensize, e->prevrawlen);
    ZIP_ENTRY_ENCODING(p + e->prevrawlensize, e->encoding);
    ZIP_DECODE_LENGTH(p + e->prevrawlensize, e->encoding, e->lensize, e->len);
    assert(e->lensize != 0); // 验证 encoding 有效
    e->headersize = e->prevrawlensize + e->lensize;
    e->p = p;
}
```

##### `zipEntrySafe`

`zipEntry` 的安全版本。主要针对非信任指针。它保证了不访问在 `ziplist` 范围外的内存。

```c
static inline int zipEntrySafe(unsigned char* zl, size_t zlbytes, unsigned char *p, zlentry *e, int validate_prevlen) {
    unsigned char *zlfirst = zl + ZIPLIST_HEADER_SIZE; // 头节点指针
    unsigned char *zllast = zl + zlbytes - ZIPLIST_END_SIZE; // 尾节点指针
    // 判断指针是否移除宏定义函数
#define OUT_OF_RANGE(p) (unlikely((p) < zlfirst || (p) > zllast))

    // 如果没有头溢出压缩列表的可能，就走捷径（最大的 lensize 和 prevrawlensize 都是5字节）
    if (p >= zlfirst && p + 10 < zllast) {
        ZIP_DECODE_PREVLEN(p, e->prevrawlensize, e->prevrawlen);
        ZIP_ENTRY_ENCODING(p + e->prevrawlensize, e->encoding);
        ZIP_DECODE_LENGTH(p + e->prevrawlensize, e->encoding, e->lensize, e->len);
        e->headersize = e->prevrawlensize + e->lensize;
        e->p = p;
        if (unlikely(e->lensize == 0)) // 检测 e->lensize 是否为0
            return 0;
        if (OUT_OF_RANGE(p + e->headersize + e->len)) // 判断是否溢出 ziplist 范围
            return 0;
        if (validate_prevlen && OUT_OF_RANGE(p - e->prevrawlen)) // 判断prevlen是否溢出
            return 0;
        return 1;
    }

    if (OUT_OF_RANGE(p)) // 检测指针是否溢出
        return 0;

    ZIP_DECODE_PREVLENSIZE(p, e->prevrawlensize); // 设置 prevrawlensize
    if (OUT_OF_RANGE(p + e->prevrawlensize))
        return 0;

    // 检测编码是否有效
    ZIP_ENTRY_ENCODING(p + e->prevrawlensize, e->encoding);
    e->lensize = zipEncodingLenSize(e->encoding);
    if (unlikely(e->lensize == ZIP_ENCODING_SIZE_INVALID))
        return 0;

    // 检测节点头编码是否溢出
    if (OUT_OF_RANGE(p + e->prevrawlensize + e->lensize))
        return 0;

    // 解码prevlen和节点头长度
    ZIP_DECODE_PREVLEN(p, e->prevrawlensize, e->prevrawlen);
    ZIP_DECODE_LENGTH(p + e->prevrawlensize, e->encoding, e->lensize, e->len);
    e->headersize = e->prevrawlensize + e->lensize;

    // 检测节点是否溢出
    if (OUT_OF_RANGE(p + e->headersize + e->len))
        return 0;

    // 检测 prevlen 是否溢出
    if (validate_prevlen && OUT_OF_RANGE(p - e->prevrawlen))
        return 0;

    e->p = p;
    return 1;
#undef OUT_OF_RANGE // 取消宏定义 OUT_OF_RANGE 
}
```

##### `zipRawEntryLengthSafe`

计算 `p` 节点占用字节总数安全版。

```c
static inline unsigned int zipRawEntryLengthSafe(unsigned char* zl, size_t zlbytes, unsigned char *p) {
    zlentry e;
    assert(zipEntrySafe(zl, zlbytes, p, &e, 0));
    return e.headersize + e.len;
}
```

##### `zipRawEntryLength`

计算 `p` 节点占用字节总数。`zipRawEntryLengthSafe` 的非安全版。

```c
static inline unsigned int zipRawEntryLength(unsigned char *p) {
    zlentry e;
    zipEntry(p, &e);
    return e.headersize + e.len;
}
```

##### `zipAssertValidEntry`

验证节点是否溢出压缩列表范围。

```c
static inline void zipAssertValidEntry(unsigned char* zl, size_t zlbytes, unsigned char *p) {
    zlentry e;
    assert(zipEntrySafe(zl, zlbytes, p, &e, 1));
}
```

##### `ziplistResize`

调整压缩列表大小。

```c
unsigned char *ziplistResize(unsigned char *zl, unsigned int len) {
    zl = zrealloc(zl,len); // 重新分配空间
    ZIPLIST_BYTES(zl) = intrev32ifbe(len); // 设置总长度
    zl[len-1] = ZIP_END; // 重新设置结束节点
    return zl;
}
```

##### `__ziplistCascadeUpdate`

级联更新。当节点被插入时，我们需要去设置下一个节点的 `prevlen` 字段等于当前插入节点的长度。这会导致长度不能编码在1个字节内，下一个节点需要扩展去保存5字节编码的 `prevlen`。 这可以自由完成，因为这只发生在一个节点已经被插入的时候（可能导致 `realloc` 和 `memmove`）。然而，编码这个 `prevlen` 可能也需要对应节点扩展。这种影响可能会贯穿整个压缩列表，当有一连串长度接近 `ZIP_BIG_PREVLEN` 的节点，所以需要去检测 `prevlen` 能否被编码在每个连续节点内。`prevlen` 字段需要收缩的反转也会造成这种影响。

```c
unsigned char *__ziplistCascadeUpdate(unsigned char *zl, unsigned char *p) { // 比较吃力
    // *p 指向不需要更新的第一个节点
    zlentry cur; // 模板节点
    size_t prevlen, prevlensize, prevoffset; // 上一个更新节点信息
    size_t firstentrylen; // 用于处理头插入
    size_t rawlen, curlen = intrev32ifbe(ZIPLIST_BYTES(zl));
    size_t extra = 0, cnt = 0, offset;
    size_t delta = 4; // 更新一个节点 prevlen 属性需要的额外字节数（5-1）
    unsigned char *tail = zl + intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl)); // 尾节点指针

    // 空压缩列表
    if (p[0] == ZIP_END) return zl;
	
    zipEntry(p, &cur); // 不需要安全版，因为该输入指针在被返回它的函数已验证
    firstentrylen = prevlen = cur.headersize + cur.len; // 当前节点总长度=头长度+内容长度
    prevlensize = zipStorePrevEntryLength(NULL, prevlen); // 计算prevlen字段长度
    prevoffset = p - zl; // 距离zl偏移
    p += prevlen; // 往后偏移。这里的prevlen就是当前节点的总长度，所以，相加 切到下一个节点

    // 迭代压缩列表为了找出为了更新它需要的额外字节数
    while (p[0] != ZIP_END) { // 一直到结束节点
        assert(zipEntrySafe(zl, curlen, p, &cur, 0)); // 有效节点

        // 当 prevlen 没有更新时中止
        if (cur.prevrawlen == prevlen) break;

        // 当节点的 prevlensize 足够大时中止
        if (cur.prevrawlensize >= prevlensize) {
            if (cur.prevrawlensize == prevlensize) { // prevlen字段长度相等
                zipStorePrevEntryLength(p, prevlen); // 设置对应长度
            } else {
                // 这会导致收缩，我们需要避免，所以，将 prevlen 设置在可用的字节
                zipStorePrevEntryLengthLarge(p, prevlen);
            }
            break;
        }

        // cur.prevrawlen 之前的头节点（可能）。如果是头节点，自然也就是为0
        // 或者 前一个节点原始长度+额外长度==前一个节点总长度
        assert(cur.prevrawlen == 0 || cur.prevrawlen + delta == prevlen);

        // 更新前一个节点信息并增加指针
        rawlen = cur.headersize + cur.len; // 当前节点原始长度
        prevlen = rawlen + delta;  // 原始长度+额外长度=当前节点总长度
        prevlensize = zipStorePrevEntryLength(NULL, prevlen); // 重新计算存储长度字段大小
        prevoffset = p - zl; // 重新计算需要被更新节点距离 zl 偏移
        p += rawlen; // 往后偏移
        extra += delta; // 叠加补充长度
        cnt++; // 补充次数？
    }

    // 额外字节为0 所有更新已完成，或不需要更新
    if (extra == 0) return zl;

    // 更新尾节点偏移在循环之后
    if (tail == zl + prevoffset) { // 
        // 当最后一个需要更新的节点恰好是尾节点时，更新尾节点偏移除非这正是更新的唯一一个节点（这种情况下，尾节点偏移不会发生变化）
        if (extra - delta != 0) { // 补充字节大于额外字节，表示该尾节点不是唯一更新的节点，前面还有需要更新的节点
            // 减去尾节点补充的额外字节，就是前面节点补充字节总量
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+extra-delta);
        }
    } else {
        // 如果不是尾节点，就更好操作了，直接原尾节点偏移+补充字节
        ZIPLIST_TAIL_OFFSET(zl) =
            intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+extra);
    }

    // 现在 p 指针就在原始压缩列表第一个不需要改变字节处
    // 将之后的数据，移到新压缩列表
    offset = p - zl; // 计算偏移 前面节点就是需要变动的节点？
    zl = ziplistResize(zl, curlen + extra); // 调整zl大小=原总长度+补充长度
    p = zl + offset; // 直接移到不需要更新字节处
    // 总长度-偏移-1 需要拷贝的长度（不变动节点总长度）
    // p+extra 目标位置 不需要更新节点的新位置
    // p 拷贝位置
    memmove(p + extra, p, curlen - offset - 1);
    p += extra; // 移到不变动节点处 然后向前偏移 prevlen

    // 从尾到头迭代所有需要被更新的节点
    while (cnt) { // 补充次数
        zipEntry(zl + prevoffset, &cur); // 获取需要第一个被更新节点，反向，也就是之前循环最后一个需要被更新节点
        rawlen = cur.headersize + cur.len; // 当前节点原始长度
        // 将节点移向尾部 重置 prevlen
        // 撇开 当前节点保存上一个节点原始长度的长度(有变，向后对齐) 实际数据长度
        memmove(p - (rawlen - cur.prevrawlensize), 
                zl + prevoffset + cur.prevrawlensize, 
                rawlen - cur.prevrawlensize);
        p -= (rawlen + delta); // 将p移到原始长度+额外长度之前
        if (cur.prevrawlen == 0) { // 头节点，将其 prevlen 更新为第一个节点长度
            zipStorePrevEntryLength(p, firstentrylen);
        } else { // 否则就增加额外长度 4字节
            zipStorePrevEntryLength(p, cur.prevrawlen+delta);
        }
        // 前移到前一个节点。没问题。
        prevoffset -= cur.prevrawlen;
        cnt--;
    }
    return zl;
}
```

##### `__ziplistDelete`

从压缩列表中，删除从 `p` 开始的 `num` 个节点。返回压缩列表指针。

```c
unsigned char *__ziplistDelete(unsigned char *zl, unsigned char *p, unsigned int num) {
    // i 循环变量
    // totlen 需要删除字节总数
    // deleted 需要删除的节点数量 deleted <= num
    unsigned int i, totlen, deleted = 0;
    size_t offset;
    int nextdiff = 0;
    zlentry first, tail; // 删除头节点
    size_t zlbytes = intrev32ifbe(ZIPLIST_BYTES(zl)); 

    zipEntry(p, &first); // 解析first
    for (i = 0; p[0] != ZIP_END && i < num; i++) { // 可能遇到ZIP_END提前中止 这也是deleted < num的原因
        p += zipRawEntryLengthSafe(zl, zlbytes, p); // p节点占用字节总数 += 直接向后偏移
        deleted++;
    }
	// p已经偏移到最后一个需要删除的节点位置
    assert(p >= first.p); // 自然偏移结果
    totlen = p-first.p; // 删除节点移除字节总数
    if (totlen > 0) { // 显然
        uint32_t set_tail;
        if (p[0] != ZIP_END) { // 未到结束节点
            // 与当前prevrawlen比较，存储当前节点的prevrawlen可能会增加或减少所需要的字节数
            // 这儿有个存储它的空间，因为它在正在删除节点时已经预先存储
            // first.prevrawlen-p.prevrawlen 0|4|-4
            nextdiff = zipPrevLenByteDiff(p,first.prevrawlen);

            // 当p回跳时，总会有空间的
            // 如果新前一个节点很大，删除节点中就会有个5字节的prevlen头的节点，所以这儿肯定至少释放5个字节，我们只需要4字节
            p -= nextdiff; // 调整prevlensize 
            assert(p >= first.p && p<zl+zlbytes-1);
            zipStorePrevEntryLength(p,first.prevrawlen); // 存储 prevlen

            // 尾节点偏移前移totlen字节
            set_tail = intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))-totlen;

            // 当尾部包含多个节点时，还是需要考虑nextdiff。否则，prevlen大小变化对尾偏移没有影响
            assert(zipEntrySafe(zl, zlbytes, p, &tail, 1)); // 解析tail
            if (p[tail.headersize+tail.len] != ZIP_END) { // 补充 nextdiff
                set_tail = set_tail + nextdiff;
            }

            // 将尾部p移到压缩列表前面
            // 由于断言p>=first.p，我们知道totlen>=0，所以p>first.p并且也是保存了不能溢出，即使节点长度遭到破坏
            size_t bytes_to_move = zlbytes-(p-zl)-1; // -1 不包含结束节点？
            memmove(first.p,p,bytes_to_move);
        } else { // 已到结束节点。尾节点已被删除，不再需要释放空间
            set_tail = (first.p-zl)-first.prevrawlen; // 计算结束节点位置
        }

        // 调整压缩列表大小
        offset = first.p-zl;
        zlbytes -= totlen - nextdiff; // 减去总共移除的字节数和对prevlen大小的调整
        zl = ziplistResize(zl, zlbytes);
        p = zl+offset;

        ZIPLIST_INCR_LENGTH(zl,-deleted); // 更新节点数量

        // 设置上面计算好的尾偏移
        assert(set_tail <= zlbytes - ZIPLIST_END_SIZE);
        ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(set_tail);

        // 当nextdiff!=0，下一个节点的原始长度已经发生了变化，所以需要级联更新贯穿压缩列表
        if (nextdiff != 0)
            zl = __ziplistCascadeUpdate(zl,p);
    }
    return zl;
}
```

##### `__ziplistInsert`

在压缩列表 `zl` 指定节点 `p` 处插入新节点 `s` 。

```c
unsigned char *__ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen) {
    // curlen 当前压缩列表字节总数
    size_t curlen = intrev32ifbe(ZIPLIST_BYTES(zl)), reqlen, newlen;
    // prevlensize prevlen的大小 prevlen 前一个节点的大小
    unsigned int prevlensize, prevlen = 0;
    size_t offset;
    int nextdiff = 0;
    unsigned char encoding = 0;
    
    long long value = 123456789; // 初始化避免警告
    zlentry tail;

    // 找到已插入节点的prevlen和prevlensize
    if (p[0] != ZIP_END) { // 不在结束节点
        ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
    } else { // 在结束节点
        unsigned char *ptail = ZIPLIST_ENTRY_TAIL(zl); // 尾节点
        if (ptail[0] != ZIP_END) {
            prevlen = zipRawEntryLengthSafe(zl, curlen, ptail);
        }
    }

    // 检测需要插入的节点能否被编码
    if (zipTryEncoding(s,slen,&value,&encoding)) { // 尝试编码
        // 根据encoding设置合适的整型编码
        reqlen = zipIntSize(encoding);
    } else {
        // encoding无法使用。然而zipStoreEntryEncoding可以使用字符串长度表明编码方式
        reqlen = slen;
    }
    // 需要存储前一个节点长度和本身数据的空间
    reqlen += zipStorePrevEntryLength(NULL,prevlen); // 补充前一个节点长度prevlen
    reqlen += zipStoreEntryEncoding(NULL,encoding,slen); // 补充本身所需空间

    // 如果出入的位置不是尾部，就需要保证，下一个节点的prevlen可以存储该新节点的长度
    int forcelarge = 0; // 强制扩展
    // 结束节点就不管。
    // reqlen.lensize-p.prevlensize 1-5
    nextdiff = (p[0] != ZIP_END) ? zipPrevLenByteDiff(p,reqlen) : 0;
    if (nextdiff == -4 && reqlen < 4) {
        nextdiff = 0;
        forcelarge = 1;
    }

    // 存储p偏移，因为realloc可能会改变zl地址
    offset = p-zl;
    newlen = curlen+reqlen+nextdiff; // 原长度+新节点长度+prevlen修正
    zl = ziplistResize(zl,newlen);
    p = zl+offset; // 重新获取p位置

    // 如果可以就应用内存移动并更新尾偏移
    if (p[0] != ZIP_END) {
        // -1 排除ZIP_END字节
        // reqlen newEntry len
        // 将p-nextdiff移到p+reqlen位置，长度为curlen-offset-1+nextdiff
        // p后就空出reqlen空间
        memmove(p+reqlen,p-nextdiff,curlen-offset-1+nextdiff);

        // 对下一个节点的上一个节点（当前节点）原始长度编码
        if (forcelarge) // p+reqlen
            zipStorePrevEntryLengthLarge(p+reqlen,reqlen);
        else
            zipStorePrevEntryLength(p+reqlen,reqlen);

        // 更新尾偏移
        ZIPLIST_TAIL_OFFSET(zl) =
            intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+reqlen);

        // 当尾部包含多个节点时，需要补充nextdiff。否则，prevlen大小的改变对尾偏移没有影响
        assert(zipEntrySafe(zl, newlen, p+reqlen, &tail, 1));
        if (p[reqlen+tail.headersize+tail.len] != ZIP_END) {
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+nextdiff);
        }
    } else { // 该节点将成为新的尾
        ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(p-zl);
    }

    // 当nextdiff非0时，下一个节点的原始长度会发生变化。所以需要级联更新贯穿整个压缩列表
    if (nextdiff != 0) {
        offset = p-zl;
        zl = __ziplistCascadeUpdate(zl,p+reqlen);
        p = zl+offset;
    }

    // 写入节点
    p += zipStorePrevEntryLength(p,prevlen); // 写入prevlen
    p += zipStoreEntryEncoding(p,encoding,slen); // 写入encoding
    // 写入entry-data
    if (ZIP_IS_STR(encoding)) { // 字符串拷贝
        memcpy(p,s,slen);
    } else {
        zipSaveInteger(p,value,encoding);
    }
    ZIPLIST_INCR_LENGTH(zl,1); // 更新节点数量
    return zl;
}
```

##### `uintCompare`

整型比较，快排。

```c
int uintCompare(const void *a, const void *b) {
    return (*(unsigned int *) a - *(unsigned int *) b);
}
```

##### `ziplistSaveValue`

将一个从 `val` 或 `lval` 读取的字符串快速存到目标结构体。

```c
/* Helper method to store a string into from val or lval into dest */
static inline void ziplistSaveValue(unsigned char *val, unsigned int len, long long lval, ziplistEntry *dest) {
    dest->sval = val;
    dest->slen = len;
    dest->lval = lval;
}
```

#### 接口函数

##### `ziplistNew`

创建一个新的压缩列表。返回 `zl` 对应指针。

```c
unsigned char *ziplistNew(void) {
    unsigned int bytes = ZIPLIST_HEADER_SIZE+ZIPLIST_END_SIZE; // 计算 头+结束 长度 空列表
    unsigned char *zl = zmalloc(bytes); // 分配空间
    ZIPLIST_BYTES(zl) = intrev32ifbe(bytes); // 设置压缩列表总长度
    ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(ZIPLIST_HEADER_SIZE); // 设置到尾节点的偏移量
    ZIPLIST_LENGTH(zl) = 0; // 设置节点数量指针
    zl[bytes-1] = ZIP_END; // 设置结束节点
    return zl; // 返回 zl 指针
}
```

##### `ziplistMerge`

合并两个压缩列表，`first` 和 `second`，将 `second` 追加到 `first`。将大的压缩列表重新分配空间去包含新的合并列表。

```c
unsigned char *ziplistMerge(unsigned char **first, unsigned char **second) {
    // 只要有一个列表为空，就中止，返回NULL
    if (first == NULL || *first == NULL || second == NULL || *second == NULL) return NULL;

    // 自然，相等也无法合并
    if (*first == *second) return NULL;

    size_t first_bytes = intrev32ifbe(ZIPLIST_BYTES(*first)); // zl1字节总量
    size_t first_len = intrev16ifbe(ZIPLIST_LENGTH(*first)); // zl1节点总量

    size_t second_bytes = intrev32ifbe(ZIPLIST_BYTES(*second)); // zl2字节总量
    size_t second_len = intrev16ifbe(ZIPLIST_LENGTH(*second)); // zl2节点总量

    int append;
    unsigned char *source, *target; // 源 & 目标
    size_t target_bytes, source_bytes;
    // 选择较大的列表作为目标列表，方便调整大小
    // 也必须知道，是追加还是前置到目标列表
    if (first_len >= second_len) { // 保留zl1，将zl2追加大zl1
        target = *first; // 目标
        target_bytes = first_bytes;
        source = *second; // 源
        source_bytes = second_bytes;
        append = 1;
    } else { // 保留zl2，将zl1前置到zl2
        target = *second;
        target_bytes = second_bytes;
        source = *first;
        source_bytes = first_bytes;
        append = 0;
    }

    // 计算最终所需字节总量（减去一对元数据 HEADER+END）
    size_t zlbytes = first_bytes + second_bytes -
                     ZIPLIST_HEADER_SIZE - ZIPLIST_END_SIZE;
    size_t zllength = first_len + second_len; // 节点总量，相加无碍

    // 联合zl节点总量应小于UINT16_MAX zllen类型限制
    zllength = zllength < UINT16_MAX ? zllength : UINT16_MAX;

    // 在开始分离内存之前保存尾偏移位置
    size_t first_offset = intrev32ifbe(ZIPLIST_TAIL_OFFSET(*first));
    size_t second_offset = intrev32ifbe(ZIPLIST_TAIL_OFFSET(*second));

    // 将目标扩展到新字节，然后追加或前置源
    target = zrealloc(target, zlbytes); // 重新分配空间 zlbytes
    if (append) { // 追加
        // 拷贝源 TARGET - END, SOURCE - HEADER
        memcpy(target + target_bytes - ZIPLIST_END_SIZE,
               source + ZIPLIST_HEADER_SIZE,
               source_bytes - ZIPLIST_HEADER_SIZE);
    } else { // 前置
        // 将目标内容移到 SOURCE-END，然后拷贝源到腾出的空间 SOURCE-END
        // SOURCE-END, TARGET-HEADER
        memmove(target + source_bytes - ZIPLIST_END_SIZE,
                target + ZIPLIST_HEADER_SIZE,
                target_bytes - ZIPLIST_HEADER_SIZE); // target整体后移
        memcpy(target, source, source_bytes - ZIPLIST_END_SIZE); // source前移
    }

    // 更新头部元信息 zlbytes zllen zltail
    ZIPLIST_BYTES(target) = intrev32ifbe(zlbytes);
    ZIPLIST_LENGTH(target) = intrev16ifbe(zllength);
    ZIPLIST_TAIL_OFFSET(target) = intrev32ifbe(
                                   (first_bytes - ZIPLIST_END_SIZE) +
                                   (second_offset - ZIPLIST_HEADER_SIZE));

    // 级联更新 主要针对 prevlen，这里从target+first_offset开始
    target = __ziplistCascadeUpdate(target, target+first_offset);

    // 源列表释放置空
    if (append) {
        zfree(*second);
        *second = NULL;
        *first = target;
    } else {
        zfree(*first);
        *first = NULL;
        *second = target;
    }
    return target;
}
```

##### `ziplistPush`

向压缩列表 `zl` 插入新节点 `s`，通过 `where` 指定从头还是尾，然后调用 `__ziplistInsert` 实现。

```c
unsigned char *ziplistPush(unsigned char *zl, unsigned char *s, unsigned int slen, int where) {
    unsigned char *p;
    p = (where == ZIPLIST_HEAD) ? ZIPLIST_ENTRY_HEAD(zl) : ZIPLIST_ENTRY_END(zl);
    return __ziplistInsert(zl,p,s,slen);
}
```

##### `ziplistIndex`

返回压缩列表迭代指定的偏移处的节点，负数从尾开始。

```c
unsigned char *ziplistIndex(unsigned char *zl, int index) {
    unsigned char *p;
    unsigned int prevlensize, prevlen = 0;
    size_t zlbytes = intrev32ifbe(ZIPLIST_BYTES(zl));
    if (index < 0) { // 从尾开始
        index = (-index)-1;
        p = ZIPLIST_ENTRY_TAIL(zl); // 获取尾节点
        if (p[0] != ZIP_END) { // 非结束节点
            ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
            while (prevlen > 0 && index--) { // prevlen>0确保没到头节点
                p -= prevlen; // 向前偏移
                assert(p >= zl + ZIPLIST_HEADER_SIZE && p < zl + zlbytes - ZIPLIST_END_SIZE); // 断言 p 在正常范围内 未溢出
                ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
            }
        }
    } else { // 从头开始
        p = ZIPLIST_ENTRY_HEAD(zl); // 获取头节点
        while (index--) {
            p += zipRawEntryLengthSafe(zl, zlbytes, p);
            if (p[0] == ZIP_END) // 到结束节点
                break;
        }
    }
    if (p[0] == ZIP_END || index > 0) // 结束节点或idx超出zllen
        return NULL;
    zipAssertValidEntry(zl, zlbytes, p); // 验证节点合法
    return p; // 返回节点
}
```

##### `ziplistNext`

返回压缩列表指定节点 `p` 的下一个节点。

```c
unsigned char *ziplistNext(unsigned char *zl, unsigned char *p) {
    ((void) zl);
    size_t zlbytes = intrev32ifbe(ZIPLIST_BYTES(zl));
	// 当前节点或下一个节点为结束节点，没有下一个节点
    if (p[0] == ZIP_END) return NULL;
    p += zipRawEntryLength(p);
    if (p[0] == ZIP_END) return NULL;
    zipAssertValidEntry(zl, zlbytes, p);
    return p;
}
```

##### `ziplistPrev`

返回压缩列表指定节点 `p` 的上一个节点。

```c
unsigned char *ziplistPrev(unsigned char *zl, unsigned char *p) {
    unsigned int prevlensize, prevlen = 0;
	// 结束节点返回NULL
    if (p[0] == ZIP_END) { // 如果指定节点是结束节点，就获取尾节点，判断是否为结束节点
        p = ZIPLIST_ENTRY_TAIL(zl);
        return (p[0] == ZIP_END) ? NULL : p;
    } else if (p == ZIPLIST_ENTRY_HEAD(zl)) { // 头节点返回NULL
        return NULL;
    } else {
        ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
        assert(prevlen > 0); // 非头节点
        p-=prevlen; // 往前偏移
        size_t zlbytes = intrev32ifbe(ZIPLIST_BYTES(zl));
        zipAssertValidEntry(zl, zlbytes, p);
        return p;
    }
}
```

##### `ziplistGet`

获取指针 `p` 处的节点，将相关信息保存在 `*sstr` 或 `sval` （取决于节点的编码）。

```c
// *sstr 字符串 slen 字符串长度
// sval 整型
unsigned int ziplistGet(unsigned char *p, unsigned char **sstr, unsigned int *slen, long long *sval) {
    zlentry entry;
    if (p == NULL || p[0] == ZIP_END) return 0;
    if (sstr) *sstr = NULL; // 重置

    zipEntry(p, &entry);
    if (ZIP_IS_STR(entry.encoding)) { // 判断是否为字符串
        if (sstr) {
            *slen = entry.len;
            *sstr = p+entry.headersize;
        }
    } else {
        if (sval) {
            *sval = zipLoadInteger(p+entry.headersize,entry.encoding);
        }
    }
    return 1;
}
```

##### `ziplistInsert`

在节点 `p` 后插入新节点。直接调用 `__ziplistInsert` 实现。

##### `ziplistDelete`

删除指定节点。

```c
unsigned char *ziplistDelete(unsigned char *zl, unsigned char **p) {
    size_t offset = *p-zl;
    zl = __ziplistDelete(zl,*p,1);
    *p = zl+offset;
    return zl;
}
```

##### `ziplistDeleteRange`

删除一系列节点。

```c
unsigned char *ziplistDeleteRange(unsigned char *zl, int index, unsigned int num) {
    unsigned char *p = ziplistIndex(zl,index);
    return (p == NULL) ? zl : __ziplistDelete(zl,p,num);
}
```

##### `ziplistReplace`

替换 `p` 处的节点为 `s`，相当于删除然后插入。

```c
unsigned char *ziplistReplace(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen) {

    // 获取当前节点的元信息
    zlentry entry;
    zipEntry(p, &entry);

    // 计算需要存储节点的长度，包括prevlen
    unsigned int reqlen;
    unsigned char encoding = 0;
    long long value = 123456789;
    if (zipTryEncoding(s,slen,&value,&encoding)) { // 尝试对s进行编码
        reqlen = zipIntSize(encoding); // 编码需要的字节数
    } else { // 字符串
        reqlen = slen; /* encoding == 0 */
    }
    reqlen += zipStoreEntryEncoding(NULL,encoding,slen); // encoding长度

    if (reqlen == entry.lensize + entry.len) { // 刚好合适
        // 简单重写节点
        p += entry.prevrawlensize;
        p += zipStoreEntryEncoding(p,encoding,slen); // encoding
        // 拷贝值
        if (ZIP_IS_STR(encoding)) {
            memcpy(p,s,slen);
        } else {
            zipSaveInteger(p,value,encoding);
        }
    } else { // 删除 & 新增
        zl = ziplistDelete(zl,&p);
        zl = ziplistInsert(zl,p,s,slen);
    }
    return zl;
}
```

##### `ziplistFind`

查找指向与指定节点相等的节点指针。

```c
unsigned char *ziplistFind(unsigned char *zl, unsigned char *p, unsigned char *vstr, unsigned int vlen, unsigned int skip) {
    int skipcnt = 0;
    unsigned char vencoding = 0;
    long long vll = 0;
    size_t zlbytes = ziplistBlobLen(zl);

    while (p[0] != ZIP_END) { // 不是结束节点
        struct zlentry e;
        unsigned char *q;

        assert(zipEntrySafe(zl, zlbytes, p, &e, 1));
        q = p + e.prevrawlensize + e.lensize;

        if (skipcnt == 0) {
            // 将当前节点和特殊节点比较
            if (ZIP_IS_STR(e.encoding)) { // 字符串
                if (e.len == vlen && memcmp(q, vstr, vlen) == 0) { // 找到就直接返回节点p
                    return p;
                }
            } else { // 整型
                if (vencoding == 0) {
                    if (!zipTryEncoding(vstr, vlen, &vll, &vencoding)) {
                        // 如果节点vstr无法被编码，就将vencoding设置为UCHAR_MAX。所以在下次就不用尝试
                        vencoding = UCHAR_MAX;
                    }
                    // 现在vencoding一定非0
                    assert(vencoding);
                }

                // 在vencoding不为UCHAR_MAX时（没有这种编码的可能性，不是有效的整型），比较当前节点和特殊节点
                if (vencoding != UCHAR_MAX) {
                    long long ll = zipLoadInteger(q, e.encoding);
                    if (ll == vll) {
                        return p;
                    }
                }
            }

            // 重置 skip 统计
            skipcnt = skip;
        } else { // 跳过节点
            skipcnt--;
        }
        p = q + e.len; // 节点偏移
    }

    return NULL;
}
```

##### `ziplistLen`

返回压缩列表的节点数量。

```c
unsigned int ziplistLen(unsigned char *zl) {
    unsigned int len = 0;
    if (intrev16ifbe(ZIPLIST_LENGTH(zl)) < UINT16_MAX) { // 如果在UINT16_MAX内
        len = intrev16ifbe(ZIPLIST_LENGTH(zl));
    } else { // 循环计数
        unsigned char *p = zl+ZIPLIST_HEADER_SIZE;
        size_t zlbytes = intrev32ifbe(ZIPLIST_BYTES(zl));
        while (*p != ZIP_END) {
            p += zipRawEntryLengthSafe(zl, zlbytes, p);
            len++;
        }

        // 如果len小于UINT16_MAX，更新压缩列表的zllen
        if (len < UINT16_MAX) ZIPLIST_LENGTH(zl) = intrev16ifbe(len);
    }
    return len;
}
```

##### `ziplistBlobLen`

返回压缩列表字节总量。

```c
size_t ziplistBlobLen(unsigned char *zl) {
    return intrev32ifbe(ZIPLIST_BYTES(zl));
}
```

##### `ziplistRepr`

标准打印压缩列表？

```c
void ziplistRepr(unsigned char *zl) {
    unsigned char *p;
    int index = 0;
    zlentry entry;
    size_t zlbytes = ziplistBlobLen(zl);

    printf(
        "{total bytes %u} "
        "{num entries %u}\n"
        "{tail offset %u}\n",
        intrev32ifbe(ZIPLIST_BYTES(zl)),
        intrev16ifbe(ZIPLIST_LENGTH(zl)),
        intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl)));
    p = ZIPLIST_ENTRY_HEAD(zl);
    while(*p != ZIP_END) {
        assert(zipEntrySafe(zl, zlbytes, p, &entry, 1));
        printf(
            "{\n"
                "\taddr 0x%08lx,\n"
                "\tindex %2d,\n"
                "\toffset %5lu,\n"
                "\thdr+entry len: %5u,\n"
                "\thdr len%2u,\n"
                "\tprevrawlen: %5u,\n"
                "\tprevrawlensize: %2u,\n"
                "\tpayload %5u\n",
            (long unsigned)p,
            index,
            (unsigned long) (p-zl),
            entry.headersize+entry.len,
            entry.headersize,
            entry.prevrawlen,
            entry.prevrawlensize,
            entry.len);
        printf("\tbytes: ");
        for (unsigned int i = 0; i < entry.headersize+entry.len; i++) {
            printf("%02x|",p[i]);
        }
        printf("\n");
        p += entry.headersize;
        if (ZIP_IS_STR(entry.encoding)) {
            printf("\t[str]");
            if (entry.len > 40) {
                if (fwrite(p,40,1,stdout) == 0) perror("fwrite");
                printf("...");
            } else {
                if (entry.len &&
                    fwrite(p,entry.len,1,stdout) == 0) perror("fwrite");
            }
        } else {
            printf("\t[int]%lld", (long long) zipLoadInteger(p,entry.encoding));
        }
        printf("\n}\n");
        p += entry.len;
        index++;
    }
    printf("{end}\n\n");
}
```

##### `ziplistValidateIntegrity`

验证数据结构的完整性。`deep` 是否深度验证，头和节点。

```c
int ziplistValidateIntegrity(unsigned char *zl, size_t size, int deep,
    ziplistValidateEntryCB entry_cb, void *cb_userdata) {
    // 检测实际读取到的头大小 HEADER+END
    if (size < ZIPLIST_HEADER_SIZE + ZIPLIST_END_SIZE) return 0;

    // 检测在头部编码的大小是否匹配分配的大小
    size_t bytes = intrev32ifbe(ZIPLIST_BYTES(zl));
    if (bytes != size) return 0;

    // 检测最后一个字节必须是结束符
    if (zl[size - ZIPLIST_END_SIZE] != ZIP_END) return 0;

    // 检测尾偏移没有溢出分配空间
    if (intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl)) > size - ZIPLIST_END_SIZE) return 0;

    // 非深度验证结束
    if (!deep) return 1;

    unsigned int count = 0;
    unsigned char *p = ZIPLIST_ENTRY_HEAD(zl); // 定位头节点
    unsigned char *prev = NULL;
    size_t prev_raw_size = 0;
    while(*p != ZIP_END) { // 没有结束字节
        struct zlentry e;
        // 解码节点头和尾
        if (!zipEntrySafe(zl, size, p, &e, 1)) return 0;

        // 确保说明前一个节点大小的准确性
        if (e.prevrawlen != prev_raw_size) return 0;

        // 选择性回调验证
        if (entry_cb && !entry_cb(p, cb_userdata)) return 0;

        // 节点偏移
        prev_raw_size = e.headersize + e.len;
        prev = p;
        p += e.headersize + e.len;
        count++;
    }

    // 确保zltail节点指向最后一个节点的开始位置
    if (prev != ZIPLIST_ENTRY_TAIL(zl)) return 0;

    // 检测在头部统计的节点数准确性
    unsigned int header_count = intrev16ifbe(ZIPLIST_LENGTH(zl));
    if (header_count != UINT16_MAX && count != header_count) return 0;

    return 1;
}
```

##### `ziplistRandomPair`

随机返回一对键和值，存储到 `key` 和 `val` 参数（`val` 可以为空在不需要的情况下）。

```c
 // total_count 是提前计算好的 压缩列表节点数量的二分之一
void ziplistRandomPair(unsigned char *zl, unsigned long total_count, ziplistEntry *key, ziplistEntry *val) {
    int ret;
    unsigned char *p;

    assert(total_count); // 避免在损坏的压缩列表被0除

    // 生成偶数，因为压缩列表要保存 k-v 对
    int r = (rand() % total_count) * 2;
    p = ziplistIndex(zl, r);
    ret = ziplistGet(p, &key->sval, &key->slen, &key->lval);
    assert(ret != 0);

    if (!val) return;
    p = ziplistNext(zl, p);
    ret = ziplistGet(p, &val->sval, &val->slen, &val->lval);
    assert(ret != 0);
}
```

##### `ziplistRandomPairs`

随便返回多对（数量由 `count` 指定 ）键值（存储到 `keys` 和 `vals` 参数），可能会重复。

```c
void ziplistRandomPairs(unsigned char *zl, unsigned int count, ziplistEntry *keys, ziplistEntry *vals) {
    unsigned char *p, *key, *value;
    unsigned int klen = 0, vlen = 0;
    long long klval = 0, vlval = 0;

    // index属性必须在第一个，由于在uintCompare要使用
    typedef struct {
        unsigned int index;
        unsigned int order;
    } rand_pick; 
    rand_pick *picks = zmalloc(sizeof(rand_pick)*count);
    unsigned int total_size = ziplistLen(zl)/2;
    assert(total_size); // 非空

    // 创建一个随机索引池（有些可能会重复）
    for (unsigned int i = 0; i < count; i++) {
        picks[i].index = (rand() % total_size) * 2; // 生成偶数索引
        picks[i].order = i; // 保持选取它们的顺序
    }
    qsort(picks, count, sizeof(rand_pick), uintCompare); // 根据index进行快排

    // 从压缩列表中获取节点到一个输出数组（按照原始顺序）
    unsigned int zipindex = 0, pickindex = 0;
    p = ziplistIndex(zl, 0); // 头节点？为啥不用ZIPLIST_ENTRY_HEAD定位
    while (ziplistGet(p, &key, &klen, &klval) && pickindex < count) {
        p = ziplistNext(zl, p); // 下一个节点
        assert(ziplistGet(p, &value, &vlen, &vlval)); // 获取val
        while (pickindex < count && zipindex == picks[pickindex].index) {
            int storeorder = picks[pickindex].order;
            ziplistSaveValue(key, klen, klval, &keys[storeorder]); // 存储key
            if (vals)
                ziplistSaveValue(value, vlen, vlval, &vals[storeorder]); // 存储val
             pickindex++;
        }
        zipindex += 2;
        p = ziplistNext(zl, p); // 下一个节点
    }

    zfree(picks);
}
```

##### `ziplistRandomPairsUnique`

随机返回多对键值（唯一版）。

```c
unsigned int ziplistRandomPairsUnique(unsigned char *zl, unsigned int count, ziplistEntry *keys, ziplistEntry *vals) {
    unsigned char *p, *key;
    unsigned int klen = 0;
    long long klval = 0;
    unsigned int total_size = ziplistLen(zl)/2;
    unsigned int index = 0;
    if (count > total_size) count = total_size;

    /* To only iterate once, every time we try to pick a member, the probability
     * we pick it is the quotient of the count left we want to pick and the
     * count still we haven't visited in the dict, this way, we could make every
     * member be equally picked.*/
    p = ziplistIndex(zl, 0);
    unsigned int picked = 0, remaining = count;
    // 只需要循环一次，每次都会尝试去选取节点。选取它的概率是想要选取的剩余计数和在字典中尚未访问到的数量的商
    while (picked < count && p) {
        double randomDouble = ((double)rand()) / RAND_MAX;
        // 剩余计数/尚未访问到的计数
        double threshold = ((double)remaining) / (total_size - index);
        if (randomDouble <= threshold) {
            assert(ziplistGet(p, &key, &klen, &klval));
            ziplistSaveValue(key, klen, klval, &keys[picked]);
            p = ziplistNext(zl, p);
            assert(p);
            if (vals) {
                assert(ziplistGet(p, &key, &klen, &klval));
                ziplistSaveValue(key, klen, klval, &vals[picked]);
            }
            remaining--;
            picked++;
        } else {
            p = ziplistNext(zl, p);
            assert(p);
        }
        p = ziplistNext(zl, p);
        index++;
    }
    return picked;
}
```

#### 本章小结

看完之后，大概知道，这是种什么数据了，一系列特殊编码的连续内存块组成的顺序型数据结构。确实很紧密，节约内存。每个内存块都有着不同的业务含义。当然，其中有些操作，还是比较迷惑，以后再说吧。