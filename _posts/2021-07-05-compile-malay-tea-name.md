---
title: 三点几嚟，饮茶先啦 —— 将大马饮料名编译成汉语
date: 2021-07-05 12:44:49 +0800
tags: [编译, Ruby, 语言学]
---

近几日马来西亚网友 Gurdip Singh 在 Facebook 发的这个「三点几嚟，饮茶先啦」非常流行。

Facebook: www.facebook.com/100009201465316/videos/2530411593942198

Bilibili 搬运：https://www.bilibili.com/video/av845257746/

不过马来语的饮料名称非常有意思。某个目前居住在新加坡的朋友 @david92 给我解释了一下，如何在店里点茶喝。基本是一个组合式的语法，非常规律。最基础的茶底是红茶（Teh）或者咖啡（Kopi）。默认饮料是带糖和炼乳的，但你可以重新定制。如果你在后面加上 O 表示不要加炼乳，而 C 表示把炼乳换成鲜奶。类似，Kosong 是无糖，Siu Dai 是少糖，而 Gah Dai 是加更多的糖。

## ebnf 语法

我查阅了一些资料进一步完善了一下这个概念，发现这个语法完全是「可编译」的，非常简单。很快，我写了一个 bnf 语法来描述这个概念：

```ebnf
<water>     ::= "Kopi" | "Teh" | "Milo"
<sugar>     ::= "Kosong" | "Siu Dai" | "Gah Dai"
<milk>      ::= "O" | "C"
<thickness> ::= "Po" | "Gau"
<extra>     ::= "Peng" | "Bubble" | "Halia"
<upsize>    ::= "Nga Lat"
<takeout>   ::= "Bungkus" 
<plastic>   ::= "Ikat"
<knot>      ::= "Mati" | "Tepi"

<drink>     ::=
  (<takeout> (" " <plastic> (" " <knot>)?)? " ")?
  (<upsize> " ")?
  <water> (" " <milk>)? (" " <sugar>)?
  (" " <thickness>)?
  (" " <extra>)*
```

其中，Po 是清淡，Gau 是浓缩，Peng 是加冰块，Bubble 是加珍珠，Halia 是加姜汁。Nga Lat 是大杯。外带的概念比校复杂，Bungkus 是外带，通常是杯状的。像是 Bernard Tee 视频里那种塑料袋装的，叫 Ikat。Ikat 有两种不同的打结方式，一种是打死结 Mati，还有一种是侧面打结，开口的叫 Tepi。

比如 Bungkus Ikat Mati Nga Lat Kopi O Siu Dai Gau Peng Bubble 就是外带塑料袋装打死结大杯少糖浓缩咖啡加冰块和珍珠。

