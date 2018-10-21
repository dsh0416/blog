---
title: Rack/Rails 服务器获取用户 IP 问题分享
date: 2017-12-26 21:58:59
tags: [编程, Ruby]
---

## 引

由于最近论坛发生了某些讨论，出现了一个恶意攻击的账号。然而去查用户 IP 时，却发现没有拿到用户的 IP。这个问题非常诡异，几经排查，终于发现了问题，问题同时发生在 Rack 和 Rails 的实现上，这意味着这个问题可能影响几乎所有 Ruby Web 服务器。故有必要写下此文，分享这一问题。

## 代理

我们先做一个假设问题，如果你是 Rack 的作者，你会如何获取客户端的 IP 地址？

最容易想到的就是直接获取 TCP 连接的 IP。这个在 Rack 中已经被 parse 成  `env['REMOTE_ADDR']` 了，于是直接获取这个值就可以了。这个做法基本上是对的，但是很快就遇到了问题。如果我的服务器躲在 HTTP 反向代理或者负载均衡之后，那么我获取到的岂不是都是代理的 IP 而不是真正的用户 IP 了吗？

早年有一些公司会自己设置 HTTP header 来解决这一问题。2004 年 IETF 通过了 [RFC 7239 标准](https://tools.ietf.org/html/rfc7239)，为这样的问题制定了通用的标准。简单来说，就是用 `X-Real-IP` header 标记用户的真实 IP，用 `X-Forwarded-For` header 标记途径路由。

这时候你可能会想 `X-Real-IP` 不是已经表示了用户的真实 IP 了吗？为什么还需要 `X-Forwarded-For` 呢？因为 `X-Real-IP` 可能是用户伪造的。当然这也取决于你的反向代理是怎么配置的，如果你在 nginx 中强写 `X-Real-IP` 其实没有这一问题。

但 Rack 作为一个通用的 Web 服务器框架，并不能相信用户会正确处理这个 header。Rack 的判断逻辑初衷是这样的：请求经过了一系列的节点，写了 `X-Forwarded-For` header，去掉可信任的路由，经过的最后一个路由，是用户 IP。如果用户伪造这个头，那也会是最后一个被最后一个内网路由加上其真实的 IP，从而避开攻击。

听起来很合理，但是这留下了一个问题：

- **哪些路由 IP 是可信任的？**

## 大于配置的约定

### 可信任 = 内网？

Rack 没有强制要求用户自己配置，其判断哪些是可信任的路由 IP 是通过判断 IP 是不是在内网下来的，这在大多数情况下，确实工作，所以我们可能根本不会注意到它。下面是代码：

```ruby
# https://github.com/rack/rack/blob/b0818268fd9f4026a1b0fb205cc1060ee38a23cc/lib/rack/request.rb#L419

def trusted_proxy?(ip)
  ip =~ /\A127\.0\.0\.1\Z|\A(10|172\.(1[6-9]|2[0-9]|30|31)|192\.168)\.|\A::1\Z|\Afd[0-9a-f]{2}:.+|\Alocalhost\Z|\Aunix\Z|\Aunix:/i
end
```

这一串正则可以说是相当的刺激，抛开后面的 unix 文件描述符、localhost 和 IPv6 不讲。前面一部分的正则非常取巧。因为在 IP 地址 3 种主要类型（A 类地址、B 类地址、C 类地址）里各有一部分可以作为私有地址。

A类地址：10.0.0.0～10.255.255.255 

B类地址：172.16.0.0～172.31.255.255 

C类地址：192.168.0.0～192.168.255.255

而 Rack 就是把它们过滤掉了，这些确实描述了可能的内网地址。无论你架在云服务商那里，还是自建机房，能分配的内网确实就是这些。这在大多数情况下工作，除非：

### 可信任 = { 内网, 可信任外网 }

如果你的代理跨内网，比如在多个可用区之间代理，又或者把服务器架在高防后，而高防和你的 IP 不在同一个内网，那你就悲剧了。

因为中间跨越了公网 IP，Rack 的正则没有 match 上，所以自然就视其为不可信任的 IP，直接取了这个 IP 作为客户端的 IP。

特别是后者，因为你切换高防是一个很仓促的事件，可能根本想不清这些事，而一旦在高防期间出了问题，就真的查不到 IP 了。

### 影响

由于这个问题出在 Ruby Web 最常用的框架 Rack 上。无论是 Sinatra、Cuba 还是 Rails 都是直接建在上面，Rails 的各种 gem 例如 Devise 也是基于这个假设是正确的。所以几乎所有的 Ruby Web 应用都会受影响。~~当然你裸写 TCP 本来就是要自己处理的，自然不受影响。~~

## 如何配置 Rack

于是，如果我们要配置，怎么配置呢？Rack 本身并没有直接暴露配置的接口，所以答案就只剩下一种：

**元编程**

例如我需要加入特殊可信任 IP 1.2.3.4，在你应用监听前，Rack 被 require 后（例如 Rails 的 initializer 里）加入形如以下的代码：

```ruby
module Rack
  class Request
    def trusted_proxy?(ip)
      ip =~ /\A127\.0\.0\.1\Z|\A(10|172\.(1[6-9]|2[0-9]|30|31)|192\.168)\.|\A::1\Z|\Afd[0-9a-f]{2}:.+|\Alocalhost\Z|\Aunix\Z|\Aunix:|\A1\.2\.3\.4Z/i
      # 自定义的判断函数，可信任的返回任意值，不可信任的返回 nil 即可。
    end
  end
end
```

另外需要注意的时，这里不适合从文件或网络 I/O 中获取判断的条件，因为这个部分会被非常频繁地调用，除非你自己实现缓存。

## Rails 用户的坑

如果你是 Rails 应用，以为做到上面就够了吗？你有没有注意到 Rails 的默认中间件中有一个 `ActionDispatch::RemoteIp`，你知道这是啥玩意吗？

Rails 也认识到了 Rack 有这个问题，为了方便配置，把获取 IP 的部分自己单独实现了一遍，默认也是信任内网，然后新建了一个参数： `request.remote_ip` 用来记录这个值。Rails 看似好意其实带来了不便，因为仍然有许多 gem 是基于 Rack 的  `request.ip` 写的，比如黑名单工具 Rack Attack，所以并不能帮我们节省配置。

所以你还要再配置一遍 Rails 才行。

在  `config/application.rb`  下执行如下代码：

```ruby
proxies = [] # proxies 这个参数接受多种形式的参数，详见 http://api.rubyonrails.org/classes/ActionDispatch/RemoteIp.html

config.middleware.swap ActionDispatch::RemoteIp, ActionDispatch::RemoteIp, true, proxies
```

解决！

**注：示例使用的是 Rails 5 的代码，Rails 3 和 4 的配置代码甚至还不一样🤦‍♀️**

旧版本详见 [Issue#5223](https://github.com/rails/rails/issues/5223) 这个被反复打开的 Issue。

## 结

这是一个很不容易遇到的场景，但一旦遇到又很容易被忽视，忽视后有可能产生很大的麻烦。而且说实话，平时哪怕读 Rack 和 Rails 的源代码也想不到会有这种场景。了解后各位 Ruby 开发者在平时开发中多注意一下吧！
