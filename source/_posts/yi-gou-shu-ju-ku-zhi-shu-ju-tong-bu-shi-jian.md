---
title: 异构数据库之数据同步实践
date: 2017-03-23 00:18:04
tags: [Node.js]
author: xizhibei
---
一般来说，我们的不少业务中，需要用到数据同步，而其中涉及到的本质无非是 **『数据一致性』**。首先，能想到的数据同步的例子肯定是数据库，由于数据库领域存在的 CAP 理论，一定会有数据同步的过程来达到数据一致性，只是那是属于相同数据库之间的同步，在不同数据库的情况下同步数据的话，叫做 **『异构同步』**。

不过目前先放下一致性原理，以现有的一个业务场景为例，来说说如何实现异构同步。

### 业务场景
线上有一个采取了分表与分库的 MySQL 数据库集群，主要存放订单等数据，业务上想要看到订单的统计结果，但是如果直接在集群上跑脚本的话，容易将数据库拖垮，因此，需要同步数据至其它数据源进行统计，比如 ES、HBase 等。

### 解决方案

#### 完整克隆
这个最容易想到，数据库 A 完整导出数据，然后导入至数据库 B，缺点显而易见，不适用于持续增长的数据，可以勉强适用于每天导出部分数据，然后用 Excel 统计。（对的，一开始业务部门要数据的时候，你就会这么干，如果你还在这么干的话，考虑下其它方案吧，后期他们一遍又一遍来找你导数据的时候，你会发飙的。）

#### 标记同步
如果业务简单，插入到数据库中的数据不会发生变化，即日志型数据表，这时候就可以根据标记，比如 ** 时间戳 ** 来同步，即使发生故障，也可以从上一次同步的位置开始重新同步。

#### 实时同步
即将线上产生的数据，实时写到两个数据库，显然会拖慢应用，可以考虑做成离线的后台任务队列去做。只是在数据实时性要求不是那么高的情况下，不建议使用。

#### 数据库日志同步
比如 MySQL 的 binlog，MongoDB 的 oplog，包含了所有的数据库记录变动记录，可以解析相应的 log 数据格式，来达到完全同步到其它数据库的目的。这种方法兼容程度高，几乎不会丢数据，只是可能需要较多的开发，工具的话也有，目前还没有尝试过。

需要提醒的一点是：如果是云服务器厂商提供的数据库实例，可能你无法获取相应的 log，就比如 MySQL 的 binlog 。即使你能获取，还有一点很头疼的是，你需要将 binlog 设置成 **row** 模式，才能去同步，这样会造成 binlog 非常大，高写入场景下不适合。


### 一些 Node.js 数据同步脚本实践经验
首先是几个比较好用的 cli 工具：

- [commander](https://github.com/tj/commander.js)：用来处理命令行参数，另外还有个最近出来，挺火的：[Caporal.js](https://github.com/mattallty/Caporal.js)；
- [chalk](https://github.com/chalk/chalk)：输出各种颜色，用来调节写脚本时候郁闷的心情；
- [progress](https://github.com/tj/node-progress)：进度条，不知道同步进度的时候会很蛋疼，而一直用 `console.log` 或者 `debug` 的话，容易输出一堆没用的 log 。（相信我，当你同事看到进度条在刷刷的动的时候，尤其是产品🐶，他们会用崇拜的眼神来看你的 😝）；

然后，注意内存，同步脚本的时候，这个脚本所占的内存会比较大，可以考虑设置大一些：

```js
node --max-old-space-size=2000 ./script.js
```

还有并发数，如果开的比较高，需要注意 ** 连接池 **。

最后，如果使用 http 协议发送数据，则需要在跑脚本的机器上设置文件句柄限制（数字自己调整即可，这里只是个例子）：

```bash
ulimit -n 100000000
```



***
原链接: https://github.com/xizhibei/blog/issues/43