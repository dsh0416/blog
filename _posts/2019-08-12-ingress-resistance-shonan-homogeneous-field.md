---
title: 休闲魔蓝玩家的湘南单人四重蓝笋
date: 2019-08-12 21:29:28 +0900
tags: [Ingress, 完全四重]
---

作为一个蓝笋教徒，上一次也是唯一一次做蓝笋就是 <span style="color: rgb(0, 112, 192);">@ngiamzsjit</span> 设计的复旦单人完全四重任务。

那个任务我做了两次，第一次做到一半，被突然出现的绿军炸了。而且那个绿军确实是无心的，一边上课一边随手甩了两个炸。由于那个炸同时涉及一类点和二类点。让修复工作变得异常困难，最后随便连了一通就回家了。

第二次是在夜里做的，当时复旦内部正在施工，有好几个 po 非常难摸，但是做到凌晨还是顺利完工，只不过出校门的时候被复旦的保安问了半天。

![ingress-homo-badge](/assets/images/ingress-homo-badge.png)

然后我就咸鱼了一年。本来以为到了日本 portal 密度变高应该可以多玩玩，结果一年下来连家门口 50 米远的 IFS 都没去。于是就考虑在暑假趁着做个单人四重竹笋把学校盖了。

## 规划

我大概一个月前开始规划。我们庆应大学湘南藤泽校区在山上，又是休校时间，上山的唯一公共交通工具是公交车，最近的车站分别是 30 分钟车程的湘南台站和辻堂站。这意味着行动一旦开始，绿军就算要来阻挡，至少也需要大概半个小时的机动时间。

校区很小，校区内大约有 20 个左右的 portal，密度很高，理论上应该会好做，但是实际情况形状不够好，在这些 portal 中找不到可能的四重解。最后考虑引入大学在坡下入口处的纪念碑和学校后门几个神社，最后找到了一组解。

![ingress-homo-plan](/assets/images/ingress-homo-plan.png)

