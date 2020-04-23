---
title: 83 行 Ruby 的蓝调即兴生成器
date: 2020-04-23 12:02:53
tags: [音乐, Ruby]
---

## 刻板印象的蓝调

如果让我用代码来生成音乐，我可能优先的类型就是爵士。像是流行音乐，要是来几颗不和谐音，这听起来可就一点都不流行了。但爵士不一样，弹错一次是弹错，如果能连续弹错两次，那就是真正的爵士艺术家了（逃）。但爵士对乐理的要求太高，要想完全靠代码把逻辑理清楚那也不是一件容易的事。一个好的爵士即兴实在是太难了，不过我们可以根据一些刻板印象来做一做。我就想到了十二小节蓝调和声进行以及蓝调音阶，单靠这两个就一下子把需要的伴奏和旋律的模式给提供了一个大概，也许可以试试用代码来自动生成看看。

合成器方面我用的是 Sonic Pi，一款基于 Ruby 的实时编码音乐合成器。其实我之前用过一两次，也不算太会用这个库，也是突发奇想然后现学的。

## 节奏生成

首先还是要有一个节奏生成器，由于是蓝调音乐，节奏其实可以比较随性。我的基本想法很简单，在给定一个小节里随机填充四分、八分音符。

```ruby
FOURTH = 0.5
EIGHTH = FOURTH / 2
SIXTEENTH = EIGHTH / 2

def generate_rhythms(candidates=[EIGHTH, FOURTH], remaining_time=FOURTH*4)
  rhythms = []
  
  while remaining_time > candidates.min
    rhythm = choose(candidates)
    rhythms << rhythm
    remaining_time -= rhythm
  end

  rhythms << remaining_time if remaining_time > 0

  rhythms
end
```

节奏上想要比较 swing 一点。一个刻板印象的 swing 就是两个八分音符前一拍长一点，后一拍短一点。所以我就在随机填充后在做了一些手动上的修改：

```ruby
def generate_rhythms(candidates=[EIGHTH, FOURTH], remaining_time=FOURTH*4)
  rhythms = []
  
  while remaining_time > candidates.min
    rhythm = choose(candidates)
    rhythms << rhythm
    remaining_time -= rhythm
  end
  
  rhythms << remaining_time if remaining_time > 0
  
  # Generate Swing Pattern
  (rhythms.length - 1).times do |i|
    if rhythms[i] == EIGHTH and rhythms[i + 1] == EIGHTH
      rhythms[i] = EIGHTH + SIXTEENTH
      rhythms[i + 1] = EIGHTH - SIXTEENTH
    end
  end
  
  rhythms
end
```

写完节奏，我们写一个测试来听一下。

```ruby
live_loop :rhythm_test do
  use_synth :pulse
  rhythms = generate_rhythms
  rhythms.each do |t|
    play :C3, release: t
    sleep t
  end
end
```

感觉还行。

<audio src="/static/blues-rhythm.mp3" controls></audio>

## 贝斯

有了节奏我们接下来就可以写贝斯。~~贝斯反正很多人听不见，也不用花很大力气写。~~

十二小节蓝调的和弦基本上就是四个一级，两个四级两个一级，一个五级一个四级再两个一级如此重复。贝斯我的想法就是写一个 walking bass（走路贝斯）。基本上就是第一个音是和弦根音（否则听起来就不像这个和弦了），后面的音就是在九和弦的三度、七度、九度音上随便爬就行。

```ruby
CHORDS_BASE = [:C2, :C2, :C2, :C2, :F2, :F2, :C2, :C2, :G2, :F2, :C2, :C2]

live_loop :bass_line do
  use_synth :fm
  
  CHORDS_BASE.each do |c|
    notes = chord(c, '9').to_a
    
    rhythms = generate_rhythms
    
    play notes[0], release: rhythms[0]
    sleep rhythms[0]
    
    rhythms[1..-1].each do |t|
      play choose([notes[1], notes[3], notes[4]]), release: t
      sleep t
    end
  end
end
```

感觉差不多。

<audio src="/static/blues-bass.mp3" controls></audio>

## 左手伴奏

写完了贝斯，最好在高一个八度再写一个左手伴奏，节奏可以和贝斯错开。因为已经有贝斯了，这个伴奏我不想要有和弦根音。然后节奏上希望有一些错落感，不要像 bass 一样从头弹到尾。于是我加了一个随机变量，当到达当前节奏时 50% 演奏和弦，50% 概率直接跳过。

```ruby
CHORDS_HARMONY = [:C3, :C3, :C3, :C3, :F3, :F3, :C3, :C3, :G3, :F3, :C3, :C3]

live_loop :chord do
  use_synth :fm
  
  CHORDS_HARMONY.each do |c|
    notes = chord(c, '9').to_a
    rhythms = generate_rhythms
    
    rhythms.each do |t|
      play [notes[1], notes[3], notes[4]], release: t if choose([true, false])
      sleep t
    end
  end
end
```

