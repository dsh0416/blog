---
title: 从零开始 Ruby on PHP
date: 2016-06-10 22:02:06 +0800
tags: [编程, Ruby, PHP]
---

## 关于什叶派 PHP 教徒
PHP 早期的设计意图下整个语言都是模板驱动的。也就是说主要就是写个静态页面，在适当的需要动态的场合插入一些短小的 CGI 代码。然而逊尼派 PHP 教徒确认为 PHP 可以变成一门大而全的语言，摇身一变，强行又是 MVC 又是 MVC 的。各种框架层出不穷，抄完 Spring 又抄 Rails，甚至在一门模板驱动的语言中再搞一个模板渲染引擎。简直是异教。

## Ruby on PHP
考虑到 Ruby 是一门强大的支持元编程的语言。很容易地我们能够将 Ruby 写成一门以模板来驱动的 CGI 语言。通过几行代码
```ruby
require 'sinatra'

get '/' do
  redirect '/index.php'
end

get /^\/(.*?)\.php$/ do |c|
  erb c.to_sym
end

post /^\/(.*?)\.php$/ do |c|
  erb c.to_sym
end
```
这样，只需要在 `./view` 下创建 erb 文件就可以当做 php 来写了，甚至连子目录都可以。
当然这样有一些小问题，比如，传进 erb 后连 request 来什么请求都不知道了。
所以我们可以稍作修改。
```ruby
require 'sinatra'

get '/' do
  redirect '/index.php'
end

get /^\/(.*?)\.php$/ do |c|
  erb c.to_sym, locals: {request: request}
end

post /^\/(.*?)\.php$/ do |c|
  erb c.to_sym, locals: {request: request}
end
```
这样我们就有了一切 request 的数据了。自己在里面继续二次处理就是了。

## 模板渲染
erb 模板是有趣的，但是像 PHP 那样一次性插十几行代码就不是很方便了。其实 erb 里也有这样的方法，只是 deprecated 而已。
```erb
<body>
  <%
    puts 'foo'
    puts 'bar'
  %>
</body>
```
然而问题来了，erb 里并不支持 echo 那样的命令。这样我们没法在一坨 Ruby 代码中输出某个东西到网页。
但是，Rails 里的 ActionView 不是支持 concat 吗？那么我们稍稍用点元编程的技巧，支持一下 echo 咯。
所以我们得到了最终代码
```ruby
# This is yet another PHP project
require 'sinatra'
require 'action_view'

lookup_context = ActionView::LookupContext.new('./views')
lookup_context.cache = false if development?
view_context = ActionView::Base.new(lookup_context)

module ActionView::Helpers::TextHelper
  def echo(string)
    output_buffer << string
  end
end

get '/' do
  redirect '/index.php'
end

get /^\/(.*?)\.php$/ do |c|
  view_context.render(file: c, locals: {request: request})
end

post /^\/(.*?)\.php$/ do |c|
  view_context.render(file: c, locals: {request: request})
end
```
我们也能在 erb 里愉快地 echo 了
```erb
<html>
  <body>
    Proudly Powered By 
    <br>
  <%
    echo 'PHP'
  %>
  </body>
</html>
```

## 副作用
根据上面的做法，我们把 Ruby 变成了一门什叶派 PHP 心目中 PHP 本该有的样子了。
顺便还带来了一点副作用那就是，Wappalyzer 把这个项目完全当做 PHP 项目了。
假如黑客试图用 PHP 的漏洞来攻击你，就等着看好戏吧。诶嘿~
