# spark mapstatus 原理

本文旨在分析 spark 中shuffle过程涉及的mapstatus 原理，以问题方式驱动

## mapstatus 在executor端是如何使用的

主要是在shuffle过程拉取数据时使用，代码可以参见BlockStoreShuffleReader。BlockStoreShuffleReader拉取的过程每次请求最多拉取配置大小的数据（默认是48M／5），所以maptask在mapstaus中的统计的写入大小有如下两个用途：

* 拉取时直接跳过size为0的block
* 拉取时用于计算一次请求拉多少block，由于我们配置了一个block manager一次请求拉取的数据量大小，所以我吗可以根据mapstatus的统计信息确认这次请求需要拉几个block



这里值得注意的是，在什么场景下空块会比较高？？可以预见的是当分桶数特别大时，而一个map处理的数据又比较小时容易出现。但实际情况如何需要测试验证。



另外一个问题是，spark为什么要这么设计？hadoop有mapstatus吗？如果没有他的拉取过程是如何做的？