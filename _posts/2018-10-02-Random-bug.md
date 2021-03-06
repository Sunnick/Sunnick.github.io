---
layout:       post
title:        "随机数使用不当引发的生产bug"
date:         2018-10-02 12:00:00
author:       "顺仔"
header-img:   "img/in-post/random.jpg"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - java
    - 架构
    - 踩坑
---

前几天负责的理财产品线上出现问题：一客户赎回失败，查询交易记录时显示某条交易记录为其他人的卡号。

交易的链路如下：
![](https://user-gold-cdn.xitu.io/2019/3/13/16976fd80edee276?w=572&h=75&f=png&s=7301)

出现该问题后，我们对日志进行了分析，发现主站收到的两笔流水号完全相同，然而主站却没有做重复校验，将两笔订单（A和B）都发往基金系统，基金系统做了重复校验，收到A之后开始处理，收到B之后直接报错返回，A处理完后又正常返回。但是主站根据流水号更新数据库状态，却将两笔订单更新错了，导致客户的交易记录出错。

该问题虽然不会造成用户的资金损失或记账出错，但是交易记录出错会带来极差的用户体验，引发客户投诉，并对公司声誉带来不良影响。因此主站通过增加重复校验来解决此问题。

但是问题的根源在于为何会产生重复的流水号，只有从源头上消灭重复的流水号，该问题才算彻底解决，因此我们对代码进行了分析。

流水号由APP -server产生，并传入后续的交易。流水号生成代码如下：
![](https://user-gold-cdn.xitu.io/2019/3/13/16976ff88d944568?w=509&h=327&f=png&s=55196)

可以看出，流水号由13位时间戳+3位随机数+固定数字“38”组成。一般情况下，该规则生成的流水号是不会重复的，因为时间戳是精确到毫秒的。但是在高并发的情况下，同一毫秒收到多个请求，此时只能由三位随机数来保证流水号的唯一性。

虽然就单次请求来说，与同一毫秒内其它请求的流水号重复的几率极小，可以忽略。假设每一毫秒有2个请求，那么这两个请求的3位随机数重复的概率为1/1000，不重复的概率为999/1000（假设是这么大的概率，没有经过数学计算）。我们通过程序来看下流水号的重复概率：

![](https://user-gold-cdn.xitu.io/2019/3/13/169770030287d3b2?w=406&h=167&f=png&s=23792)

程序运行结果如下（为了方便查看，随机数加了-用来分隔）：
![](https://user-gold-cdn.xitu.io/2019/3/13/169770098bc09be0?w=199&h=198&f=png&s=32360)

程序运行多次，也无法复现流水号重复的问题。但无法复现不代表没有问题，只能说明发生概率较小，因此需要调大循环次数。

循环次数调大后，log输出已无法靠肉眼去看是否重复，需要将每个流水号出现的次数存入Map，最后再看有多少个次数大于1的流水号。代码片段如下：
![](https://user-gold-cdn.xitu.io/2019/3/13/16977011c0507139?w=636&h=154&f=png&s=43036)
![](https://user-gold-cdn.xitu.io/2019/3/13/1697701596257a85?w=510&h=255&f=png&s=16566)
![](https://user-gold-cdn.xitu.io/2019/3/13/1697701a85e3acf1?w=607&h=459&f=png&s=89384)
执行以上代码，结果如下：
![](https://user-gold-cdn.xitu.io/2019/3/13/1697702145d5a8a3?w=293&h=336&f=png&s=16721)
可以看出，随着统计样本的扩大，出现重复的流水号的几率也在增加。也就是说，在系统长时间处于高并发的情况下，每一毫秒都会有重复的概率产生（如1/1000），随着时间的推移，在相当长的一段时间内，不发生重复的概率为999/1000 * 999/1000 * ........，不重复的概率越来越小，发生重复的概率越来越大。

如何避免发生重复呢？目前我想到的有以下几种方法：

* 使用数据库的自增id作为流水号，但这样会增加数据库IO开销，降低性能；
* 使用Redis存储流水号，每次使用时到Redis获取并加1，配合着分布式锁一同使用。同方案1一样，会增加IO开销，降低性能；
* 使用开源的发号器，如Snowflake等;
* 使用UUID，但UUID生成是字符串，不是数字，有些场景不一定适用。
 
**如果各位有好的想法，欢迎关注我的公众号（程序员顺仔）留言讨论～**
![](https://user-gold-cdn.xitu.io/2019/3/13/169770417414bccc?w=254&h=241&f=png&s=43837)