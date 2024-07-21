---
title: 基于 Prolog 的 SATB 四部和声生成器
date: 2024-07-21 17:30:00 +0800
tags: [Prolog, 逻辑学, 音乐, 乐理, 合唱]
---

最近在研究 SATB 四部和声，然后感觉如果完全按照和声学教材里的内容，尤其是 18 世纪的和声学概念，几乎就是给了一大堆规则，然后写的是有把人当成一个栈机 (stack machine) 不停搜索不要违背这些规则。既然如此，为什么我们不能设计一个搜索程序，直接将四部和声的配法规约出来呢？

## Prolog

很多年前，我写过一篇关于 Prolog 的[博客](/2020/01/18/archetypes-skeptic)，Prolog 是一个非常方便的，给定一系列逻辑，然后让它从中规约出结果的一门语言。一开始其实我想和之前[某篇博客](/2020/04/23/auto-blues-generator) 一样使用 Ruby 和 Sonic Pi 来实现，因为可以所见即所得的获得但发现随着规则越写越多，搜索的实现变得越来越麻烦。这时候就轮到我们 Prolog 出场了。

## 和弦

如果我们只考虑单个和弦，不处理和弦之间连接的话，我们先定义一个音符，是由其音名、升降调符号和所在的八度构成的，为了方便比较，我们不妨把这个音对应的 MIDI 调值算出来，于是我们可以先写一个这样的代码：

```prolog
% Notes
key_note(c). key_note(d). key_note(e). key_note(g). key_note(a). key_note(b).
semitone_offset(c, 0). semitone_offset(d, 2). semitone_offset(e, 4). semitone_offset(f, 5).
semitone_offset(g, 7). semitone_offset(a, 9). semitone_offset(b, 11).
note_offset(-2). note_offset(-1). note_offset(0). note_offset(1). note_offset(2).
octave(-2). octave(-1). octave(0). octave(1). octave(2). octave(3). octave(4). octave(5). octave(6). octave(7). octave(8). octave(9).

% Define note/3 predicate to check if a given Key, Offset, and Octave form a valid note
note(Key, Offset, Octave) :- key_note(Key), note_offset(Offset), octave(Octave).

% Define note_value/4 predicate to calculate the MIDI value for a given Key, Offset, and Octave
note_value(Key, Offset, Octave, Value) :-
  note(Key, Offset, Octave),
  semitone_offset(Key, KeyOffset),
  Value is 12 * (Octave + 1) + KeyOffset + Offset.

% Define note_value/2 predicate to calculate the MIDI value for a given note
note_value([Key, Offset, Octave], Value) :- note_value(Key, Offset, Octave, Value).
```

围绕这个系统，我们可以开始定义音域，我这里所使用的音域规则如下（括号之中的是 MIDI 调值）：
1. Soprano: C4 (60) - A5 (81)
2. Alto: F3 (53) - D5 (74)
3. Tenor: C3 (48) - A4 (69)
4. Bass: F2 (41) - D4 (62)

围绕这个机制，我们可以写出这样的规则：

```prolog
voice(soprano). voice(alto). voice(tenor). voice(bass).
voice_range(soprano, 60, 81).
voice_range(alto, 53, 74).
voice_range(tenor, 48, 69).
voice_range(bass, 41, 62).

% Define legal_note_range/4 predicate to check if a note is within the range for a given voice
legal_note_range(Voice, Key, Offset, Octave) :-
  voice(Voice), note(Key, Offset, Octave),
  voice_range(Voice, Min, Max), note_value(Key, Offset, Octave, Value), Value >= Min, Value =< Max.
```

除了各个声部本身能支持的音高，我们还需要处理两个问题：SATB 四部必须是严格地，S 的音高大于 A 的音高大于 T 的音高大于 B 的音高，以及女高和女低音、女低音和男高音之间不能超过一个完全八度，男高音和男低音之间不能超过完全十二度，这时候我们计算的 MIDI 调值可以很方便于我们构建这个规则：

