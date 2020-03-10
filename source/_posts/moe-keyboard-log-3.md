---
title: 「Keyboard Moe」从零开始自制键盘（三）：分区
date: 2020-03-11 01:48:08
tags: [键盘]
---

ESP32 的片上 Flash 应该有 4MB，但是我之前刷入 ROM 的时候，提示可用的空间只有 1280KB，这让我非常困扰。引入一个 BLE 库就已经吃掉了 75% 的 Flash 空间，我一度怀疑我是不是能在有限的空间里完成程序。ESP32 的一个很有意思的特性就是非常容易实现基于 WiFi 的 OTA 固件升级系统。于是我今天研究了一下，结果一研究明白了之前问题的原因，简单来说就是分区表。

PlatformIO 默认的分区表如下：

```csv
# Name,   Type, SubType, Offset,  Size, Flags
nvs,      data, nvs,     0x9000,  0x5000,
otadata,  data, ota,     0xe000,  0x2000,
app0,     app,  ota_0,   0x10000, 0x140000,
app1,     app,  ota_1,   0x150000,0x140000,
spiffs,   data, spiffs,  0x290000,0x170000,
```

也就是说，因为要支持 OTA，所以固件被分成了两块，每块 `0x140000 bytes`，然后他们有一个共享的 spiffs 分区（SPI Flash Filing System），这可以之后被我们用来存储配置文件。既然我觉得不够用，那我还是先来调整一下分区吧，我改过后的分区如下：

```csv
# Name,   Type, SubType, Offset,  Size, Flags
nvs,      data, nvs,     0x9000,  0x5000,
otadata,  data, ota,     0xe000,  0x2000,
app0,     app,  ota_0,   0x10000, 0x180000,
app1,     app,  ota_1,   0x190000,0x180000,
spiffs,   data, spiffs,  0x310000,0x90000,
```

应该我的配置文件不会超过 576KB，所以给固件的两个分区每个都多分类了 256KB。固件的可用容量扩大到了 1536KB，缓解了燃眉之急。
