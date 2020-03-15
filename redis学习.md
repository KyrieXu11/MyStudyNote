#  配置文件解析

```
daemonize yes
```

这一条配置的意思就是在后台启动，以守护进程的方式启动



```
pidfile /var/run/redis/redis-server.pid
```

这条的配置就是将redis-server启动之后的pid写入到这个文件，pid就是进程号



```
port 6379
```

redis-server的默认端口号就是6379



```
bind 127.0.0.1
```

绑定的地址，默认是只能本地连接redis-server，如果要远程连接redis-server，就把这一条配置注释掉就行了



```
timeout 0
```

当客户端闲置多长时间的时候关闭连接，如果指定是0那就是禁用这个功能



```
loglevel verbose
```

redis的日志级别，默认是verbose，这个级别还有：debug、notice、warning



```
logfile /var/log/redis/redis-server.log
```

如果redis是以守护进程启动的，而这个logfile又是stdout的话，那么会默认输出到/dev/null



```
databases 16
```

默认支持16个数据库



```
saved <seconds> <changes>
```

设置多少秒之后又changes更改，就是持久化操作，将key-value写到硬盘上



```
dbfilename dump.rdb
```

指定本地数据库名称，它所在的目录是下面这条配置

```
dir /var/lib/redis
```



```
requirepass foobared
```

设置redis的密码，默认是关闭的



```
maxclients 10000
```

同一时间的最大客户端连接数量



```
maxmemory <bytes>
```

redis的最大缓存，设置的值不要超过服务器的内存的最大值，比如说，1g的服务器，不能设置超过1g的redis最大内存。redis启动时，会把数据加载到内存中去，达到最大内存之后，redis会先尝试清除已经到期的key-value，但是当这个方法也已经处理过之后仍然达到了最大内存设置，将无法进行写操作，但是可以进行读操作。redis新的vm机制，会把key放入内存，value放入swap区。

如果redis一直增加数据，就会让redis的内存溢出，导致redis宕机，所以有两种解决方案：

1. 为数据设置一个合理的过期时间

