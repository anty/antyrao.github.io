# Implemtantion of Timer_set in Seastar Framework
# 索引
- 是什么
- 红黑树实现
- `Timer_set`实现
- 总结

------------

# 是什么
`Timer_set`是Seastar框架中用于管理定时器的一个数据结构, 作为一个定时器的管理结构,需要高效支持如下两个操作:
- 插入和删除定时器
- 给定一个时间点,得到所有超时定时器集合

红黑树作为定时器管理的一种常见数据结构

------------

# 红黑树实现
红黑树是一种有序的数据结构,插入时,定时器超时时间点作为key插入到红黑树中, 即在插入过程中对定时器进行排序. 由于是有序的, 所以能相对高效的支持上述几种操作,大部分应用场景下也是我们首选的数据结构

------------

# Timer_set实现
`Timer_set`的设计是基于这样一种观察: 大部分定时器,插入到定时器集合中后,其实很快就被删除了,并没有等到其超时发生.因为Seastar中所有IO操作都是异步的,那么需要有一个定时器来跟踪其超时,例如跟踪TCP发包的定时器,绝大部分情况下都不会发生超时.
所以`Timer_set`的设计更加偏重于插入和删除的效率,expire获取超时定时器集合效率次之
## 数据结构
`Timer_set`内部实现,类似一个hash table,主要包含两个组成部分
- `std::numeric_limits<timestamp_t>::digit + 1`数组桶(i.e.: 表示时间bit位总数,64位/32位)
- 每个数组桶成员是一个`boost::intrusive::list`结构(暂时可以简单对应为`std::list`)

## 插入
插入流程很straightforward:根据定时器的超时时间计算一个hash值,对应到数组中的一个位置,然后`push_back`到对应位置的list中. 其使用的hash函数包含很多的tricks,比较特殊 ,重点介绍一下,先看代码
```cpp
int get_index(timestamp_t timestamp) const
{
    if (timestamp <= _last)
    {
        return n_buckets - 1;
    }

    auto index = bitsets::count_leading_zeros(timestamp ^ _last);
    assert(index < n_buckets - 1);
    return index;
}
```
其他部分可以忽略,重点就一行
`auto index = bitsets::count_leading_zeros(timestamp ^ _last);`
其中`_last`标记为上次expire时获取超时定时器集合的输入,初始为0

当last为0时,取timestamp(定时器超时时间点)的leading zeros的数量作为index.这样timestamp值越大,对应index的值越小. 也就是说这个hash算法, 能够保留hash后的key的顺序性(逆序), e.g. :
```cpp
 timestamp   binary                                       index
     10      0x0000 0000 0000 0000 0000 0000 0000 1010     28
     20      0x0000 0000 0000 0000 0000 0000 0001 0100     27  
```
由于20 > 10, 20对应索引值肯定小于等于10对应索引值,绝对不会大于

但`_last`是可变的(必须单调递增的),粗看有点蒙,这个hash算法是如何保证当`_last`变化后,同一个key的hash位置不变的呢? 介绍其实现之前,还是先介绍一下改变`_last`值的expire超时过程

expire时,输入为一个时间点T, 作为新的`_last`.`Timer_set`筛选出所有expire的定时器,也就是所有定时器的expire时间点比给定时间点T小. 操作流程如下:
 - 根据输入时间点,通过hash计算得到一个索引值i1, 所有index> i1的list包含的定时器肯定都小于T,添加到超时集合
 - 同时遍历i1对应的list筛选出大于T的定时器(没有超时,需保留),重新插入(基于新的last值);所有小于T的定时器添加到超时集合
 - 所有index < i1的list包含的定时器超时时间肯定大于T,不会超时

介绍完expire过程,回到前面的问题,由于expire操作,`_last`被更新,那么一个问题:没有超时的定时器是如何被正确定位到的. 举个例子说明Hash计算过程:
一个大于T的定时器timer, 与新的`_last` = T 异或时, 由于timer比last大,分两种情况:
1. timer的最高位与last相同,这种情况下,表示timer与last位于同一个index,在expire过程中,timer会被重新插入
```cpp
    timer    0x00001101   
     last     0x00001001

            ^
            = 0x00000100
```
从索引位置32 - 4 = 28,重新插入到索引位置 32 - 3 = 29中.基于新`_last`,能够正常索引到timer
2. timer的最高位大于`_last`时,其实是不会影响最终的结果的
```cpp
timer       0x00001001               0x00001001
 last       0x0                      0x00000110
            ^                        ^
            = 28                     =   28
```
也就说虽然last变化了,不影响结果正确性

## `_last`的作用
last为什么要更新,不更新可不可以?
如果last不更新,一致为0. 那么由于定时器值的特点(一个时间值),大部分最高位其实都不变的,那么最后会hash到同一个list中,也就退化为一个无序的list,
执行expire操作时,需要遍历无序list筛选出所有超时定时器,效率差

但如果用当前时间now来expire `Timer_set`一次,last变为当前时间,
后续添加的定时器与last的间隔不会太大(大部分应用场景是这样),最高的几位大部分相同,异或后变为0,能够更加离散hash到不同的桶中

比如对于如下几个timestamp
```cppp
1476801191     0x0101 1000 0000 0110 0011 0010 1010 0111
1476801193     0x0101 1000 0000 0110 0011 0010 1010 1001
1476801195     0x0101 1000 0000 0110 0011 0010 1010 1011
1476801200     0x0101 1000 0000 0110 0011 0010 1011 0000
```
如果last不跟新,会被hash到同一个桶中,如果last更新为1476801191,则hash 1476801193和1476801195 hash到一个桶中, 1476801200 hash到另外一个桶中,当根据1476801195获取超时定时器集合时, 能够相对快速获取集合
如果时间精确到纳秒, 则超时时间被hash到不同桶的概率相对更高

------------

# 总结
回头说下`boost::intrusive::list `结构,其实是一个双向链表,插入和删除的时间复杂度为O(1),对于频繁的定时器插入和删除操作来说,是最高效的. 但对于expire来说,时间复杂度为O(n), 效果最差,`Timer_set`为了保留插入和删除的效率,而expire不至于太差,利用了一个hash的思想,而这个hash算法能够保证输入的顺序性,来提升expire效率
这种思路也是一种tradeoff,重点优化common case
