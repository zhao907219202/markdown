#### 链表
废话不多说，今天继续学习Redis的基本数据结构——链表和哈希表。
先看一个例子，以下展示的integers列表键包含了从1到1024共一千零二十四个整数：

```
redis-> LLEN integers
(integer) 1024
redis-> LRANGE integers 0 5
1)"1"
2)"2"
3)"3"
4)"4"
5)"5"
```
integers 列表键的底层实现就是一个链表，链表中的每个节点都保存了一个整数值。链表在Redis使用十分广泛，发布和订阅、慢查询、监视器等功能都用到了链表，Redis服务器本身用链表保存了多个客户端的状态信息,以及使用链表来构建客户端输出缓冲区(output buffer)。

Redis的链表和数据结构中的双向链表的设计是一致的。每个链表节点的声明在adlist.h/listNode结构中：
```
typedef struct listNode{
    struct listNode * prev;
    struct listNode * next;
    void * value;
} listNode;
```
节点包含了前驱、后继和值。
```
typedef struct list{
    listNode * head;
    listNode * tail;
    unsigned long len;
    void * (*dup) (void * ptr);
    void (*free) (void * ptr);
    int (*match) (void * ptr,void * key);
} list;
```
表头、表尾、长度计数器。后面三个为函数指针，分别用于复制节点所保存的值、释放节点所保存的值和对比链表节点所保存的值是否和另一个输入值是否相等。链表相关的操作也相对比较简单，这里不多做赘述，下面继续看一个很重要的数据结构——哈希表。

#### 字典
Redis的字典使用哈希表作为底层实现，一个哈希表里面可以有多个哈希表节点，而每个哈希表节点就保存了字典中的一个键值对。实际上，Redis数据库底层也是用哈希表来存储键值对的。先来看看字典这个数据结构的定义。
```
typedef struct dictht {
    dictEntry ** table;
    unsigned long size;
    unsigned long sizemask;
    unsugned long used;
}
```
table属性是一个数组，数组中的每一个元素都是一个指向dict.h/dictEntry结构的指针，每个dictEntry结构都保存着一个键值对。size属性记录了哈希表的大小，依旧是table数组的大小，而used属性则记录了哈希表目前已有节点（键值对）的数量。sizemask属性的值总是等于size-1，这个属性是用来和哈希值一起决定一个键应该被放到table数组的哪个索引上面。

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-dict-1-20180503.png)

上图展示了table的存储信息。table 相当于一个二维指针，每个元素都是一个一维数组，数组元素为dictEntry。下面再来看看dictEntry的结构声明：
```
typedef struct dictEntry{
    void * key;
    union{
        void * val;
        uint64_t u64;
        int64_t s64;
    } v;
    struct dictEntry * next;
} dictEntry;

```
key属性保存着键值对的键，而v属性则保存着键值对中的值，其中键值对的值可以是一个指针，或者是一个uint64_t整数，又或者是一个 int64_t整数。next属性是指向另一个哈希表节点的指针，这个指针可以将多个哈希值相同的键值对连接在一起，以此来解决键冲突（collision）的问题。这点类似java中的HashMap，熟悉java的coder这里应该不会觉得陌生。
```
typedef struct dict {
    dictType * type;
    void * privdata;
    dictht ht[2];
    int rehashidx;
}
```
type属性和privdata属性是针对不同类型的键值对，为创建多态字典而设置的。type属性是一个指向dictType结构的指针，每个dictType结构保存了一簇用于操作特定类型键值对的函数，这里就不展开说了。privdata属性则保存了需要传给那些类型特定函数的可选参数。ht是含有两个项的数组，数组中每个元素都是一个dictht哈希表，一般情况下，字典使用ht[0],ht[1]只会在对ht[0]rehash的时候使用。除了ht[1]外，rehashidx也与rehash有关，它记录了rehash的进度，如果没有进行rehash，则为-1。图为普通状态下的字典：
![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-dict-0-20180502.png)

