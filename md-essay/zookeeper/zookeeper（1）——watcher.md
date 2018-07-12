Zookeeper提供了分布式数据的发布/订阅功能。一个典型的发布/订阅模型系统定义了一种一对多的订阅关系，能够让多个订阅者同时监听某一个主题对象。
在Zookeeper中，引入了Watcher机制来实现这种分布式的通知功能。在使用Zookeeper的Api创建连接、获取数据、获取孩子节点等等操作的时候，都可以
向服务端注册一个Watcher，服务端在监测到这些被注册的节点状态被修改之后，会向客户端发送事件通知，从而触发客户端注册的事件处理机制。
这里我们就详细介绍下Zookeeper的Watcher机制。
#### Watcher 和 WatcherEvent
```
public interface Watcher {

    public interface Event {

        public enum KeeperState {...}

        public enum EventType {...}
    }

    public enum WatcherType {...}

    abstract public void process(WatchedEvent event);
}
```
看上面代码可以看到watcher是个接口，抽象方法process其实就是事件触发的处理逻辑。上面的的子接口是事件的定义，其中的两个枚举分别为连接状态和事件类型。中间的枚举是监视器的类型。
那到底有多少种事件状态？《从Paxos到Zookeeper 分布式一致性原理与实践》给出了很清晰的答案，这里引用一下。