```prolog
% Define legal_chord_range/4 predicate to check if a chord is within the ranges and respects voice leading
legal_chord_range(Soprano, Alto, Tenor, Bass) :-
  Soprano = [SopranoKey, SopranoOffset, SopranoOctave],
  Alto = [AltoKey, AltoOffset, AltoOctave],
  Tenor = [TenorKey, TenorOffset, TenorOctave],
  Bass = [BassKey, BassOffset, BassOctave],
  legal_note_range(soprano, SopranoKey, SopranoOffset, SopranoOctave),
  legal_note_range(alto, AltoKey, AltoOffset, AltoOctave),
  legal_note_range(tenor, TenorKey, TenorOffset, TenorOctave),
  legal_note_range(bass, BassKey, BassOffset, BassOctave),
  note_value(SopranoKey, SopranoOffset, SopranoOctave, SopranoValue),
  note_value(AltoKey, AltoOffset, AltoOctave, AltoValue),
  note_value(TenorKey, TenorOffset, TenorOctave, TenorValue),
  note_value(BassKey, BassOffset, BassOctave, BassValue),
  % Check for proper voice leading
  SopranoValue > AltoValue, AltoValue > TenorValue, TenorValue > BassValue,
  % Check for proper voice spacing
  SopranoValue - AltoValue =< 12,
  AltoValue - TenorValue =< 12,
  TenorValue - BassValue =< 19.
```

写到这里我们需要处理一下重复音的问题，写四部和声的时候，当给定一个三和弦，至少会出现一个重复音，在一些情况下，我们也会跳过某个三度或者五度。而七和弦比较特殊，一般七和弦的七音是一定要出现的，但是不能重复出现。另外，三全音（增四度或减五度）只能出现一次，调中的七音也不能重复出现，于是我们需要处理一下这几种情况：

