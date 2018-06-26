除了RDB持久化，Redis还提供了AOF（Append Only File）持久化功能。AOF不同于RDB，其保存Redis服务器所执行的写命令来记录数据库状态。

#### AOF持久化的实现
##### 命令追加
AOF持久化分为命令的追加（append）、文件写入、文件同步（sync）三个步骤。AOF打开时，服务器执行一个写命令后，会以协议格式将被执行的写命令追加到服务器的状态aof_buf缓冲区的末尾：
```
struct redisServer{
    // ...
    // AOF 缓冲区
    sds aof_buf;
    // ...
}
```

##### AOF文件的写入与同步
Redis的服务器进程就是一个事件循环（loop），这个循环中的文件事件负责接收客户端的命令请求，以及向客户端发送命令回复，而时间事件则负责执行像serverCron函数这样需要定时运行的函数。

因为服务器在处理文件事件时可能会执行写命令，使得一些内容追加到aof_buf缓冲区里面，所以在服务器每次结束一个事件循环之前，它都会调用flushAppendOnlyFile函数，考虑是否要将缓冲区的内容
保存到AOF文件里面。后面事件章节会详细赘述这个循环内容，伪代码标识如下：

```
def eventLoop():
    while True:

        # 处理文件事件，接受命令请求以及发送命令回复
        processFileEvents()

        # 处理时间事件
        processTimeEvents()

        # 考虑是否要将aof_buf中的内容写入和保存到AOF文件里面
        flushAppendOnlyFile()
```

flushAppendOnlyFile 函数的行为由服务器配置的appendfsync 选项的值来决定，不同的值产生的行为不同（默认everysec）：

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-aof-0-20180621.png)

这里说一下文件的写入和同步，在程序调用write函数，将一些数据写入到文件的时候，操作系统通常将写入数据暂存在内存缓冲区中，等到缓冲被填满或者超过指定的时限后，才会真正的将缓冲的数据写入磁盘。
这种做法提高了效率，但是也为写入数据带来了安全问题，因为如果计算机突然停机，那么缓冲的数据将会丢失。为此，系统提供fsync和fdatasync两个同步函数，可以强制同步到磁盘，确保数据安全。

###### AOF持久化的效率和安全性

服务器配置 appendfsync选项的值直接决定AOF持久化功能的效率和安全性。当配置为always时，服务器在每个时间循环都会把aof_buf缓存写入并且同步到AOF文件。这样不用担心机器故障停机，虽然安全，但是效率最低
如果配置为everysec，服务器在每次事件循环都要将aof_buf缓冲区中所有内容写入到AOF文件，并且每隔一秒就要在子线程中对AOF文件进行一次同步。从效率上讲，这样足够快，并且即使停机，也只丢失一秒钟的数据。
如果配置为no，服务器每个事件循环都将aof_buf写入AOF文件，何时同步，交给操作系统处理。这样AOF写入速度很快，但是同步周期长，故障丢失大。

#### AOF文件的载入与数据还原

因为AOF包含了重建数据库所需的所有写命令，所以服务器只要读入并重新执行一遍AOF文件保存的写命令，就可以还原服务器关闭之前的数据库状态。详细步骤如下：

1. 创建一个不带网络连接的伪客户端（fake client）: 因为Redis命令只能在客户端上下文中执行，而载入AOF文件时所使用的命令直接来源于AOF文件而不是网络连接，所以服务器使用了一个没有网络连接
的伪客户端来执行AOF文件保存的写命令。
2. 从AOF文件中分析并取出一条写命令
3. 使用伪客户端执行被读出的写命令。
4. 一直执行步骤2 和 步骤3，直到AOF文件中所有写命令都被处理完毕

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-aof-1-20180622.png)

#### AOF 重写
因为AOF持久化是通过保存被执行的写命令来记录数据库状态的，所以随着服务器运行时间的流逝，AOF文件中的内容会越来越多，文件的体积也会越来越大，如果不加以
控制的话，体积过大的AOF文件很可能对Redis服务器、甚至整个宿主计算机造成影响，并且AOF文件的体积越大，使用AOF文件来进行数据还原所需的时间就越多。

