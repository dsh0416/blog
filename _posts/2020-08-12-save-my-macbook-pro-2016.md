---
title: 拯救老婆 —— MacBook Pro 维修计划
date: 2020-08-12 18:19:58
tags: [Macbook Pro, 散热, 维修]
---

## 我的老婆不行了

我的 MacBook Pro 15 (late-2016) 真的不行了。

作为一款 16 GB 内存，i7-7820HQ CPU 的电脑，实在是卡得不行。特别是在我搬家后，机器已经卡到一个完全不能忍受的地步。`kernel_task` 进程自己能吃掉 6 GB 的内存，还能吃掉 250% 的 CPU，一丁点道理都没有。我打开 Activity Monitor 仔仔细细研究着这个 `kernel_task` 到底在干什么。慢慢地我发现了几个问题。

## 软件着手研究问题

首先是内存，我的各种后台程序即使在刚刚开机的情况下也能吃掉 > 20 GB 的内存。而物理内存只有 16 GB，这意味着 `swap` 介入了。这直接体现就是我的硬盘吞吐量惊人。运行一天 `kernel_task` 硬盘吞吐量高达 2TB。如果这么下去，先不说机器卡不卡，我对我 SSD 的寿命感到怀疑。特别是这一代的 SSD 并不是可更换的，一旦健康状况出问题，会变得非常麻烦。

说到硬盘我就发现了另一个问题，一旦我从 NAS 上大量拉数据到硬盘，CPU 占用也会狂飙。这时候我就想到 Mac 的一个芯片问题 —— Apple T1 Chip。在后续机型中，苹果使用的 T2 芯片是能够负责硬盘加解密的，而 T1 芯片其实主要就是 SMC (EC) 芯片的替代，只是加入了 TouchID 指纹芯片的一些特性。这意味着我的电脑的硬盘加解密是跑在 CPU 上的。在我 CPU 如此卡顿的情况下，我必须要从软件上先处理这一问题。

于是我把系统备份，然后重装成了 macOS Big Sur，同时关闭了 FileVault，暂停了全盘加解密。

![Big Sur](/assets/images/big-sur-beta.jpg)

## 是什么在导致 CPU 疯狂降频？

然而重装后虽然 `swap` 的问题减轻了，但是机器并不能算快。一旦运行一些负载稍微大一点的应用，比如 Chrome 或者 Firefox，机器还是会变卡。这意味着还有别的问题在困扰我的老婆。

既然主要问题来自 CPU，于是我去下载了一个 [Intel Power Gadget](https://software.intel.com/content/www/us/en/develop/articles/intel-power-gadget.html#attachment-heading) 来查看 CPU 的具体情况。结果发现，当 CPU 降频发生时，CPU 温度只有 60 多摄氏度，而风扇却已经满载了。严重时，频率降到了 1.0GHz，远远低于 CPU 的基础频率。这意味着，有别的东西在引发降频。正当我一筹莫展的时候，我把充电线一拔，突然 CPU 频率就回去了（？？？）

于是我尝试搜索了充电和降频的问题，果然很快就找到了一些[文章](https://www.forbes.com/sites/barrycollins/2020/04/24/why-you-shouldnt-charge-your-macbook-pro-from-the-left-hand-side/#15349b4f78ff)。在左侧的 Type-C 接口充电，会导致 MacBook Pro 过热降频。所以，你必须 Charge Your MacBook Pro “Right”。当我把充电线移动到右边后，至少 MacBook Pro 可以不那么卡了。

## 风扇转速为什么还是那么高？

然而在移动了充电线后，CPU 的温度倒是上去了，风扇一直在高负载转动，这也太奇怪了。如果我们在软件或者外设上找不到原因，可能硬件上确实出了问题。于是我先简单把外壳先拆了，发现整个壳子内的灰已经满到结成毛团了。在我用刷子先把壳子里的灰尘和风扇上的灰尘清理后，机器的降频现象得到了改善。

![Intel Gadget Result 1](/assets/images/intel-gadget-1.jpg)

## CPU 满载时为什么还会降频？

于是我开始对机器进行压力测试。可惜压力测试 5 分钟后， CPU 还是达到 100 摄氏度触发过热降频，虽然频率只会降到 2.2GHz，但我觉得鳍片上没有处理到的灰尘可能是原因。于是我决定给机器来个大扫除，清理灰尘，更换硅脂。

我打开 ifixit 开始看这台机器的[拆解](https://zh.ifixit.com/Guide/MacBook+Pro+15-Inch+Touch+Bar+Late+2016+Thermal+Paste+Replacement/131546?lang=en)。结果发现这款机器要想更换硅脂，几乎要把所有东西都拆了，要拆掉 114514 个螺丝。我想不行我还是找个店铺来帮我弄吧。于是开始找日本的 Mac 修理的店铺。结果发现光拆机器的手工费就要 15000 日元。看着这个价格，我想我一晚上的劳动也能省这 15000 日元吧。

于是我照着教程一个个把螺丝拆下来。果然鳍片上全是灰尘，而且苹果的这个便宜硅脂也已经发干了。最后我给 CPU 和 GPU 重新上了 MX-4 的硅脂，小心安装回去，一个螺丝也没有多。开机，轻松点亮。

## 我的老婆救回来了

更换完散热的结果如何呢？

![Intel Gadget Result 2](/assets/images/intel-gadget-2.jpg)

以默认风扇 profile，烤机 2 分钟风扇甚至都不会满载。而强制风扇满载后，CPU 烤机温度也只有 86 摄氏度，不但不降频，甚至还能睿频！至此，我的老婆终于被我抢救了回来，再也不用过着靠插管度日的生活，可以开开心心地安享她的下半生了。