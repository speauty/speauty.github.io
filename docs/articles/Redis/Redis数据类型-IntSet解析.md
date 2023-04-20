# Redis源码之数据类型解析-IntSet {docsify-ignore}

当前分析 `Redis` 版本为6.2，需要注意。

整数集合（`IntSet`），`Redis` 用于保存整数值的集合抽象数据结构，可以保存 `int16_t`、`int32_t` 或者 `int64_t` 的整数值，并且集合满足唯一性（集合不包含重复项）和有序性（集合中的元素按照从小到大有序排序）。

#### 基础结构

```c
typedef struct intset { // 整数集合
    // encoding可选值 及其范围
    // INTSET_ENC_INT16 int16_t -2^15(min) ~ 2^16-1(max)
    // INTSET_ENC_INT32 int32_t -2^31 ~ 2^31-1
    // INTSET_ENC_INT64 int64_t -2^63 ~ 2^63-1
    // 在 intset.c 中定义
    // #define INTSET_ENC_INT16 (sizeof(int16_t))
    // #define INTSET_ENC_INT32 (sizeof(int32_t))
    // #define INTSET_ENC_INT64 (sizeof(int64_t))
    uint32_t encoding; // 编码方式，指定contents数组中元素的类型
    uint32_t length; // 集合包含元素数量 contents长度
    int8_t contents[]; // 保存元素的数组 虽然这里类型是int8_t，实际上是看encoding属性
} intset;
```

#### 私有函数

##### `_intsetValueEncoding`

获取指定数值的编码方式。最小为 `int16_t`。

```c
static uint8_t _intsetValueEncoding(int64_t v) {
    if (v < INT32_MIN || v > INT32_MAX)
        return INTSET_ENC_INT64; // int64_t
    else if (v < INT16_MIN || v > INT16_MAX)
        return INTSET_ENC_INT32; // int32_t
    else
        return INTSET_ENC_INT16; // int16_t
}
```

##### `_intsetGetEncoded`

获取整数集合给定位置处的值，已知编码方式 `uint8_t enc`。

```c
static int64_t _intsetGetEncoded(intset *is, int pos, uint8_t enc) {
    // 三个返回值定义
    int64_t v64;
    int32_t v32;
    int16_t v16;

    if (enc == INTSET_ENC_INT64) {
        memcpy(&v64,((int64_t*)is->contents)+pos,sizeof(v64)); // 直接拷贝
        memrev64ifbe(&v64); // 将指针所指的64位无符号整数从小端数切换到大端数
        return v64;
    } else if (enc == INTSET_ENC_INT32) {
        memcpy(&v32,((int32_t*)is->contents)+pos,sizeof(v32));
        memrev32ifbe(&v32); // 32位
        return v32;
    } else {
        memcpy(&v16,((int16_t*)is->contents)+pos,sizeof(v16));
        memrev16ifbe(&v16); // 16位
        return v16;
    }
}
```

##### `_intsetGet`

返回整数集合指定位置的值，编码方式由集合指定 `is->encoding` （需要调用 `intrev32ifbe` 对值进行转换，从小端切到大端），然后直接调用 实现 `_intsetGetEncoded`。

##### `_intsetSet`

设置整数集合指定位置的值，编码方式由集合指定。

```c
static void _intsetSet(intset *is, int pos, int64_t value) {
    uint32_t encoding = intrev32ifbe(is->encoding);

    if (encoding == INTSET_ENC_INT64) {
        ((int64_t*)is->contents)[pos] = value;
        memrev64ifbe(((int64_t*)is->contents)+pos); // 大小端切换
    } else if (encoding == INTSET_ENC_INT32) {
        // ...
    } else {
        // ...
    }
}
```

##### `intsetResize`

调整集合大小，采用 `zrealloc`。当集合元素类型变动时调用？

```c
static intset *intsetResize(intset *is, uint32_t len) {
    uint32_t size = len*intrev32ifbe(is->encoding); // 计算contents长度
    is = zrealloc(is,sizeof(intset)+size); // 集合长度+contents长度 的空间
    return is;
}
```

##### `intsetSearch`

在整数集合中查找整数值。

