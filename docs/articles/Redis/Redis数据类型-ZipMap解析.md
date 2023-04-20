# Redis源码之数据类型解析-ZipMap {docsify-ignore}

当前分析 `Redis` 版本为6.2，需要注意。

`ZipMap`，压缩字典。

#### 基础结构

##### `ZipMap`

压缩字典。其内存布局，`<zmlen><len>"key1"<len><free>"val1"<len>"key2"<len><free>"val2"<zmend>`。

实例：\x02\x03foo\x03\x00bar\x05hello\x05\x00world\xff

* `1 byte zmlen`，压缩字典的长度，即键和值的数量，如果该长度大于254(即ZIPMAP_BIGLEN)，需要遍历压缩字典获取实际长度。
* `5 bytes len`，接下来的字符串(键或值)的长度，如果第一个字节为无符号字节，则为单字节长度；如果是254，则后面跟着一个四字节无符号整数（按主机字节顺序）。值255用于表示压缩字典结束符。
* `uint8_t free`，字符串后面剩下未用的字节数量。
* `uint8_t zmend`，标记压缩字典尾端的特殊值 `0xFF`。

#### 宏定义

```c
#define ZIPMAP_BIGLEN 254 // 压缩字典最大长度
#define ZIPMAP_END 255 // 压缩字典结束符标记
#define ZIPMAP_VALUE_MAX_FREE 4 // 值后面最大的尾字节数量
// 返回压缩字典的长度编码的字节数
#define ZIPMAP_LEN_BYTES(_l) (((_l) < ZIPMAP_BIGLEN) ? 1 : sizeof(unsigned int)+1)
```

#### 私有函数
##### `sdslen`

#### 接口函数

##### `zipmapNew`

创建压缩字典。

```c
unsigned char *zipmapNew(void) {
    unsigned char *zm = zmalloc(2); // 直接分配两个字节的空间，存一个zmlen和end
    zm[0] = 0; /* Length */
    zm[1] = ZIPMAP_END; // 0xff
    return zm;
}
```

##### `zipmapDecodeLength`

解析编码长度。

```c
static unsigned int zipmapDecodeLength(unsigned char *p) {
    unsigned int len = *p; // 第一个字节就是zmlen，在内存布局中
    if (len < ZIPMAP_BIGLEN) return len; // 如果小于zm最大长度，直接返回即可，没毛病
    memcpy(&len,p+1,sizeof(unsigned int));
    memrev32ifbe(&len);
    return len;
}
```