---
title: 分布式锁的Redis与Zookeeper实现
tags:
  - 分布式
categories:
  - blog
date: 2020-10-12 09:44:00
---
在分布式系统中，会有来自于不同实例的线程访问同一个临界资源，这时我们需要一种分布式的协调技术来对线程进行调度。其中的核心实现为分布式锁。
<!--more-->

## 分布式锁的特性

- 在分布式环境下，同一个临界资源\临界操作只能同时被1个机器的1个线程访问\执行。
- 高可用的锁获取与锁释放
- 高性能的锁获取与锁释放
- 可重入性，同一任务可多次获取锁
- 具备锁失效机制
- 可实现非阻塞锁

## Redis简单实现

#### 加锁

```
SETNX lock_id 1
```

`SETNX` 命令为“**SET** if **N**ot e**X**ists”的简写。当key不存在时返回1，key存在时返回0。当该key存在值时我们可以认为为对应id的资源加上了锁。在一个线程执行该命令时，如果返回0，则说明该资源已被加锁，获取锁失败。当返回1时，说明资源之前未被加锁，当前线程成功获取了锁。

#### 释放锁

```
DEL lock_id
```

`DEL` 命令通过删除lock_id来释放锁，从而使其他线程在运行`SETNX lock_id`时可以获取到锁。

#### 防止死锁

如果一个线程在加锁后，还没来得及解锁便崩溃了，就会导致这个锁无人释放从而形成死锁。为了防止这种情况，我们需要给锁设置超时时间。如果简单的使用下方的语句设置超时会产生问题：

```
EXPIRE lock_id 30
```

产生问的原因是，`SETNX`与`EXPIRE`两次操作之间是非原子性的，也就会导致如果线程在运行`SETNX`和`EXPIRE`之间崩溃了，会产生死锁。对此正确的解决方式为使用如下语句：

```
SET lock_id 1 NX EX 30
```

