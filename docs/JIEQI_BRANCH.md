# 揭棋分支说明 / Jieqi Branch Documentation

本文档说明揭棋（Jieqi/Banqi）分支是如何基于原始 Pikafish 分支创建的。

This document explains how the Jieqi (揭棋/Banqi) branch was created based on the original Pikafish branch.

## 概述 / Overview

揭棋是中国象棋的一种变体，其中棋子最初是背面朝上放置的（暗棋），只有在移动时才会翻开。Jieqi 分支通过在原始 Pikafish 引擎之上添加特定修改来实现这种变体。

Jieqi is a variant of Chinese chess (Xiangqi) where pieces start face-down (dark) and are revealed when moved. The jieqi branch implements this variant by adding specific modifications on top of the original Pikafish engine.

## 分支历史 / Branch History

Jieqi 分支从 Pikafish master 分支的提交 `1f451f43`（"Fix comment typos"）创建而来。在该基础上添加了以下揭棋相关的提交：

The jieqi branch was created from the Pikafish master branch at commit `1f451f43` ("Fix comment typos"). The following jieqi-specific commits were added on top:

1. **c8e8b49** - `Basic changes required for jieqi`（揭棋所需的基本修改）
2. **10cc3d9** - `Search and eval tune (40k iter)`（搜索和评估调优）
3. **e344482** - `Update benchmark positions for jieqi`（更新揭棋基准测试局面）
4. **1db02ba** - `Truncate the PV after the dark move`（在暗棋移动后截断主要变例）
5. **10334fe** - `Fix mate in flip_search`（修复翻棋搜索中的将杀）
6. **e27d5de** - `Search and eval tune (~8k iter)`（搜索和评估调优）
7. **9b963f7** - `Fix score display`（修复分数显示）

## 主要代码修改 / Key Code Changes

### 1. 类型系统修改 / Type System Changes (`src/types.h`)

添加了 `DARK` 棋子类型和 `DARK_PIECE` 枚举值：

Added `DARK` piece type and `DARK_PIECE` enum value:

```cpp
enum PieceType : std::int8_t {
    NO_PIECE_TYPE, ROOK, ADVISOR, CANNON, PAWN, KNIGHT, BISHOP, KING, DARK, KNIGHT_TO = 8, PAWN_TO,
    // ...
};

enum Piece : std::int8_t {
    // ...
    DARK_PIECE, PIECE_NB = DARK_PIECE
};
```

扩展了 `DirtyPiece` 结构以跟踪翻棋操作：

Extended `DirtyPiece` structure to track flip operations:

```cpp
struct DirtyPiece {
    Piece  pc;
    Square from, to;
    Square remove_sq, add_sq;
    Piece  remove_pc, add_pc;
    bool requires_refresh[2];
};
```

### 2. 棋盘表示 / Position Representation (`src/position.cpp`, `src/position.h`)

添加了暗棋的初始布局和相关数据结构：

Added dark piece initial layout and related data structures:

- `DarkPieces` - 暗棋初始布局 / Initial dark piece layout
- `byTypeBB[DARK]` - 暗棋位棋盘 / Dark pieces bitboard
- `restPieces` - 剩余未翻开棋子计数 / Remaining unrevealed pieces count

新增核心函数：

Added core functions:

- `is_dark(Square s)` - 检查指定格是否为暗棋 / Check if square has dark piece
- `move_dark(Move m)` - 检查移动是否涉及暗棋 / Check if move involves dark piece
- `do_flip(Square, Piece, ...)` - 执行翻棋操作 / Execute flip operation
- `undo_flip(Square, Piece)` - 撤销翻棋操作 / Undo flip operation
- `rest_pieces(Color)` - 获取剩余暗棋 / Get remaining dark pieces

### 3. 搜索算法 / Search Algorithm (`src/search.cpp`, `src/search.h`)

添加了 `flip_search` 函数来处理翻棋时的概率搜索：

Added `flip_search` function to handle probabilistic search when flipping:

```cpp
template<NodeType nodeType>
Value Search::Worker::flip_search(
    Position& pos, Stack* ss, Value alpha, Value beta, 
    bool isQsearch, Depth depth, bool cutNode);
```

该函数的核心逻辑：

The core logic of this function:

1. 获取当前颜色剩余的所有暗棋类型 / Get all remaining dark piece types for current color
2. 对每种可能的翻棋结果进行搜索 / Search for each possible flip result
3. 根据每种棋子的剩余数量计算加权胜率 / Calculate weighted win rate based on remaining count of each piece
4. 使用 sigmoid 函数进行分数转换 / Use sigmoid function for score conversion

在 `search()` 和 `qsearch()` 中添加了翻棋检测：

Added flip detection in `search()` and `qsearch()`:

```cpp
// Dive into flip search when the last move is moving a dark pieces
if ((ss - 1)->currentMove.is_ok() && pos.is_dark((ss - 1)->currentMove.to_sq()))
{
    constexpr auto nt = PvNode ? PV : NonPV;
    return flip_search<nt>(pos, ss, alpha, beta, false, depth, cutNode);
}
```

### 4. 规则修改 / Rule Changes

将60步和棋规则改为40步：

Changed 60-move rule to 40-move rule:

- `rule60` → `rule40`
- `rule60_count()` → `rule40_count()`
- 相关阈值从120调整为80 / Related thresholds adjusted from 120 to 80

### 5. 走法生成 / Move Generation (`src/movegen.cpp`)

修改了走法生成以考虑暗棋的特殊移动规则：

Modified move generation to consider special movement rules for dark pieces:

- 暗棋只能移动到相邻格子 / Dark pieces can only move to adjacent squares
- 仕（士）作为暗棋时可以移出九宫 / Advisor as dark piece can move outside palace

### 6. NNUE 评估 / NNUE Evaluation

更新了 NNUE 累加器以处理翻棋操作：

Updated NNUE accumulator to handle flip operations:

- `src/nnue/features/half_ka_v2_hm.cpp`
- `src/nnue/features/half_ka_v2_hm.h`
- `src/nnue/nnue_accumulator.cpp`

### 7. FEN 格式扩展 / FEN Format Extension

FEN 字符串格式扩展以包含剩余暗棋信息：

Extended FEN string format to include remaining dark pieces:

```
局面 颜色 剩余暗棋 步数规则计数 回合数
position color remaining_pieces rule_count move_number
```

示例 / Example:
```
rnbakabnr/9/1c5c1/p1p1p1p1p/9/9/P1P1P1P1P/1C5C1/9/RNBAKABNR w R2A2C2P5N2B2K1r2a2c2p5n2b2k1 0 1
│                                                              │ │                                │ │
│                                                              │ │                                │ └─ 回合数/move number
│                                                              │ │                                └─── 步数规则计数/rule count
│                                                              │ └─────────────────────────────────── 剩余暗棋(棋子+数量)/remaining pieces(piece+count)
│                                                              └──────────────────────────────────── 走棋方/side to move
└──────────────────────────────────────────────────────────────────────────────────────────────────── 棋盘布局/board layout
```

其中剩余暗棋部分格式为：`棋子类型+数量`，例如 `R2` 表示红车剩余2个。

The remaining pieces section format is: `piece_type+count`, e.g., `R2` means 2 red rooks remaining.

## 如何创建类似分支 / How to Create a Similar Branch

1. **从 master 分支创建新分支 / Create new branch from master**
   ```bash
   git checkout master
   git checkout -b your-variant-branch
   ```

2. **修改类型系统 / Modify type system** (`src/types.h`)
   - 添加新的棋子类型 / Add new piece types
   - 修改相关数据结构 / Modify related data structures

3. **修改棋盘表示 / Modify position representation** (`src/position.cpp`, `src/position.h`)
   - 实现变体特有的规则 / Implement variant-specific rules
   - 添加必要的辅助函数 / Add necessary helper functions

4. **修改搜索算法 / Modify search algorithm** (`src/search.cpp`, `src/search.h`)
   - 实现变体特有的搜索逻辑 / Implement variant-specific search logic
   - 调整评估参数 / Adjust evaluation parameters

5. **更新走法生成 / Update move generation** (`src/movegen.cpp`)
   - 适应新的移动规则 / Adapt to new movement rules

6. **训练新的 NNUE 网络 / Train new NNUE network**
   - 生成变体特有的训练数据 / Generate variant-specific training data
   - 训练新的神经网络权重 / Train new neural network weights

7. **更新基准测试 / Update benchmarks** (`src/benchmark.cpp`)
   - 添加变体特有的测试局面 / Add variant-specific test positions

## 相关文件列表 / Related Files

主要修改的文件 / Main modified files:

- `src/types.h` - 类型定义 / Type definitions
- `src/position.cpp` / `src/position.h` - 棋盘表示 / Position representation
- `src/search.cpp` / `src/search.h` - 搜索算法 / Search algorithm
- `src/movegen.cpp` - 走法生成 / Move generation
- `src/benchmark.cpp` - 基准测试 / Benchmarks
- `src/nnue/features/half_ka_v2_hm.cpp` / `src/nnue/features/half_ka_v2_hm.h` - NNUE 特征
- `src/nnue/nnue_accumulator.cpp` / `src/nnue/nnue_accumulator.h` - NNUE 累加器

## 参考资料 / References

- [Pikafish 官方仓库 / Official Repository](https://github.com/official-pikafish/Pikafish)
- [揭棋规则 / Jieqi Rules](https://zh.wikipedia.org/wiki/%E6%8F%AD%E6%A3%8B)
- [Stockfish 官方仓库 / Stockfish Repository](https://github.com/official-stockfish/Stockfish)
