---
title: MySQL 惊险恢复记
date: 2018-01-05 21:48:59 +0800
tags: [编程, 运维, MySQL, Ruby]
---

昨天接到朋友的一个电话，说服务器被攻击了。上去一看，Windows 主机弱密码，直接被远程登录中了勒索病毒，要价就是好几十比特币。所有文件都被加密。遇到这个问题，第一反应就是完蛋，直接恢复备份吧。然而这是台物理机，而不是云主机，备份没有那么方便，所以这台机器

**没有备份。**

这基本上已经基本判了死刑了，当时的解决是立刻切断服务器电源，挂载到安全的电脑上找一些还没来得及被加密的文件抢救。

## 初探

由于朋友当时不知道怎么拆 RAID 阵列，所以就直接把服务器整机搬到了我家里。🤦‍

不过后来我拆开一看，其实只是一个 RAID 1，所以任何一块盘的内容是一样的，随便拔一块下来。联想收购 ThinkServer 后干的最大的大好事就是把硬盘架上的 T6H 螺丝换成了标准的十字螺丝。（我之前在 ThinkServer 螺丝问题上的坑爹遭遇参考[此贴](/2017/02/15/build-up-x79-workstation)）

SATA 连上电脑检查文件。硬盘的核心文件是一个 MySQL 数据库，其它文件都是 stateless 的代码，倒是很容易恢复。根据恢复 MySQL 数据库的经验，第一反应是去找 MySQL 二进制日志文件。许多数据灾难最后都是通过二进制日志恢复的，而且这些已经都有非常成熟的工具可以直接使用。

然而找到 `log` 的目录的时候发现所有的 `.bin` 文件已经变成了 `.bin.RESERVE`，显然是已经被加密没救了。

## 转机

然而接下来，我突然发现，这个病毒的实现有个 Bug，一个实实在在的 Bug，为这个事情发现了转机。病毒没有加密所有文件，因为如果这样，它也无法发送自己的勒索信息，所以它保留了其自己生成出来文件名的文件不被加密。然而为了方便判断这个「自己生成」的概念，病毒是直接依靠后缀的。病毒保留了 `.html` `.txt` 和 `.log` 后缀的所有文件。

`.log`？这个通用的日志格式不也是 MySQL General Query Log 用的吗？虽然一般情况下，我们不会打开 MySQL General Query Log 这个那么低级别的 Log，**但是** 这台机器完全是一群不懂运维的人维护的。所有 MySQL 竟然是用 WAMP 全家桶安装的，还真的默认开启了 MySQL General Query Log。把 `mysql.log` 文件 dump 下来，果然完整叙述了这台机器运行一年间所有的连接和查询信息。依靠这个看来可以把所有数据都还原了！

## Oh General! My General!

MySQL General Query Log 虽然信息全，但并没有好用的直接把数据倒回去的工具（毕竟会开这个选项的可能性就很小吧）。那么没有就自己写咯。

简单写了个 MySQL General Query Log 恢复工具：[my_general](https://github.com/dsh0416/my-general)

这个工具的原理很简单，就是 parse log，然后判断这一行是不是一个 QUERY。然后简单判断了一下这个 QUERY 是不是一个 `SELECT` 或者 `SHOW`。如果是，就直接跳过；如果不是，那么就在数据库上运行一下。

然而这个还是有一些问题，有一些 QUERY 是在 `DATABASE` 层做的，有些是连接上具体 `DATABASE` 进行的。但是由于问题比较简单，我就先手动建立 `DATABASE`，然后过滤所有 `CREATE DATABASE` 和 `CREATE USER` 的操作。

```
> gem install my_general
> my_general -l mysql.log -d connection.yaml

Log File:           mysql.log
Database YAML:      connection.yml
[1/4] ⌚  Counting Complexity...
[1/4] ⌚  Counting Complexity... OK [Complexity: 17793561]
[2/4] 🔌  Dailing Database...
[2/4] 🔌  Dailing Database... OK [Connected]
[3/4] ⏳  Importing Data...
Progress: |=====================================================================|
[3/4] ⏳  Importing Data... OK [17793561/17793561]
[4/4] 🚩  Finished!
```

解决！

不过这玩意有一些问题，因为分隔符是换行号，如果主动比如在 `CREATE TABLE` 的时候自己拼一些换行号可能会爆炸。如果是用 ORM 做的 Migration 和日常使用，倒是没有这种特殊符号的问题。

## 结

做备份啊，做备份。备份不到用的时候永远被低估，一旦用到就悔恨万分。

保持良好的用机习惯，不要没事上服务器下片。这服务器上连天猫、淘宝的浏览记录都有，还把 IE 的安全级别调低了。

还有就是，遇到这种非常规的问题，不能紧张，保持乐观，要仔细寻找线索。

反正修不好是常态，万一修好了呢。（逃