```prolog
% Sclaes
scale(cmaj, [(c, 0), (d, 0), (e, 0), (f, 0), (g, 0), (a, 0), (b, 0)]).
scale(dmaj, [(d, 0), (e, 0), (f, 1), (g, 0), (a, 0), (b, 0), (c, 1)]).
scale(emaj, [(e, 0), (f, 1), (g, 1), (a, 0), (b, 0), (c, 1), (d, 1)]).
scale(fmaj, [(f, 0), (g, 0), (a, 0), (b, -1), (c, 0), (d, 0), (e, 0)]).
scale(gmaj, [(g, 0), (a, 0), (b, 0), (c, 0), (d, 0), (e, 0), (f, 1)]).
scale(amaj, [(a, 0), (b, 0), (c, 1), (d, 0), (e, 0), (f, 1), (g, 1)]).
scale(bmaj, [(b, 0), (c, 1), (d, 1), (e, 0), (f, 1), (g, 1), (a, 1)]).
scale(cmin, [(c, 0), (d, 0), (e, -1), (f, 0), (g, 0), (a, -1), (b, -1)]).
scale(dmin, [(d, 0), (e, 0), (f, 0), (g, 0), (a, 0), (b, 0), (c, 1)]).
scale(emin, [(e, 0), (f, 0), (g, 0), (a, 0), (b, 0), (c, 1), (d, 1)]).
scale(fmin, [(f, 0), (g, 0), (a, -1), (b, -1), (c, 0), (d, 0), (e, -1)]).
scale(gmin, [(g, 0), (a, 0), (b, 0), (c, 0), (d, 0), (e, 0), (f, 1)]).
scale(amin, [(a, 0), (b, 0), (c, 0), (d, 0), (e, 0), (f, 1), (g, 1)]).
scale(bmin, [(b, 0), (c, 1), (d, 1), (e, 0), (f, 1), (g, 1), (a, 1)]).

% Define is_augmented_fourth/2 predicate to check if two notes are an augmented fourth apart
is_augmented_fourth(note_value(Value1), note_value(Value2)) :-
  Interval is abs(Value1 - Value2),
  Interval =:= 6.

is_seventh_note_of_scale(Key, Offset, Scale) :-
  scale(Scale, Notes),
  last(Notes, (Key, Offset)).

% Define is_same_note_value/2 predicate to check if two notes have the same MIDI value
is_same_note_value(Value1, Value2) :-
  Value1 // 12 =:= Value2 // 12.

% Define legal_chord_repeat/4 predicate to check if a chord has repeated notes
legal_chord_repeat(Scale, Soprano, Alto, Tenor, Bass) :-
  Soprano = [SopranoKey, SopranoOffset, SopranoOctave],
  Alto = [AltoKey, AltoOffset, AltoOctave],
  Tenor = [TenorKey, TenorOffset, TenorOctave],
  Bass = [BassKey, BassOffset, BassOctave],
  note_value(SopranoKey, SopranoOffset, SopranoOctave, SopranoValue),
  note_value(AltoKey, AltoOffset, AltoOctave, AltoValue),
  note_value(TenorKey, TenorOffset, TenorOctave, TenorValue),
  note_value(BassKey, BassOffset, BassOctave, BassValue),
  % Check for argumented fourth
  % if the Soprano and Alto are argumented fourth apart, then the Tenor and Base should not use the same note
  \+ (is_augmented_fourth(SopranoValue, AltoValue), (is_same_note_value(SopranoValue, TenorValue); is_same_note_value(SopranoValue, BassValue); is_same_note_value(AltoValue, TenorValue); is_same_note_value(AltoValue, BassValue))),
  % if the Soporano and Tenor are argumented fourth apart, then the Alto and Base should not use the same note
  \+ (is_augmented_fourth(SopranoValue, TenorValue), (is_same_note_value(SopranoValue, AltoValue); is_same_note_value(SopranoValue, BassValue); is_same_note_value(TenorValue, AltoValue); is_same_note_value(TenorValue, BassValue))),
  % if the Soprano and Bass are argumented fourth apart, then the Alto and Tenor should not use the same note
  \+ (is_augmented_fourth(SopranoValue, BassValue), (is_same_note_value(SopranoValue, AltoValue); is_same_note_value(SopranoValue, TenorValue); is_same_note_value(BassValue, AltoValue); is_same_note_value(BassValue, TenorValue))),
  % if the Alto and Tenor are argumented fourth apart, then the Soprano and Bass should not use the same note
  \+ (is_augmented_fourth(AltoValue, TenorValue), (is_same_note_value(SopranoValue, AltoValue); is_same_note_value(SopranoValue, BassValue); is_same_note_value(AltoValue, TenorValue); is_same_note_value(BassValue, TenorValue))),
  % if the Alto and Bass are argumented fourth apart, then the Soprano and Tenor should not use the same note
  \+ (is_augmented_fourth(AltoValue, BassValue), (is_same_note_value(SopranoValue, AltoValue); is_same_note_value(SopranoValue, TenorValue); is_same_note_value(AltoValue, TenorValue); is_same_note_value(BassValue, TenorValue))),
  % if the Tenor and Bass are argumented fourth apart, then the Soprano and Alto should not use the same note
  \+ (is_augmented_fourth(TenorValue, BassValue), (is_same_note_value(SopranoValue, TenorValue); is_same_note_value(SopranoValue, BassValue); is_same_note_value(AltoValue, TenorValue); is_same_note_value(AltoValue, BassValue))),
  % Check for if it is the seventh note of the scale
  % if the Soprano is the seventh note of the scale, then the Alto, Tenor, and Bass should not use the same note
  \+ (is_seventh_note_of_scale(SopranoKey, SopranoOffset, Scale), (is_same_note_value(SopranoValue, AltoValue); is_same_note_value(SopranoValue, TenorValue); is_same_note_value(SopranoValue, BassValue))),
  % if the Alto is the seventh note of the scale, then the Soprano, Tenor, and Bass should not use the same note
  \+ (is_seventh_note_of_scale(AltoKey, AltoOffset, Scale), (is_same_note_value(AltoValue, SopranoValue); is_same_note_value(AltoValue, TenorValue); is_same_note_value(AltoValue, BassValue))),
  % if the Tenor is the seventh note of the scale, then the Soprano, Alto, and Bass should not use the same note
  \+ (is_seventh_note_of_scale(TenorKey, TenorOffset, Scale), (is_same_note_value(TenorValue, SopranoValue); is_same_note_value(TenorValue, AltoValue); is_same_note_value(TenorValue, BassValue))),
  % if the Bass is the seventh note of the scale, then the Soprano, Alto, and Tenor should not use the same note
  \+ (is_seventh_note_of_scale(BassKey, BassOffset, Scale), (is_same_note_value(BassValue, SopranoValue); is_same_note_value(BassValue, AltoValue); is_same_note_value(BassValue, TenorValue))).  

legal_chord_contents(Chord, Soprano, Alto, Tenor, Bass) :-
  Soprano = [SopranoKey, SopranoOffset, _],
  Alto = [AltoKey, AltoOffset, _],
  Tenor = [TenorKey, TenorOffset, _],
  Bass = [BassKey, BassOffset, _],
  SopranoSignature = (SopranoKey, SopranoOffset),
  AltoSignature = (AltoKey, AltoOffset),
  TenorSignature = (TenorKey, TenorOffset),
  BassSignature = (BassKey, BassOffset),
  % all four parts should be in the chord
  member(SopranoSignature, Chord), member(AltoSignature, Chord), member(TenorSignature, Chord), member(BassSignature, Chord),
  % if the chord is a seventh chord (four members), do not repeat the seventh note
  (length(Chord, ChordSize), (ChordSize =\= 4; (
    (last(Chord, SecenthNote),
    % length [SopranoSignature, AltoSignature, TenorSignature, BassSignature] whose note is the same as the seventh note
    exclude(=(SecenthNote), [SopranoSignature, AltoSignature, TenorSignature, BassSignature], FilteredSeventhNote),
    length(FilteredSeventhNote, FilteredSeventhNoteSize), 4 - FilteredSeventhNoteSize =:= 1)))).
```