我们可以在 [ebnf playground](https://bnfplayground.pauliankline.com/) 测试这个语法。这个网站甚至能生成随机的符合某个语法（比如这里 Drink）的字符串，来让我们人工检查这个语法对不对。于是截止这里我们还可以实现一个自动生成饮料名的生成器：https://www.bilibili.com/video/BV1SK4y197hc

## 实现到汉语的编译器

Ruby 中有一个 gem 叫 ebnf 可以读取 ebnf 文件然后生成对应的 parser，然后我们写一个输出中文的 generator 即可将马来西亚语翻译成中文。

由于 ebnf 的语法并没有规范，Ruby ebnf 库和我们刚刚 playground 中的语法有细微不同，这里做了一些变更。

```ruby
require "ebnf"

TEA_GRAMMER = <<-EOF
Water     ::= "Kopi" | "Teh" | "Milo"
Sugar     ::= "Kosong" | "Siu Dai" | "Gah Dai"
Milk      ::= "O" | "C"
Thickness ::= "Po" | "Gau"
Extra     ::= "Peng" | "Bubble" | "Halia"
Upsize    ::= "Nga Lat"
Knot      ::= "Mati" | "Tepi"
Plastic   ::= "Ikat" Knot?
Takeout   ::= "Bungkus" Plastic?

Drink     ::= Takeout? Upsize? Water Milk? Sugar? Thickness? Extra*
EOF
```

由于这个库执行 parse 需要先转换成解析表达文法（Parsing Expression Grammar）从而生成更多的子规则，我们先打印一下自动生成的子规则便于之后开发 generator。

```ruby
EBNF.parse(TEA_GRAMMER).make_peg.ast

=begin
(rule Water (alt "Kopi" "Teh" "Milo"))
(rule Sugar (alt "Kosong" "Siu Dai" "Gah Dai"))
(rule Milk (alt "O" "C"))
(rule Thickness (alt "Po" "Gau"))
(rule Extra (alt "Peng" "Bubble" "Halia"))
(rule Upsize (seq "Nga Lat"))
(rule Knot (alt "Mati" "Tepi"))
(rule Plastic (seq "Ikat" _Plastic_1))
(rule _Plastic_1 (opt Knot))
(rule Takeout (seq "Bungkus" _Takeout_1))
(rule _Takeout_1 (opt Plastic))
(rule Drink (seq _Drink_1 _Drink_2 Water _Drink_3 _Drink_4 _Drink_5 _Drink_6))
(rule _Drink_1 (opt Takeout))
(rule _Drink_2 (opt Upsize))
(rule _Drink_3 (opt Milk))
(rule _Drink_4 (opt Sugar))
(rule _Drink_5 (opt Thickness))
(rule _Drink_6 (star Extra))
=end
```

我们创建一个 generator 类，如下：

```ruby
class MalayTea
  include EBNF::PEG::Parser
  attr_reader :rules
   def initialize
    @rules = EBNF.parse(TEA_GRAMMER).make_peg.ast
  end

  def evaluate(input)
    parse(input, :Drink, @rules)
  end
end

p MalayTea.new.evaluate(gets.chomp)
```

 这时候输入一个句子，其会如实输出 AST 抽象语法树。而我们要做的就是根据 rule 名称来规约这些语法树直到某个特定输出。

```ruby
production(:_Drink_1, clear_packrat: true) do |value|
    if value.nil?
        ""
    elsif value.length == 1
        "外带"
    else
        "外带#{value[-1].values.join}"
    end
end

production(:Plastic, clear_packrat: true) do |value|
    if value.nil?
        ""
    elsif value.length == 1
        "塑料袋装"
    else
        "塑料袋装#{value[-1].values.join}"
    end
end

production(:_Plastic_1, clear_packrat: true) do |value|
    value.nil? ? "" : { Mati: "打死结", Tepi: "侧面打结"}[value.to_sym]
end

production(:_Drink_2, clear_packrat: true) do |value|
    value.nil? ? "" : "大杯"
end

production(:Water, clear_packrat: true) do |value|
    { Kopi: "咖啡", Teh: "红茶", Milo: "美禄" }[value.to_sym]
end

production(:_Drink_3, clear_packrat: true) do |value|
    value.nil? ? "炼乳" : { O: "", C: "鲜奶" }[value.to_sym]
end

production(:_Drink_4, clear_packrat: true) do |value|
    value.nil? ? "" : { Kosong: "无糖", "Siu Dai": "少糖", "Gah Dai": "加糖"}[value.to_sym]
end

production(:_Drink_5, clear_packrat: true) do |value|
    value.nil? ? "" : { Gau: "浓缩", Po: "清淡"}[value.to_sym]
end

production(:_Drink_6, clear_packrat: true) do |value|
    extras = value.map do |a|
        { Peng: "冰块", Bubble: "珍珠", Halia: "姜汁" }[a.to_sym]
    end
    extras.empty? ? "" : "加#{extras.join}"
end
```

我们规约最后一步的时候需要特殊处理鲜奶红茶和鲜奶咖啡这两个词语，因为一般我们直接叫奶茶和咖啡拿铁。

```ruby
production(:Drink, clear_packrat: true) do |value|
    h = value.inject(&:merge)
    if h[:Water] == "红茶" and h[:_Drink_3] == "鲜奶"
        h[:Water] = "奶茶"
        h[:_Drink_3] = ""
    elsif h[:Water] == "咖啡" and h[:_Drink_3] == "鲜奶"
        h[:Water] = "咖啡拿铁"
        h[:_Drink_3] = ""
    end
    "#{h[:_Drink_1]}#{h[:_Drink_2]}#{h[:_Drink_4]}#{h[:_Drink_3]}#{h[:_Drink_5]}#{h[:Water]}#{h[:_Drink_6]}"
end
```

## 成果

最后我们得到了一个将马来西亚语的饮料名编译到汉语的编译（翻译）器。

完整代码如下：

```ruby
require "ebnf"

TEA_GRAMMER = <<-EOF
Water     ::= "Kopi" | "Teh" | "Milo"
Sugar     ::= "Kosong" | "Siu Dai" | "Gah Dai"
Milk      ::= "O" | "C"
Thickness ::= "Po" | "Gau"
Extra     ::= "Peng" | "Bubble" | "Halia"
Upsize    ::= "Nga Lat"
Knot      ::= "Mati" | "Tepi"
Plastic   ::= "Ikat" Knot?
Takeout   ::= "Bungkus" Plastic?

Drink     ::= Takeout? Upsize? Water Milk? Sugar? Thickness? Extra*
EOF

class MalayTea
  include EBNF::PEG::Parser
  attr_reader :rules

  production(:_Drink_1, clear_packrat: true) do |value|
    if value.nil?
      ""
    elsif value.length == 1
      "外带"
    else
      "外带#{value[-1].values.join}"
    end
  end

  production(:Plastic, clear_packrat: true) do |value|
    if value.nil?
      ""
    elsif value.length == 1
      "塑料袋装"
    else
      "塑料袋装#{value[-1].values.join}"
    end
  end

  production(:_Plastic_1, clear_packrat: true) do |value|
    value.nil? ? "" : { Mati: "打死结", Tepi: "侧面打结"}[value.to_sym]
  end

  production(:_Drink_2, clear_packrat: true) do |value|
    value.nil? ? "" : "大杯"
  end

  production(:Water, clear_packrat: true) do |value|
    { Kopi: "咖啡", Teh: "红茶", Milo: "美禄" }[value.to_sym]
  end

  production(:_Drink_3, clear_packrat: true) do |value|
    value.nil? ? "炼乳" : { O: "", C: "鲜奶" }[value.to_sym]
  end

  production(:_Drink_4, clear_packrat: true) do |value|
    value.nil? ? "" : { Kosong: "无糖", "Siu Dai": "少糖", "Gah Dai": "加糖"}[value.to_sym]
  end

  production(:_Drink_5, clear_packrat: true) do |value|
    value.nil? ? "" : { Gau: "浓缩", Po: "清淡"}[value.to_sym]
  end

  production(:_Drink_6, clear_packrat: true) do |value|
    extras = value.map do |a|
      { Peng: "冰块", Bubble: "珍珠", Halia: "姜汁" }[a.to_sym]
    end
    extras.empty? ? "" : "加#{extras.join}"
  end

  production(:Drink, clear_packrat: true) do |value|
    h = value.inject(&:merge)
    if h[:Water] == "红茶" and h[:_Drink_3] == "鲜奶"
      h[:Water] = "奶茶"
      h[:_Drink_3] = ""
    elsif h[:Water] == "咖啡" and h[:_Drink_3] == "鲜奶"
      h[:Water] = "咖啡拿铁"
      h[:_Drink_3] = ""
    end
    "#{h[:_Drink_1]}#{h[:_Drink_2]}#{h[:_Drink_4]}#{h[:_Drink_3]}#{h[:_Drink_5]}#{h[:Water]}#{h[:_Drink_6]}"
  end

  def initialize
    @rules = EBNF.parse(TEA_GRAMMER).make_peg.ast
  end

  def evaluate(input)
    parse(input, :Drink, @rules)
  end
end

loop do
  puts "输入饮料名："
  print "> "
  puts  "< #{MalayTea.new.evaluate(gets.chomp)}"
end
```

我们来运行一下看看：

```
❯ ruby main.rb
输入饮料名：
> Bungkus Ikat Mati Nga Lat Kopi O Siu Dai Gau Peng Bubble
< 外带塑料袋装打死结大杯少糖浓缩咖啡加冰块珍珠
```

欢迎大家带着这个程序去马来西亚／新加坡饮茶。
