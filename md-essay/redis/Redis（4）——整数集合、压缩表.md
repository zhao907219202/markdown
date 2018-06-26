#### 整数集合
整数集合（intset）是集合键的底层实现之一，当一个集合只包含整数值元素，并且这个集合的元素数量不多时，Redis就会使用整数集合作为集合键的底层实现。
```
redis-> SADD numbers 1 3 5 7 9
(integers) 5
redis-> OBJECT ENCODING numbers
"intset"
```
看看整数集合的定义
```
typedef struct intset{
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
}
```
contents数组是整数集合的低层实现：整数集合的每个元素都是contents数组的一个数组项（item），各个项在数组中按值的大小从小到大有序地排列，并且不包含重复项。
length属性记录了整数集合包含的元素数量，也就是contents数组的长度。虽然intset结构将contents数组声明为int8_t类型，但是其真正类型取决于enconding属性的值。
+ 如果encoding属性的值为INTSET_ENC_INT16，那么contents就是一个int16_t类型的数组，数组中的每一个项都是一个int16_t类型的整数值（最小值-32768，最大32767）
+ 如果encoding属性的值为INTSET_ENC_INT32，那么contents就是一个int32_t类型的数组，数组中的每一个项都是一个int32_t类型的整数值（最小值-2147483648，最大值2147483647）
+ 如果encoding属性的值为INTSET_ENC_INT64，那么contents就是一个int64_t类型的数组，数组中里的每一项就是一个int64_t类型的整数值（最小值-9223372036854775808，最大值9223372036854775807）

下图展示了一个整数集合：
1. encoding属性是INTSET_ENC_INT64,表示整数集合的底层实现为int64_t类型的数组，而数组中保存的都是int64_t类型的整数值
2. length属性的值为4，表示整数集合包含4个元素
3. contents数组按从小到大的顺序保存着集合中的四个元素，因为每个元素都是int64_t类型的整数值，所以contents数组的大小为sizeof（int64_t)*4 = 64 * 4 = 256 位
![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-intset-0-20180505.png)

#### 整数集合的升级
当我们把一个新元素添加到整数集合中时，如果新元素比整数集合现有的所有元素的类型都要长，整数集合需要先进行升级(upgrade),然后才能将新元素添加到整数集合里面。
升级一共分为三个步骤：
+ 根据新元素的类型，扩展整数集合底层数组的空间大小，并为新元素分配空间
+ 将底层数组现有的所有元素都转换成与新元素相同的类型，并将类型转换后的元素放置到正确的位置上，而且在放置元素的过程中，需要继续位置底层数组的有序性不变。
+ 将新元素添加到底层数组里面。

升级的过程要分配空间并且扩展每个项的字节数并移动到相应的位置，所以复杂度是O(N)。升级的好处主要有两个：提升灵活性，整数集合自动升级来适应新元素，不必担心类型错误；节约内存，集合可以同时保存三种不同类型的值，又可以确保升级操作只会在有需要的时候进行。注意的是整数集合不支持降级。

#### 压缩表
压缩表(ziplist)是列表键和哈希键的底层实现之一。当一个列表键只包含少量列表项，并且每个列表项要么就是小整数值，要么就是长度比较短的字符串，那么redis就会使用压缩列表来做列表键的底层实现。另外当一个哈希键只包含少量键值对，并且这个键值对的键和值要么就是小整数值要么就是长度比较短的字符串，那么redis就会使用压缩列表来做哈希键的底层实现。

压缩列表是redis为了节约内存而开发的，有由一系列特殊编码的连续内存块组成的顺序型(squential)数据结构。一个压缩列表可以包含多个节点(entry)，每个节点可以保存一个字节数组或者一个整数值。
下图展示了压缩列表的各个组成部分：
![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-ziplist-0-20180506.png)
![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-ziplist-1-20180506.png)

举个例子：

* 列表zlbytes属性的值为0x50,十进制80，表示压缩列表总长80字节
* 列表zltail属性的值为0x30,十进制60，表示如果我们有一个指向压缩列表起止地址的指针p,那么只要用指针p加上偏移量60就可以计算出表尾节点entry3的地址
* 列表zllen属性的值为0x3,十进制3，表示压缩列表包含三个节点
![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-ziplist-2-20180506.png)

