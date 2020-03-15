---
title: 6 小时重构一个字幕工具
date: 2020-03-15 10:20:37
tags: [字幕, YouTube]
---

## 背景

某位来自伊朗，先居住在加拿大温哥华的翻译人员曾说过：「ass 能解决所有问题，对于我来说。」

![lodz](/static/lodz.jpg)

[Aegisub](http://www.aegisub.org/) 确实是一个非常好用的工具，提供了非常丰富的字幕特效，以及相对高的字幕效率，除了程序很容易崩溃。但对于简单的纯文本字幕，其实流程是可以进一步被优化的。

## 字幕流程

字幕流程其实就三个部分：文字稿整理（听写或翻译）、打轴、校对。我们可以围绕这三个流程来设计一套最简单最高效的工作流。昨天看 [NiceChord 好和弦](https://www.youtube.com/watch?v=Ath3BX9DBRs)的时候发现 wiwi 老师写了个类似想法的工具，主要是围绕提高打轴效率的一套程序。我尝试用了一下这套工具，想法是很好，但这代码是写的有点糟糕，介面也基本没有。

于是我考虑重构。重构主要目的如下：

- 用 Vue.js 实现一套可用的 UI
- 默认提供视频的播放控制
- 移除运行时的 Node.js 和 http-server 依赖
- 基于 File API 读取字幕文件和视频文件
- 基于 GitHub Actions 和 GitHub Pages 实现自动部署
- 基于 MIT 开源协议
- 提供 ESLint 代码风格 Linting
- 新增动态响应实现控制
- 新增 i18n 支持
- 新增字幕的实时预览功能
- 新增 SRT 字幕编辑有效性校验
- 新增对 WebVTT 格式的支持

重新实现所有已有功能大约花了 2 小时，加的那堆新功能花了 4 小时。共计 6 小时把事情给搞完了。

[Live Demo](https://srt.coderemixer.com/)

[GitHub Repo](https://github.com/dsh0416/fast-srt-subtitle/)

[GitHub Pull Request](https://github.com/wiwikuan/fast-srt-subtitle/pull/4)

## 已知 bug

发现 Firefox 的 CC 字幕 track 不能加载 blob url。这使得实时预览功能在 Firefox 上存在一定的问题，但 Chrome 和 Safari 都没有问题。准备这两天准备一下各种细节情况，向 Mozilla 回报一下。

其实也可以用 Electron 封装一下离线使用。

美其名曰，统一体验。

实则内存爆灹。
