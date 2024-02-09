---
title: "硬核解谜 Prolog 入门 —— 13 Archetypes: Skeptic"
date: 2020-01-18 22:29:29
tags: [Prolog, 逻辑学, 游戏, 设计, Ingress]
---

13 Archetypes: Skeptic 的谜题由三个单独的谜题组成。这三个谜题唯一的共同点就是，它们都有一个错误。

![13 Archetypes: Skeptic](/assets/images/archetypes-skeptic.jpg)

第二个谜题是「数迷」，第三个就是个数学的填字游戏（crossword），其实没什么好讲的，我们来好好讲一下第一个谜题。

## 起因

第一个谜题是一个可以完全用一阶谓词逻辑来描述的问题，也就是使用「断言」「量化」和命题的组合来描述。

上了一年我们学校萩野達也教授的计算理论 + 逻辑课。萩野教授是日本研究第五代计算机时候成长出来的计算机科学家，数学基础异常过硬，和现在天天写 JavaScript、Python 还叫苦不迭的这些 CS 学生完全不在一个级别上。

上了萩野一年课之后，我觉得我翅膀也硬了。既然这玩意可以用一阶谓词逻辑来描述，那么应该可以用 Prolog 来解决。第一个题目因为答案选项里有 E、1、3、A、R，因为 13 Archetypes 所有题目都以 13AR 结束，所以显然根本不用看题就知道答案是 E13AR。但如果仔仔细细做一做的话你会发现，其实 10 分钟也做完了。

但是我不管，我就要用 Prolog 来解决，重现一下上世纪 90 年代人工智能研究的荣耀。

## Prolog 是什么？

Prolog 是 Programming in Logic 的缩写。Prolog 程序基于一阶谓词逻辑理论。基本就是描述逻辑，然后让 Prolog 解释器来求解。我这里使用的 Prolog 解释器是 SWI-Prolog。SWI-Prolog 是一个从 1987 年开始开发的自由的 Prolog 解释器。我毕竟也不是 Prolog 语言的专家，对我来说找个开源软件临时用一下就行了。

Prolog 的语法非常简单，几分钟就上手了。Prolog 里大写开头的字母就是变量，其他的就是常量，每个语句必须以句号结尾。第一步是描述事实，比如 Alice 是一个人：

```prolog
human(alice).
```

事实可以是一个常量的事实，也可以是多个常量之间的，比如 Alice 喜欢 Bob：

```
likes(alice, bob).
```

然后就是推理关系。符号是 `:-` ，符号后面的如果成立，那么符号前面的也会成立，比如两个人互相喜欢，那么他们就是朋友：

```prolog
friend(X, Y) :-
	likes(X, Y),
	likes(Y, X).
```

其中 `,` 逗号表示「且 (and)」，如果是「或 (or)」那么就是 `;` 分号。然后 `\+` 表示否，最后还可以用 `()` 来组合它们，基本上这就是基本逻辑了。

比如冬马喜欢春希，雪菜喜欢冬马和春希，春希没有去音乐会。

```prolog
human(haruki).
human(touma).
human(setsuna).

likes(touma, haruki).
likes(setsuna, haruki).
likes(setsuna, touma).

friend(X, Y) :-
	human(X),
	human(Y),
	likes(X, Y),
	likes(Y, X).
```

然后我们用 SWI-Prolog 加载这个程序，来看看他们之间谁是朋友。

```
❯ swipl white.pl
Welcome to SWI-Prolog (threaded, 64 bits, version 8.0.3)
SWI-Prolog comes with ABSOLUTELY NO WARRANTY. This is free software.
Please run ?- license. for legal details.

For online help and background, visit http://www.swi-prolog.org
For built-in help, use ?- help(Topic). or ?- apropos(Word).

?- friend(X, Y).
false.
```

程序返回了 `false.` 显然他们三个当不成朋友。

## 写 Prolog

一开始我花了一个小时，吭哧吭哧就写完了程序。一开始约束条件写得太松散，嵌套太多太复杂，结果跑了好几个小时都没跑出来。

> 第一次有了喜欢的事情（指数学）。有了能做一辈子的事业（指写代码）。两件快乐事情重合在一起。而这两份快乐，又给我带来更多的快乐。得到的，本该是像梦境一般幸福的时间……但是，为什么，会变成这样呢……

