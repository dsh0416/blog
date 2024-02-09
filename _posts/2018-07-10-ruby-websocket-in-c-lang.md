---
title: 在 Ruby 中实现一个 WebSocket 掩码
date: 2018-7-10 21:47:45
tags: [编程, Ruby]
---

## 引

在 WebSocket 协议中有一个重要的实现就是 masking（掩码），根据 RFC 6455 [10.3 章节](https://tools.ietf.org/html/rfc6455#section-10.3) 的讨论，客户端连接服务器必须要打开掩码功能，当服务器收到一个客户端帧没有打开掩码时，应到立刻终止这一连接。关于为什么必须有一方开启 mask，可以参见 [这个回答](https://security.stackexchange.com/questions/36930/how-does-websocket-frame-masking-protect-against-cache-poisoning)。

一个 WebSocket 帧长成下面这样：

```
      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-------+-+-------------+-------------------------------+
     |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
     |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
     |N|V|V|V|       |S|             |   (if payload len==126/127)   |
     | |1|2|3|       |K|             |                               |
     +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
     |     Extended payload length continued, if payload len == 127  |
     + - - - - - - - - - - - - - - - +-------------------------------+
     |                               |Masking-key, if MASK set to 1  |
     +-------------------------------+-------------------------------+
     | Masking-key (continued)       |          Payload Data         |
     +-------------------------------- - - - - - - - - - - - - - - - +
     :                     Payload Data continued ...                :
     + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
     |                     Payload Data continued ...                |
     +---------------------------------------------------------------+
```

举例来说：

当客户端向服务器发送一个 "Hello" 字符串的时候，二进制流长成下面这样：

```
0x81 0x85 0x37 0xfa 0x21 0x3d 0x7f 0x9f 0x4d 0x51 0x58
```

其中：

- 第一个 Byte 0x81，也就是 0b10000001，指的是 FIN=1，代表这帧传输了完整的数据，没有分片。Opcode 是 0x1，传输的是一个字符串。
- 第二个 Byte 0x85，也就是 0b10000101，指使用 Mask，Payload 的长度是 5。
- 第三到第六个 Byte 是 Mask 掩码 Key。
- 第七到十一个 Byte 原先是 0x48(H) 0x65(e) 0x6c(l) 0x6c(l) 0x6f(o)，但现在要被掩码盖住，掩码的计算方式就是循环进行异或运算。解码时只需要对着 Mask Key 再做一次异或就会回去，因为 a ^ b ^ b = a。
  - 0x48 ^ 0x37 = 0x7f
  - 0x65 ^ 0xfa = 0x9f
  - 0x6c ^ 0x21 = 0x4d
  - 0x6c ^ 0x3d = 0x51
  - 0x6f ^ 0x37 = 0x58

这个过程在 Ruby 中应该怎么实现呢？

## 尝试

```ruby
stream = StringIO.new([0x81, 0x85, 0x37, 0xfa, 0x21, 0x3d, 0x7f, 0x9f, 0x4d, 0x51, 0x58].pack('C*')) # 模拟网络传进来的 byte 流

def mask(data)
  first_byte = data.getbyte
  opcode = first_byte & 0b0001111 # 先不考虑 fin

  second_byte = data.getbyte
  raise 'NotMaskedError' unless (second_byte & 0b10000000) == 128

  payload = second_byte & 0b01111111
  mask = Array.new(4) { data.getbyte }
  masked_msg = Array.new(payload) { data.getbyte }

  # Start Decoding
  masked_msg = masked_msg.map.with_index do |byte, i|
    byte ^ mask[i % 4]
  end

  return masked_msg.pack('C*') if [0x1, 0x9, 0xA].include? opcode # String message
  return masked_msg
end

puts mask(stream) # => Hello
```

Bravo! 我们正确实现了解码过程，这段代码的性能如何呢？

```ruby
require 'benchmark'

N = 1_000_000
Benchmark.bm do |x|
  x.report { N.times do mask(StringIO.new([0x81, 0x85, 0x37, 0xfa, 0x21, 0x3d, 0x7f, 0x9f, 0x4d, 0x51, 0x58].pack('C*'))) end }
end
```

```
       user     system      total        real
   5.083038   0.025399   5.108437 (  5.154494)
```

emmm... 处理 1000 万个 byte 就这种性能… 恐怕用 WebSocket 做实时通讯不太行啊。

不过 [Donald Knuth](http://en.wikipedia.org/wiki/Donald_Knuth) 说过：「我们应该忘记细小的性能提升，在 97% 的情况下，过早的优化都是万恶之源。」

**好了，收工回家了。**

（完）



















## 才怪

高德纳还说过：「我们千万不能放弃剩下的 3%」而解码过程是每个请求都必须跑的，对于一个 I/O-bound 的 Web 应用，如果因为一个 decode 变成 CPU-bound 那可是个非常严重的问题，能优化一点都有很大的帮助。

于是，要想给一段代码找到性能问题，第一个想到的工具当然就是：

**profiler** !!!

要在 Ruby 上使用 profiler 很简单，只需要引入 profile 库即可。不过 profiler 会让程序运行得很慢，要注意节制在潜在的性能问题上去单独跑。

```bash
ruby -rprofile test.rb
```

报告如下：

```
  %   cumulative   self              self     total
 time   seconds   seconds    calls  ms/call  ms/call  name
 49.28     2.94      2.94   150000     0.02     0.05  Object#mask
 11.59     3.63      0.69    20000     0.03     0.09  Array#initialize
  7.05     4.05      0.42    20000     0.02     0.10  Array#map
  5.65     4.39      0.34   110000     0.00     0.00  StringIO#getbyte
  4.16     4.64      0.25    10002     0.02     1.78  nil#
  3.37     4.84      0.20    20002     0.01     0.10  Class#new
  2.62     5.00      0.16    50000     0.00     0.00  Array#[]
  2.60     5.15      0.16    50000     0.00     0.00  Integer#^
  2.53     5.30      0.15    50000     0.00     0.00  Integer#%
  1.77     5.41      0.11    10001     0.01     0.01  StringIO#initialize
  1.75     5.51      0.10    10000     0.01     0.21  Enumerator#with_index
  1.73     5.62      0.10    10001     0.01     0.02  StringIO.new
  1.57     5.71      0.09    30000     0.00     0.00  Integer#&
  1.41     5.79      0.08    20001     0.00     0.00  Array#pack
  1.26     5.87      0.08        1    75.09  5962.83  Integer#times
  0.53     5.90      0.03    10000     0.00     0.00  Array#include?
  0.52     5.93      0.03    10000     0.00     0.00  Integer#==
  0.52     5.96      0.03    10001     0.00     0.00  BasicObject#initialize
...
```

这个 profiler 结果是我们最不愿意看到的，因为各个方法都平均地散落在哪里，并没有哪个方法特别占用时间。面对这种情况我们该怎么做呢？

## AOT？

由于我们的操作几乎都是位操作，而以对象为单位来操作大大增加了各种开销。也许等之后 2.6.0 Ruby 有 JIT 的话，也许会随着跑的时间变长而优化，不过。。。我们可以 AOT 编译这些代码啊。

什么？你问 Ruby 什么时候支持 AOT 了？

Ruby 不支持，你自己判断这段代码是瓶颈，自己把这段代码写成 C 语言编译到二进制不就是 AOT 了？

在这篇 [博客](https://blog.jcoglan.com/2012/07/29/your-first-ruby-native-extension-c/) 中作者介绍了他是如何通过实现 C 扩展来提升 Faye WebSocket 的性能的。按这个思路，我之前对 [Midori](https://github.com/midori-rb/midori.rb) 库中 WebSocket 的 Mask 是类似实现的：

```c
#include <ruby.h>

VALUE WebSocket = Qnil;
VALUE MidoriWebSocket = Qnil;

void Init_midori_ext();
VALUE method_midori_websocket_mask(VALUE self, VALUE payload, VALUE mask);

void Init_midori_ext()
{
  Midori = rb_define_module("Midori");
  MidoriWebSocket = rb_define_class_under(Midori, "WebSocket", rb_cObject);
  rb_define_protected_method(MidoriWebSocket, "mask", method_midori_websocket_mask, 2);
}

VALUE method_midori_websocket_mask(VALUE self, VALUE payload, VALUE mask)
{
  long n = RARRAY_LEN(payload), i, p, m;
  VALUE unmasked = rb_ary_new2(n);

  int mask_array[] = {
      NUM2INT(rb_ary_entry(mask, 0)),
      NUM2INT(rb_ary_entry(mask, 1)),
      NUM2INT(rb_ary_entry(mask, 2)),
      NUM2INT(rb_ary_entry(mask, 3))};

  for (i = 0; i < n; i++)
  {
    p = NUM2INT(rb_ary_entry(payload, i));
    m = mask_array[i % 4];
    rb_ary_store(unmasked, i, INT2NUM(p ^ m));
  }
  return unmasked;
}
```

从理论上讲，这很好的减少了各种内存的分配，应该快很多，那么到底快了多少呢？

```
       user     system      total        real
   5.293516   0.044790   5.338306 (  5.442116)
   4.627439   0.023852   4.651291 (  4.699278)
```

1.16x 的速度。

## 能不能快一点？

写 Ruby 上的 C 扩展一个比较麻烦的地方就在于这些 VALUE，VALUE 其实是一个指向 Ruby 对象的 uintptr_t 指针。如果我们减少对 Ruby 对象本身的调用，减少一些中间过程，理论上应该会更快。比如说，当我们发现 0x1 0x9 0xA 的 Opcode 是字符串类型，我们便可以不去创建 RARRAY 类型，而直接将字符串类型返回回去。

于是我们可以再实现一个 C 扩展函数：

```c
#include <ruby.h>
#include <ruby/encoding.h>

VALUE Midori = Qnil;
VALUE MidoriWebSocket = Qnil;

void Init_midori_ext();
VALUE method_midori_websocket_mask(VALUE self, VALUE payload, VALUE mask);
VALUE method_midori_websocket_mask_str(VALUE self, VALUE payload, VALUE mask);

void Init_midori_ext()
{
  Midori = rb_define_module("Midori");
  MidoriWebSocket = rb_define_class_under(Midori, "WebSocket", rb_cObject);
  rb_define_protected_method(MidoriWebSocket, "mask", method_midori_websocket_mask, 2);
  rb_define_protected_method(MidoriWebSocket, "mask_str", method_midori_websocket_mask_str, 2);
}

VALUE method_midori_websocket_mask(VALUE self, VALUE payload, VALUE mask)
{
  long n = RARRAY_LEN(payload), i, p, m;
  VALUE unmasked = rb_ary_new2(n);

  int mask_array[] = {
      NUM2INT(rb_ary_entry(mask, 0)),
      NUM2INT(rb_ary_entry(mask, 1)),
      NUM2INT(rb_ary_entry(mask, 2)),
      NUM2INT(rb_ary_entry(mask, 3))};

  for (i = 0; i < n; i++)
  {
    p = NUM2INT(rb_ary_entry(payload, i));
    m = mask_array[i % 4];
    rb_ary_store(unmasked, i, INT2NUM(p ^ m));
  }
  return unmasked;
}

VALUE method_midori_websocket_mask_str(VALUE self, VALUE payload, VALUE mask)
{
  long n = RARRAY_LEN(payload), i, p, m;
  char result[n];

  int mask_array[] = {
      NUM2INT(rb_ary_entry(mask, 0)),
      NUM2INT(rb_ary_entry(mask, 1)),
      NUM2INT(rb_ary_entry(mask, 2)),
      NUM2INT(rb_ary_entry(mask, 3))};

  for (i = 0; i < n; i++)
  {
    p = NUM2INT(rb_ary_entry(payload, i));
    m = mask_array[i % 4];
    result[i] = p ^ m;
  }

  return rb_enc_str_new(result, n, rb_utf8_encoding());
}
```

```
       user     system      total        real
   5.372697   0.044040   5.416737 (  5.502921)
   4.994706   0.047653   5.042359 (  5.147833)
   3.745804   0.015971   3.761775 (  3.796537)
```

这下，我们得到了 45% 的性能提升，这是个好开始。

## What if...

既然我们知道减少和 Ruby 对象的交互可以增加性能，如果我们把整个 decode 过程都用 C 实现一遍会怎么样呢？

```C
#include <ruby.h>
#include <ruby/encoding.h>

VALUE Midori = Qnil;
VALUE MidoriWebSocket = Qnil;

void Init_midori_ext();
VALUE method_midori_websocket_mask(VALUE self, VALUE payload, VALUE mask);
VALUE method_midori_websocket_mask_str(VALUE self, VALUE payload, VALUE mask);
VALUE method_midori_websocket_decode_c(VALUE self, VALUE data);

void Init_midori_ext()
{
  Midori = rb_define_module("Midori");
  MidoriWebSocket = rb_define_class_under(Midori, "WebSocket", rb_cObject);
  rb_define_protected_method(MidoriWebSocket, "mask", method_midori_websocket_mask, 2);
  rb_define_protected_method(MidoriWebSocket, "mask_str", method_midori_websocket_mask_str, 2);
  rb_define_method(MidoriWebSocket, "decode_c", method_midori_websocket_decode_c, 1);
}

VALUE method_midori_websocket_mask(VALUE self, VALUE payload, VALUE mask)
{
  long n = RARRAY_LEN(payload), i, p, m;
  VALUE unmasked = rb_ary_new2(n);

  int mask_array[] = {
      NUM2INT(rb_ary_entry(mask, 0)),
      NUM2INT(rb_ary_entry(mask, 1)),
      NUM2INT(rb_ary_entry(mask, 2)),
      NUM2INT(rb_ary_entry(mask, 3))};

  for (i = 0; i < n; i++)
  {
    p = NUM2INT(rb_ary_entry(payload, i));
    m = mask_array[i % 4];
    rb_ary_store(unmasked, i, INT2NUM(p ^ m));
  }
  return unmasked;
}

VALUE method_midori_websocket_mask_str(VALUE self, VALUE payload, VALUE mask)
{
  long n = RARRAY_LEN(payload), i, p, m;
  char result[n];

  int mask_array[] = {
      NUM2INT(rb_ary_entry(mask, 0)),
      NUM2INT(rb_ary_entry(mask, 1)),
      NUM2INT(rb_ary_entry(mask, 2)),
      NUM2INT(rb_ary_entry(mask, 3))};

  for (i = 0; i < n; i++)
  {
    p = NUM2INT(rb_ary_entry(payload, i));
    m = mask_array[i % 4];
    result[i] = p ^ m;
  }

  return rb_enc_str_new(result, n, rb_utf8_encoding());
}

VALUE method_midori_websocket_decode_c(VALUE self, VALUE data)
{
  int byte, opcode;
  ID getbyte = rb_intern("getbyte");

  byte = NUM2INT(rb_funcall(data, getbyte, 0));
  opcode = byte & 0xf;

  byte = NUM2INT(rb_funcall(data, getbyte, 0));
  if ((byte & 0x80) != 0x80)
  {
    rb_raise(rb_eRuntimeError, "NotMaskedError");
  }

  int n = byte & 0x7f;
  char result[n];

  int mask_array[] = {
      NUM2INT(rb_funcall(data, getbyte, 0)),
      NUM2INT(rb_funcall(data, getbyte, 0)),
      NUM2INT(rb_funcall(data, getbyte, 0)),
      NUM2INT(rb_funcall(data, getbyte, 0))};

  for (int i = 0; i < n; i++)
  {
    result[i] = NUM2INT(rb_funcall(data, getbyte, 0)) ^ mask_array[i % 4];
  }

  if (opcode == 0x1 || opcode == 0x9 || opcode == 0xA)
    return rb_enc_str_new(result, n, rb_utf8_encoding());

  VALUE result_arr = rb_ary_new2(n);
  for (int i = 0; i < n; i++)
  {
    rb_ary_store(result_arr, i, INT2NUM(result[i]));
  }
  return result_arr;
}

```

跑一下 benchmark：

```
       user     system      total        real
   5.020994   0.029096   5.050090 (  5.108254)
   4.836846   0.035304   4.872150 (  4.953138)
   3.826166   0.021345   3.847511 (  3.892810)
   2.021958   0.014087   2.036045 (  2.066971)
```

2.47x 速度，至此，我们将解码速度提高了 147%。

## 副作用

然而显然，用 C 语言来写这样的代码安全性是很难保证的。特别是大家的 C 语言水平通常都不会太高超，一不小心来个内存泄漏分分钟就 gg 了。大多数时候，我们解决的性能问题都不是直接封装成库，如果是企业内部用，那么我们可以对编译过程提一些要求。一个比较有趣的尝试是 Rust 上的 [helix](https://usehelix.com/)。可以在保障内存安全的情况下写一些辅助函数，并且通过 Rust 宏的支持下，使得代码也没有 C 语言那么难看。

如果在你的业务场景中也有遇到运算密集（注意不是 I/O 密集瓶颈）时，通过 profiler 确认，不妨也可以试试用 C 扩展来实现一下吧！

最后留一个小问题，如果收到的 WebSocket 流非常长，能不能通过什么魔法使得编译器优化过程中触发 CPU 的 SIMD 特性，从而得到更快的速度呢？
