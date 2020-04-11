---
title: 用 Ruby 学习基本乐理（一）：音高
date: 2020-04-11 14:36:38
tags: [音乐, 乐理, 编程, Ruby, 教程]
---

## 序

音乐很有趣，理解音乐很难。但音乐背后的物理、数学原理并没有那么复杂，此所谓乐理。Ruby 是一门编程语言。编程编的是程序，所谓程序，是计算机执行的指令，是阐明计算过程的方式。我们在此使用 Ruby 语言描述乐理，以简单的例子提高自己对乐理的理解，也可以精进自己的 Ruby 编程技能。

## 从基本音高定义开始

关于音高的基本定义如下：

1. 标准音高 A4 = 440Hz
2. 一个八度有 12 个半音：C, C#, D, D#, E, F, F#, G, G#, A, A#, B
3. 两个八度之间的频率关系差 2 倍
4. 以十二平均律调音，两个相邻音的频率关系差 $ 2^\frac{1}{12} $ 倍。

我们先设计一个根据到 A4 半音数量计算音高频率的 Ruby 程序：

```ruby
STANDARD_TUNING = 440.0
def frequency_by_offset(offset)
  STANDARD_TUNING * (2.0 ** (1.0 / 12)) ** offset
end
```

然后我们要解析音名，从而处理到音高的关系。音乐中的音名主要有七个：C、D、E、F、G、A、B，之间关系是大调音阶关系，即全音、全音、半音、全音、全音、全音、半音。但一个八度内半音数量有 12，所以后面可以接升降记号 # 或 b，记号可以叠加。音高到音名的关系是一对多的，D、C## 可以表示同一个音高（取决于作曲家的具体需求）。最后加上一个八度的标记来表示超过一个八度的音，我们可以用一个正则表达式 `^[CDEFGAB][#,b]*\d$` 来匹配。

```ruby
class Tone
  STANDARD_TUNING = 440.0
  attr_reader :frequency
  attr_reader :name

  def initialize(name)
    @name = name
    offset = parse_name(name)
    @frequency = frequency_by_offset(offset)
  end
  
  def parse_name(name)
    raise AugumentError unless name.match?(/^[CDEFGAB][#,b]*\d$/)
    tone = name[0]
    sharps = name[1...-1]
    range = name[-1].to_i
    
    # Calculate offset
    major_scale = [-9, -7, -5, -4, -2, 0, 2]
    offset = major_scale['CDEFGAB'.index(tone)] + (range - 4) * 12 # Offset without sharps or flats
    sharps.chars.each do |c|
      c == '#' ? offset += 1 : offset -= 1
    end
    offset
  end
  
  def frequency_by_offset(offset)
    STANDARD_TUNING * (2.0 ** (1.0 / 12)) ** offset
  end
end
```

我们来验证一下这个程序：

```
2.7.0 :070 > Tone.new('A4')
 => #<Tone:0x00007f828c865b58 @name="A4", @frequency=440.0>
2.7.0 :071 > Tone.new('A#5')
 => #<Tone:0x00007f828c8653b0 @name="A#5", @frequency=932.3275230361803> 
2.7.0 :072 > Tone.new('C3')
 => #<Tone:0x00007f828c864d48 @name="C3", @frequency=130.8127826502992>
```