第二天想到了这个问题和一个经典逻辑问题「谁养鱼」是比较类似的，翻了翻正好看到这个问题在 SWI-Prolog 的[例子](https://swish.swi-prolog.org/p/fish_puzzle_marijke.pl)里。学习了一下别人是怎么写的，改了一下程序竟然就秒解了。最后的程序如下：

```prolog
% 1. Alice is level 7.
% 2. The level 7 agent was in squad Foot-E.
% 3. The agent in squad Foot-E started playing in 2016.
% 4. Alice started playing in 2018.
% 5. The level 16 agent has a founders badge.
% 6. Bob has a gold innovator badge.
% 7. David started playing before Eve, who started playing before Clarie.
% 8. The agent who started playing in 2014 was on squad Bike-1.
% 9. The agent who started playing in 2015 is the recursed level 12 agent.
% 10. The agent in squad Bike-3 was level 9.
% 11. Clarie started playing in 2016.
% 12. The two agents who were on bikes were the agent who started in 2016 and the level 13 agent.
% 13. The level 16 agent was in squad Foot-A.
% 14. The agent who started in 2015 was in squad Foot-R.
% 15. Bob was on a bike.
% 16. The agent who started playing in 2013 was in squad Foot-A.

year(2013). year(2014). year(2015). year(2016). year(2017). year(2018). year(2019).
earlier(A, B) :- year(A), year(B), A @< B.

agents(Ag):-
  % each agent in the list Ag of agent is represented as:
  %      h(Name, Level, Year, Type, Squad)
  length(Ag, 5),
  member(h(alice, 7, _, _, _), Ag), % 1
  member(h(_, 7, _, foot, e), Ag), % 2
  member(h(_, _, 2016, foot, e), Ag), % 3
  member(h(alice, _, 2018, _, _), Ag), % 4
  member(h(_, 16, 2013, _, _), Ag), % 5
  (member(h(bob, _, 2014, _, _), Ag); member(h(bob, _, 2013, _, _), Ag)), % 6
  member(h(david, _, Yd, _, _), Ag), % 7
  member(h(eve, _, Ye, _, _), Ag),
  member(h(clarie, _, Yc, _, _), Ag),
  earlier(Yd, Ye), earlier(Ye, Yc),
  member(h(_, _, 2014, bike, one), Ag), % 8
  member(h(_, 12, 2015, _, _), Ag), % 9
  member(h(_, 9, _, bike, three), Ag), % 10
  member(h(clarie, _, 2016, _, _), Ag), % 11
  member(h(A, _, _, bike, _), Ag), % 12
  member(h(B, _, _, bike, _), Ag),
  \+ A=B,
  ((member(h(A, 13, _, _, _), Ag), member(h(B, _, 2016, _, _), Ag));
  (member(h(B, 13, _, _, _), Ag), member(h(A, _, 2016, _, _), Ag))),
  member(h(_, 16, _, foot, a), Ag), % 13
  member(h(_, _, 2015, foot, r), Ag), % 14
  member(h(bob, _, _, bike, _), Ag), % 15
  member(h(_, _, 2013, foot, a), Ag). % 16

```

其中几个需要注意的是，一个是加入年份，为了程序能够在有限的整数空间里搜索，设定了条件：

```prolog
year(2013). year(2014). year(2015). year(2016). year(2017). year(2018). year(2019).
earlier(A, B) :- year(A), year(B), A @< B.
```

然后 Bob 有 Innovator 牌，光这个条件是不能判断具体是 2013 还是 2014 加入游戏的，所以写了一个或条件：

```prolog
(member(h(bob, _, 2014, _, _), Ag); member(h(bob, _, 2013, _, _), Ag)), % 6
```

还有一条就是两个自行车队的一个是 2016 年加入的，另一个是 13 级，这句话转换成谓词逻辑有点绕：

```prolog
member(h(A, _, _, bike, _), Ag), % 12
member(h(B, _, _, bike, _), Ag),
\+ A=B,
((member(h(A, 13, _, _, _), Ag), member(h(B, _, 2016, _, _), Ag));
(member(h(B, 13, _, _, _), Ag), member(h(A, _, 2016, _, _), Ag))),
```

跑了一下，当然是 `false.`。因为其中有一个条件是错误的。于是从第一个条件开始尝试一个个注释掉。

注释到第四个，秒解：

```
❯ swipl archetype.pl 
Welcome to SWI-Prolog (threaded, 64 bits, version 8.0.3)
SWI-Prolog comes with ABSOLUTELY NO WARRANTY. This is free software.
Please run ?- license. for legal details.

For online help and background, visit http://www.swi-prolog.org
For built-in help, use ?- help(Topic). or ?- apropos(Word).

?- agents(Ag).
Ag = [h(alice, 7, 2016, foot, e), h(david, 16, 2013, foot, a), h(bob, 13, 2014, bike, one), h(eve, 12, 2015, foot, r), h(clarie, 9, 2016, bike, three)] .
```

程序给出的第一组解如下（不唯一）：

| 名字   | 级别 | 加入年份 | Squad 类型 | Squad 编号 |
| ------ | ---- | -------- | ---------- | ---------- |
| Alice  | 7    | 2016     | Foot       | E          |
| Bob    | 13   | 2014     | Bike       | 1          |
| Clarie | 9    | 2016     | Bike       | 3          |
| David  | 16   | 2013     | Foot       | A          |
| Eve    | 12   | 2015     | Foot       | R          |

答案是 `E13AR`，完全没有错。

## 总结

这个例子很好展示了上世纪 90 年代人工智能研究的奥妙。还有就是不要钻牛角尖，几秒钟就解出来的题不要浪费一天去写代码。但我写了一天才写出来只能说明是我菜，这程序运行只花了几毫秒就找到解了还是挺厉害的。数学的发展可能关乎着真正强人工智能的诞生，希望大家多学数学，少写 JavaScript。