2. 采用lru(最近最少使用)淘汰策略进行页面置换

   ![redis_strategy.png](https://i.loli.net/2020/02/14/J4u2BIoRgOG7Upd.png)

依次从上到下说说每个算法的作用：

​	1.设定的超时时间数据中，删除最不常用的数据

​	2.查询所有的key中最近最不常用的数据，进行删除

​	5.随机在超时时间的数据删除

​	6.查询全部设定超时时间的数据并进行排序，将马上就过期的数据删除

​	7.溢出的话，报错返回，默认使用这个策略

# redis的一些命令

## redis登陆命令

redis-cli [-h host -p port -a password]，如果是默认的没改地址、没改端口号、没设置密码，那么就是正常的redis-cli启动客户端直接连接redis-server。

## redis的关闭命令

1. 使用linux的kill 的9号信号杀死redis进程，容易造成数据丢失
2. 使用shutdown关闭

## redis的查询命令

## 教程网址

https://www.redis.net.cn/tutorial/3501.html



### keys命令

![redis_instructions_list.png](https://i.loli.net/2020/02/15/69uOBsJXviPcg2k.png)

### string命令

string是redis的最最基本的数据类型，一个key对应一个value。

并且其是二进制安全的，因为在redis存储的string是可序列化的，而序列化之后，都是用二进制表示的数据，编码不会被篡改，所以是二进制安全的，可以包含任何的数据，如jpg图片或者是序列化对象(java中实现serializable接口的对象，其他语言不清楚)。

一个key能存储最大512mb的数据，但是建议不超过1024个字节即1kb。

关于下面的指令有几点补充的地方：

1.getrange key start end 是全闭区，左右两端都能取的。

2.incr key 如果在没有key的情况下，会创建这个key，并且初始化为0 。

3.setnx key value,分布式锁的命令，面试可能会问。

4.string常存储json数据

![redis_string_instructions.png](https://i.loli.net/2020/02/15/Dm5jgQlpOZudVSk.png)

### hash命令：

哈希就是将多对，field-value存储在key中，查询可以根据key查询，这样查出来的就是多个field-value。也可以根据key field查询，这样查出来的就是对应field的value。

下面的指令说两句补充的：

1.hmset key field.. value..，这个必须redis中没有这个key，不然会报错。

2.hash常存储一个对象

![redis_hash_instrcutions.png](https://i.loli.net/2020/02/15/hiO436mfsUc2YWL.png)





1.查询所有的key：keys *，也可以查询某个特定的key，也可以根据key模糊查询，比如

​	![redis_key_select.png](https://i.loli.net/2020/02/15/AMIVWPX3rpKfaid.png)

现在有两个key:`test_key`和`test_key1`，可以通过这种模糊查询来查询key值

2.查询对应key的value：get \<key-name>，此 命令用于获取指定 key 的值。如果 key 不存在，返回 nil 。如果key 储存的值不是字符串类型，返回一个错误。查询的时候，**key区分大小写**。

3.删除key：del key1 key2 key3...，可以删除一个key也可以删除多个key，但是不能使用模糊删除。如果删除成功就会返回删除了几个key。要删除所有的key是：flushall

![redis_del_key_vague.png](https://i.loli.net/2020/02/15/QrlX9N8Gm5DeywR.png)

4.序列化给定的key，并且返回序列化的值：dump key，但是源key是不变的。

5.判断key是否存在：exists key... ，存在返回存在key的数量，不存在返回0

6.给key设置过期时间：expire key seconds

7.查看还有多长时间存活：ttl key，如果返回的是-1代表永久存活，-2代表无效的key

![redis_expire_key_10seconds.png](https://i.loli.net/2020/02/15/T5WpzrIhOePyNt1.png)

# key的命名规范

redis单个key可以存入512m大小

1. key不要太长，不要超过1024个字节；也不要太短，影响可读性。

2. 因为nosql是数据与数据之间没有任何关联的，所以只有通过命名来产生关联关系，举个小例子

   现在有两个key，表示的都是用户名，可是如果都是user_name的话，会导致之前的user_name的value被后面的user_name的value取代掉。如果是在mysql数据库中，那么我们通常会加一个`id`字段来表示不同的用户，所以在这里，最好使用user_id_name，这种形式的key来存储。就是`user_1_name` 的`value`表示`xu`，`user_2_name`的`value`表示`kyrie`，这样就区分了两个user_name的区别。可是由于一些`sql`数据库中表的字段的名称，可能带有下划线，所以这里是命名方式是：`user:id:name`



# 缓存穿透

## 概念

就是缓存和数据库中都没有数据，而用户不断的发起请求，会导致数据库压力增大，甚至导致宕机。比如用户不断查询id=-1的值。

## 解决方法

1. 接口层增加校验，如用户鉴权校验，id做基础校验，id<=0的直接拦截；
2. 从缓存取不到的数据，在数据库中也没有取到，这时也可以将key-value对写为key-null，缓存有效时间可以设置短点，如30秒（设置太长会导致正常情况也没法使用）。这样可以防止攻击用户反复用同一个id暴力攻击

# 缓存雪崩

## 概念

缓存雪崩就是，当redis的key大面积过期的时候，数据库的压力倍增。举个不会发生的例子：当双十一来临的时候，设置首页商品过期时间是1个小时，当人们抢购完成之后，此时的用户量还是很大，缓存过期了，只有从数据库中查询数据库，这么大的访问量，数据库承受的压力过大，甚至导致宕机。

## 解决方法

1. 缓存数据的过期时间设置随机，防止同一时间大量数据过期现象发生。
2. 如果缓存数据库是分布式部署，将热点数据均匀分布在不同的缓存数据库中。
3. 设置热点数据永远不过期。

## 和缓存击穿的区别

缓存雪崩是大量的缓存数据过期，而缓存击穿是一条数据正在被高并发的使用，从而导致数据库宕机。

# Jedis怎么操作redis

```java
InputStream stream = this.getClass().getClassLoader().getResourceAsStream("jedis.properties");
        assert stream != null;
//        处理流，加快文件读写速度
        BufferedInputStream in = new BufferedInputStream(stream);
        Properties properties = new Properties();
        properties.load(in);
        String host = properties.getProperty("jedis.host");
        String portStr = properties.getProperty("jedis.port");
        int port = Integer.parseInt(portStr);
//        1. 获取连接
        Jedis jedis=new Jedis(host,port);
//        2. 设置密码，如果redis不设置密码的话，不能连接redis
        jedis.auth("123");
//        3. 检测是否连接成功，如果连接成功则继续
        String res = jedis.ping();
        if("PONG".equals(res)){
            jedis.setnx("jedis:key","jedis:value");
            jedis.close();
        }
```

在redis查询对应的记录：

![redis_jedis_test.png](https://i.loli.net/2020/02/15/N1WFYw9eC2dmvRj.png)



# Redis持久化原理

Redis 分别提供了 RDB 和 AOF 两种持久化机制：

- RDB 将数据库的快照（snapshot）以二进制的方式保存到磁盘中。
- AOF 则以协议文本的方式，将所有对数据库进行过写入的命令（及其参数）记录到 AOF 文件，以此达到记录数据库状态的目的。

## AOF

Redis 将所有对数据库进行过**写入的命令（及其参数）**记录到 **AOF 文件**， 以此达到记录数据库状态的目的。

如果有对redis进行**修改的**指令的话，那么这几条指令就会被记录在aof文件中。为了处理的方便， AOF 文件使用**网络通讯协议**的格式来保存这些命令。

同步命令到 AOF 文件的整个过程可以分为三个阶段：

1. **命令传播**：Redis 将执行完的命令、命令的参数、命令的参数个数等信息发送到 AOF 程序中。redis服务器会根据redis客户端的请求的协议文本来调用不同的函数，将参数转换成字符串对象。
2. **缓存追加**：AOF 程序根据接收到的命令数据，将命令转换为网络通讯协议的格式，然后将协议内容追加到服务器的 AOF 缓存中。
3. **文件写入和保存**：AOF 缓存中的内容被写入到 AOF 文件末尾，如果设定的 AOF 保存条件被满足的话， fsync 函数或者 fdatasync 函数会被调用，将写入的内容真正地保存到磁盘中。

### 命令传播

就是当一个 redis客户端要执行命令的时候，客户端就会通过网络连接，将协议文本传输给redis服务器，服务器在接收到客户端请求的时候，会根据不同的协议文本调用不同的命令函数，并且将各个参数从字符串文本转换成字符串对象。

比如说， 要执行命令 `SET KEY VALUE` ， 客户端将向服务器发送文本 `"*3\r\n$3\r\nSET\r\n$3\r\nKEY\r\n$5\r\nVALUE\r\n"` 。

针对上面的 `SET`命令例子， Redis 将客户端的命令指针指向实现 `SET `命令的 `setCommand` 函数， 并创建三个 Redis 字符串对象， 分别保存 `SET` 、 `KEY` 和 `VALUE` 三个参数（命令也算作参数）。

![Aof](https://redisbook.readthedocs.io/en/latest/_images/graphviz-a5c804211267a10a5c3ffa47c5b600727191a3be.svg)

### 缓存追加

1. 接受命令、命令的参数、以及参数的个数、所使用的数据库等信息。
2. 将命令还原成 Redis 网络通讯协议。
3. 将协议文本追加到 `aof_buf` 末尾。

### 文件写入和保存

每当服务器常规任务函数被执行、 或者事件处理器被执行时， aof.c/flushAppendOnlyFile 函数都会被调用， 这个函数执行以下两个工作：

WRITE：根据条件，将 aof_buf 中的缓存写入到 AOF 文件。

SAVE：根据条件，调用 fsync 或 fdatasync 函数，将 AOF 文件保存到磁盘中。

#### AOF的保存模式

Redis 目前支持三种 AOF 保存模式，它们分别是：

1. `AOF_FSYNC_NO` ：不保存。
2. `AOF_FSYNC_EVERYSEC` ：每一秒钟保存一次。
3. `AOF_FSYNC_ALWAYS` ：每执行一个命令保存一次。

##### 1. 不保存

在这种模式下， 每次调用 flushAppendOnlyFile 函数， WRITE 都会被执行， 但 SAVE 会被略过。

在这种模式下， SAVE 只会在以下任意一种情况中被执行：

1. Redis 被关闭
2. AOF 功能被关闭
3. 系统的写缓存被刷新（可能是缓存已经被写满，或者定期保存操作被执行

这三种情况下的 SAVE 操作都会引起 Redis **主进程阻塞**。

##### 2.每一秒钟保存一次

在这种模式中， SAVE 原则上每隔一秒钟就会执行一次， 因为 SAVE 操作是由后台子线程调用的， 所以它**不会引起服务器主进程阻塞**。

![每一秒保存一次](https://redisbook.readthedocs.io/en/latest/_images/graphviz-1b226a6d0f09ed1b61a30d899372834634b96504.svg)

##### 3.每一条命令保存一次

每次执行完一个命令之后，write和save都会被执行一次，此时redis主进程是会被堵塞的

# redis的一些坑的总结

1.使用`Jedis`连接redis的时候，必须设置redis的密码，否则会和我一样浪费很长时间来找错:slightly_smiling_face: 而且也不用通过`ssh`通道的方式来连接redis，不过我有点好奇，那些`redis`的管理工具是怎么实现ssh方式连接redis，还能够不用密码的。

