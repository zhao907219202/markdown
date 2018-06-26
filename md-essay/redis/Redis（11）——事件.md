Redis服务器是一个事件驱动程序，服务器主要处理两类事件：

+ 文件事件（file event）：Redis服务器通过套接字与客户端（或者其他Redis服务器）进行连接，而文件事件就是服务器对套接字的抽象。服务器与客户端（或者其他服务器）
的通信会产生相应的文件事件，而服务器则通过监听并处理事件来完成一系列网络通信操作。
+ 时间事件（time event）：Redis服务器中的一些操作（比如serverCron函数）需要给定的时间点执行，而时间事件就是服务器对这类定时操作的抽象。

#### 文件事件
Redis是基于Reacror模式开发了自己的网络事件处理器，也就是文件事件处理器（file event handler）:

+ 文件事件处理器使用I/O多路复用（multiplexing）程序来同时监听多个套接字，并根据套接字目前执行的任务来为套接字关联不同的事件处理器。
+ 当被监听的套接字准备好执行连接应答（accept）、读取（read）、写入（write）、关闭（close）等操作时，与操作相对应的的文件事件就会产生，这时文件事件
处理器就会调用套接字之前关联好的事件处理器来处理这些事件。

虽然文件事件处理器以单线程的方式运行，但通过I/O多路复用程序来监听多个套接字，文件事件处理器既实现了高性能的网络通信模型，又可以很好地与Redis服务器中
其他同样的以单线程方式运行的模块进行连接，这保持了Redis内部单线程设计的简单性。

##### 文件事件处理器的构成
文件事件处理器有四个组成部分，分别是套接字、I/O多路复用程序、文件事件分派器（dispatcher）以及事件处理器。文件事件是对套接字操作的抽象，每当一个套接字准备好执行
连接应答（accept）、写入、读取、关闭等操作时，都会产生一个文件事件。因为一个服务器通常会连接多个套接字，所以多个文件事件有可能并发的出现。

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-event-0-20180623.png)


I/O多路复用程序负责监听多个套接字，并向文件事件分派器传送那些产生了事件的套接字。尽管多个文件事件可能会并发出现，但I/O多路复用程序总是会将所有产生事件
的套接字都放到一个队列里面，然后通过这个队列，以有序（sequentially）、同步（synchronously）、每次一个套接字的方式向文件事件分配器传送套接字。当上一个
套接字产生的事件处理完毕之后（该套接字为事件所关联的事件处理器执行完毕），I/O多路复用程序才会继续向文件事件分派器传送下一个套接字。文件事件分派器接受
I/O多路复用程序传来的套接字，并根据套接字产生的事件的类型，调用相应的事件处理器。服务器会为执行不同任务的套接字关联不同的事件处理器，这些处理器是一个个函数，
它们定义了某个事件发生时，服务器应该执行的动作。

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-event-1-20180623.png)

##### I/O多路复用程序的实现
Redis的I/O多路复用程序的所有功能都是通过包装常见的sleect、epoll、evport和kqueue这些I/O多路复用函数库来实现的，每个I/O多路复用函数库在Redis源码中都对应了一个单独的文件，
比如ae_select.c、ae_epoll.c、ae_kqueue.c。因为Redis为每个I/O多路复用程序都实现了相同的API，所以I/O多路复用程序的底层实现是可以互换的。

ae.c里面宏定义部分选择I/O多路复用的函数库：

```
/* Include the best multiplexing layer supported by this system.
 * The following should be ordered by performances, descending. */
#ifdef HAVE_EVPORT
#include "ae_evport.c"
#else
    #ifdef HAVE_EPOLL
    #include "ae_epoll.c"
    #else
        #ifdef HAVE_KQUEUE
        #include "ae_kqueue.c"
        #else
        #include "ae_select.c"
        #endif
    #endif
#endif
```

底层实现如下图所示：

