---
title: 柯洁能成为九冠王吗？
date: 2024-05-20 02:40:00 +0800
tags: [统计学, Elo]
---

柯洁（九段）的上一个世界冠军可以追溯到 2020 年三星杯 2-0 胜申真谞（九段），连续两年没有获得世界冠军，上一次和世界冠军失之交臂是 2023 年亚运会负許皓鋐（九段）。不由让人疑问，柯洁在职业生涯再拿一个世界冠军的概率有多少？为了回答这个问题，我们需要一个衡量棋手水平的模型。一个比较简单的方法就是通过 Elo 等级分。

## Bradley-Terry 模型

今天的 Elo 等级分和 Arpad Elo 的原始假设有比较大的差异，尤其是在选择的分布上。一般我们会假设使用 Bradley-Terry 模型构造 elo 等级分模型，即对于两个选手 A 和 B，A 获胜的概率是
$$
P(A>B)=\frac{10^\frac{R(A)}{400}}{10^\frac{R(A)}{400}+10^\frac{R(B)}{400}}
$$
然而当两位选手下完一盘棋后，后验的结果如何反馈到 elo 分的修正上是一个比较大的问题。一个比较好的算法是 [WHR Algorithm](https://www.remi-coulom.fr/WHR/)，也就是 [Go Ratings](https://goratings.org) 使用的算法。

举个例子来说，我们以截止到 2024 年 5 月 10 日数据的最新等级分来看的话，如果我们比较柯洁和战鹰（二段）的等级分的话，柯洁的等级分是 3694 分，而战鹰的等级分是 2824 分。如果明天就有一盘柯洁和申真谞的比赛的话，那么战鹰战胜柯洁的概率是 0.66%，从中可以看出顶尖职业棋手间微妙的差距。

申真谞的等级分是 3877 分，所以柯洁对战申真谞胜出的概率是 25.86%。而柯洁前两天爆冷负刘宇航（六段），刘宇航的等级分是 3536 分，发生这样爆冷的概率是 28.70%。从这个角度来看，柯洁能赢申真谞本身也属于一种爆冷了。

## 建模

然而锦标赛并不是单纯的胜率对比，一名棋手在淘汰赛中需要进行多轮比赛，即使胜率总是 90%，连续赢 8 轮的概率也只有 43%。本着「量子力学量力学，随机过程随机过」的原则，我们不如使用蒙特卡洛方法来模拟这些锦标赛的规则，从而能更好估计夺冠的分布。

我们考虑下面的个人世界围棋赛事：应式杯、春兰杯、三星杯、LG 杯、梦百合杯、烂柯杯。应式杯每四年举办一次，而亚运会下一届是不是正式项目不确定。北海新绎杯由于今年第一年办，不确定之后的赛制以及会不会继续办，先不纳入计算。

为了简化计算，我们不妨做下面的假设：

1. 我们定义淘汰赛为跳过全部轮空轮次的淘汰赛。
2. 进入淘汰赛 n 个名额的总是世界排名前 n 名的选手。
3. 柯洁总能进入淘汰赛。
4. 进入淘汰赛的 n 名棋手的比赛顺序完全随机，忽略部分比赛的同一国家 / 地区选手回避原则。
5. 模拟过程中各选手 elo 分不再变化。

于是我们简化各个杯赛模拟规则如下：

| 杯赛     | 本赛选手数 | 规则                               |
| -------- | ---------- | ---------------------------------- |
| 应式杯   | 16         | 单败淘汰，半决赛三番棋，决赛五番棋 |
| 春兰杯   | 16         | 单败淘汰，决赛三番棋               |
| 三星杯   | 16         | 单败淘汰，决赛三番棋               |
| LG 杯    | 32         | 单败淘汰，决赛三番棋               |
| 梦百合杯 | 32         | 单败淘汰，半决赛三番棋，决赛五番棋 |
| 烂柯杯   | 32         | 单败淘汰，决赛三番棋               |

## 胜率

由于选手数量不会超过 32 人，我们可以先根据等级分计算一下世界前 32 强棋手之间的胜率矩阵来加速计算：

```ruby
ELO_RATINGS = {
  "Shin Jinseo": 3877,
  "Ke Jie": 3694,
  "Park Junghwan": 3672,
  "Wang Xinghao": 3671,
  "Gu Zihao": 3660,
  "Li Qincheng": 3660,
  "Ding Hao": 3658,
  "Li Xuanhao": 3655,
  "Byun Sangil": 3653,
  "Yang Dingxin": 3652,
  "Zhao Chenyu": 3637,
  "Fan Tingyu": 3620,
  "Yang Kaiwen": 3617,
  "Dang Yifei": 3616,
  "Li Weiqing": 3608,
  "Lian Xiao": 3608,
  "Mi Yuting": 3594,
  "Iyama Yuta": 3585,
  "Xu Jiayang": 3584,
  "Shin Minjun": 3583,
  "Ichiriki Ryo": 3582,
  "Liao Yuanhe": 3581,
  "Xu Haohong": 3581,
  "Xie Erhao": 3563,
  "Xie Ke": 3562,
  "Shi Yue": 3560,
  "Kang Dongyun": 3558,
  "Jiang Weijie": 3556,
  "Shibano Toramaru": 3555,
  "Tan Xiao": 3548,
  "Tuo Jiaxi": 3541,
  "Kim Myounghoon": 3536,
}

PLAYERS_INDEXES = ELO_RATINGS.each_with_index.map do |pair, index|
  [pair[0].to_s, index]
end.to_h

WINNING_RATES = (ELO_RATINGS.map do |_, rating_1|
  ELO_RATINGS.map do |_, rating_2|
    10 ** (rating_1.to_f / 400) / (10 ** (rating_1.to_f / 400) + 10 ** (rating_2.to_f / 400))
  end
end)

# Returns the possibility of winning
def beats(a, b)
  WINNING_RATES[PLAYERS_INDEXES[a]][PLAYERS_INDEXES[b]]
end
```

## 构造比赛规则

我们构造一个辅助函数，能选出排名前 n 的选手：

```ruby
# Pick first n players
def pick_players(n)
  ELO_RATINGS.keys[0...n]
end
```

然后我们模拟两选手下 n 番棋，返回胜者：

```ruby
# Emulate a game between two players
def emulate_game(a, b, num)
  num.times.map { rand < beats(a, b) }.count(true) > num / 2 ? a : b
end
```

然后我们先模拟一下应式杯的规则：

```ruby
def simulate_yingshi_tournament
  players = pick_players(16).shuffle
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 1) } # 8 winners
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 1) } # 4 winners
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 3) } # 2 winners
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 5) } # 1 winner
  players.first
end
```

我们来模拟 1000 次，看看结果：

```ruby
p 1000.times.map { simulate_yingshi_tournament }.tally.sort_by { |_, v| -v }.to_h

# {"Shin Jinseo"=>553, "Ke Jie"=>68, "Park Junghwan"=>54, "Li Qincheng"=>44, "Wang Xinghao"=>40, "Li Xuanhao"=>33, "Byun Sangil"=>32, "Gu Zihao"=>31, "Ding Hao"=>30, "Yang Dingxin"=>28, "Zhao Chenyu"=>24, "Li Weiqing"=>16, "Dang Yifei"=>16, "Fan Tingyu"=>12, "Yang Kaiwen"=>11, "Lian Xiao"=>8}
```

申真谞赢下了其中的 553 个冠军，而柯洁赢下了其中的 68 个。

## 模拟

我们先试图模拟一下 2024 年的情况，也就是全部 6 个杯赛都会举行。

```ruby
def simulate_yingshi_tournament
  players = pick_players(16).shuffle
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 1) } # 8 winners
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 1) } # 4 winners
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 3) } # 2 winners
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 5) } # 1 winner
  players.first
end

def simulate_chunlan_tournament
  players = pick_players(16).shuffle
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 1) } # 8 winners
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 1) } # 4 winners
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 1) } # 2 winners
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 3) } # 1 winner
  players.first
end

def simulate_samsung_tournament
  players = pick_players(16).shuffle
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 1) } # 8 winners
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 1) } # 4 winners
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 1) } # 2 winners
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 3) } # 1 winner
  players.first
end

def simulate_lg_tournament
  players = pick_players(32).shuffle
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 1) } # 16 winners
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 1) } # 8 winners
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 1) } # 4 winners
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 1) } # 2 winner
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 3) } # 1 winner
  players.first
end

def simulate_mengbaihe_tournament
  players = pick_players(32).shuffle
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 1) } # 16 winners
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 1) } # 8 winners
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 1) } # 4 winners
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 3) } # 2 winner
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 5) } # 1 winner
  players.first
end

def simulate_lanke_tournament
  players = pick_players(32).shuffle
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 1) } # 16 winners
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 1) } # 8 winners
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 1) } # 4 winners
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 1) } # 2 winner
  players = players.each_slice(2).map { |a, b| emulate_game(a, b, 3) } # 1 winner
  players.first
end

def has_kejie_won_tournament
  [simulate_chunlan_tournament, simulate_samsung_tournament, simulate_lg_tournament, simulate_mengbaihe_tournament, simulate_lanke_tournament].count("Ke Jie")
end
```

模拟 10000 次后，柯洁期望是每年获得 0.3881 个世界冠军。如果应式杯不举行，模拟了 10000 次，柯洁获得至少一个冠军的概率是 0.3256。由于应式杯每四年举行一次，我们取其期望，也就是柯洁每年能获得 **0.3412** 个世界冠军。

## 生涯估计

整体来说，淘汰赛如果不采用单败淘汰制，例如在半决赛和决赛种引入三番棋和五番棋，那么其就会体现为一个方差更小的二项分布，会更能体现实力本身的差距，相对来说运气成分会变小。像应式杯和梦百合杯这种决赛五番棋的，对于柯洁来说，如果正好对上申真谞就会变得极为不利，反之就会变得极为有利。在这个大前提下，锦标赛引入的这种不确定性增大了柯洁夺冠的概率。

然而，对于一名围棋棋手而言，其生涯是有限的，我们无法相信柯洁会一直维持他职业生涯的巅峰状态，随着时间，他的状态会越来越差。如果我们假设柯洁能在未来 5 年内都能维持这样的表现不下滑，那么其**至少再拿一个世界冠军的概率是 88.60%**；如果在未来 8 年内都能维持这样的表现不下滑，那么其**至少再拿一个世界冠军的概率是 96.56%**。整体来说，成为九冠王的概率还是比较大的。

