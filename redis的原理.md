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

底层是跳跃列表。

## hyperloglog 的底层原理

应用场景，统计UV

![t40JQf.png](https://s1.ax1x.com/2020/06/09/t40JQf.png)

通过低位的连续0位maxbit记为K来推测随机数的数量N

pf的内存占用总是12kb的原因：在redis中 hyperloglog 的实现使用了 16384 个桶，即$2^{14}$，每个桶的maxbit需要6个bit，所以占用的内存就是 $2^{14}*\frac{6}{8}$，即12kb

## 布隆过滤器的底层原理

应用场景：比如新闻app，在推荐新闻的时候不会推荐已经看过的新闻，就可以使用布隆过滤器对推荐的新闻看看历史记录是不是在里面

采用大型位数组和3个无偏 hash 函数，计算三个hash(key)的值，在位数组对应位置标记上1，如果这个key存在的话，这三个位置都应该为1，有一个为0则返回不存在，但是并不是三个位置都为1则一定存在，可能是其他的key导致有一个位置为1的，所以位数组越稀疏，正确率越大。

