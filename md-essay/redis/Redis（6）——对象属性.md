#### 类型检查和命令多态
Redis中用于操作键的命令基本上可以分为两种类型：

1. 对任何键都可以执行的命令，比如DEL、EXPIRE、RENAME、OBJECT
2. 只能对特定类型的键执行，比如

+ SET、GET、APPEND、SRRLEN 只能对字符串键执行
+ HDEL、HSET、HGET、HLEN 只能对哈希键执行
+ RPUSH、LPOP、LINSERT、LLEN 只能对列表键执行
+ SADD、SPOP、SINTER、SCARD 只能对集合键执行
+ ZADD、ZCARD、ZRANK、ZSCORE 只能对有序集合键执行

##### 类型检查的实现
在执行一个类型特定的命令之前，Redis会先检查输入键的类型是否正确，然后再决定是否执行给定的命令。类型检查是通过redisobject结构的type属性来实现的。举个例子，执行LLEN时，服务器会先检查是否为列表对象类型，具体如下图：

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-object-7-20180530.png)

##### 多态命令的实现
Redis除了会根据值对象的类型来判断键是否能够执行指定命令外，还会根据值对象的编码格式，选择正确的命令实现代码来执行命令。举个例子，列表对象有ziplist和linkedlist两种编码，两种编码分别使用不同的api实现对应的数据结构上的操作。那么我们可以理解为LLEN这个命令在编码方式上是多态的，无论是那种编码类型，都会正常执行。DEL、EXPIRE、TYPE等命令实际上也是多态命令，是基于type的多态命令。

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-object-8-20180530.png)

#### 内存回收
C语言不存在自动内存回收功能，所以Redis在自己的对象系统中构建了一个引用计数器（reference counting）技术实现的内存回收机制，通过这一机制，程序可以通过跟踪对象的引用技术信息，在适当的时候自动释放对象并进行内存回收。
每个对象的引用计数信息有redisobject结构的refcount属性记录：
```
typedef struct redisobject{
    // ...
    // 引用计数
    int refcount;
    // ...
}
```

创建一个新对象，引用计数初始化为1，被新程序引用，增1，不再被程序使用，减1，为0时对象所占用的内存会被释放。

#### 对象共享
对象的引用计数还带有对象共享的作用，在Redis中让多个键共享同一个值对象需要执行两个步骤：
+ 将数据键的值指针指向一个现有的值对象
+ 将被共享的值对象引用计数器增1

redis会初始化服务器时，创建0-9999所有的整数值，一万个字符串对象作为共享对象。

#### 对象的空转时长
redisobject中除了type、encoding、ptr和refcount属性外，还有一个lru属性用来计算空转时长。OBJECT IDLETIME命令可以打印出给定键的空转时长，是用当前时间减去键的lru时间计算得出的。OBJECT IDLETIME命令是特殊的，这个命令在访问键的对象时，不会修改值对象的lru属性。

键的空转时长还有一个作用，如果服务器打开了maxmemory选项，并且服务器用于回收内存的算法是volatile-lru 或者 allkeys-lru，那么当服务器占用的内存数超过了maxmemory选项所设置的上限值时，空转时长较高的那部分键会优先被服务器释放，从而回收内存。配置文件的maxmemory选线和maxmemory-policy选项说明介绍了关于这方面的更多信息。