![image](https://raw.githubusercontent.com/zhao907219202/markdown/master/md-picture/redis/redis-event-2-20180625.png)

##### 事件的类型
I/O多路复用程序可以监听多个套接字的ae.h/AE_READABLE 时间和ae.h/AE_WRITABLE事件。
```
#define AE_READABLE 1   /* Fire when descriptor is readable. */
#define AE_WRITABLE 2   /* Fire when descriptor is writable. */
```
这两类事件和套接字操作之间的对应关系如下：

+ 当套接字变得可读时（客户端对套接字执行了write操作，或者执行close操作），或者有新的可应答（acceptable）套接字出现时（客户端对服务器的监听套接字执行connect操作），套接字产生
AE_READABLE时间
+ 当套接字变得可写时（客户端对套接字执行read操作），套接字产生AE_WRITABLE事件

I/O多路复用程序允许服务器同时监听套接字的AE_READABLE事件和AE_WRITABLE事件，如果一个套接字同时产生了这两种事件，那么文件事件分派器会优先处理AE_READABLE事件，等到AE_READABLE
处理完之后，才处理AE_WRITABLE事件。如果一个套接字又可读又可写的话，那么服务器将先读套接字后写套接字。

##### API
+ ae.c/aeCreateFileEvent 函数接受一个套接字描述符、一个事件类型，以及一个事件处理器作为函数，将给定套接字的的给定事件加入到I/O多路复用程序的监听范围之内，并对事件和事件处理器进行关联。
+ ae.c/aeDeleteFileEvent 函数接受一个套接字描述符和一个监听事件类型作为参数，让I/O多路复用程序取消对给定套接字的给定事件的监听，并取消事件与事件处理器之间的关联。
+ ae.c/aeGetFileEvents 函数接受一个套接字描述符，返回该套接字正在被监听的事件类型：
     + 如果套接字没有任何事件被监听，那么函数返回AE_NONE
     + 如果套接字的读事件正在被监听，那么函数返回AE_READABLE
     + 如果套接字的写事件正在被监听，那么函数返回AE_WRITABLE
     + 如果套接字的读事件和写事件正在被监听，难么函数返回AE_READABLE|AE_WRITABLE
+ ae.c/aeWait 函数接受一个套接字描述符、一个事件类型和一个毫秒数为参数，在给定的时间内阻塞并等待套接字的给定类型事件产生，当事件成功生成，或者等待超时之后，函数返回。
+ ae.c/aeApiPoll 函数接受一个sys/time.h/struct timeval 结构为参数，并在指定的时间内，阻塞并等待所有被aeCreateFileEvent函数设置成监听状态的套接字产生文件事件，当有至少一个事件
产生，或者等待超时后，函数返回。
+ ae.c/aeProcessEvents 函数式文件事件分派器，它先调用aeApiPoll函数来等待事件产生，然后遍历所有已产生的事件，并调用相应的事件处理器来处理这些事件。
+ ae.c/aeGetApiName 函数返回I/O多路复用程序底层所使用的I/O多路复用函数库的名称，返回epoll表示底层为epoll函数库，返回select表示底层为select函数库。

##### 文件事件处理器
Redis为文件事件编写了多个处理器，这些事件处理器分别用于实现不同的网络通信需求。比如：
1. 为了对连接服务器的各个客服端进行应答，服务器要为监听套接字关联连接应答处理器
2. 为了接受客户端传来的命令请求，服务器要为客户端套接字关联命令请求回复处理器
3. 为了向客户端返回命令的执行结果，服务器要为客户端套接字关联命令回复处理器
4. 当主服务器和从服务器进行复制操作时，主从服务器都需要关联特别为复制功能编写的复制处理器

###### 连接应答处理器
Redis进行初始化的时候，程序会将这个连接应答处理器和服务器监听套接字的AE_READABLE事件关联起来，当有客户端调用sys/socket.h/connect 函数连接服务器监听套接字的时候，
套接字就会产生AE_READABLE事件，引发连接应答处理器执行。

###### 命令请求处理器
当一个客户端通过连接应答处理器成功连接服务器之后，服务器会将客户端套接字的AE_READABLE事件和命令请求处理器关联起来，当客户端向服务器发送命令请求的时候，套接字就会产生
AE_READABLE事件，引发命令请求处理器执行，并执行相应的读入操作。在客户端连接服务器的过程中，服务器一直都会为客户端套接字的AE_READABLE事件关联命令请求处理器。

###### 命令回复处理器
当服务器有命令回复需要传送给客户端的时候，服务器会将客户端套接字的AE_WRITABLE事件和命令回复处理器关联起来，当客户端准备好接受服务器传回的命令回复时，就会产生AE_WRITABLE
事件，引发命令回复处理器执行，并执行套接字写入操作。当回复传送完毕，就会解除两个之间的关联。
