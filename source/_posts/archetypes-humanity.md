---
title: "元胞自动机 101 —— 13 Architypes: Humanist"
date: 2020-01-14 14:22:47
tags: [元胞自动机, 混沌, 游戏, 设计, Ingress]
---

## 序

前一晚改论文改睡着了，结果错过了。早上醒来发现题目还挺简单的，来聊一聊谜题的出题思路吧。

![题目](https://storage.googleapis.com/ingress-internal-event-data/13archetypes/humanist/humanist2_a6f9854f-6e0c-5050-a10d-91f4aaa4c382.png)

## 一种新科学

元胞自动机是一种由元胞、元胞状态、领域、和状态更新规则构成的一个自动机，其数学表达为：

$$ A = (L, d, S, N, f) $$

换成人话就是说，对于一个 d 维度的有限状态矩阵，每次每个格点的状态更新取决于其上一次中周围格点的状态。元胞自动机是研究混沌现象的一个重要模型，在 Wolfram 出版的书籍《一种新科学》（我个人对这是不是一种新科学持保留态度）中被进行了非常深入的探讨。

Wolfram 把初等一维元胞自动机的全部 256 种规则进行了分类，分成了四类：平稳型、周期型、混沌型、复杂型。

- 平稳型就是无论初始局面如何，最后终会变成稳定均匀状态。
- 周期性则是无论初始局面如何，最终会变成稳定的震荡结构，随机性会被过滤。
- 混沌型指初始局面会演化成一个伪随机或混沌状状态。
- 复杂型则是上面的一系列组合，可能会出现混沌，而一部分的初始结构会稳定保留，结构于结构会相互作用。（比如像规则 110 被证明是图灵完备的，那么其局面自然会随着初始输入变化有很大变化，可能稳定可能混沌了。）

但这个分类有个很大的问题，就是每一个类别几乎都能找到例外。所以这到底是一种新科学还是一种伪科学还是有点迷幻的。

## 最有名的元胞自动机

最有名的元胞自动机是 1970 年由英国数学家约翰·何顿·康威提出的康威生命游戏（Conway's Game of Life）其规则如下：

- 元胞自动机的维度是 2（运行在 2 维平面上）
- 状态只有两种，存活和死亡
- 当周围存活元胞是 0 个或 1 个时，其会死亡（太孤独了，无聊死了）
- 当周围存活元胞是 2 个或 3 个时，保持原样
- 当周围存活元胞是 4 个时，该元胞死亡（没饭吃饿死了）
- 当周围存活元胞是 3 个，且当前元胞是死亡状态时，该元胞变成存活（模拟繁殖）

这个游戏按 Wolfram 的分类应该是第四种复杂型。而我们的这个谜题，就是把图输入到这种自动机里，然后跑一下就结束了。

## 怎么解题

模拟它应该是要写个程序的，毕竟手推太麻烦了。Game of Life 的实现大概满世界都是吧。打开 [LeetCode](https://leetcode.com/problems/game-of-life/solution/) 的题解抄下来稍微改改就有了。

```python
import collections

def gameOfLifeInfinite(live):
  ctr = collections.Counter((I, J)
                            for i, j in live
                            for I in range(i-1, i+2)
                            for J in range(j-1, j+2)
                            if I != i or J != j)
  return {ij
    for ij in ctr
    if ctr[ij] == 3 or ctr[ij] == 2 and ij in live}

def gameOfLife(board):
  live = {(i, j) for i, row in enumerate(board) for j, live in enumerate(row) if live}
  live = gameOfLifeInfinite(live)
  for i, row in enumerate(board):
    for j in range(len(row)):
      row[j] = int((i, j) in live)

board = [[0,1,0],[0,0,1],[1,1,1],[0,0,0]]
gameOfLife(board)
print(board)
```

比较恶心的是，我们怎么去把图片转换成矩阵。据说大多数人都是手弄的。我觉得有点过分了，毕竟也是一个 27 x 27 的超大矩阵，手抄一抄得抄 729 个数字。咱这也不是庙会的抄数字赢钱的摊，咱还是找个自动化的方法来处理吧。

我们先裁切一下图片，减少一下代码量，变成：

![裁切后图像](/static/game-of-life-ingress.png)

然后我们写点 OpenCV 的代码来做图像处理和转换，简单来说就是二值化，然后压缩成 27x27（因为像素冗余有够大，直接压大概不会被边框影响到），然后直接除以 255 变成 01 矩阵，跑完打印出来。

```python
import collections
import cv2

def gameOfLifeInfinite(live):
  ctr = collections.Counter((I, J)
                            for i, j in live
                            for I in range(i-1, i+2)
                            for J in range(j-1, j+2)
                            if I != i or J != j)
  return {ij
    for ij in ctr
    if ctr[ij] == 3 or ctr[ij] == 2 and ij in live}

def gameOfLife(board):
  live = {(i, j) for i, row in enumerate(board) for j, live in enumerate(row) if live}
  live = gameOfLifeInfinite(live)
  for i, row in enumerate(board):
    for j in range(len(row)):
      row[j] = int((i, j) in live)

img = cv2.imread('/Users/delton/Desktop/game-of-life-ingress.png')

# Convert RGB to Binary Image
ret, bw_img = cv2.threshold(img, 127, 255, cv2.THRESH_BINARY)

# Convert to 27x27
# 反正像素冗余那么大，直接转换多半不会出问题
resized_image = cv2.resize(bw_img, (27, 27))

# Pick Channel 0 only
board = resized_image[:, :, 0] / 255
gameOfLife(board)
print(board)

# Revert Black and White
cv2.imshow("Result", 1.0 - board)
cv2.waitKey(0)
cv2.destroyAllWindows()
```

结果：

![解码结果](/static/humanity-result.png)

扫码：

![扫码结果](/static/humanity-scan.jpg)

结束。

## 扩展阅读

- [生命游戏：另一种计算机 —— 混乱博物馆](https://youtu.be/GQNREcMVPHY)

- [混沌与分形 —— MommyTalk](https://www.youtube.com/watch?v=2lfVFOXzonY)