现在我们把这些东西组合起来，不妨来试试效果：

```prolog
% Combination of all predicates to check if a chord is legal
legal_chord(Scale, Chord, Soprano, Alto, Tenor, Bass) :-
  legal_chord_range(Soprano, Alto, Tenor, Bass),
  legal_chord_repeat(Scale, Soprano, Alto, Tenor, Bass),
  legal_chord_contents(Chord, Soprano, Alto, Tenor, Bass).
```

我们给定一个 G 大调下的一组 G 和弦，让它判断这是不是非法的：

```prolog
?- legal_chord(gmaj, [(g, 0), (b, 0), (d, 0)], [g, 0, 5], [b, 0, 4], [d, 0, 4], [g, 0, 2]).
true.
```

非常好！

## 连接

连接的问题比和弦本身更麻烦，我们有六个问题需要解决：
1. 避免四个声部沿着完全相同方向移动
2. 避免声部超越（也就是两个连续的、包含至少两个独立声部的和音，当后和音中较低的声部的音比前和音中较高的声部的音要高，或者当后和音中较高的声部的音比前和音中较低的声部的音要低）
3. 避免出现方向相同的平行八度
4. 避免出现方向相同的平行五度
5. 避免出现隐伏八度
6. 避免出现隐伏五度

写出来大概是这样的：