我们和[调音网站](https://pages.mtu.edu/~suits/notefreqs.html)比对一下结果：

| 音名 | 计算结果 | 调音网站标准值 |
| ---- | -------- | -------------- |
| A4   | 440.00   | 440.00         |
| A#5  | 932.328  | 932.33         |
| C3   | 164.813  | 130.81         |

结果基本准确。

## 频率反查

我们根据音名计算除了频率，下一步是要根据频率计算出接近的音名。但是频率是一个浮点数，浮点数不适合直接用等于号比较。在许多计算中，我们会设置一个阈值 $\epsilon$，即计算两个值的差是否小于这个阈值来判断是否相等。但是这个在音乐中是不适合的，因为八度是一个在频率上呈指数增长的东西，于是当音越高，其对阈值越宽容；当音越低，其对阈值越严格。这在和弦上还发展出了「低音程限制」的问题，我们会在之后具体提到。

在此我们采用一个粗暴搜索的算法来处理，即二分查找算法 (binary search algorithm)。在 Ruby 中，Array 有自带的二分查找实现，我们不妨使用它。

```ruby
ALL_TONES = (0..9).map {|range| ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B'].map {|name| "#{name}#{range}"}}.flatten.map { |name| Tone.new(name) }

def by_frequency(freq)
  ALL_TONES.bsearch { |x| x.frequency >= freq }
end

p by_frequency(440) # => #<Tone:0x00007f9b62815498 @name="A4", @frequency=440.0>
```

我们把这套系统合并到我们的类中，我们可以得到：

```ruby
class Tone
  STANDARD_TUNING = 440.0
  attr_reader :frequency
  attr_reader :name

  def initialize(name)
    @name = name
    offset = parse_name(name)
    @frequency = frequency_by_offset(offset)
  end

  def parse_name(name)
    raise AugumentError unless name.match?(/^[CDEFGAB][#,b]*\d$/)
    tone = name[0]
    sharps = name[1...-1]
    range = name[-1].to_i
    
    # Calculate offset
    major_scale = [-9, -7, -5, -4, -2, 0, 2]
    offset = major_scale['CDEFGAB'.index(tone)] + (range - 4) * 12 # Offset without sharps or flats
    
    sharps.chars.each do |c|
      c == '#' ? offset += 1 : offset -= 1
    end

    offset
  end
  
  def frequency_by_offset(offset)
    STANDARD_TUNING * (2.0 ** (1.0 / 12)) ** offset
  end

  class << self
    ALL_TONES = (0..9).map {|range| ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B'].map {|name| "#{name}#{range}"}}.flatten.map { |name| Tone.new(name) }

    def by_frequency(freq)
      ALL_TONES.bsearch { |x| x.frequency >= freq }
    end
  end
end

p Tone.by_frequency(440) # => #<Tone:0x00007fb0cd8164a8 @name="A4", @frequency=440.0>
```



## 泛音列

物体发声体除了整体振动（基音）以外，还会分段振动。因此除了基础的频率，通常还会产生两倍、三倍、四倍...频率的泛音。站在频率上非常好理解，但是要想记住泛音列的组成音的音名还是挺难记忆的。我们来看看能不能利用我们的音高定义和频率反查系统来自动计算泛音列吧。

由于理论上泛音列是无穷的，只要是整数倍都可以。所以我们可以利用 Ruby 2.6.0 开始引入的 endless range 特性让代码变得好看一些。由于我们反查的上限是 B9 所以当搜索超过 B9 我们就忽略之后的泛音列。另外在特高频率上，可能多个频率对应的是某同一个音名，所以在计算完，我们可以 `uniq` 一下。

```ruby
def harmonic_series(max=Tone.new('B9').frequency)
  tones = []
  (2..).each do |n|
    break if @frequency * n > max
    tones << Tone.by_frequency(@frequency * n)
  end
  tones.uniq { |t| t.name }
end

p Tone.new('C7').harmonic_series # => [#<Tone:0x00007f859703d8c8 @name="C8", @frequency=4186.009044809585>, #<Tone:0x00007f859700ec80 @name="G#8", @frequency=6644.875161279136>, #<Tone:0x00007f859700c868 @name="C9", @frequency=8372.018089619174>, #<Tone:0x00007f85980424a8 @name="E9", @frequency=10548.081821211863>, #<Tone:0x00007f8598040130 @name="G#9", @frequency=13289.750322558277>, #<Tone:0x00007f8597063438 @name="A#9", @frequency=14917.240368578916>]
```

根据 C 的泛音列表，其前 6 个泛音是 C G C1 E1 G1 Bb1。我们的计算器把其中两个 G 都计算成了 G#。这是因为我们的频率反查器可以接受一定程度的偏高，但不能接受偏低。考虑到我们的搜索范围只有 120 个音，我们其实没有必要将时间复杂度优化到 $O(logn)$，我们大可以使用 $O(n)$ 的复杂度。于是我把频率反查改成了下面的代码：

```ruby
def by_frequency(freq)
  errs = ALL_TONES.map {|x| (x.frequency - freq).abs }
  ALL_TONES[errs.rindex(errs.min)]
end
```

于是我们的泛音列结果变成了：

```ruby
p Tone.new('C7').harmonic_series # => [#<Tone:0x00007fc27787a2d0 @name="C8", @frequency=4186.009044809585>, #<Tone:0x00007fc277878c78 @name="G8", @frequency=6271.926975708001>, #<Tone:0x00007fc277877850 @name="C9", @frequency=8372.018089619174>, #<Tone:0x00007fc277876f40 @name="E9", @frequency=10548.081821211863>, #<Tone:0x00007fc277875960 @name="G9", @frequency=12543.853951416007>, #<Tone:0x00007fc277874330 @name="A#9", @frequency=14917.240368578916>]
```

与泛音表完全一致。

## 小结

至此，我们用 Ruby 处理了乐理中与音高有关的常见问题。让我们对 Ruby 和乐理的熟练程度都进一步提升了。接下来，我会介绍如何进一步用 Ruby 来处理音程与和弦。