#### 哈希算法
当要将一个新的键值添加到字典里面时，程序需要先根据键值对的键计算出哈希值和索引值，然后再根据索引值，将包含新键值对的哈希表节点放到哈希表数组的指定索引上面。Redis计算哈希值和索引值的方法如下:
```
//计算哈希值
hash = dict->type->hashFunction(key);
//使用哈希表的sizemask属性和哈希值计算索引值
//根据情况不同，ht[x]可以是ht[0]或者ht[1]
index = hash & dict -> ht[x].sizemask;
```
字典被用作数据库的底层实现或者哈希键的实现时，Redis使用MurmurHash2算法来计算键的哈希值。

#### 解决键冲突
当有两个或以上数量的键被分配到了哈希表数组的同一个索引上时，就发生了键冲突。Redis的哈希表使用链地址法（separate chaining）来解决键冲突，每个哈希节点都有一个next指针，多个哈希表节点可以用next指针构成一个单向链表，被分配到同一个索引上的多个节点可以用这个单向链表连接起来，这就解决了键冲突的问题。因为dictEntry节点组成的链表没有指向链表表尾的指针，所以为了速度考虑，程序总是将新节点添加到链表的表头位置(复杂度为O(1))，排在其他已有节点的前面。

#### rehash
随着操作的不断执行，哈希表保存的键值对会逐渐地增多或减少，为了让哈希表的负载因子（load factor）维持在一个合理的范围之内，当哈希表保存的键值对数量太多或者太少时，程序需要对哈希表的大小进行相应的扩展或者收缩。扩展和收缩哈希表的工作可以通过执行rehash操作来完成，Redis对字典的哈希表执行rehash的步骤如下：
* 为字典ht[1]哈希表分配空间，这个哈希表的大小取决于要执行的操作，以及ht[0]当前包含的键值对数量（也就是ht[0].used属性的值）
    * 如果执行的是扩展操作，那么ht[1]的大小为第一个大于等于ht[0].used * 2的2次幂
    * 如果执行的是收缩操作，那么ht[1]的大小为第一个大于等于ht[0].used 的2次幂
* 将保存在ht[0]中的所有键值对rehash到ht[1]上面：rehash指的是重新计算键的哈希值和索引值，然后将键值对放置到ht[1]哈希表的指定位置上
* 当ht[0]包含的所有键值对都迁移到ht[1]之后（ht[0]变为空表），释放ht[0]，将ht[1]设置为ht[0]，并在ht[1]新创建一个空白哈希表，为下一次rehash做准备。

以下条件中的任意一个满足时，程序会自动开始对哈希表执行扩展操作：
+ 服务器目前没有在执行BASAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于1
+ 服务器目前正在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于5

负载因子公式 load_factor = ht[0].used / ht[0].size

根据BGSAVE命令或BGREWRITEAOF命令是否正在执行，服务器执行扩展操作所需的负载因子并不相同，这是因为在执行BGSAVE命令或者BGREWRITEAOF命令的过程中，Redis需要创建当前服务器进程的子进程，而大多数操作系统都采用写时复制（copy-on-write）来优化子进程的使用效率，所以在子进程存在期间，服务器会提高负载因子的阈值，从而避免在子进程存在期间进行扩展操作，从而节约内存。

另一方面，负载因子小于0.1时，程序开始对哈希表执行收缩操作。

#### 渐进式rehash
rehash的动作并不是一次性、集中式的完成的，而是分多次、渐进式的完成的，原因肯定是数据量大时，会导致服务器在一段时间内停止服务。详细步骤如下：
+ 为ht[1]分配空间，让字典同时持有ht[0] 和 ht[1]两个哈希表
+ 在字典中维持一个索引计数器rehashidx，并将它的值设置为0，标识rehash工作正式开始
+ 在rehash期间，每次对字典执行添加、删除、查找或者更新操作时，程序除了执行指定的操作之外，还会顺带将ht[0]哈希表在rehashidx索引上的所有键值对rehash到ht[1]，完成后将rehashidx的值增一
+ 随着字典操作不断执行，最终在某个时间点，全部rehash完毕，rehashidx设置为-1，表示完成

在渐进式rehash期间，删改查会在两个ht上操作，例如要查找某个键值，现在ht[0]中查找，没找到再去ht[1]中查找。新添加的键值对会直接添加到ht[1]中，意图和目的十分明显，ht[0]只减不增，最终为空表释放。


