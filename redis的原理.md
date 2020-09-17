# redis的底层原理

## string 的底层原理

类似于java中的 ArrayList 提前分配空间。当容量小于1mb的时候，扩容的动作是扩大到自身容量的2倍；当容量大于1mb的时候，每次扩容1mb；最大是512mb

## list 的底层原理

类似java的 LinkedList 但是不光全部是这个，而是一个 quicklist ,当元素比较少的时候，采用连续的存储结构存储元素，称之为压缩列表( ziplist )；当元素比较多的时候，才会改成 quicklist ，将多个 ziplist 使用双向链表串起来就是 quicklist.

## hash 的底层原理

类似java中的 HashMap ，无序字典，数组加链表。

在rehash的时候是采用渐进式hash，就是将原来的 hash 结构和新的 hash 结构同时保留，在后续的操作中逐渐将旧的hash结构迁移到新的hash结构上面去。

![图片1](http://redisbook.com/_images/graphviz-4c43eaf38cbca10d8d368a5144db6f3c69ab3d84.png)

![](http://redisbook.com/_images/graphviz-b91705b0d7a6c7fd5e37332a930534e0e136ae73.png)

![](http://redisbook.com/_images/graphviz-9e2996e6ca9665776062470cdac346e8fc255374.png)

![](http://redisbook.com/_images/graphviz-c871b5de1a7910aea237ca9dc86508b48da94769.png)

![](http://redisbook.com/_images/graphviz-3b31e4e08cc3e212f986039eb08ae77224cdeec9.png)

![](http://redisbook.com/_images/graphviz-86f810ac65c4e6ee58b17105dfeaa06973d8dd16.png)

## set 的底层原理

相当于 java中的 hashset 无序，唯一。

## zset 的底层原理

底层是跳跃列表。按照score排序，如果score相同，则按照value排序

## hyperloglog 的底层原理

应用场景，统计UV

![t40JQf.png](https://s1.ax1x.com/2020/06/09/t40JQf.png)

通过低位的连续0位maxbit记为K来推测随机数的数量N

pf的内存占用总是12kb的原因：在redis中 hyperloglog 的实现使用了 16384 个桶，即$2^{14}$，每个桶的maxbit需要6个bit，所以占用的内存就是 $2^{14}*\frac{6}{8}$，即12kb

## 布隆过滤器的底层原理

应用场景：比如新闻app，在推荐新闻的时候不会推荐已经看过的新闻，就可以使用布隆过滤器对推荐的新闻看看历史记录是不是在里面

采用大型位数组和3个无偏 hash 函数，计算三个hash(key)的值，在位数组对应位置标记上1，如果这个key存在的话，这三个位置都应该为1，有一个为0则返回不存在，但是并不是三个位置都为1则一定存在，可能是其他的key导致有一个位置为1的，所以位数组越稀疏，正确率越大。

# 哨兵选举机制

可参考[哨兵选举](https://blog.csdn.net/qq_28165595/article/details/104102693) 和 [官方文档](https://redis.io/topics/sentinel)

在哨兵配置的时候有下面几个选项

```shell
sentinel monitor <master-group-name> <ip> <port> <quorum>
sentinel monitor mymaster 127.0.0.1 6379 2 	# 选举主机
sentinel down-after-milliseconds mymaster 60000 # 主观下线的超时时间
sentinel failover-timeout mymaster 180000 # 
sentinel parallel-syncs mymaster 1

sentinel monitor resque 192.168.1.3 6380 4
sentinel down-after-milliseconds resque 10000
sentinel failover-timeout resque 180000
sentinel parallel-syncs resque 5
```

## 主观下线

每一个哨兵在一段时间内会向节点（包括主机和从机）发送心跳检测，如果在`down-after-milliseconds`这个时间内有节点没有向哨兵发送心跳包，则被检测出没有心跳包的哨兵认定该节点为主观下线。

## 客观下线

一个哨兵认定主观下线没用啊，必须要其他哨兵也觉得节点下线了才行，所以在数量**超过**`quorum`的哨兵觉得这个节点挂了，则这个节点就被认定为客观下线。

## 哨兵选举

那要是主机被认定为客观下线了，那谁来做老大就成了一个难题了，决定权在哨兵手里，所以哨兵要选一个老大出来，由这个老大来决定选哪个节点为主机，那这个老大怎么选呢？当一个哨兵确认了主机客观下线了之后，就会给自己投一票，然后会发送一个请求要其他的哨兵也投自己一票，如果得到的票数>= max(`quorum`和`Sentinel节点数/2+1`)，则该哨兵成为老大，如果没有选出来则进行下一次选举。

正常情况下，哪个 sentinel节点最先确认master客观下线，哪个sentinel节点就会成为执行故障转移的 leader。

## 选择主节点

不会选择不健康的节点

+ 主观下线的从机
+ 大于等于5s未响应的从机

对健康的从节点进行一个排序

1. 选择优先级最高（priority最小）的节点，如果priority = 0 则表示不能成为主机，如果相同则转到2
2. 选择复制主机最完整的节点，如果相等则转3
3. 选择runid最小的节点，这个不可能相等了

## 执行故障转移

+ 让选出来的新节点成为主机

+ 让其他的节点成为新主节点的从机
+ 将老主机设置为从机，并且会一直关注这个老主机，当它恢复的时候命令其去复制新的master

## 可能出现的问题

### 脑裂

#### 描述

就是在让其他节点变成新节点的从机的时候，原来的主机苏醒了并且还有节点连接在老主机上，这个时候就不知道怎么办了。

#### 解决方案

待办。。

