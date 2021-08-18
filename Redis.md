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

## 四、Redis持久化

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
4. aof重写的过程会对数据库中的键进行检查，已经过期的键不会被保存到aof文件中

## 六、Redis主从复制

## 七、Redis集群

hash环