其中`NX`与`EX`为Redis在2.6.12版本后引入的`SET`命令选项，`NX`类似`SETNX`，仅在key不存在时使`SET`生效。`EX`为设置该key的过期时间。具体的选项可见官方文档[SET key value [EX seconds] [PX milliseconds] [NX|XX]](http://www.redis.cn/commands/set.html)。

#### 防止误删

在一些情况下可能会导致锁误删，即线程获取的锁被其他线程删除。为了防止这种情况，我们可以将锁的值设定为线程或客户端的id：

```
SET lock_id THREAD_ID NX EX 30
```

之后在删除前获取该锁的值，与自身id进行比较，仅在id相同时才删除锁，这样就会涉及执行两条命令：

```
GET lock_id
# 若id相同：
DEL lock_id
```

看到执行两条命令，我们就又发现了问题：这个操作没有原子性，如果在`GET`命令和`DEL`命令之间，锁的值发生了变化，那就还会产生误删的情况。为了解决这个问题，官方给出的[解决方案](https://redis.io/topics/distlock)是Lua脚本：

```
EVAL "if redis.call('get',KEYS[1]) == ARGV[1] then return redis.call('del',KEYS[1]) else return 0 end" 1 lock_id THREAD_ID
```

其中`EVAL`第一个参数为Lua语句，第二个参数为个数(此例中为一个key和一个参数)。之后的参数为key和传入Lua的参数。Redis执行Lua脚本具有原子性，执行的Lua如下：

```Lua
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

至此我们完成了简单的Redis锁实现。

## ZooKeeper分布式锁原理

#### ZooKeeper的四种节点

- 持久节点（PERSISTENT）默认的节点类型。创建节点的客户端与 Zookeeper 断开连接后，该节点依旧存在。
- 持久节点顺序节点（PERSISTENT_SEQUENTIAL）所谓顺序节点，就是在创建节点时，Zookeeper 根据创建的时间顺序给该节点名称进行编号。
- 临时节点（EPHEMERAL）和持久节点相反，当创建节点的客户端与 Zookeeper 断开连接后，临时节点会被删除。
- 临时顺序节点（EPHEMERAL_SEQUENTIAL）顾名思义，临时顺序节点结合和临时节点和顺序节点的特点：在创建节点时，Zookeeper 根据创建的时间顺序给该节点名称进行编号；当创建节点的客户端与 Zookeeper 断开连接后，临时节点会被删除。

#### 加锁

首先，在 Zookeeper 当中创建一个持久节点 。当第一个客户端想要获得锁时，需要在这个节点下面创建一个**临时顺序节点** Lock1。

之后，Client1 查找 该节点下面所有的临时顺序节点并排序，判断自己所创建的节点 Lock1 是不是顺序最靠前的一个。如果是第一个节点，则成功获得锁。

Client2 查找下面所有的临时顺序节点并排序，判断自己所创建的节点 Lock2 是不是顺序最靠前的一个，结果发现节点 Lock2 并不是最小的。于是，Client2 向排序仅比它靠前的节点 Lock1 注册 Watcher，用于监听 Lock1 节点是否存在。这意味着 Client2 抢锁失败，进入了等待状态。这便形成了一个等待队列

#### 解锁

当任务完成时，Client1 会显示调用删除节点 Lock1 的指令。由于 Client2 一直监听着 Lock1 的存在状态，当 Lock1 节点被删除，Client2 会立刻收到通知。这时候 Client2 会再次查询 父节点下面的所有节点，确认自己创建的节点 Lock2 是不是目前最小的节点。如果是最小，则 Client2 顺理成章获得了锁。

#### 实现

```java
public class ZkLockImpl implements ZkLock {

    private static final Logger LOG = LoggerFactory.getLogger(ZkLock.class);

    private final String lockPath;
    private final ZkClient zkClient;
    private final ThreadLocal<String> curNode = new ThreadLocal<>();
    private final ThreadLocal<String> preNode = new ThreadLocal<>();

    /**
     * Constructor for basic ZkLock
     *
     * @param servers Servers list for zookeeper, see {@link ZkClient#ZkClient(java.lang.String, int, int)}
     * @param sessionTimeout Session Timeout for zookeeper, see {@link ZkClient#ZkClient(java.lang.String, int, int)}
     * @param connectionTimeout Connection Timeout for zookeeper, see {@link ZkClient#ZkClient(java.lang.String, int, int)}
     * @param lockPath the path of this lock in zookeeper
     */
    public ZkLockImpl(String servers, int sessionTimeout, int connectionTimeout, String lockPath){
        this.lockPath = lockPath;
        this.zkClient = new ZkClient(servers, sessionTimeout, connectionTimeout);
        if(!zkClient.exists(lockPath)){
            zkClient.createPersistent(lockPath);
            LOG.info("Connected to [{}], lock path:[{}] created",servers,lockPath);
        }else {
            LOG.info("Connected to [{}], lock path:[{}] existed",servers,lockPath);
        }
    }

    /**
     * Try if the thread occupied lock currently.
     *
     * @return Ture if thread occupied the lock
     */
    private boolean tryLock() {
        List<String> lockQueue = zkClient.getChildren(lockPath);
        Collections.sort(lockQueue);
        if(lockQueue.size()>0&&(lockPath+"/"+lockQueue.get(0)).equals(curNode.get())){
            LOG.debug("Lock [{}] acquired",lockPath);
            return true;
        }else{
            int index = lockQueue.indexOf(curNode.get().substring(lockPath.length()+1));
            preNode.set(lockPath+"/"+lockQueue.get(index-1));
            LOG.debug("Lock [{}] is occupied, set preNode to [{}]",lockPath,preNode.get());
            return false;
        }
    }

    public void lock() {
        CountDownLatch latch = new CountDownLatch(1);

        IZkDataListener listener = new IZkDataListener() {
            @Override
            public void handleDataChange(String dataPath, Object data) {

            }

            @Override
            public void handleDataDeleted(String dataPath) {
                LOG.debug("Node [{}] has been deleted",dataPath);
                latch.countDown();
            }
        };

        if(null==curNode.get()){
            curNode.set(zkClient.createEphemeralSequential(lockPath+"/","Lock"));
            LOG.debug("curNode [{}] has been created",curNode.get());
        }else {
            throw new LockException("ZkLock is not reentrant");
        }

        if(!tryLock()&&zkClient.exists(preNode.get())){
            zkClient.subscribeDataChanges(preNode.get(),listener);
            while(zkClient.exists(preNode.get())&&!tryLock()){
                try {
                    LOG.debug("Thread blocked in lock [{}]",lockPath);
                    latch.await();
                } catch (InterruptedException e) {
                    LOG.error(e.getMessage());
                }
            }
            zkClient.unsubscribeDataChanges(preNode.get(),listener);
        }
    }

    public boolean releaseLock() {
        if(null!=curNode.get()&&zkClient.exists(curNode.get())&&tryLock()){
            if(zkClient.delete(curNode.get())){
                LOG.debug("Lock [{}] released, Node [{}] deleted",lockPath,curNode.get());
                curNode.remove();
                return true;
            }
        }
        LOG.error("Illegally lock release");
        return false;
    }
}
```

项目见我的[GitHub](https://github.com/GaryXiongxiong/ZkLock)

## 参考

> http://www.imodou.com.cn/article/67
>
> https://www.funtl.com/zh/apache-dubbo-zookeeper/Zookeeper-%E5%A6%82%E4%BD%95%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81.html#zookeeper-%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81%E7%9A%84%E5%8E%9F%E7%90%86