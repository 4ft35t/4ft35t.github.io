---
title: "按人的思维方式解开数独"
description: 
date: 2024-08-07T21:24:32+08:00
image: https://upload.wikimedia.org/wikipedia/commons/thumb/e/e0/Sudoku_Puzzle_by_L2G-20050714_standardized_layout.svg/722px-Sudoku_Puzzle_by_L2G-20050714_standardized_layout.svg.png
math: 
hidden: false
comments: true
tags: ['soduku', 'backtracking', 'human', 'algorithm', 'python']
categories: ['Algorithm']
draft: false
---

数独是一种逻辑游戏，目标是填充一个 9x9 的网格，使得每一行、每一列和每一个 3x3 的子网格中的数字都是 1 到 9。

数独的规则非常简单，但是解决数独问题却是一个复杂的任务。在这篇文章中，我们将介绍一种按照人的思维方式解开数独的方法，而不是使用传统的回溯算法。

## 人是如何解开数独的
我自己解数独的时候，通常会按照以下步骤进行：
- 先找已有 5 个数字的行、列、块，看看是否有数字只能填在某个位置
- 已出现次数较多的某个特定数字，利用行和列的关系，综合确定其在剩余块中的位置

以下图为例，![](https://upload.wikimedia.org/wikipedia/commons/thumb/e/e0/Sudoku_Puzzle_by_L2G-20050714_standardized_layout.svg/722px-Sudoku_Puzzle_by_L2G-20050714_standardized_layout.svg.png)

一眼看过去，`5` 比较多
1. 上边的三个块，左边块的 5 在第一行，中间块的 5 在第二行，所以右边块的 5 只能在第三行
2. 右边第三行中间已有数字，只有左右可以填
3. 再看右边的三个块，有没有 5 出现，右下角的 5 在最右侧，所以右上角的 5 只能在最左侧

人解题的过程，实际是用排除法，逐步缩小数字的可能位置，直到确定唯一位置。

## 用 Python 实现
### 设计思路
1. 初始化时，每个格子的可能值都是 1-9
2. 当一个格子确定了数字 n，那么这个格子所在的行、列、块中的其他格子, 都不能填 n, 从它们的可能值中删除 n
3. 当一个格子的可能值只有一个时，填入这个数字
4. 重复 2 和 3，直到所有格子都填满

### 伪代码
```python
class Cell:
    def __init__(self, x, y, n=0):
        self.x = x
        self.y = y
        self.value = None
        self.values = list(range(1, 10))

    def set_value(self, n):
        self.value = n
        self.values = [n]

        self.del_near_value(n)

    def del_near_value(self, n):
        for cell in near_cells:
            cell.values.remove(n)

class Sudoku:
    def __init__(self, matrix):
        self.sells = []
        for y, row in enumerate(matrix):
            for x, n in enumerate(row):
                cell = Cell(x, y, n)
                self.sells.append(cell)

    def solve(self):
        while True:
            for cell in cell_with_one_value:
                cell.set_value(cell.values[0])

            if all(cell.value for cell in self.sells):
                break

        return self.cells
```

### 存在的问题
上面的方法在处理初始数字比较多的数独时，效果很好。但是对于初始数字比较少的数独，可能会陷入死循环。因为这种方法只能处理确定的数字，而不能处理可能的数字。

### 解决方案
在每次的 while 循环中，增加一个判断，如果没有任何一个格子的可能值只有一个，那么就选择可能值最少的一个格子，填入可能值的第一个数字。

这么做的原因是，这种初始数字少的数独，可能会有多个解，我们只需要找到一个解即可。

### 完整代码
https://gist.github.com/4ft35t/d284860ff96ddc4d7aae5a4262af907a