```c
static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos) {
    // value 给定整数值 *pos 查到的索引，在集合中
    int min = 0, max = intrev32ifbe(is->length)-1, mid = -1;
    int64_t cur = -1;

    if (intrev32ifbe(is->length) == 0) { // 空集合，自然
        if (pos) *pos = 0;
        return 0;
    } else {
        if (value > _intsetGet(is,max)) { // 大于最大值
            if (pos) *pos = intrev32ifbe(is->length);
            return 0;
        } else if (value < _intsetGet(is,0)) { // 小于最小值
            if (pos) *pos = 0;
            return 0;
        }
    }

    while(max >= min) { // 二分查找
        mid = ((unsigned int)min + (unsigned int)max) >> 1;
        cur = _intsetGet(is,mid);
        if (value > cur) {
            min = mid+1;
        } else if (value < cur) {
            max = mid-1;
        } else {
            break;
        }
    }

    if (value == cur) {
        if (pos) *pos = mid;
        return 1;
    } else { // 未找到 pos 为该插入的位置
        if (pos) *pos = min;
        return 0;
    }
}
```

##### `intsetUpgradeAndAdd`

将整数集合的编码方式升级，并插入给定整数值（该值类型比原集合编码方式高，需要升级处理）。

```c
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
    uint8_t curenc = intrev32ifbe(is->encoding); // 当前集合编码方式
    uint8_t newenc = _intsetValueEncoding(value); // 给定值的编码方式
    int length = intrev32ifbe(is->length); // 集合长度
    int prepend = value < 0 ? 1 : 0; // 判断在前还是在后,该值肯定比集合所有现有值要大或小

    // 设置集合新编码方式,并调整集合大小 intsetResize
    // length+1 1为新元素 value的空间
    is->encoding = intrev32ifbe(newenc); // 设置集合
    is = intsetResize(is,intrev32ifbe(is->length)+1);

    // 升级原则(不需要重写值)
    // 需要添加的元素往往在头或尾,所以我们需要确保contents的头或尾有一个空空间即可
    while(length--) // 但从这里来看,是重写了吧(get -> set)? 反向
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));

    // 在头或尾设置值
    if (prepend)
        _intsetSet(is,0,value);
    else
        _intsetSet(is,intrev32ifbe(is->length),value);
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1); // 更新集合长度
    return is;
}
```

##### `intsetMoveTail`

将整数集合中指定位置 `from` 以及之后的数据移到 指定位置 `to`。整体后移。`to-from` 中间位置的元素不是存在重复？

```c
static void intsetMoveTail(intset *is, uint32_t from, uint32_t to) {
    void *src, *dst;
    uint32_t bytes = intrev32ifbe(is->length)-from;
    uint32_t encoding = intrev32ifbe(is->encoding);

    if (encoding == INTSET_ENC_INT64) {
        src = (int64_t*)is->contents+from; // 源位置
        dst = (int64_t*)is->contents+to; // 目标位置
        bytes *= sizeof(int64_t);
    } else if (encoding == INTSET_ENC_INT32) {
        // ...
    } else {
        // ...
    }
    memmove(dst,src,bytes);
}
```



#### 接口函数

##### `intsetNew`

创建一个新的整数集合。编码方式默认为 `INTSET_ENC_INT16`。

```c
intset *intsetNew(void) {
    intset *is = zmalloc(sizeof(intset)); // 分配空间
    is->encoding = intrev32ifbe(INTSET_ENC_INT16); // 指定默认编码方式
    is->length = 0; // 默认长度
    return is;
}
```

##### `intsetAdd`

向整数集合中添加一个整数。

```c
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
    uint8_t valenc = _intsetValueEncoding(value); // 计算整数的编码方式
    uint32_t pos;
    if (success) *success = 1; // 重置初始值？

    // 检测是否有必要升级编码。
    if (valenc > intrev32ifbe(is->encoding)) {
        // 该操作总是成功的，所以不需要更新 *success
        return intsetUpgradeAndAdd(is,value);
    } else {
        // 检测该值是否已在集合中存在
        // 该操作会定位在未找到该元素时需要插入的正确位置
        if (intsetSearch(is,value,&pos)) {
            if (success) *success = 0;
            return is;
        }

        is = intsetResize(is,intrev32ifbe(is->length)+1); // 更新大小
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1); // 整体后移
    }

    _intsetSet(is,pos,value); // 更新pos处的值
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1); // 更新长度
    return is;
}
```