为了解决AOF文件体积膨胀的问题，Redis提供了AOF重写（rewrite）功能。重写可以创建一个新的AOF文件来替代现有的AOF文件，新旧两个AOF文件保存的数据库
状态时相同的，但新的AOF文件不会包含任何浪费空间的冗余命令，所以以新AOF文件的体积通常比旧的要小的多。

##### AOF 文件重写的实现
AOF重写不需要对现有的AOF文件进行任何读取，分析或者写入操作，这个功能是通过读取服务器当前的数据库状态来实现的。现有的AOF可能对某个key做了不止一次的更新操作，导致
这个key的对应AOF记录有多条，但是新生成的AOF文件只需要一条，直接记录这个key的值，这就是AOF重写的好处。整个重写过程用如下伪代码表示为：

```
def aof_rewrite(new_aof_file_name):
    f = create_file(new_aof_file_name)
    for db in redisServer.db:
        if db.is_empty(): continue
        f.write_command("SELECT" + db.id)
        for key in db；
            if key.is_expired(): continue
            if key.type == String:
                rewrite_string(key)
            elif key.type == List:
                rewrite_list(key)
            elif key.type == Hash:
                rewrite_hash(key)
            elif key.type == Set:
                rewrite_set(key)
            elif key.type == SortedSet:
                rewrite_sorted_set(key)

            if key.have_expire_time()；
                rewrite_expire_time(key)
    f.close()
```

上面所有rewrite函数就是往新的AOF文件中写入对应的命令，比如SET、RPUSH、HMSET、SADD、ZADD、PEXPIREAT。实际中，在处理集合键（列表、哈希表、集合、有序集合）时
会先检查键所包含的元素数量，为了避免客户端输入缓冲区的溢出，如果元素数量超过了redis.h/REDIS_AOF_REWRITE_ITEMS_PER_CMD常量值，那么重写程序将使用多条命令记录键的值，目前这个值是64，也就是说一条语句最多添加64个元素。

##### AOF 后台重写
Redis服务器单个线程来处理命令请求，如果服务器直接调用aof_rewrite函数的话，那么在AOF重写期间，服务器显然将如法继续服务。所以Redis将AOF重写程序放到子进程里执行，这样同时达到两个目的：

1. 子进程进行AOF重写期间，服务器进程可以继续服务，处理命令请求
2. 子进程带有服务器进程的数据副本，使用子进程而不是线程，可以在避免使用锁的情况下，保存数据的安全性

子进程也有一个问题需要解决，因为子进程在进行AOF重写期间，服务器进程还在继续处理命令请求，而新的命令可能对现有的数据库状态进行修改，从而使得服务器当前的数据库状态和重写后
的AOF文件所保存的数据库状态不一致。

为了解决数据不一致的问题，Redis服务器设置了一个AOF重写缓冲区，这个缓冲区在服务器创建子进程后开始使用，当Redis服务器执行完一个写命令后，它会同时将这个写命令发送给AOF缓冲区和AOF重写缓冲区，

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-aof-2-20180622.png)

这样一来可以保证，AOF缓冲区的内容会定期被写入和同步到AOF文件，对现有的AOF文件的处理工作会如常进行；从创建子进程开始，所有的写操作也会写到AOF重写缓冲区

当子进程完成AOF重写工作之后，它会向父进程发送一个信号，父进程在收到该信号之后，会调用一个信号处理函数，并执行以下工作（执行过程中对服务器进程阻塞）：

+ 将AOF重写缓冲区的所有内容写入到新的AOF文件中，这时新AOF文件所保存的数据库状态和服务器当前的数据库状态一致
+ 对新的AOF文件进行改名，原子的覆盖现有的AOF文件，完成新旧两个AOF文件的替换

整个AOF后台重写过程中，只有信号处理函数会对服务器造成阻塞，在其他时候，AOF后台重写都不会阻塞父进程，这将AOF重写服务器性能造成的影响降到了最低。举个例子：

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-aof-3-20180622.png)







