---
title: 「Keyboard Moe」从零开始自制键盘（一）：硬件设计
date: 2020-02-16 15:43:03 +0900
tags: 键盘
---

## 需求探究

我从去年就考虑做一把自己的键盘。其需要满足下面的需求：

- 便携性：提供不少于 70 键的键盘矩阵，支持蓝牙连接。
- 电池管理：能够对锂电池进行充放电，并监控电池电量状态。
- 接口现代：支持 Type-C 接口，可以自动在蓝牙和 USB HID 模式切换（方便装机等场景）。
- 矢量控制：提供一个类似于小红点的推杆，可以临时替代鼠标。
- 人体工学布局：由于我的打字键位并不是非常标准，导致现有的人体工学布局用起来非常难受。所以我考虑做一个有两排重复按键的人体工学配列。

一个简单的想法就是基于开源的 GH60 方案二次开发。我手上有一把白嫖来的 [Chicory](https://help.ydkb.io/doku.php?id=en:keyboards:chicory)，其设计基本还是和 GH60 一致的围绕 Atmega32u4 设计的方案，蓝牙部分再单独连接到 MDBT40 模组（nRF51822）。这个方案的优缺点很明显，优点是可以很好沿用 AVR 相关的软件栈（特别是 [tmk_keyboard](https://github.com/tmk/tmk_keyboard)）。缺点方面一个是贵，官方套件卖到了 120 美金，光 MDBT40-256RV3 模组就要 7 美金；另一方面 GH60 方案已经几乎把 Atmega32u4 的 I/O 口用完了，要想再加一些我上面的那些功能就很困难。

于是本着吃螃蟹的想法，我决定自己设计一个键盘方案出来。我自己没有专业学习过模拟电路、数字电路，所以也是一遍做一遍学，折腾了大半个月，在去年的 9 月左右进行了第一次的打样。打样的结果非常失败，由于一些设计缺陷，导致最后连上电都存在问题。

![第一版设计图](/assets/images/pcb-moe-legacy.jpg)

芯片方案这边选的是 ESP32，虽然这芯片本职是做 WiFi 的，但是作为蓝牙芯片也是非常便宜的。但是 ESP32 不支持 USB 主控，于是最后又迫不得已加了 Atmega32u4 来做 USB 主控。实际拿到生产的板子发现，USB Type-C 接口沉板没做，硬飞上去发现 CC 没有接下拉电阻导致识别也有问题。在我的设计中，ESP32 芯片的固件会由 Atmega32u4 刷入，光整明白这个刷机流程就非常困难。耦合度太高，导致调试非常困难，光布线就遇到很多难题，最后 PCB 弄了四层板进一步提高了生产成本。最后 BOM 单下来，一片得有个好几百。就算不考虑钱的问题，这键盘的尺寸很尴尬，上面一堆贴片件，开钢网又特别大，焊一次就得一天，一直试错下去也不是个办法。

## Moe (Modular Open Ergonomics)

在第一次失败后，我就考虑把设计拆成两个部分，进行模块化设计。

一个是具备所有逻辑功能的核心板 (moe-core) 和键盘矩阵灯阵板 (moe-71)。核心板包括电源管理、蓝牙管理、USB 管理、固件管理等所有功能，通过一个 FPC 连接器连接到矩阵板上。这样我可以先优先调试核心板，在面包板上随便搭个矩阵就能调试。FPC 提供 1A 的 5V、800mA 的 3.3V、800mA 的 GND 以及 4 个 ADC pin（用来做小红点推杆），剩下的就是一个 20 个键盘矩阵（默认配置 15x5）。所以也方便之后出不同配列的键盘，只需要换个矩阵板就行，不需要再画那堆复杂的逻辑功能。新的 MCU 选型还是 ESP32，但是去掉了 Atmega32u4。

Atmega32u4 首先不便宜，对于那个性能来说，非常划不来。相反，我上了一片 [CH9329](http://www.wch.cn/products/CH9329.html)，芯片内已经做好了 HID 鼠标和键盘的封装，BOM 成本只有 3.8 元一片，非常划算。

另外还给这次的板子加上了一片 [CP2102](https://www.silabs.com/products/development-tools/software/usb-to-uart-bridge-vcp-drivers)，从而提供了板上的 USB 刷机口，即插即刷。还有一点吸取教训的，就是在一些辅助功能上都加了一个 0 Ohm 的贴片电阻。如果测试下来硬件上有问题，可以直接把电阻撬了，也不影响其它功能的调试。

电源方面基本就找了个充电宝的外围芯片 IP5206，能做到 1.5A 充电和 2A 放电，还自带电量查询和手电筒功能（虽然被我串了个电容禁用了，应该没有人想要把键盘当手电筒用）。出来后接到 LDO 上，做一个 3.3V 1A 的输出。

最后设计出来的核心板的尺寸是 14cm x 3cm。如果使用 [nRF5340](https://www.nordicsemi.com/Products/Low-power-short-range-wireless/nRF5340) 的方案应该尺寸能进一步减少，还能用上蓝牙 5.1 的新特性。就是 nRF5340 太贵，而且模块难买（业余项目没有精力去调射频参数）就放弃了。

花了一上午来布线，最后折腾来折腾去，把布线控制在了两层板里，于是不需要用到四层板进一步降低了成本。最后合算下来的 BOM 成本在每套 45 元，加上板子和 SMT 的钱，核心板套件的成本在 65 元以内。比 Chicory 功能更丰富，加上配列板的推荐成本还是能控制在 90 元以内，还是颇有价格优势的。

![电路图](/assets/images/moe-core-feb.png)

![正面](/assets/images/pcb-moe-feb-a.png)

![背面](/assets/images/pcb-moe-feb-b.png)

工程（MIT 协议）：[https://oshwhub.com/dsh0416/moe-core](https://oshwhub.com/dsh0416/moe-core)

## 挖坑

等疫情过去了流出来调试。

- 2019.2.17 更新，加了一个退耦电容以防万一。修正了螺丝孔位的位置，并使固定螺丝焊盘接地。
- 2019.2.18 更新，修改了三个肖特基二极管以支持充电电压。