```prolog
legal_connection(Chord1, Chord2) :-
  Chord1 = [Soprano1, Alto1, Tenor1, Bass1],
  Chord2 = [Soprano2, Alto2, Tenor2, Bass2],
  note_value(Soprano1, SopranoValue1), note_value(Alto1, AltoValue1), note_value(Tenor1, TenorValue1), note_value(Bass1, BassValue1),
  note_value(Soprano2, SopranoValue2), note_value(Alto2, AltoValue2), note_value(Tenor2, TenorValue2), note_value(Bass2, BassValue2),
  % all four parts should not move in parallel motion
  \+ (SopranoValue1 > SopranoValue2, AltoValue1 > AltoValue2, TenorValue1 > TenorValue2, BassValue1 > BassValue2),
  \+ (SopranoValue1 < SopranoValue2, AltoValue1 < AltoValue2, TenorValue1 < TenorValue2, BassValue1 < BassValue2),
  \+ (SopranoValue1 = SopranoValue2, AltoValue1 = AltoValue2, TenorValue1 = TenorValue2, BassValue1 = BassValue2),
  % avoid voice overlapping
  SopranoValue2 >= AltoValue1,
  AltoValue2 >= TenorValue1, AltoValue2 =< SopranoValue1,
  TenorValue2 >= BassValue1, TenorValue2 =< AltoValue1,
  BassValue2 =< TenorValue1,
  % no parallel 8th
  \+ ((SopranoValue1 - AltoValue1) // 12 =:= 0, (SopranoValue2 - AltoValue2) // 12 =:= 0, SopranoValue2 > SopranoValue1, AltoValue2 > AltoValue1),
  \+ ((SopranoValue1 - AltoValue1) // 12 =:= 0, (SopranoValue2 - AltoValue2) // 12 =:= 0, SopranoValue2 < SopranoValue1, AltoValue2 < AltoValue1),
  \+ ((SopranoValue1 - AltoValue1) // 12 =:= 0, (SopranoValue2 - AltoValue2) // 12 =:= 0, SopranoValue2 = SopranoValue1, AltoValue2 = AltoValue1),
  \+ ((SopranoValue1 - TenorValue1) // 12 =:= 0, (SopranoValue2 - TenorValue2) // 12 =:= 0, SopranoValue2 > SopranoValue1, TenorValue2 > TenorValue1),
  \+ ((SopranoValue1 - TenorValue1) // 12 =:= 0, (SopranoValue2 - TenorValue2) // 12 =:= 0, SopranoValue2 < SopranoValue1, TenorValue2 < TenorValue1),
  \+ ((SopranoValue1 - TenorValue1) // 12 =:= 0, (SopranoValue2 - TenorValue2) // 12 =:= 0, SopranoValue2 = SopranoValue1, TenorValue2 = TenorValue1),
  \+ ((SopranoValue1 - BassValue1) // 12 =:= 0, (SopranoValue2 - BassValue2) // 12 =:= 0, SopranoValue2 > SopranoValue1, BassValue2 > BassValue1),
  \+ ((SopranoValue1 - BassValue1) // 12 =:= 0, (SopranoValue2 - BassValue2) // 12 =:= 0, SopranoValue2 < SopranoValue1, BassValue2 < BassValue1),
  \+ ((SopranoValue1 - BassValue1) // 12 =:= 0, (SopranoValue2 - BassValue2) // 12 =:= 0, SopranoValue2 = SopranoValue1, BassValue2 = BassValue1),
  \+ ((AltoValue1 - TenorValue1) // 12 =:= 0, (AltoValue2 - TenorValue2) // 12 =:= 0, AltoValue2 > AltoValue1, TenorValue2 > TenorValue1),
  \+ ((AltoValue1 - TenorValue1) // 12 =:= 0, (AltoValue2 - TenorValue2) // 12 =:= 0, AltoValue2 < AltoValue1, TenorValue2 < TenorValue1),
  \+ ((AltoValue1 - TenorValue1) // 12 =:= 0, (AltoValue2 - TenorValue2) // 12 =:= 0, AltoValue2 = AltoValue1, TenorValue2 = TenorValue1),
  \+ ((AltoValue1 - BassValue1) // 12 =:= 0, (AltoValue2 - BassValue2) // 12 =:= 0, AltoValue2 > AltoValue1, BassValue2 > BassValue1),
  \+ ((AltoValue1 - BassValue1) // 12 =:= 0, (AltoValue2 - BassValue2) // 12 =:= 0, AltoValue2 < AltoValue1, BassValue2 < BassValue1),
  \+ ((AltoValue1 - BassValue1) // 12 =:= 0, (AltoValue2 - BassValue2) // 12 =:= 0, AltoValue2 = AltoValue1, BassValue2 = BassValue1),
  \+ ((TenorValue1 - BassValue1) // 12 =:= 0, (TenorValue2 - BassValue2) // 12 =:= 0, TenorValue2 > TenorValue1, BassValue2 > BassValue1),
  \+ ((TenorValue1 - BassValue1) // 12 =:= 0, (TenorValue2 - BassValue2) // 12 =:= 0, TenorValue2 < TenorValue1, BassValue2 < BassValue1),
  \+ ((TenorValue1 - BassValue1) // 12 =:= 0, (TenorValue2 - BassValue2) // 12 =:= 0, TenorValue2 = TenorValue1, BassValue2 = BassValue1),
  % % no parallel 5th
  \+ ((SopranoValue1 - AltoValue1) // 7 =:= 0, (SopranoValue2 - AltoValue2) // 7 =:= 0, SopranoValue2 > SopranoValue1, AltoValue2 > AltoValue1),
  \+ ((SopranoValue1 - AltoValue1) // 7 =:= 0, (SopranoValue2 - AltoValue2) // 7 =:= 0, SopranoValue2 < SopranoValue1, AltoValue2 < AltoValue1),
  \+ ((SopranoValue1 - AltoValue1) // 7 =:= 0, (SopranoValue2 - AltoValue2) // 7 =:= 0, SopranoValue2 = SopranoValue1, AltoValue2 = AltoValue1),
  \+ ((SopranoValue1 - TenorValue1) // 7 =:= 0, (SopranoValue2 - TenorValue2) // 7 =:= 0, SopranoValue2 > SopranoValue1, TenorValue2 > TenorValue1),
  \+ ((SopranoValue1 - TenorValue1) // 7 =:= 0, (SopranoValue2 - TenorValue2) // 7 =:= 0, SopranoValue2 < SopranoValue1, TenorValue2 < TenorValue1),
  \+ ((SopranoValue1 - TenorValue1) // 7 =:= 0, (SopranoValue2 - TenorValue2) // 7 =:= 0, SopranoValue2 = SopranoValue1, TenorValue2 = TenorValue1),
  \+ ((SopranoValue1 - BassValue1) // 7 =:= 0, (SopranoValue2 - BassValue2) // 7 =:= 0, SopranoValue2 > SopranoValue1, BassValue2 > BassValue1),
  \+ ((SopranoValue1 - BassValue1) // 7 =:= 0, (SopranoValue2 - BassValue2) // 7 =:= 0, SopranoValue2 < SopranoValue1, BassValue2 < BassValue1),
  \+ ((SopranoValue1 - BassValue1) // 7 =:= 0, (SopranoValue2 - BassValue2) // 7 =:= 0, SopranoValue2 = SopranoValue1, BassValue2 = BassValue1),
  \+ ((AltoValue1 - TenorValue1) // 7 =:= 0, (AltoValue2 - TenorValue2) // 7 =:= 0, AltoValue2 > AltoValue1, TenorValue2 > TenorValue1),
  \+ ((AltoValue1 - TenorValue1) // 7 =:= 0, (AltoValue2 - TenorValue2) // 7 =:= 0, AltoValue2 < AltoValue1, TenorValue2 < TenorValue1),
  \+ ((AltoValue1 - TenorValue1) // 7 =:= 0, (AltoValue2 - TenorValue2) // 7 =:= 0, AltoValue2 = AltoValue1, TenorValue2 = TenorValue1),
  \+ ((AltoValue1 - BassValue1) // 7 =:= 0, (AltoValue2 - BassValue2) // 7 =:= 0, AltoValue2 > AltoValue1, BassValue2 > BassValue1),
  \+ ((AltoValue1 - BassValue1) // 7 =:= 0, (AltoValue2 - BassValue2) // 7 =:= 0, AltoValue2 < AltoValue1, BassValue2 < BassValue1),
  \+ ((AltoValue1 - BassValue1) // 7 =:= 0, (AltoValue2 - BassValue2) // 7 =:= 0, AltoValue2 = AltoValue1, BassValue2 = BassValue1),
  \+ ((TenorValue1 - BassValue1) // 7 =:= 0, (TenorValue2 - BassValue2) // 7 =:= 0, TenorValue2 > TenorValue1, BassValue2 > BassValue1),
  \+ ((TenorValue1 - BassValue1) // 7 =:= 0, (TenorValue2 - BassValue2) // 7 =:= 0, TenorValue2 < TenorValue1, BassValue2 < BassValue1),
  \+ ((TenorValue1 - BassValue1) // 7 =:= 0, (TenorValue2 - BassValue2) // 7 =:= 0, TenorValue2 = TenorValue1, BassValue2 = BassValue1),
  % % no hidden 8th
  \+ (SopranoValue2 - SopranoValue1 > 5, BassValue2 > BassValue1, (SopranoValue2 - BassValue2) // 12 =:= 0),
  \+ (SopranoValue1 - SopranoValue2 > 5, BassValue1 > BassValue2, (SopranoValue2 - BassValue2) // 12 =:= 0),
  % % no hidden 5th
  \+ (SopranoValue2 - SopranoValue1 > 5, BassValue2 > BassValue1, (SopranoValue2 - BassValue2) // 7 =:= 0),
  \+ (SopranoValue1 - SopranoValue2 > 5, BassValue1 > BassValue2, (SopranoValue2 - BassValue2) // 7 =:= 0).
```

