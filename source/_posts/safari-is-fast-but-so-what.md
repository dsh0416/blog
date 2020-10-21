---
title: 如果 Safari 不能做到正确的行为，何来的快？
date: 2020-10-21 11:52:12
tags: [Safari, Apple, Web, W3C, JavaScript]
---

For English version, click [here](/2020/10/21/safari-is-fast-but-so-what-english/)

## 一个困扰了一周的 bug

2016 年的一天，当我们发现 iPhone 上的浏览器不能正确通过我们的 CDN 鉴权后，我们花了数天的时间来 debug。简单来说当时的情况是，我们需要同时上传 3 个文件，我们会用用户 token 来换 3 个独立的随机数 id，这三个 id 会被 CDN 服务器认为合法，用户可以直接上传到 CDN 上而无需在我们自己服务器上中转。

但 iOS 用户很快就出现了一个奇怪的问题，用户 3 个文件只能成功上传 1 个，剩下 2 个无法正常上传。再进一步调试后我们发现，在上传任意一个文件后，剩下两个 id 变成了非法。再进一步地，我们发现 Safari 获得的 3 个 id 竟然是完全相同的？！

## 复现

我很快设计出了能够构建出这个问题的重现：

```ruby
require 'sinatra'

get '/' do
  <<-EOF
<html>
<script type="text/javascript">
function reqListener () {
  console.log(this.responseText);
}

for (i = 0; i < 3; i++) {
  var oReq = new XMLHttpRequest();
  oReq.addEventListener("load", reqListener);
  oReq.open("GET", "/count");
  oReq.send();
}
</script>
</html>
EOF
end

count = 0
get '/count' do
  count += 1
  count.to_s
end

```

在 Firefox 上，你会得到 1 2 3 的输出，而在 Safari 上你会得到 3 个 1。

针对同一个 API 接口，只要请求参数完全一致，并且在一个请求返回前，相同的请求已经被发出，那么这些请求都会得到完全相同的结果。

## 问题分析

一般来说我们认为 RESTful API 的 GET 是幂等的，如果我们把 x 当作服务器的状态，f 是 GET 请求操作，那么我们有：
$$
f(f(x)) = f(x)
$$
这确保了多次调用接口产生的副作用，和一次调用是一致的。这可以得到推论认为每次 GET 请求的返回都应该是一样的。但一方面这是 RESTful 风格的特性，而不是标准的 HTTP 请求的特性；另一方面一些接口必然无法满足这样的要求，比如统计接口和随机数接口。

如果我们假设这个幂等的可靠性，那么我们自然可以把请求参数进行哈希，从而提高事件回调时事件引擎的处理速度。但是显然这个假设是错的。但显然 Safari 做了相关的优化，从而导致了问题。

## Bug Report

![Screenshot](/static/safari-js-bug.png)

我在 2016 年通过 Apple 当时非常丑的 Feedback 系统提交了这个 bug。然而从这个 Feedback 系统升级到了 Feedback Assistant，Mac OS X 改名成了 macOS，从 El Capitan 升级到了 Big Sur，这个 Bug 不但在最新的 Safari 14.0.1 (16610.2.8.1.1) 依然存在。这个 Ticket 也没有得到任何回复。

## 结论

Safari 很快，Safari 效率很高，Safari 很省电。但如果连基本的 W3C Web API 的兼容性、可靠性都不能保证，我们怎么敢使用这个浏览器？但好在由于 App Store 的垄断性，iOS 设备被要求不允许使用第三方 Webview，包括 iOS 上的 Chrome 和 Firefox。从而迫使我们这些 Web 开发者为 Safari 的无下限进行妥协。就连邪恶的微软 IE，也没有敢利用操作系统的垄断来强制浏览器的规格。

Safari 何止是新的 IE，它比 IE 邪恶多了。Apple 根本就是自由互联网的摧毁者，这和中国开发者痛恨的微信浏览器又有什么本质上的区别呢？

F**k you, Apple.

