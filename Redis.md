# Redis

> Redis版本：redis: stable 6.2.5 (bottled), HEAD

## 一、Redis命令分类

### String相关命令

[https://redis.io/commands#string](https://redis.io/commands#string)

### List相关命令

[https://redis.io/commands#list](https://redis.io/commands#list)

### Map相关命令

[https://redis.io/commands#hash](https://redis.io/commands#hash)

### 集合Set相关命令

[https://redis.io/commands#set](https://redis.io/commands#set)

### 有序集合Zset相关命令

[https://redis.io/commands#sorted_set](https://redis.io/commands#sorted_set)

## 二、Redis底层数据结构

### 简单动态字符串（sds）

> sds是redis在c字符串基础上改造的保存字符串的结构

<img src="https://raw.githubusercontent.com/liuyuede123/image/main/image-20210817094107297.png" alt="image-20210817094107297" style="zoom:50%;" />

```c
struct sdshdr {
    // 记录buf数组中已使用字节的数量
    // 等于SDS所保存字符串的长度
    int len;
    // 记录buf数组中未使用字节的数量
    int free;
    // 字节数组，用于保存字符串
    char buf[];
};
```

**相比于c字符串的优点**

**1、常数复杂度获取字符串长度**

c字符串时间复杂度位O(n), sds动态字符串为O(1)。动态字符串结构体中保存了字符串长度的属性len

**2、杜绝缓冲区溢出**

c字符串扩展操作，如果没有事先判断长度的话，会导致溢出。sds提供了一套API，会在扩展的时候判断len是否超出，从而分配更多空间。

**3、减少修改字符串带来的内存重分配次数**

如果增加字符串长度之后，内存空间不够，忘记重新分配内存，会产生缓冲区溢出。如果缩减字符串长度之后，空出来的内存不及时释放的话会产生内存溢出。像redis对字符串修改操作比较频繁的数据库，如果采用c字符串，会频繁的重分配内存，带来性能开销，会占用修改字符串的一大半时间。sds通过free字段，解除了字符串长度和底层数组长度之间的关联。sds里面，数组长度可以包含未使用的字节。通过free，sds实现了，空间预分配和惰性空间释放2种优化策略。

**01、空间预分配**

当修改后字节数小于1M时，程序会分配和len属性同大小的内存空间，比如修改后13字节，重分配后13+13+1，其中1字节保存空字符串。

当修改后字节数大于1M时，比如修改后30M，重分配后30M+1M+1

**02、惰性空间释放**

当缩短sds字符串后，程序不会立即释放空间，而是记录到free中，等待将来使用。sds也有对应的api，在有需要时，也可以立即释放未使用的空间。

**4、二进制安全**

c字符串中的字符必须符合某种编码，而且除了字符串的末尾之外，不能包含空字符，否则被程序最先读入的空字符会被误认为字符串的结束。所以c字符串只能保存文本类型的数据，不能保存图片、视频等其他二进制数据。sds就可以避免这个问题，因为它不是通过空字符判断数据结束，而是通过len属性判断。

**5、兼容部分c字符串函数**

sds和c字符保持一致，会在字符串的结尾加一个空字符，可以复用c字符的函数，而不需要重写一些字符串处理的函数。

### 双向链表（ListNode）

**c语言没有内置的链表结构，redis自己实现了链表**

<img src="https://raw.githubusercontent.com/liuyuede123/image/main/image-20210817132425363.png" alt="image-20210817132425363" style="zoom:50%;" />

每个链表节点的结构，双向链表

```c
typedef struct listNode {
    // 前置节点
    struct listNode * prev;
    // 后置节点
    struct listNode * next;
    // 节点的值
    void * value;
}listNode;
```

list结构持有链表

```c
typedef struct list {
    // 表头节点
    listNode * head;
    // 表尾节点
    listNode * tail;
    // 链表所包含的节点数量
    unsigned long len;
    // 节点值复制函数
    void *(*dup)(void *ptr);
    // 节点值释放函数
    void (*free)(void *ptr);
    // 节点值对比函数
    int (*match)(void *ptr,void *key);
} list;
```

1. 双向链表，获取某个节点的前置和后继节点的时间复杂度为1
2. 无环，链表的头尾都指向null，对链表的访问以null为终点
3. 带表头指针和表尾指针，通过head和tail获取链表的头尾节点的时间复杂度为1
4. 带链表长度计数器，通过len获取列表的长度时间复杂度为1
5. 链表节点使用void*指针来保存节点值，并且可以通过list结构的dup、free、match三个属性为节点值设置类型特定函数，所以链表可以用于保存各种不同类型的值。


### 压缩列表（ziplist）

> “压缩列表（ziplist）是列表键和哈希键的底层实现之一。当一个列表键只包含少量列表项，并且每个列表项要么就是小整数值，要么就是长度比较短的字符串，那么Redis就会使用压缩列表来做列表键的底层实现。”

> “另外，当一个哈希键只包含少量键值对，比且每个键值对的键和值要么就是小整数值，要么就是长度比较短的字符串，那么Redis就会使用压缩列表来做哈希键的底层实现。”

**“压缩列表是一种为节约内存而开发的顺序型数据结构。”**

压缩列表结构

<img src="https://raw.githubusercontent.com/liuyuede123/image/main/image-20210817165045639.png" alt="image-20210817165045639" style="zoom:50%;" />



压缩列表节点结构

<img src="https://raw.githubusercontent.com/liuyuede123/image/main/image-20210817165131938.png" alt="image-20210817165131938" style="zoom:50%;" />

1. 可以从尾部遍历压缩列表
2. 连续的空间地址，对cpu缓存友好
3. 可能会导致连锁更新，但是概率比较小

**连锁更新**

这时，如果我们将一个长度大于等于254字节的新节点new设置为压缩列表的表头节点，那么new将成为e1的前置节点

<img src="https://raw.githubusercontent.com/liuyuede123/image/main/image-20210817165839675.png" alt="image-20210817165839675" style="zoom:50%;" />


因为e1的previous_entry_length属性仅长1字节，它没办法保存新节点new的长度，所以程序将对压缩列表执行空间重分配操作，并将e1节点的previous_entry_length属性从原来的1字节长扩展为5字节长。

后面套娃

### 哈希表（ditcht）

**c语言中没有内置字典数据结构，redis自己实现**

- redis中对字符串操作`set name liu`，其中`name`就是字典中的键，`liu`就是字典中的值
- 当一个哈希键包含的键值对比较多，又或者键值对中的元素都是比较长的字符串时，Redis就会使用字典作为哈希键的底层实现。`hset`系列命令

<img src="https://raw.githubusercontent.com/liuyuede123/image/main/image-20210817135252611.png" alt="image-20210817135252611" style="zoom:50%;" />

hash表结构定义

```c
typedef struct dictht {
    // 哈希表数组
    dictEntry **table;
    // 哈希表大小
    unsigned long size;
    //哈希表大小掩码，用于计算索引值
    //总是等于size-1
    unsigned long sizemask;
    // 该哈希表已有节点的数量
    unsigned long used;
} dictht;
```

hash表节点的定义

```c
typedef struct dictEntry {
    // 键
    void *key;
    // 值
    union{
        void *val;
        uint64_tu64;
        int64_ts64;
    } v;
    // 指向下个哈希表节点，形成链表。解决hash冲突
    struct dictEntry *next;
} dictEntry;
```

字典的定义

```c
typedef struct dict {
    // 类型特定函数
    dictType *type;
    // 私有数据(类型特定函数的可选参数)
    void *privdata;
    // 哈希表。ht属性是一个包含两个项的数组，数组中的每个项都是一个dictht哈希表，一般情况下，字典只使用ht[0]哈希表，ht[1]哈希表只会在对ht[0]哈希表进行rehash时使用
    dictht ht[2];
    // rehash索引
    //当rehash不在进行时，值为-1
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */
} dict;
```

类型特定函数

```c
typedef struct dictType {
    // 计算哈希值的函数
    unsigned int (*hashFunction)(const void *key);
    // 复制键的函数
    void *(*keyDup)(void *privdata, const void *key);
    // 复制值的函数
    void *(*valDup)(void *privdata, const void *obj);
    // 对比键的函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    // 销毁键的函数
    void (*keyDestructor)(void *privdata, void *key);
    // 销毁值的函数
    void (*valDestructor)(void *privdata, void *obj);
} dictType;
```

**1、hash算法**

```c
# 使用字典设置的哈希函数，计算键key的哈希值
hash = dict->type->hashFunction(key);
#使用哈希表的sizemask属性和哈希值，计算出索引值
#根据情况不同，ht[x],可以是ht[0]或者ht[1]
index = hash & dict->ht[x].sizemask;
```

**2、hash冲突**

redis的hash冲突采用链地址法，不同key分配到的同一个hash表的同一个key上

![image-20210817140317012](https://raw.githubusercontent.com/liuyuede123/image/main/image-20210817140317012.png)

**3、rehash**

> “了让哈希表的负载因子（load factor）维持在一个合理的范围之内，当哈希表保存的键值对数量太多或者太少时，程序需要对哈希表的大小进行相应的扩展或者收缩。”

rehash步骤

1. 为字典的ht[1]hash表分配空间，hash表空间的大小取决于要执行的操作，以及ht[0]当前包含的键值对个数（ht[0].used属性的值）

   ·如果执行的是扩展操作，那么ht[1]的大小为第一个大于等于ht[0].used*2的2 n（2的n次方幂）；
   ·如果执行的是收缩操作，那么ht[1]的大小为第一个小于等于ht[0].used的2 n。

2. 将ht[0]上面的所有键值对，rehash到ht[1]上面。rehash指的是重新计算键的哈希值和索引值，然后将键值对放置到ht[1]哈希表的指定位置上。

3. 当ht[0]包含的所有键值对都迁移到了ht[1]之后（ht[0]变为空表），释放ht[0]，将ht[1]设置为ht[0]，并在ht[1]新创建一个空白哈希表，为下一次rehash做准备。



准备rehash

<img src="https://raw.githubusercontent.com/liuyuede123/image/main/image-20210817141527434.png" alt="image-20210817141527434" style="zoom:50%;" />



对ht[1]分配空间

<img src="https://raw.githubusercontent.com/liuyuede123/image/main/image-20210817142446504.png" alt="image-20210817142446504" style="zoom:50%;" />



h[0]的数据，迁移到h[1]

<img src="https://raw.githubusercontent.com/liuyuede123/image/main/image-20210817142555304.png" alt="image-20210817142555304" style="zoom:50%;" />



释放ht[0]，并将ht[1]重命名为ht[0]，然后为th[1]分配一个空白hash表

<img src="https://raw.githubusercontent.com/liuyuede123/image/main/image-20210817144259377.png" alt="image-20210817144259377" style="zoom:50%;" />

**hash表的扩展与收缩**

1. 服务器目前没有在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于1
2. 服务器目前正在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于5
3. 当哈希表的负载因子小于0.1时，程序自动开始对哈希表执行收缩操作

**注意⚠️**

“根据BGSAVE命令或BGREWRITEAOF命令是否正在执行，服务器执行扩展操作所需的负载因子并不相同，这是因为在执行BGSAVE命令或BGREWRITEAOF命令的过程中，Redis需要创建当前服务器进程的子进程，而大多数操作系统都采用写时复制（copy-on-write）技术来优化子进程的使用效率，所以在子进程存在期间，服务器会提高执行扩展操作所需的负载因子，从而尽可能地避免在子进程存在期间进行哈希表扩展操作，这可以避免不必要的内存写入操作，最大限度地节约内存。”

负载因子计算

```c
# 负载因子= 哈希表已保存节点数量/ 哈希表大小
load_factor = ht[0].used / ht[0].size
```

**4、渐进式hash**

“rehash动作并不是一次性、集中式地完成的，而是分多次、渐进式地完成的”

“如果ht[0]里只保存着四个键值对，那么服务器可以在瞬间就将这些键值对全部rehash到ht[1]；但是，如果哈希表里保存的键值对数量不是四个，而是四百万、四千万甚至四亿个键值对，那么要一次性将这些键值对全部rehash到ht[1]的话，庞大的计算量可能会导致服务器在一段时间内停止服务”

**渐进式rehash步骤**

1. 为ht[1]分配空间，让字典同时持有ht[0]和ht[1]2个hash表
2. 在字典中维持一个索引计数器变量rehashidx，并将它的值设置为0，表示rehash工作正式开始
3. 在rehash进行期间，每次对字典执行添加、删除、查找或者更新操作时，程序除了执行指定的操作以外，还会顺带将ht[0]哈希表在rehashidx索引上的所有键值对rehash到ht[1]，当rehash工作完成之后，程序将rehashidx属性的值增一。
4. 随着字典操作的不断执行，最终在某个时间点上，ht[0]的所有键值对都会被rehash至ht[1]，这时程序将rehashidx属性的值设为-1，表示rehash操作已完成。

> “因为在进行渐进式rehash的过程中，字典会同时使用ht[0]和ht[1]两个哈希表，所以在渐进式rehash进行期间，字典的删除（delete）、查找（find）、更新（update）等操作会在两个哈希表上进行。例如，要在字典里面查找一个键的话，程序会先在ht[0]里面进行查找，如果没找到的话，就会继续到ht[1]里面进行查找，诸如此类。
> 另外，在渐进式rehash执行期间，新添加到字典的键值对一律会被保存到ht[1]里面，而ht[0]则不再进行任何添加操作，这一措施保证了ht[0]包含的键值对数量会只减不增，并随着rehash操作的执行而最终变成空表。”

### 跳表（SkipList）

> “Redis使用跳跃表作为有序集合键的底层实现之一，如果一个有序集合包含的元素数量比较多，又或者有序集合中元素的成员（member）是比较长的字符串时，Redis就会使用跳跃表来作为有序集合键的底层实现。”

**“跳跃表中的所有节点都按分值从小到大来排序。”**

跳表节点结构

```c
typedef struct zskiplistNode {
    // 层
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 跨度
        unsigned int span;
    } level[];
    // 后退指针
    struct zskiplistNode *backward;
    // 分值
    double score;
    // 成员对象
    robj *obj;
} zskiplistNode;
```

跳表结构

```c
typedef struct zskiplist {
    // 表头节点和表尾节点
    struct skiplistNode *header, *tail;
    // 表中节点的数量
    unsigned long length;
    // 表中层数最大的节点的层数
    int level;
} zskiplist;
```



**表头遍历跳表**

<img src="https://raw.githubusercontent.com/liuyuede123/image/main/image-20210817155824651.png" alt="image-20210817155824651" style="zoom:50%;" />



1. 迭代程序首先访问跳跃表的第一个节点（表头），然后从第四层的前进指针移动到表中的第二个节点。
2. 在第二个节点时，程序沿着第二层的前进指针移动到表中的第三个节点。
3. 在第三个节点时，程序同样沿着第二层的前进指针移动到表中的第四个节点。
4. 当程序再次沿着第四个节点的前进指针移动时，它碰到一个NULL，程序知道这时已经到达了跳跃表的表尾，于是结束这次遍历。


**跨度**

1. 两个节点的跨度越大，它们的距离就越远
2. 指向NULL的所有前进指针的跨度都为0，因为它们没有连向任何节点。
3. 跨度是用来计算排位的，从头遍历，经历的跨度之和就是排位。

**后退指针**

“跟可以一次跳过多个节点的前进指针不同，因为每个节点只有一个后退指针，所以每次只能后退至前一个节点”

<img src="https://raw.githubusercontent.com/liuyuede123/image/main/image-20210817160529334.png" alt="image-20210817160529334" style="zoom:50%;" />



**分值和成员对象**

分值可以重复，成员对象必须唯一。分值相同情况下，按照成员对象字典序排序。

### 整数集合（IntSet）

> “整数集合（intset）是集合键的底层实现之一，当一个集合只包含整数值元素，并且这个集合的元素数量不多时，Redis就会使用整数集合作为集合键的底层实现。”

整数集合结构

```c
typedef struct intset {
    // 编码方式
    uint32_t encoding;
    // 集合包含的元素数量
    uint32_t length;
    // 保存元素的数组
    int8_t contents[];
} intset;
```

<img src="https://raw.githubusercontent.com/liuyuede123/image/main/image-20210817161722299.png" alt="image-20210817161722299" style="zoom:50%;" />



**升级**

1. “根据新元素的类型，扩展整数集合底层数组的空间大小，并为新元素分配空间。”
2. “将底层数组现有的所有元素都转换成与新元素相同的类型，并将类型转换后的元素放置到正确的位上，而且在放置元素的过程中，需要继续维持底层数组的有序性质不变。”
3. “将新元素添加到底层数组里面。”


### 快表（quicklist）



### 对象

> “Redis并没有直接使用这些数据结构来实现键值对数据库，而是基于这些数据结构创建了一个对象系统，这个系统包含字符串对象、列表对象、哈希对象、集合对象和有序集合对象这五种类型的对象，每种对象都用到了至少一种我们前面所介绍的数据结构。”

**1、对象的类型与编码**

“Redis使用对象来表示数据库中的键和值，每次当我们在Redis的数据库中新创建一个键值对时，我们至少会创建两个对象，一个对象用作键值对的键（键对象），另一个对象用作键值对的值（值对象）。”

```c
typedef struct redisObject {
    // 类型
    unsigned type:4;
    // 编码
    unsigned encoding:4;
    // 指向底层实现数据结构的指针
    void *ptr;
    // ...
} robj;
```

“TYPE命令的实现方式也与此类似，当我们对一个数据库键执行TYPE命令时，命令返回的结果为数据库键对应的值对象的类型，而不是键对象的类型”



每种类型都至少用了2中不同的编码

![image-20210817172156444](https://raw.githubusercontent.com/liuyuede123/image/main/image-20210817172156444.png)



**2、字符串对象**

“字符串对象的编码可以是int、raw或者embstr。”

2.1、“如果一个字符串对象保存的是整数值，并且这个整数值可以用long类型来表示，那么字符串对象会将整数值保存在字符串对象结构的ptr属性里面（将void*转换成long），并将字符串对象的编码设置为int。”

2.2、“如果字符串对象保存的是一个字符串值，并且这个字符串值的长度大于32字节，那么字符串对象将使用一个简单动态字符串（SDS）来保存这个字符串值，并将对象的编码设置为raw。”。会调用2次系统分配，分别创建redisObject和sdshdr结构。

2.3、“如果字符串对象保存的是一个字符串值，并且这个字符串值的长度小于等于32字节，那么字符串对象将使用embstr编码的方式来保存这个字符串值。”。只会调用1次系统分配，在一块连续的空间依次包含redisObject和sdshdr结构。

2.4、编码的转换

1. int改为字符串的话，编码会从int转为raw
2. embstr是只读的，一旦修改就会转化为raw编码

**3、列表对象**

![image-20210817175710871](https://raw.githubusercontent.com/liuyuede123/image/main/image-20210817175710871.png)

1. 列表对象会使用压缩列表和双端链表2中编码结构，其中双端链表节点中会嵌套字符串对象。字符串对象是Redis五种类型的对象中唯一一种会被其他四种类型对象嵌套的对象。
2. 列表对象保存的所有字符串元素的长度都小于64字节，列表对象保存的元素数量小于512个。不能满足这2个条件的列表对象需要使用linkedlist编码。

**4、hash对象**

1. 哈希对象的编码可以是ziplist或者hashtable
2. 哈希对象保存的所有键值对的键和值的字符串长度都小于64字节；哈希对象保存的键值对数量小于512个；不能满足这两个条件的哈希对象需要使用hashtable编码。

**5、集合对象**

1. 集合对象的编码可以是intset或者hashtable
2. 集合对象保存的所有元素都是整数值；集合对象保存的元素数量不超过512个。不能满足这两个条件的集合对象需要使用hashtable编码。

**6、有序集合对象**

1. 有序集合的编码可以是ziplist或者skiplist
2. 跳表配合字典使用，是为了查找对应成员的分值，时间复杂度为1。跳表查找某个成员logn
3. 有序集合保存的元素数量小于128个；有序集合保存的所有元素成员的长度都小于64字节；不能满足以上两个条件的有序集合对象将使用skiplist编码。

**7、内存回收**

```c
typedef struct redisObject {
    // ...
    // 引用计数
    int refcount;
    // ...
} robj;
```

1. “在创建一个新对象时，引用计数的值会被初始化为1；”
2. “当对象被一个新程序使用时，它的引用计数值会被增一；”
3. “当对象不再被一个程序使用时，它的引用计数值会被减一；”
4. “当对象的引用计数值变为0时，对象所占用的内存会被释放。”

**8、对象共享**

1. 将数据库键的值指针指向一个现有的值对象，refcount加一
2. redis不共享字符串的对象，因为可能判断是否相等会消耗更多cpu的性能

**9、对象的空转时长**

```c
typedef struct redisObject {
    // ...
    unsigned lru:22;
    // ...
} robj;
```



1. “当服务器设置了maxmemory选项，并且服务器用于回收内存的算法为volatile-lru或者allkeys-lru，那么当服务器占用的内存数超过了maxmemory选项所设置的上限值时，空转时长较高的那部分键会优先被服务器释放，从而回收内存。”



## 三、Redis网络模型

### io多路复用

### 文件事件处理器

## 四、Redis持久化

### rdb持久化

> “RDB持久化功能所生成的RDB文件是一个经过压缩的二进制文件，通过该文件可以还原生成RDB文件时的数据库状态”

**1、bgSave保存条件**

- “服务器在900秒之内，对数据库进行了至少1次修改。”
- “服务器在300秒之内，对数据库进行了至少10次修改。”
- “服务器在60秒之内，对数据库进行了至少10000次修改。”

自动执行bgSave命令结构，一般服务会每隔100毫秒检查一次

根据dirty执行次数，和lastsave上次执行save的时间戳

![image-20210818102126111](https://raw.githubusercontent.com/liuyuede123/image/main/image-20210818102126111.png)

**2、rdb文件结构**

| REDIS | db_version | databases | EOF  | check_sum |
| ----- | ---------- | --------- | ---- | --------- |

1. 以REDIS符号开头，5字节
2. db_version长度为4字节，它的值是一个字符串表示的整数，这个整数记录了RDB文件的版本号，比如"0006"就代表RDB文件的版本为第六版。
3. databases部分包含着零个或任意多个数据库，以及各个数据库中的键值对数据
4. EOF常量的长度为1字节，这个常量标志着RDB文件正文内容的结束，当读入程序遇到这个值的时候，它知道所有数据库的所有键值对都已经载入完毕了。
5. check_sum是一个8字节长的无符号整数，保存着一个校验和，这个校验和是程序通过对REDIS、db_version、databases、EOF四个部分的内容进行计算得出的。服务器在载入RDB文件时，会将载入数据所计算出的校验和与check_sum所记录的校验和进行对比，以此来检查RDB文件是否有出错或者损坏的情况出现。


### aof持久化

> 直接保存redis文本命令，只会同步写命令

**1、aof持久化的实现**

“AOF持久化功能的实现可以分为命令追加（append）、文件写入、文件同步（sync）三个步骤。”

1. 当AOF持久化功能处于打开状态时，服务器在执行完一个写命令之后，会以协议格式将被执行的写命令追加到服务器状态的aof_buf缓冲区的末尾
2. 在服务器每次结束一个事件循环之前，它都会调用flushAppendOnlyFile函数，考虑是否需要将aof_buf缓冲区中的内容写入和保存到AOF文件里面
3. 会有2个步骤，写入只是每个事件循环调用write将aof_buffer里面的数据写到操作系统缓冲区，同步就是把操作系统缓冲区的内容同步到aof文件

**2、aof的载入与数据还原**

1. redis构造一个伪客户端用来执行aof中的命令，知道aof文件中的命令都执行完。

**3、aof重写**

1. 比如aof中有6条写命令，aof重写之后直接读取数据库重新生成一条写命令

**4、aof后台重写**

1. “子进程进行AOF重写期间，服务器进程（父进程）可以继续处理命令请求”
2. “子进程带有服务器进程的数据副本，使用子进程而不是线程，可以在避免使用锁的情况下，保证数据的安全性。”
3. 同步过程主进程产生新的写请求，子进程如何同步？这个时候主进程会将写命令同时发送给aof缓冲区和aof重写缓冲区。当子进程完成重写工作之后，会向父进程发送一个信号，父进程受到信号会调用一个信号处理函数。“将AOF重写缓冲区中的所有内容写入到新AOF文件中，这时新AOF文件所保存的数据库状态将和服务器当前的数据库状态一致。”，“对新的AOF文件进行改名，原子地（atomic）覆盖现有的AOF文件，完成新旧两个AOF文件的替换。”，“在整个AOF后台重写过程中，只有信号处理函数执行时会对服务器进程（父进程）造成阻塞，在其他时候，AOF后台重写都不会阻塞父进程，这将AOF重写对服务器性能造成的影响降到了最低。”



## 五、Redis缓存淘汰策略

### 设置键的过期时间

**1、设置过期时间**

1. “EXPIRE、PEXPIRE、EXPIREAT三个命令都是使用PEXPIREAT命令来实现的：无论客户端执行的是以上四个命令中的哪一个，经过转换之后，最终的执行效果都和执行PEXPIREAT命令一样。”

**2、保存过期时间**

1. “redisDb结构的expires字典保存了数据库中所有键的过期时间，我们称这个字典为过期字典”
2. “过期字典的键是一个指针，这个指针指向键空间中的某个键对象（也即是某个数据库键）”
3. “过期字典的值是一个long long类型的整数，这个整数保存了键所指向的数据库键的过期时间——一个毫秒精度的UNIX时间戳。”

```c
typedef struct redisDb {
    // ...
    // 过期字典，保存着键的过期时间
    dict *expires;
    // ...
} redisDb;
```

**3、过期键的判定**

1. “检查给定键是否存在于过期字典：如果存在，那么取得键的过期时间。”
2. “检查当前UNIX时间戳是否大于键的过期时间：如果是的话，那么键已经过期；否则的话，键未过期。”

**4、过期键的删除策略**

1、“定时删除：在设置键的过期时间的同时，创建一个定时器（timer），让定时器在键的过期时间来临时，立即执行对键的删除操作。”

优点：“定时删除策略可以保证过期键会尽可能快地被删除，并释放过期键所占用的内存”

缺点：“在过期键比较多的情况下，删除过期键这一行为可能会占用相当一部分CPU时间，在内存不紧张但是CPU时间非常紧张的情况下，将CPU时间用在删除和当前任务无关的过期键上，无疑会对服务器的响应时间和吞吐量造成影响。”

2、“惰性删除：放任键过期不管，但是每次从键空间中获取键时，都检查取得的键是否过期，如果过期的话，就删除该键；如果没有过期，就返回该键。”

优点：“惰性删除策略对CPU时间来说是最友好的：程序只会在取出键时才对键进行过期检查，这可以保证删除过期键的操作只会在非做不可的情况下进行，并且删除的目标仅限于当前处理的键，这个策略不会在删除其他无关的过期键上花费任何CPU时间”

缺点：“惰性删除策略的缺点是，它对内存是最不友好的：如果一个键已经过期，而这个键又仍然保留在数据库中，那么只要这个过期键不被删除，它所占用的内存就不会释放。”

3、“定期删除：每隔一段时间，程序就对数据库进行一次检查，删除里面的过期键。至于要删除多少过期键，以及要检查多少个数据库，则由算法决定。”

优点：“定期删除策略每隔一段时间执行一次删除过期键操作，并通过限制删除操作执行的时长和频率来减少删除操作对CPU时间的影响”

定期删除策略的难点是确定删除操作执行的时长和频率：“函数每次运行时，都从一定数量的数据库中取出一定数量的随机键进行检查，并删除其中的过期键”

**5、AOF、RDB和复制功能对过期键的处理**

1. “在执行SAVE命令或者BGSAVE命令创建一个新的RDB文件时，程序会对数据库中的键进行检查，已过期的键不会被保存到新创建的RDB文件中”
2. 主服务器载入rdb文件会过滤掉过期键。从服务载入rdb文件依然会加载过期键，不做处理，但是主服务器同步到从服务器时，会删除掉过期键。
3. 如果数据库中的键已经过期，但是还没有惰性删除或者定期删除，aof文件保持之前的记录，但是当惰性删除或者定期删除之后，会往aof文件中追加一条删除的命令。
4. aof重写的过程会对数据库中的键进行检查，已经过期的键不会被保存到重写后aof文件中
5. 从服务器不会对过期键做判断删除处理，只有主服务器删除键，同步del命令到从服务器才会删除。所以即使键过期了，从服务器依然会查出来返回给客户端。

## 六、Redis主从复制

> slaveof 127.0.0.1 6379     将6380服务器设置为6379的从服务器

### 同步

“PSYNC命令具有完整重同步（full resynchronization）和部分重同步（partial resynchronization）两种模式：”

1. “完整重同步用于处理初次复制情况：完整重同步的执行步骤和SYNC命令的执行步骤基本一样，它们都是通过让主服务器创建并发送RDB文件，以及向从服务器发送保存在缓冲区里面的写命令来进行同步。”，“主服务器将记录在缓冲区里面的所有写命令发送给从服务器，从服务器执行这些写命令，将自己的数据库状态更新至主服务器数据库当前所处的状态。”
2. “部分重同步则用于处理断线后重复制情况：当从服务器在断线后重新连接主服务器时，如果条件允许，主服务器可以将主从服务器连接断开期间执行的写命令发送给从服务器，从服务器只要接收并执行这些写命令，就可以将数据库更新至主服务器当前所处的状态。”

**部分重同步的实现：**

1. 复制偏移量：“主服务器每次向从服务器传播N个字节的数据时，就将自己的复制偏移量的值加上N。”，“从服务器每次收到主服务器传播来的N个字节的数据时，就将自己的复制偏移量的值加上N。”，当主服务器的偏移量和从服务器的偏移量不一致时，说明需要部分重同步。
2. 复制积压缓冲区：“复制积压缓冲区是由主服务器维护的一个固定长度（fixed-size）先进先出（FIFO）队列，默认大小为1MB。”。当主服务器进行命令传播时，不仅要同步到从服务器，还会将命令放到复制积压缓冲区队列，同时会累加偏移量。
3. 从服务器重连之后，会把offset传给主服务器，主服务器判断offset+1开始的数据是否在复制积压缓冲区里面
4. 如果在里面，开始部分重同步
5. 如果不在里面，开始完整同步
6. 每个redis服务都有一个运行ID，从服务器绑定主服务器的时候会保存主服务器的运行ID。当短线重连后，从服务器会把保存的主服务器运行ID发送给当前连接的主服务器
7. 如果两者相等，可以执行部分重同步
8. 如果两者不想等，则执行完整重同步

### 命令传播

“主服务器会将自己执行的写命令，也即是造成主从服务器不一致的那条写命令，发送给从服务器执行，当从服务器执行了相同的写命令之后，主从服务器将再次回到一致状态。”

### 心跳检测

“在命令传播阶段，从服务器默认会以每秒一次的频率，向主服务器发送命令：”

1. “主从服务器可以通过发送和接收REPLCONF ACK命令来检查两者之间的网络连接是否正常：如果主服务器超过一秒钟没有收到从服务器发来的REPLCONF ACK命令，那么主服务器就知道主从服务器之间的连接出现问题了。”
2. “Redis的min-slaves-to-write和min-slaves-max-lag两个选项可以防止主服务器在不安全的情况下执行写命令。”，“那么在从服务器的数量少于3个，或者三个从服务器的延迟（lag）值都大于或等于10秒时，主服务器将拒绝执行写命令，这里的延迟值就是上面提到的INFO replication命令的lag值。”
3. “如果因为网络故障，主服务器传播给从服务器的写命令在半路丢失，那么当从服务器向主服务器发送REPLCONF ACK命令时，主服务器将发觉从服务器当前的复制偏移量少于自己的复制偏移量，然后主服务器就会根据从服务器提交的复制偏移量，在复制积压缓冲区里面找到从服务器缺少的数据，并将这些数据重新发送给从服务器。”



## 七、Sentinel

<img src="https://raw.githubusercontent.com/liuyuede123/image/main/image-20210818200257063.png" alt="image-20210818200257063" style="zoom:50%;" />



“当server1的下线时长超过用户设定的下线时长上限时，Sentinel系统就会对server1执行故障转移操作：”

1. “首先，Sentinel系统会挑选server1属下的其中一个从服务器，并将这个被选中的从服务器升级为新的主服务器。”
2. “之后，Sentinel系统会向server1属下的所有从服务器发送新的复制指令，让它们成为新的主服务器的从服务器，当所有从服务器都开始复制新的主服务器时，故障转移操作执行完毕。”
3. “另外，Sentinel还会继续监视已下线的server1，并在它重新上线时，将它设置为新的主服务器的从服务器。”

### 启动sentinel

1. “Sentinel本质上只是一个运行在特殊模式下的Redis服务器，所以启动Sentinel的第一步，就是初始化一个普通的Redis服务器”

2. “启动Sentinel的第二个步骤就是将一部分普通Redis服务器使用的代码替换成Sentinel专用代码”

3. “接下来，服务器会初始化一个sentinel.c/sentinelState结构（后面简称“Sentinel状态”），这个结构保存了服务器中所有和Sentinel功能有关的状态（服务器的一般状态仍然由redis.h/redisServer结构保存）”

4. “Sentinel状态中的masters字典记录了所有被Sentinel监视的主服务器的相关信息”

   <img src="https://raw.githubusercontent.com/liuyuede123/image/main/image-20210818201421590.png" alt="image-20210818201421590" style="zoom:50%;" />

5. “初始化Sentinel的最后一步是创建连向被监视主服务器的网络连接，Sentinel将成为主服务器的客户端，它可以向主服务器发送命令，并从命令回复中获取相关的信息”

6. “另一个是订阅连接，这个连接专门用于订阅主服务器的__sentinel__:hello频道。”

### 获取主服务器信息

“Sentinel默认会以每十秒一次的频率，通过命令连接向被监视的主服务器发送INFO命令，并通过分析INFO命令的回复来获取主服务器的当前信息。”

<img src="https://raw.githubusercontent.com/liuyuede123/image/main/image-20210818201857109.png" alt="image-20210818201857109" style="zoom:50%;" />

### 获取从服务器信息

“当Sentinel发现主服务器有新的从服务器出现时，Sentinel除了会为这个新的从服务器创建相应的实例结构之外，Sentinel还会创建连接到从服务器的命令连接和订阅连接。”

### 向主服务器和从服务器发送信息

“在默认情况下，Sentinel会以每两秒一次的频率，通过命令连接向所有被监视的主服务器和从服务器发送命令”

### 接收来自主服务器和从服务器的频道信息

“当Sentinel与一个主服务器或者从服务器建立起订阅连接之后，Sentinel就会通过订阅连接，向服务器发送命令”

“Sentinel对__sentinel__:hello频道的订阅会一直持续到Sentinel与服务器的连接断开为止。”

### 多sentinel

“对于监视同一个服务器的多个Sentinel来说，一个Sentinel发送的信息会被其他Sentinel接收到，这些信息会被用于更新其他Sentinel对发送信息Sentinel的认知，也会被用于更新其他Sentinel对被监视服务器的认知”

<img src="https://raw.githubusercontent.com/liuyuede123/image/main/image-20210818202432118.png" alt="image-20210818202432118" style="zoom:50%;" />

### 创建连向其他Sentinel的命令连接

“当Sentinel通过频道信息发现一个新的Sentinel时，它不仅会为新Sentinel在sentinels字典中创建相应的实例结构，还会创建一个连向新Sentinel的命令连接，而新Sentinel也同样会创建连向这个Sentinel的命令连接，最终监视同一主服务器的多个Sentinel将形成相互连接的网络：Sentinel A有连向Sentinel B的命令连接，而Sentinel B也有连向Sentinel A的命令连接。”

<img src="https://raw.githubusercontent.com/liuyuede123/image/main/image-20210818202552513.png" alt="image-20210818202552513" style="zoom:50%;" />



“Sentinel在连接主服务器或者从服务器时，会同时创建命令连接和订阅连接，但是在连接其他Sentinel时，却只会创建命令连接，而不创建订阅连接。这是因为Sentinel需要通过接收主服务器或者从服务器发来的频道信息来发现未知的新Sentinel，所以才需要建立订阅连接，而相互已知的Sentinel只要使用命令连接来进行通信就足够了。”

### 检测主观下线状态

“在默认情况下，Sentinel会以每秒一次的频率向所有与它创建了命令连接的实例（包括主服务器、从服务器、其他Sentinel在内）发送PING命令，并通过实例返回的PING命令回复来判断实例是否在线。”

### 检测客观下线状态

“当Sentinel将一个主服务器判断为主观下线之后，为了确认这个主服务器是否真的下线了，它会向同样监视这一主服务器的其他Sentinel进行询问，看它们是否也认为主服务器已经进入了下线状态（可以是主观下线或者客观下线）。当Sentinel从其他Sentinel那里接收到足够数量的已下线判断之后，Sentinel就会将从服务器判定为客观下线，并对主服务器执行故障转移操作。”

### 选举领头sentinel

“当一个主服务器被判断为客观下线时，监视这个下线主服务器的各个Sentinel会进行协商，选举出一个领头Sentinel，并由领头Sentinel对下线主服务器执行故障转移操作。”

1. “所有在线的Sentinel都有被选为领头Sentinel的资格，换句话说，监视同一个主服务器的多个在线Sentinel中的任意一个都有可能成为领头Sentinel”
2. “每次进行领头Sentinel选举之后，不论选举是否成功，所有Sentinel的配置纪元（configuration epoch）的值都会自增一次。配置纪元实际上就是一个计数器，并没有什么特别的。”
3. “在一个配置纪元里面，所有Sentinel都有一次将某个Sentinel设置为局部领头Sentinel的机会，并且局部领头一旦设置，在这个配置纪元里面就不能再更改。”
4. “每个发现主服务器进入客观下线的Sentinel都会要求其他Sentinel将自己设置为局部领头Sentinel”
5. “当一个Sentinel（源Sentinel）向另一个Sentinel（目标Sentinel）发送SENTINEL is-master-down-by-addr命令，并且命令中的runid参数不是*符号而是源Sentinel的运行ID时，这表示源Sentinel要求目标Sentinel将前者设置为后者的局部领头Sentinel。”
6. “Sentinel设置局部领头Sentinel的规则是先到先得：最先向目标Sentinel发送设置要求的源Sentinel将成为目标Sentinel的局部领头Sentinel，而之后接收到的所有设置要求都会被目标Sentinel拒绝。”
7. “目标Sentinel在接收到SENTINEL is-master-down-by-addr命令之后，将向源Sentinel返回一条命令回复，回复中的leader_runid参数和leader_epoch参数分别记录了目标Sentinel的局部领头Sentinel的运行ID和配置纪元。”
8. “源Sentinel在接收到目标Sentinel返回的命令回复之后，会检查回复中leader_epoch参数的值和自己的配置纪元是否相同，如果相同的话，那么源Sentinel继续取出回复中的leader_runid参数，如果leader_runid参数的值和源Sentinel的运行ID一致，那么表示目标Sentinel将源Sentinel设置成了局部领头Sentinel。”
9. “如果有某个Sentinel被半数以上的Sentinel设置成了局部领头Sentinel，那么这个Sentinel成为领头Sentinel。”
10. “因为领头Sentinel的产生需要半数以上Sentinel的支持，并且每个Sentinel在每个配置纪元里面只能设置一次局部领头Sentinel，所以在一个配置纪元里面，只会出现一个领头Sentinel。”
11. “如果在给定时限内，没有一个Sentinel被选举为领头Sentinel，那么各个Sentinel将在一段时间之后再次进行选举，直到选出领头Sentinel为止。”

### 故障转移

“在选举产生出领头Sentinel之后，领头Sentinel将对已下线的主服务器执行故障转移操作，该操作包含以下三个步骤：”

1. “在已下线主服务器属下的所有从服务器里面，挑选出一个从服务器，并将其转换为主服务器。”
   1. “删除列表中所有处于下线或者断线状态的从服务器，这可以保证列表中剩余的从服务器都是正常在线的。”
   2. “删除列表中所有最近五秒内没有回复过领头Sentinel的INFO命令的从服务器，这可以保证列表中剩余的从服务器都是最近成功进行过通信的。”
   3. “删除所有与已下线主服务器连接断开超过down-after-milliseconds*10毫秒的从服务器：down-after-milliseconds选项指定了判断主服务器下线所需的时间，而删除断开时长超过down-after-milliseconds*10毫秒的从服务器，则可以保证列表中剩余的从服务器都没有过早地与主服务器断开连接，换句话说，列表中剩余的从服务器保存的数据都是比较新的。”
   4. “之后，领头Sentinel将根据从服务器的优先级，对列表中剩余的从服务器进行排序，并选出其中优先级最高的从服务器。”

2. “让已下线主服务器属下的所有从服务器改为复制新的主服务器。”
3. “将已下线主服务器设置为新的主服务器的从服务器，当这个旧的主服务器重新上线时，它就会成为新的主服务器的从服务器。”



## 八、Redis集群

hash环