## 尝试生成

现在我们有了判断连接和和弦的东西，我们只需要给出低音声部和和弦，接下来就可以让 Prolog 全自动生成四部和声了，

```prolog
generate_chord(_, []).
generate_chord(Scale, [Ans1, Ans2 | Rest]) :-
  Ans1 = [Chord1, Soprano1, Alto1, Tenor1, Bass1],
  Ans2 = [Chord2, Soprano2, Alto2, Tenor2, Bass2],
  legal_chord(Scale, Chord1, Soprano1, Alto1, Tenor1, Bass1),
  legal_chord(Scale, Chord2, Soprano2, Alto2, Tenor2, Bass2),
  legal_connection([Soprano1, Alto1, Tenor1, Bass1], [Soprano2, Alto2, Tenor2, Bass2]),
  generate_chord(Scale, [Ans2 | Rest]).
generate_chord(Scale, [Ans]) :-
  Ans = [Chord, Soprano, Alto, Tenor, Bass],
  legal_chord(Scale, Chord, Soprano, Alto, Tenor, Bass).
```

我从网上找了个例题，题目如下：

![SATB 样例题目](/assets/images/satb-example-question.png)

Prolog 这么做还是有一些缺陷，比如它不会总是选那个离自己最近的音来过渡，其实可以让 Prolog 把可能性都生成出来，然后在外面写一个某种评估函数进行排序其实就能解决问题，我这里就直接手动挑选了比较正常的结果，确实也不累。

下面请欣赏 Prolog 生成的四部和声：

![SATB 生成答案](/assets/images/satb-example-answer.png)

<audio src="/assets/audio/satb-generated.mp3" controls></audio>

您还别说，还别说，真的还行其实。