我一开始使用的是 <span style="color: rgb(0, 112, 192);">@NanoApe</span> 开发的 [Konano/Ingress-Field-Design](https://github.com/Konano/Ingress-Field-Design/) 工具进行规划在。运行过程中发现两个问题，一个是这个项目一开始设计给 Windows 执行的，程序完全由 GB2312 编码编写和编译。实际运行后发现由于我的 portal 名中有日语字符，直接 crash 了整个程序。最后我把编码都换成了 UTF-8 重新编译，输出了四重规划。

然后是输出单人四重的路径，[Konano/Ingress-Field-Design](https://github.com/Konano/Ingress-Field-Design/) 希望用户手动上传一个经过所有 portal 的路径。而我嫌麻烦，用 Mathematica 随手写了个脚本随手写了个枚举所有点 Dijkstra 找最短路的程序，算法时间复杂度是 O(N^2)。结果就被自己写的程序坑了，因为选定的好几组 portal 在两个坡上，这使得我之后行动时不停上坡下坡。

## 行动开始

在行动开始前有个好消息就是学校的整个区域被一条从农舍间的 portal 射出来的多个蓝色 field 盖场。这使得学校内 portal 在行动开始前完全是没有 link 的状态，这是完美的行动条件。

但不如人意的是我从 7 号开始感冒发烧，于是行动一直被推迟 ~~（反正画过就是做过）~~。一直到 10 号 JST 6:00 在双倍经验结束前几个小时，我实在睡不着，想出去散步清醒一下，于是当即决定开始行动。

首先我坐巴士上到山一半的远藤。在一片杂草中见到那个用来盖场的 portal，插八然后毒，接下来徒步上山。

![ingress-grass](/assets/images/ingress-grass.png)

## 插曲 1

然而行动刚刚开始一小时，我刚把第一个支点的 key 摸全，开始连线就被绿军 <span style="color: rgb(0, 176, 80);">@sicksQ </span> 打了。

![ingress-comm-1](/assets/images/ingress-comm-1.png)

这直接把我原来的计划打乱。于是我和 Intel 组正在帮我截图的 <span style="color: rgb(0, 112, 192);">@DarkKowalski</span> 开始用工地日语向对方询问。

![ingress-comm-2](/assets/images/ingress-comm-2.png)

对方随后表示接受，并且让我当心中暑，行动得以继续。但是 key 缺了几把，于是又不得不回去重新摸。根据攻击的范围判断，对方可能就是住在学校附近的住民，万万没想到千算万算在这里失算了。

![ingress-comm-3](/assets/images/ingress-comm-3.png)

一直到大约 9 点，问题 portal 基本修复完毕。而我因为这个感冒头昏脑胀，学校因为休校，关闭了自动贩卖机，我差点脱水。趁着修复的过程下山买了几瓶运动饮料灌下去又坐了一会，可以继续行动。

## 插曲 2

行动进行到大约 9:30 发现 Ingress Prime 有一个很麻烦的 bug。在 link 完成后，默认提示的下一个 link portal，经常图片没有刷新但是文字没有刷新。这使得我不小心连接错了一个 portal，还好是四类点影响不大，毒了重新摸，再重新连。

但是当我到最后一个 portal 的时候发现自己忘记带 SoftBank Ultra Link 了。一共要往外射 9 条 link。而我当时已经把支点连接完毕了。

当时 Intel 组的 <span style="color: rgb(0, 112, 192);">@DarkKowalski</span> 当即表示他有一个 pack 没有兑换，里面有 SoftBank Ultra Link 可以合法空投。这个 pack 放在 <span style="color: rgb(0, 112, 192);">@alexanderjunyin</span> 那里。在连续电话、微信电话、Telegram 电话各两通后，我确认，这人 **睡着 ** 了。

![telegram-sleep](/assets/images/telegram-sleep.png)

于是不得不毒了打掉。从坡下的另一个支点连过来。毒的过程中 8 个 key 只掉了 2 把，库存里还有 2 把，又绕学校一圈重新摸了 4 把。

最终终于完成了完全四重。

![ingress-homo-result](/assets/images/ingress-homo-result.gif)

![ingress-homo-result](/assets/images/ingress-homo-result.png)

## 感想

虽然最后没有在双倍经验结束前做完，但还是坚持做完了这个非常酷的完全四重。不过我下次再也不想在生病的时候去做这种大任务了，感觉真的要死。另外那位放我一马的重生绿军大佬 <span style="color: rgb(0, 176, 80);"> @sicksQ </span> 觉得这个 CF 很好看，一直到晚上都没有来打。

这里不愧是乡下，做了一个完全四重 COMM 里完全寂静，而且也没有人约饭。最后只好去便利店买了两个三明治，再去药店买了一盒感冒药回家了。

不过下个月就要回国，原本考虑做超级大坑清明上河图，但似乎里面有好几个任务没有办法摸到。于是考虑一下是不是有可能去做个单人五重。之前虽然做过 <span style="color: rgb(0, 112, 192);">@ngiamzsjit</span> 的教程，但是觉得对于多重的理解并不是很深刻，在这次自己把自己差点坑死后，感觉自己对完全多重的理解进步很大。


## 总结

地面组：<span style="color: rgb(0, 112, 192);">@DELTOND</span>

清障组：<span style="color: rgb(0, 112, 192);">@DELTOND</span>

提供 Key：<span style="color: rgb(0, 112, 192);">@DELTOND</span>

规划程序协力：<span style="color: rgb(0, 112, 192);">@NanoApe</span>

Intel & 截图：<span style="color: rgb(0, 112, 192);">@DarkKowalski</span>

感谢绿军：<span style="color: rgb(0, 176, 80);">@sicksQ </span>

睡觉组：<span style="color: rgb(0, 112, 192);">@alexanderjunyin</span>
