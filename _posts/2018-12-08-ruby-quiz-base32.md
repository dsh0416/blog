---
title: "[Ruby Quiz] Base32 字母表 —— 加密猫基因解码"
date: 2018-12-08 10:54:43 +0900
tags: [Ruby, Ruby Quiz]
---

![Crypto Kitties](/assets/images/cryptokitties.png)

刚刚在 ruby-talk 的邮件列表读到一个很有意思的 Ruby Quiz，题目可以[见此](https://github.com/planetruby/quiz/tree/master/008)。想到好久没有看到 Ruby Quiz 了，就做了并翻译了一下。

## 题目翻译

庆祝 CryptoKitties 一周年 —— 区块链上诞生了超过 100 万只可爱的小猫！

我们来尝试转换 "sekretoooo" 加密猫的基因，一个 240 位的整数，以每 5 位进行分割，再通过 base32 (2^5=32) 进行转换，转至 kai 标注。

Q: 什么是 kai 标注？

Kai 标注（因为 [Kai Turner](https://medium.com/@kaigani/the-cryptokitties-genome-project-on-dominance-inheritance-and-mutation-b73059dcd0a4) 解码了加密猫的基因而命名）是一种针对 240 位整数分割成 5 位块的 base58 的变种（子集）。每个 5 位块含有 32 种可能性，240 位基因可以给分割成 12 组，每组 4 (x 5 位）基因。

举例：

|Kai    |Binary |Num|Kai    |Binary |Num|Kai    |Binary |Num|Kai    |Binary |Num|
|-------|-------|---|-------|-------|---|-------|-------|---|-------|-------|---|
| **1** | 00000 | 00 | **9** | 01000 | 08 | **h** | 10000 |16 | **q** | 11000 |24 |
| **2** | 00001 | 01 | **a** | 01001 | 09 | **i** | 10001 |17 | **r** | 11001 |25 |
| **3** | 00010 | 02 | **b** | 01010 | 10 | **j** | 10010 |18 | **s** | 11010 |26 |
| **4** | 00011 | 03 | **c** | 01011 | 11 | **k** | 10011 |19 | **t** | 11011 |27 |
| **5** | 00100 | 04 | **d** | 01100 | 12 | **m** | 10100 |20 | **u** | 11100 |28 |
| **6** | 00101 | 05 | **e** | 01101 | 13 | **n** | 10101 |21 | **v** | 11101 |29 |
| **7** | 00110 | 06 | **f** | 01110 | 14 | **o** | 10110 |22 | **w** | 11110 |30 |
| **8** | 00111 | 07 | **g** | 01111 | 15 | **p** | 10111 |23 | **x** | 11111 |31 |

**注意：数字 0 和字母 l 不会在 kai 中被使用。**

我们开始编程吧！举例来说：

``` ruby

# A 240-bit super "sekretoooo" integer genome

# hexadecimal (base 16)
genome = 0x4a52931ce4085c14bdce014a0318846a0c808c60294a6314a34a1295b9ce
# decimal (base 10)
genome = 512955438081049600613224346938352058409509756310147795204209859701881294
# binary (base 2)
genome = 0b010010100101001010010011000111001110010000001000010111000001010010111101110011100000000101001010000000110001100010000100\
           011010100000110010000000100011000110000000101001010010100110001100010100101000110100101000010010100101011011100111001110
```

我们可以把 10 进制数转换成 16 进制或 2 进制数：

``` ruby
p genome    # printed as decimal (base 10) by default
# => 512955438081049600613224346938352058409509756310147795204209859701881294

p genome.to_s(16)
# => "4a52931ce4085c14bdce014a0318846a0c808c60294a6314a34a1295b9ce"

p genome.to_s(2)
# => "10010100101001010010011000111001110010000001000010111000001010010111101110011100000000101001010000000110001100010000100\
#     011010100000110010000000100011000110000000101001010010100110001100010100101000110100101000010010100101011011100111001110"

bin = '%0240b' % genome     # note: adds leading zeros - to_s(2) does not
p bin.size
# => 240
p bin
# => "010010100101001010010011000111001110010000001000010111000001010010111101110011100000000101001010000000110001100010000100\
#     011010100000110010000000100011000110000000101001010010100110001100010100101000110100101000010010100101011011100111001110"

hex = '%060x' % genome     # note: adds leading zeros - to_s(16) does not
p hex.size
# => 60
p hex
# => 60
# => "4a52931ce4085c14bdce014a0318846a0c808c60294a6314a34a1295b9ce"
```

最终要得到的效果是：

``` ruby
kai = kai_encode( genome )   ## number to base32 kai notation
p kai
# => "aaaa788522f2agff16617755e979244166677664a9aacfff"
```

挑战：创建 `kai_encode` 方法并通过 RubyQuizTest :-).

``` ruby
def kai_encode( num )
  # ...
end
```

对于 Level 1 挑战，你需要把 240 位整数转换成 Base 32 的 Kai 标注。
对于 Level 2 挑战，你要把结果 4 个 4 个进行分组，将

``` ruby
"aaaa788522f2agff16617755e979244166677664a9aacfff"
```

转换成

``` ruby
"aaaa 7885 22f2 agff 1661 7755 e979 2441 6667 7664 a9aa cfff"
```


``` ruby
def kai_fmt( kai )
  # ...
end
```


你可以从头开始编码，也可以使用任何你想用的库 / gem。你需要通过如下的测试：

``` ruby
require 'minitest/autorun'

class RubyQuizTest < MiniTest::Test

  ################################
  # test data
  def genomes
     [
       [0x00004a52931ce4085c14bdce014a0318846a0c808c60294a6314a34a1295b9ce,
        "aaaa 7885 22f2 agff 1661 7755 e979 2441 6667 7664 a9aa cfff"]
     ]  
  end

  #############
  # tests
  def test_kai_encode
    genomes.each do |pair|
      num       = pair[0]
      exp_value = pair[1].gsub(' ','')   # note: remove spaces

      assert_equal exp_value, kai_encode( num )
    end
  end # method test_kai_encode

  def test_kai_fmt
    genomes.each do |pair|
      kai       = pair[1].gsub(' ','') # remove spaces
      exp_value = pair[1]

      assert_equal exp_value, kai_fmt( kai )
    end
  end # method test_kai_fmt

end # class RubyQuizTest
```

注：对于解码后与基因的对应关系表实在太长了，而且和题目无关。如果想看可以查[原文](https://github.com/planetruby/quiz/blob/master/008/README.md)。

## 剧透时间

我想出来的使用 Ruby 的一行解：

```ruby
def kai_encode(num)
  num.to_s(2).rjust(240, '0').scan(/.{5}/).map {|n| '123456789abcdefghijkmnopqrstuvwx'[n.to_i(2)]}.join
end

def kai_fmt(kai)
  kai.scan(/.{4}/).join(' ')
end

kai_encode(0x00004a52931ce4085c14bdce014a0318846a0c808c60294a6314a34a1295b9ce) # => "aaaa788522f2agff16617755e979244166677664a9aacfff"
kai_fmt(kai_encode(genome)) # => "aaaa 7885 22f2 agff 1661 7755 e979 2441 6667 7664 a9aa cfff"
```

好在 Ruby 的 `rjust` 方法是内建的，而不需要 import 一个叫 `left-pad` 的包，一旦被删除了就天下大乱了（逃
