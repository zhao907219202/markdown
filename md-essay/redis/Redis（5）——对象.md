前面说了Redis使用的主要数据结构，但是Redis并没有直接使用这些数据结构来实现，而是基于这些数据结构创建了一个对象系统，包含字符串对象、列表对象、哈希对象、集合对象和有序集合对象。除此之外，redis还有基于引用计数器的内存回收机制，以及对象共享机制。

#### 对象的类型和编码
```
typedef struct redisObject{
    unsigned type；
    unsigned encoding；
    void * ptr;
}
```
type属性记录的对象类型，redis的键都是字符串对象，而值可以下面表中的任意一个对象。Type命令可以查看某个redis对象的类型，下图也列举了所有type命令的输出。
```
reids-> SET msg "hello world"
OK
redis-> TYPE msg
string
redis-> RPUSH numbers 1 3 5
(integer) 3
redis-> TYPE numbers 
list
```

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-object-0-20180520.png)

##### 编码和底层实现
对象的ptr指针指向对象的底层实现的额数据结构，而这些数据结构由对象的encoding属性决定。每种类型的对象都至少使用了两种不同的编码，其对应关系如下图所示:

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-object-1-20180520.png)
使用OBJECT ENCODING 命令可以查看一个数据库键的值对象编码：
```
redis-> SET msg "hello world"
OK
redis-> OBJECT ENCODING msg
"embstr"
redis-> SADD numbers 1 3 5
(integer) 3
redis-> OBJECT ENCODING numbers
"intset"
redis-> SADD numbers "seven"
(integer) 1
redis-> OBJECT ENCODING numbers
"hashtable"
```

对象所使用的数据结构| 编码常量 |OBJECT ENCODING 命令输出
---|---|----
整数 | REDIS_ENCODING_INT | "int"
embstr编码的简单动态字符串 | REDIS_ENCODING_EMBSTR|"embstr"
简单动态字符串|REDIS_ENCODING_RAW|"raw"
字典|REDIS_ENCODING_HT|"hashtable"
双端链表|REDIS_ENCODING_LINKEDLIST|"linkedlist"
压缩列表|REDIS_ENCODING_ZIPLIST|"ziplist"
整数集合|REDIS_ENCODING_INTSET|"intset"
跳跃表和字典|REDIS_ENCODING_SKIPLIST|"skiplist"

下面我们分别介绍各个对象
#### 字符串对象
字符串对象的编码可以是int，raw或者embstr。
字符串对象保存的值是整数，那么redisObject中的ptr属性将作为long存储该值（void* 转换成long），编码设置为int。
embstr 用于存储字符串长度小于32字节的情况（SET 命令时根据值的大小redis选择相应的编码格式）。embstr是对raw的一种优化，raw需要调用两次内存分配函数分别创建redisObject 和 sdshdr，但是embstr调用一次创建连续的内存空间，包含上面说的两个结构。浮点类型也是通过字符串对象存储的，对浮点类型的操作有内部执行，操作完成后再转成字符串存储。

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-object-2-20180520.png)

```
redis-> SET pi 3.14
OK
redis-> OBJECT ENCODING pi
"embstr"
redis-> INCRBYFLOAT pi 2.0
"5.14"
redis-> OBJECT ENCODING pi
"embstr"
```
int和embstr 编码的字符串在条件满足的情况下，会被转换成raw。并且embstr是只读不可修改的，对其修改命令（APPEND、SETRANGE）时只能转换成raw编码的字符串再进行后续操作。

#### 列表对象
列表对象的编码可以是ziplist 或者 linkedlist。
ziplist 编码的列表使用亚索表实现，每个压缩节点（entry）保存了一个列表元素。举个例子：
```
redis-> RPUSH numbers 1 "three" 5
(integer) 3
```
上面命令执行后的数据结构如图所示：

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-object-4-20180529.png)

##### 编码转换
当列表对象可以同时满足以下两个条件时，列表对象使用ziplist编码：
+ 列表对象保存的所有字符串元素的长度都小于64字节
+ 列表对象保存的元素数量小于512个

不能满足这两个条件的列表对象需要使用linkedlist编码

#### 哈希对象
哈希对象的编码可以是ziplist或者hashtable
ziplist编码的哈希对象使用亚索表作为底层实现，添加键值对是，会先把键的压缩列表节点推入到列表表尾，然后再对保存了值的压缩列表节点推入到压缩列表表尾。
另一方面，hashtable编码的哈希对象使用字典作为底层实现，哈希对象中的每个键值对都使用一个字典的键值对来保存：
+ 字典的每个键都是一个字符串对象，对象中保存了键值对的键
+ 字典的每个值都是一个字符串对象，对象中保存了键值对的值

下图就是哈希对象用hashtable数据结构存储的例子：

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-object-3-20180529.png)

##### 编码转换
当哈希对象同时满足以下两个条件时，哈希对象使用ziplist编码：
+ 哈希对象保存的所有键值对的键和值的字符串长度都小于64字节
+ 哈希对象保存的键值对数量小于512个

不能满足这两个条件的哈希对象需要使用hashtable编码

#### 集合对象
集合对象的编码可以是intset或者hashtable
数据结构简图如下，具体的存储方式前面章节已经介绍过。

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-object-5-20180529.png)

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-object-6-20180529.png)

##### 编码转换
当集合对象可以同时满足以下两个条件时，对象使用intset编码
+ 集合对象保存的所有元素都是整数值
+ 集合对象保存的元素数量不超过512个
不能满足这两个条件的集合对象需要使用hashtable编码

#### 有序集合对象
有序集合对象使用ziplist或者skiplist编码。使用压缩列表作为底层实现时，每个集合元素使用两个紧挨在一起的压缩列表节点来保存，第一个节点保存成员，第二节点保存分值。

使用跳跃表作为底层实现时使用zset的数据结构，具体如下：

```
typedef struct zset {
    zskiplist * zsl;
    dict * dict;
} szet;
```
zsl 是跳跃表，按分值大小保存所有集合元素，dict创建了从成员到分值的映射，字典中的每个键值对保存了一个集合元素，通过字典，方便查询给定成员的分值，复杂度O(1)。zsl 和 dict 使用引用共享成员（object）和分值（string），这样不至于浪费内存。使用dict + zskiplist的方式实现有序结合也是典型的以空间换时间的例子，只用dict不方便排序，只用zskiplist不方便查询。

##### 编码转换
当有序集合对象可以满足下面两个条件，用ziplist编码：
+ 有序集合对象保存的元素个数小于128个
+ 有序集合保存的所有元素成员的长度都小于64字节

不能满足以上两个条件的有序集合对象将使用skiplist编码


