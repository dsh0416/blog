---
title: 如果 Safari 做不到对，快有何用？
date: 2020-10-21 11:52:12 +0900
tags: [Safari, Apple, Web, W3C, JavaScript]
---

For English version, click [here](/2020/10/21/safari-is-fast-but-so-what-english)

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

HTTP/1.1 规格上 API 的 GET 是幂等的，如果我们把 x 当作服务器的状态，f 是 GET 请求操作，那么我们有：

$$
f(f(x)) = f(x)
$$

这确保了多次调用接口产生的副作用，和一次调用是一致的。我们似乎可以得到推论认为每次 GET 请求的返回都应该是一样的。

但如果我们仔细来看 [rfc7231](https://tools.ietf.org/html/rfc7231#section-4.2.2) 对于 HTTP 幂等的定义，其只是为了确保请求重新发送的可靠性，而不是不允许后端进行任何非幂等的操作。

如果我们把 GET 换成 POST 结果如何呢？

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
  oReq.open("POST", "/count");
  oReq.send();
}
</script>
</html>
EOF
end

post '/count' do
  count += 1
  count.to_s
end
```

竟然得到的也是三个 1！任何关于 HTTP 的规格都没有这样的描述，这对于 HTTP 动词的基本概念相违背，显然会带来非常严重的问题。

我们检查一下后端的输出，只有一个 1。也就是这三个 POST 请求被 Safari 缓存了？！

一些接口必然无法满足幂等的要求，比如统计接口和随机数接口。

如果我们假设这个幂等的可靠性，那么我们自然可以把请求参数进行哈希，缓存降低响应时间，以及提高事件回调时事件引擎的处理速度。但是显然这个假设是错的。但显然 Safari 做了相关的优化，从而导致了问题。

## Bug Report

![Screenshot](/assets/images/safari-js-bug.png)

如果仔细找一下会发现，2012 年左右开始几乎每年都有人在网上问 Safari cache POST 请求和 Safari cache GET requests with cache disabled 的问题。

我在 2016 年通过 Apple 当时非常丑的 Feedback 系统提交了这个 bug。然而从这个 Feedback 系统升级到了 Feedback Assistant，Mac OS X 改名成了 macOS，从 El Capitan 升级到了 Big Sur，这个 Bug 不但在最新的 Safari 14.0.1 (16610.2.8.1.1) 依然存在。这个 Ticket 也没有得到任何回复。

## 结论

Safari 很快，Safari 效率很高，Safari 很省电。但如果连基本的 W3C Web API 的兼容性、可靠性都不能保证，我们怎么敢使用这个浏览器？但好在由于 App Store 的垄断性，iOS 设备被要求不允许使用第三方 Webview，包括 iOS 上的 Chrome 和 Firefox。在 iOS 之前不会有人理 Safari 的，但现在我们这些 Web 开发者被迫为 Safari 的无下限进行妥协。就连邪恶的微软 IE，也没有敢利用操作系统的垄断来强制浏览器的规格。

Safari 何止是新的 IE，它比 IE 邪恶多了。Apple 根本就是自由互联网的摧毁者，这和中国开发者痛恨的微信浏览器又有什么本质上的区别呢？

F**k you, Apple.

