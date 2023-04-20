# Redis源码之数据类型解析-List {docsify-ignore}

当前分析 `Redis` 版本为6.2，需要注意。

`List`，链表，这里是双向链表，线性结构，这是一种常见的数据结构，拥有很高节点访问和重排的效率，`Redis` 自行实现的，`C` 语言没有这一数据结构。相关文件主要有 `adlist.h` 和 `adlist.c`。

#### 基础结构

这里倒是简单，就节点和持有节点的链表结构。

##### 链表节点

```c
typedef struct listNode { // prev和next的实现，在获取当前节点的左右节点非常高效
    struct listNode *prev; // 前（左）节点地址
    struct listNode *next; // 后（右）节点地址
    void *value; // 当前值，采用 void 指针，不单独指定类型，也就是说，可以存储各种类型的数据
} listNode;
```

##### 链表迭代器

```c
typedef struct listIter {
    listNode *next; // 下一个节点位置
    int direction; // 指定方向，左或右 0|1
    // 有相应常量指定
    // #define AL_START_HEAD 0 从头部开始
`   // #define AL_START_TAIL 1 从尾部开始
} listIter;
```

##### 双向链表

以下简称链表。

```c
typedef struct list {
    listNode *head; // 头节点位置
    listNode *tail; // 尾节点位置
    // 复制释放和匹配的句柄独立实现，多态需要，毕竟不同数据类型，处理也不大一致的
    void *(*dup)(void *ptr); // 复制句柄 复制节点保存的值
    void (*free)(void *ptr); // 释放句柄 释放节点保存的值
    int (*match)(void *ptr, void *key); // 匹配句柄 
    unsigned long len; // 链表长度
} list;
```

#### 宏定义函数

##### `listLength`

获取链表长度，`list->len`。

##### `listFirst`

获取链表头元素，也就是第一个元素，`list->head`。

##### `listLast`

获取链表尾元素，也就是最后一个元素，`list->tail`。

##### `listPrevNode`

获取链表节点的前一个节点，`node->prev`。

##### `listNextNode`

获取链表节点的后一个节点，`node->next`。

##### `listNodeValue`

获取链表节点的值，`node->value`。

##### `listSetDupMethod`

设置链表的复制句柄，`list->dup=fn`。

##### `listSetFreeMethod`

设置链表的释放句柄，`list->free=fn`。

##### `listSetMatchMethod`

设置链表的匹配句柄，`list->match=fn`。

##### `listGetDupMethod`

获取链表的复制句柄，`list->dup`。

##### `listGetFreeMethod`

获取链表的释放句柄，`list->free`。

##### `listGetMatchMethod`

获取链表的匹配句柄，`list->match`。

#### 接口函数

##### `listCreate`

创建链表，这种方式创建的链表可以通过 `listRelease` 释放，但是其中每个节点的值需要在调用 `listRelease`之前释放，或者通过设置链表的释放句柄。创建失败（主要是指分配空间失败），返回 `NULL`。成功就是新链表的指针。

```c
list *listCreate(void)
{
    struct list *list;

    if ((list = zmalloc(sizeof(*list))) == NULL) // 分配空间
        return NULL;
    // 以下皆是默认值设置处理
    list->head = list->tail = NULL;
    list->len = 0;
    list->dup = NULL;
    list->free = NULL;
    list->match = NULL;
    return list;
}
```

##### `listEmpty`

移除链表的所有节点，并不会销毁链表本身。清洗？

```c
void listEmpty(list *list)
{
    unsigned long len; // 链表长度，循环变量 递减
    listNode *current, *next;

    current = list->head; // 当前节点 从 head 开始
    len = list->len;
    while(len--) { // 该过程不能出错，否则 list->head 无效 
        next = current->next;
        if (list->free) list->free(current->value); // 如果指针了释放句柄，就调用处理
        zfree(current); // 释放节点本身
        current = next; // 节点偏移
    }
    // 初始值设置
    list->head = list->tail = NULL;
    list->len = 0;
}
```

##### `listRelease`

释放整个链表，首先调用 `listEmpty` 处理节点，在使用 `zfree` 释放链表本身。

##### `listAddNodeHead`

从头新加节点。

```c
list *listAddNodeHead(list *list, void *value)
{
    listNode *node;

    if ((node = zmalloc(sizeof(*node))) == NULL) // 分配节点空间
        return NULL;
    node->value = value; // 保存节点值
    if (list->len == 0) { // 如果空链表，直接设置头尾节点
        list->head = list->tail = node;
        node->prev = node->next = NULL; // 防止循环链表
    } else { // 非空链表 更新头节点
        node->prev = NULL;
        node->next = list->head;
        list->head->prev = node;
        list->head = node;
    }
    list->len++; // 更新链表长度
    return list;
}
```

##### `listAddNodeTail`

从尾新加节点。基本上，和从头部新加节点差不多。

```c
list *listAddNodeTail(list *list, void *value)
{
    listNode *node;

    if ((node = zmalloc(sizeof(*node))) == NULL)
        return NULL;
    node->value = value;
    if (list->len == 0) {
        list->head = list->tail = node;
        node->prev = node->next = NULL;
    } else { // 更新尾节点
        node->prev = list->tail;
        node->next = NULL;
        list->tail->next = node;
        list->tail = node;
    }
    list->len++;
    return list;
}
```

##### `listInsertNode`

向指定节点的左或右（通过 `after` 判断）新加节点。

```c
list *listInsertNode(list *list, listNode *old_node, void *value, int after) {
    listNode *node;
    // 新节点
    if ((node = zmalloc(sizeof(*node))) == NULL)
        return NULL;
    node->value = value;
    if (after) { // 新节点在指定节点之后
        node->prev = old_node;
        node->next = old_node->next;
        if (list->tail == old_node) { // 如果指定节点是尾节点
            list->tail = node;
        }
    } else { // 之前
        node->next = old_node;
        node->prev = old_node->prev;
        if (list->head == old_node) { // 如果指定节点是头节点
            list->head = node;
        }
    }
    // 更新左|右节点对当前节点的连接
    if (node->prev != NULL) { // 左节点
        node->prev->next = node;
    }
    if (node->next != NULL) { // 右节点
        node->next->prev = node;
    }
    list->len++; // 更新链表长度
    return list;
}
```

##### `listDelNode`

删除链表节点，该函数不会失败。

```c
void listDelNode(list *list, listNode *node)
{
    if (node->prev) // 更新左节点的右节点连接 或头节点直接指定
        node->prev->next = node->next;
    else
        list->head = node->next;
    if (node->next) // 更新右节点的左节点连接 或尾节点直接指定
        node->next->prev = node->prev;
    else
        list->tail = node->prev;
    
    if (list->free) list->free(node->value); // 释放值
    zfree(node); // 释放节点
    list->len--; // 更新链表长度
}
```

##### `listGetIterator`

获取链表迭代器，不会失败。

```c
listIter *listGetIterator(list *list, int direction)
{
    listIter *iter;

    if ((iter = zmalloc(sizeof(*iter))) == NULL) return NULL;
    if (direction == AL_START_HEAD)
        iter->next = list->head; // 从头开始
    else
        iter->next = list->tail; // 从尾开始
    iter->direction = direction; // 指定方向
    return iter;
}
```

##### `listReleaseIterator`

释放链表迭代器。和 `listGetIterator` 配套的。

##### `listRewind`

重置从头开始的链表迭代器。

```c
void listRewind(list *list, listIter *li) {
    li->next = list->head;
    li->direction = AL_START_HEAD;
}
```

##### `listRewindTail`

重置从尾开始的链表迭代器。

```c
void listRewindTail(list *list, listIter *li) {
    li->next = list->tail;
    li->direction = AL_START_TAIL;
}
```

`listNext`

链表迭代器偏移。返回对应节点。

```c
listNode *listNext(listIter *iter)
{
    listNode *current = iter->next;

    if (current != NULL) { // 非空情况
        if (iter->direction == AL_START_HEAD) // 判断方向
            iter->next = current->next; // 从头开始
        else
            iter->next = current->prev; // 从尾开始
    }
    return current;
}
```

##### `listDup`

复制整个链表，如果内存溢出返回 `NULL` 。主要拷贝句柄和节点。

```c
list *listDup(list *orig)
{
    list *copy;
    listIter iter;
    listNode *node;

    if ((copy = listCreate()) == NULL) // 创建新链表
        return NULL;
    // 拷贝相应句柄
    copy->dup = orig->dup;
    copy->free = orig->free;
    copy->match = orig->match;
    
    listRewind(orig, &iter); // 重置原链表迭代器（从头开始）
    while((node = listNext(&iter)) != NULL) { // 开始迭代
        void *value;

        if (copy->dup) {
            // 值复制
            value = copy->dup(node->value);
            if (value == NULL) {
                listRelease(copy);
                return NULL;
            }
        } else
            value = node->value;
        // 添加新节点
        if (listAddNodeTail(copy, value) == NULL) {
            listRelease(copy);
            return NULL;
        }
    }
    return copy; // 返回新链表
}
```

##### `listSearchKey`

在指定链表查找关键值 `key`。成功返回相应节点，失败为 `NULL`。

```c
listNode *listSearchKey(list *list, void *key)
{
    listIter iter;
    listNode *node;

    listRewind(list, &iter); // 重置迭代器
    while((node = listNext(&iter)) != NULL) {
        // 匹配过程
        if (list->match) {
            if (list->match(node->value, key)) {
                return node;
            }
        } else {
            if (key == node->value) {
                return node;
            }
        }
    }
    return NULL;
}
```

##### `listIndex`

返回指定索引的节点，0是头节点，-1是尾节点，正从头开始，负从尾开始。`idx` 溢出，返回头节点或尾节点。

```c
listNode *listIndex(list *list, long index) {
    listNode *n;

    if (index < 0) {
        index = (-index)-1; // 重新计算idx
        n = list->tail; // 从尾开始
        while(index-- && n) n = n->prev;
    } else {
        n = list->head; // 从头开始
        while(index-- && n) n = n->next;
    }
    return n;
}
```

##### `listRotateTailToHead`

将尾节点转移到头节点。

```c
void listRotateTailToHead(list *list) {
    if (listLength(list) <= 1) return;

    // 分离当前尾节点
    listNode *tail = list->tail;
    list->tail = tail->prev;
    list->tail->next = NULL;
    // 将它添加到头节点
    list->head->prev = tail;
    tail->prev = NULL;
    tail->next = list->head; 
    list->head = tail; // 更新头节点
}
```

##### `listRotateHeadToTail`

将头节点转移到尾节点。

```c
void listRotateHeadToTail(list *list) {
    if (listLength(list) <= 1) return;

    listNode *head = list->head;
    // 分离当前头节点
    list->head = head->next;
    list->head->prev = NULL;
    // 添加到尾节点
    list->tail->next = head;
    head->next = NULL;
    head->prev = list->tail;
    list->tail = head; // 更新尾节点
}
```

##### `listJoin`

将指定链表 `o` 的所有节点转移到指定链表 `l` 的尾部。只需要处理头或尾即可。最后将 `o` 链表置空。

```c
void listJoin(list *l, list *o) {
    if (o->len == 0) return;

    o->head->prev = l->tail; // 将 o 头节点的左节点连接到l的尾节点

    if (l->tail) // 如果 l 尾节点存在，则其右节点连接到 o 头节点
        l->tail->next = o->head;
    else // 否则，直接赋值头节点
        l->head = o->head;
	// 更新 l 链表的尾节点和长度
    l->tail = o->tail;
    l->len += o->len;

    // 将 o 链表置空
    o->head = o->tail = NULL;
    o->len = 0;
}
```

#### 本章小结

也看到了，链表结构的简洁，在对元素的操作上确实高效，很少涉及到循环处理。