##### `intsetRemove`

从整数集合中移除一个整数。

```c
intset *intsetRemove(intset *is, int64_t value, int *success) {
    uint8_t valenc = _intsetValueEncoding(value); // 计算需要移除值的编码方式
    uint32_t pos;
    if (success) *success = 0; 
	// 移除值的编码在集合编码方式之内 且在该集合内
    if (valenc <= intrev32ifbe(is->encoding) && intsetSearch(is,value,&pos)) {
        uint32_t len = intrev32ifbe(is->length);

        // 可移除
        if (success) *success = 1;

        // 从尾重写值。整体前移
        if (pos < (len-1)) intsetMoveTail(is,pos+1,pos); 
        is = intsetResize(is,len-1); // 调整和更新长度
        is->length = intrev32ifbe(len-1);
    }
    return is;
}
```

##### `intsetFind`

判断某值在整数集合中是否存在。

```c
uint8_t intsetFind(intset *is, int64_t value) {
    uint8_t valenc = _intsetValueEncoding(value);
    return valenc <= intrev32ifbe(is->encoding) && intsetSearch(is,value,NULL);
}
```

##### `intsetRandom`

随机返回一个元素。

```c
int64_t intsetRandom(intset *is) {
    uint32_t len = intrev32ifbe(is->length);
    assert(len); // 避免在损坏的整数集有效负载上被零除
    return _intsetGet(is,rand()%len);
}
```

##### `intsetGet`

获取指定索引处的整数值。

```c
uint8_t intsetGet(intset *is, uint32_t pos, int64_t *value) {
    if (pos < intrev32ifbe(is->length)) { // 判断索引在正常范围内
        *value = _intsetGet(is,pos);
        return 1;
    }
    return 0;
}
```

##### `intsetLen`

获取整数集合的长度。

```c
uint32_t intsetLen(const intset *is) {
    return intrev32ifbe(is->length);
}
```

##### `intsetBlobLen`

计算整数集合的字节大小。

```c
size_t intsetBlobLen(intset *is) {
    return sizeof(intset)+(size_t)intrev32ifbe(is->length)*intrev32ifbe(is->encoding);
}
```

##### `intsetValidateIntegrity`

验证整数集合结构的完整性。

```c
int intsetValidateIntegrity(const unsigned char *p, size_t size, int deep) {
    // deep 是否深度验证
    // 0 只验证头部结构
    // 1 需要验证集合的唯一性和有序性
    intset *is = (intset *)p;
    if (size < sizeof(*is)) // 检测头部大小
        return 0;

    uint32_t encoding = intrev32ifbe(is->encoding);

    size_t record_size;
    if (encoding == INTSET_ENC_INT64) {
        record_size = INTSET_ENC_INT64;
    } else if (encoding == INTSET_ENC_INT32) {
        record_size = INTSET_ENC_INT32;
    } else if (encoding == INTSET_ENC_INT16){
        record_size = INTSET_ENC_INT16;
    } else { // 非标准编码方式
        return 0;
    }

    uint32_t count = intrev32ifbe(is->length);
    if (sizeof(*is) + count*record_size != size) // 整体结构大小检测
        return 0;

    if (count==0) // 验证空集合
        return 0;

    if (!deep) // 不需要深度验证，这里即可返回成功标识
        return 1;

    // 检测唯一性和有序性
    int64_t prev = _intsetGet(is,0);
    for (uint32_t i=1; i<count; i++) {
        int64_t cur = _intsetGet(is,i);
        if (cur <= prev)
            return 0;
        prev = cur;
    }

    return 1;
}
```

#### 本章小结

该数据结构需要关注的是升级，这一策略，提高了集合的灵活性，并且节约内存。将不同整型分类放置到对应的整数集合。必要时会触发升级，从范围小的升到范围大的类型。不过，没有降级的策略。