#### 压缩列表节点的构成
每个压缩列表节点可以保存一个字节数组或者一个整数值，其中，字节数组可以是以下三种长度的其中一种：

+ 长度小于等于63（2^6-1) 字节的字节数组
+ 长度小于等于16383（2^14-1）字节的字节数组
+ 长度小于等于4294967295（2^32-1）字节的字节数组

而整数值则可以是以下六种长度的一种：

* 4位长，介于0至12之间的无符号整数
* 1字节长的有符号整数
* 3字节长的有符号整数
* int16_t 类型整数
* int32_t 类型整数
* int64_t 类型整数

每个压缩列表的节点都是有previous_entry_length,encoding,content三个部分组成，如下图所示。
![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-ziplist-3-20180506.png)

#### previous_entry_length
节点的previous_entry_length属性以字节为单位，记录了压缩列表前一个节点的长度。如果前一个节点长度小于254字节，那么previous_entry_length的长度为1字节；如果前一节点长度大于等于254字节，那么previous_entry_length属性的长度为5字节，其中第一字节为被设置成0xFE（十进制254），而之后的四个字节则用于保存前一节点的长度。

因为节点的previous_entry_length属性记录了前一节点的长度，所以程序可以通过指针运算，根据当前节点的起始地址来计算出前一个节点的起始地址。如果我们有一个指向当前节点的指针c，那么我们只要用指针c减去当前节点previous_entry_length属性的值，就可以得出一个指针指向前一个节点起始地址的指针p。压缩列表从表尾向表头遍历就是使用这个原理实现的。

#### encoding
节点的encoding属性记录了节点的content属性所保存数据的类型以及长度：
+ 一字节、两字节或者五字节长，值得最高位为00、01或者10的是字节数组编码：这种编码表示节点的content属性保存着字节数组，数组的长度由编码除去最高两位之后的其他位记录
+ 一字节长，值的最高位以11开头的是整数编码：这种编码表示节点的content属性保存的是整数值，整数值的类型和长度由编码除去最高两位之后的其他位记录

字节数组编码：
![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-ziplist-4-20180506.png)
整数编码
![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-ziplist-5-20180506.png)

#### content
节点的content 属性负责保存节点的值，节点值可以是一个字节数组或者整数，值的类型和长度由节点的encoding属性决定。

 previous_entry_length | encoding | content
:---:|:---:|:---:
... | 00001011 | "hello world" 
... | 11000000 |  10086

#### 连锁更新
前面说过，每个节点的previous_entry_length属性都记录了前一个节点的长度，现在考虑这样一种情况，在一个压缩列表中，有多个连续的、长度介于250字节到253字节之间的e1至eN，因为e1至eN的所有节点的长度都小于254字节，所以记录这些节点的长度只需要1字节的previous_entry_length。

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-ziplist-6-20180506.png)

这时我们将一个长度大于254字节的新节点new设置为压缩列表的表头节点，那么new将成为e1的前置节点。

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-ziplist-7-20180506.png)

因为e1的previous_entry_length属性为1字节，它没办法保存new节点的长度，所以程序要对它进行空间重新分配，并由原来的1字节变成5字节，现在麻烦来了，由于e1处于250字节到253字节之间，这就导致了e1的长度介于254字节和257字节之间，以此类推，后面的所有节点都需要重新分配空间，这种操作就是连锁更新。

删除节点时也有可能会触发连锁更新，被删除节点的后续节点的previous_entry_length如果是一字节，而被删除节点的前面节点长度大于等于254字节，就需要对后面节点重新分配，而如果该节点也是在250-253之间，并且后面都是，那么同样的会出现连锁更新。

因为连锁更新在最坏的情况下需要对压缩列表执行N次空间分配，而每次空间重分配的最坏时间复杂度为O(N)，所以连锁更新时间复杂度为O(N^2),但是需要注意的是，虽然连锁更新复杂度高，但是发生的概率很小，压缩表的性能几乎不会被其受影响。







