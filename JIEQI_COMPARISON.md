# Comparison between `jieqi` and `jieqi_old` Branches

This document explains the differences between the `jieqi` branch (current) and the `jieqi_old` branch, and answers the question of why the compiled engine from the current branch is not a Jieqi (揭棋/暗棋) engine.

## Summary

The current `jieqi` branch is based on a newer Pikafish/Stockfish codebase that has been updated significantly but **does not contain the Jieqi-specific features**. The `jieqi_old` branch contains the original Jieqi implementation with support for dark/unknown pieces.

## What is Jieqi (揭棋)?

Jieqi (揭棋), also known as Banqi (暗棋) or Dark Chess, is a variant of Xiangqi (Chinese Chess) where:
- Pieces start face-down (dark/unknown)
- Players flip pieces to reveal them during the game
- The game involves probabilistic reasoning about unknown pieces

## Key Differences

### 1. Jieqi Features in `jieqi_old` (Missing in Current Branch)

The `jieqi_old` branch contains these Jieqi-specific features that are **not present** in the current branch:

#### Dark Pieces Support
```cpp
// In types.h (jieqi_old)
enum Dark { KNOWN, UNKNOWN, DARK_NB = 2 };

// Dark piece handling
#define USE_NNUEEVAL 0
constexpr int DARKVALRATE = 2862;  // 5000-10000
constexpr int DARKMAXDIFF = 4812;  // 500-5000
```

#### Rest Pieces (Unplaced Pieces)
```cpp
// In position.h/cpp (jieqi_old)
RestList restPieces[COLOR_NB];  // Pieces not yet placed on the board

// Methods for handling dark pieces
bool isDark(Square s) const;
Dark Darkof(Square s) const;
Dark Darkof(Piece p) const;
```

#### Dark Search Depth
```cpp
// In position.h (jieqi_old)
#define MAXDARKDEPTH    4
#define QDARKDEPTH      1
#define MAXDARKTYPES    43

// State tracking for dark pieces
int darkDepth;
int darkTypes;
int darkTypeIndex;
```

#### Piece Flipping/Revealing
```cpp
// In position.cpp (jieqi_old)
bool getDark(StateInfo& newSt, int& typecount, bool& isDarkDepth);
void setDark();
bool gives_check(Move m, PieceType flipped);  // Support for flipped pieces
```

### 2. Codebase Differences

| Aspect | `jieqi_old` | `jieqi` (Current) |
|--------|-------------|-------------------|
| Stockfish Base | 2022 version | 2025 version |
| NNUE Support | Conditional (`USE_NNUEEVAL`) | Always enabled |
| Dark Pieces | Full implementation | Partial (type defined but not used) |
| Rest Pieces | Full tracking | Not implemented |
| Search | Modified for dark pieces | Standard Xiangqi search |
| Evaluation | Handles unknown pieces | Standard evaluation |

### 3. Why the Compiled Engine is Not a Jieqi Engine

When you compile the current branch, you get a standard Xiangqi engine because:

1. **No Dark Piece Logic**: While `DARK` is defined as a piece type in `types.h`, the logic for handling unknown/dark pieces is not implemented.

2. **No Rest Pieces Tracking**: The `RestList` and related functionality for tracking pieces that haven't been placed/revealed is missing.

3. **No Probabilistic Search**: The search algorithm doesn't account for the probabilistic nature of dark pieces.

4. **Standard Evaluation**: The evaluation function doesn't consider unknown pieces or calculate expected values for dark positions.

## What Needs to Be Done

To convert the current branch into a Jieqi engine, you would need to:

1. **Port the Dark Piece Implementation** from `jieqi_old`:
   - Add `Dark` enum with `KNOWN`/`UNKNOWN` states
   - Implement `isDark()`, `Darkof()` methods in Position class
   - Add dark piece handling in move generation

2. **Port the Rest Pieces System**:
   - Add `RestList` class for tracking unplaced pieces
   - Implement piece flipping/revealing logic
   - Add expected value calculations for unknown pieces

3. **Modify the Search Algorithm**:
   - Add dark depth tracking
   - Implement probabilistic search for unknown pieces
   - Handle piece reveal scenarios

4. **Adapt the Evaluation**:
   - Handle evaluation of positions with unknown pieces
   - Calculate expected values based on remaining piece probabilities

## Recommendation

If you need a working Jieqi engine, use the `jieqi_old` branch. The current `jieqi` branch appears to be an attempt to update the codebase to newer Pikafish/Stockfish but the Jieqi features have not been fully ported yet.

## References

- `jieqi_old` branch: Contains full Jieqi implementation (commit `23b9466c`)
- `jieqi` branch: Updated codebase without Jieqi features (commit `9b963f72`)
- Original Jieqi commits: "增加暗棋，随机棋子" (add dark chess, random pieces)