听起来是这样的。这蓝调的味道一下子就来了。

<audio src="/static/blues-chords.mp3" controls></audio>

## 即兴

布鲁斯的即兴旋律其实和写中国五声音阶的即兴旋律差不多。是一个上上下下的循环，然后在里面再做一些变化。我这里选用的音阶是 C Eb F Gb G Bb 和 C1，然后两个小节一生成。默认根据其在小节中的大概位置生成一个上升或下降的旋律，然后提供一个 offset 从 -2 到 2 之间随机。在原先旋律上加上一个 offset，如果超出了我们的音程就 clip 掉。这样应该就能产生一个整体上有上下，但是实际上又有很多变化的旋律了。

和贝斯以及和弦不太一样的是，为了有更多的变化感，节奏要从二分、四分、八分、十六分里任意随机。然后被跳过的概率不用 50% 那么大，10% 应该就可以。

```ruby
BLUES_SCALE = [:C4, :Eb4, :F4, :Gb4, :G, :Bb4, :C5]

melody_reverse = false
live_loop :melody do
  use_synth :beep
  
  rhythm = generate_rhythms([FOURTH*2, FOURTH, EIGHTH, SIXTEENTH], remaining_time=FOURTH*8)
  scale_notes = BLUES_SCALE
  scale_notes = scale_notes.reverse if melody_reverse
  melody_reverse = !melody_reverse
  
  rhythm.each_with_index do |t, i|
    offset = choose([-2, -1, 1, 2])
    
    new_index = (i.to_f / rhythm.length).to_i + offset
    new_index = 0 if new_index < 0
    new_index = scale_notes.length - 1 if new_index >= scale_notes.length
    
    note = scale_notes[new_index]
    play note, release: t if rrand(0, 1) < 0.9
    sleep t
  end
end
```

最后我们得到了这样的成品：

<audio src="/static/blues-generated.mp3" controls></audio>

## 总结

完整代码如下：

```ruby
CHORDS_BASE =    [:C2, :C2, :C2, :C2, :F2, :F2, :C2, :C2, :G2, :F2, :C2, :C2]
CHORDS_HARMONY = [:C3, :C3, :C3, :C3, :F3, :F3, :C3, :C3, :G3, :F3, :C3, :C3]
BLUES_SCALE = [:C4, :Eb4, :F4, :Gb4, :G, :Bb4, :C5]

FOURTH = 0.5
EIGHTH = FOURTH / 2
SIXTEENTH = EIGHTH / 2

def generate_rhythms(candidates=[EIGHTH, FOURTH], remaining_time=FOURTH*4)
  rhythms = []
  
  while remaining_time > candidates.min
    rhythm = choose(candidates)
    rhythms << rhythm
    remaining_time -= rhythm
  end
  
  rhythms << remaining_time if remaining_time > 0
  
  # Generate Swing Pattern
  (rhythms.length - 1).times do |i|
    if rhythms[i] == EIGHTH and rhythms[i + 1] == EIGHTH
      rhythms[i] = EIGHTH + SIXTEENTH
      rhythms[i + 1] = EIGHTH - SIXTEENTH
    end
  end
  
  rhythms
end

live_loop :bass_line do
  use_synth :fm
  
  CHORDS_BASE.each do |c|
    notes = chord(c, '9').to_a
    
    rhythms = generate_rhythms
    
    play notes[0], release: rhythms[0]
    sleep rhythms[0]
    
    rhythms[1..-1].each do |t|
      play choose([notes[1], notes[3], notes[4]]), release: t
      sleep t
    end
  end
end

live_loop :chord do
  use_synth :fm
  
  CHORDS_HARMONY.each do |c|
    notes = chord(c, '9').to_a
    rhythms = generate_rhythms
    
    rhythms.each do |t|
      play [notes[1], notes[3], notes[4]], release: t if choose([true, false])
      sleep t
    end
  end
end

melody_reverse = false
live_loop :melody do
  use_synth :beep
  
  rhythm = generate_rhythms([FOURTH * 2, FOURTH, EIGHTH, SIXTEENTH], remaining_time=FOURTH*8)
  scale_notes = BLUES_SCALE
  scale_notes = scale_notes.reverse if melody_reverse
  melody_reverse = !melody_reverse
  
  rhythm.each_with_index do |t, i|
    offset = choose([-2, -1, 1, 2])
    
    new_index = (i.to_f / rhythm.length).to_i + offset
    new_index = 0 if new_index < 0
    new_index = scale_notes.length - 1 if new_index >= scale_notes.length
    
    note = scale_notes[new_index]
    play note, release: t if rrand(0, 1) < 0.9
    sleep t
  end
end
```

一共 83 行，好像还挺像那么回事的。而且由于是一个 Loop，放着程序可以无穷无尽地生成下去，作为什么免费无版权的背景音乐素材好像还是个挺不错的来源。
