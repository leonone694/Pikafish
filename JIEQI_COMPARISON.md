# Comparison between `jieqi` and `jieqi_old` Branches

This document explains the differences between the `jieqi` branch (current) and the `jieqi_old` branch, and addresses the question of whether the compiled engine from the current branch is a Jieqi (揭棋/暗棋) engine.

## Summary

**Good news: The current `jieqi` branch IS a Jieqi engine!** It has been updated to a newer Pikafish/Stockfish codebase (2024) while maintaining Jieqi-specific features. The implementation approach differs from `jieqi_old`, but both branches support Jieqi gameplay.

## What is Jieqi (揭棋)?

Jieqi (揭棋), also known as Banqi (暗棋) or Dark Chess, is a variant of Xiangqi (Chinese Chess) where:
- Pieces start face-down (dark/unknown)
- Players flip pieces to reveal them during the game
- The game involves probabilistic reasoning about unknown pieces

## Jieqi Features in Current Branch

The current branch has the following Jieqi-specific implementations:

### Dark Pieces Support
```cpp
// In types.h - DARK piece type
NO_PIECE_TYPE, ROOK, ADVISOR, CANNON, PAWN, KNIGHT, BISHOP, KING, DARK, ...
DARK_PIECE, PIECE_NB = DARK_PIECE

// In position.h/cpp
bool is_dark(Square s) const;  // Check if a square has a dark piece
inline bool Position::is_dark(Square s) const { return pieces(DARK) & s; }
```

### Rest Pieces Tracking
```cpp
// In position.h
int restPieces[PIECE_NB];  // Track count of each piece type not yet revealed

// Access methods
const int& rest_piece(Piece pc) const;
int& rest_piece(Piece pc);
std::vector<std::pair<Piece, int>> rest_pieces(Color c) const;
```

### Piece Flipping/Revealing
```cpp
// In position.cpp - Full flip implementation
Piece do_flip(Square s, Piece pc, DirtyPiece* dp, const TranspositionTable* tt);
void undo_flip(Square s, Piece fromPc);
```

### Probabilistic Search for Dark Pieces
```cpp
// In search.cpp - Handles flip scenarios during search
auto restPieces = pos.rest_pieces(~pos.side_to_move());
for (auto& [piece, num] : restPieces) {
    Piece flipped_piece = pos.do_flip((ss - 1)->currentMove.to_sq(), piece, &dp, &tt);
    Value value = search<nodeType>(pos, ss, alpha, beta, depth, cutNode);
    pos.undo_flip((ss - 1)->currentMove.to_sq(), flipped_piece);
    results.push_back({value, num});
}
```

### FEN Format Support
```cpp
// In position.cpp - Parsing rest pieces from FEN
// Format includes rest piece counts like "R2A2C2P5..." at the end
restPieces[pc] = token - '0';
```

## Key Differences Between Branches

| Aspect | `jieqi_old` | `jieqi` (Current) |
|--------|-------------|-------------------|
| Stockfish Base | ~2022 version | 2024 version |
| NNUE Support | Conditional (`USE_NNUEEVAL`) | Integrated with NNUE |
| Dark Piece Enum | `enum Dark { KNOWN, UNKNOWN }` | Using `DARK` PieceType |
| Rest Pieces | `RestList` class | `int restPieces[PIECE_NB]` array |
| Search Integration | Custom dark depth tracking | Integrated in main search |
| Flipping | `getDark()`/`setDark()` | `do_flip()`/`undo_flip()` |

### Implementation Approach Differences

**`jieqi_old` approach:**
- Uses separate `Dark` enum to track known/unknown status
- Has dedicated `RestList` class with more features (e.g., `evgValue()` for expected value)
- Explicit dark depth limits (`MAXDARKDEPTH`, `QDARKDEPTH`)
- Tuning parameters (`DARKVALRATE`, `DARKMAXDIFF`)

**Current `jieqi` approach:**
- Uses `DARK` as a piece type directly
- Simple array for rest piece counting
- Probabilistic search integrated into main search with expected value calculations
- Cleaner integration with newer NNUE architecture

## Troubleshooting: Engine Not Behaving as Jieqi Engine

If the compiled engine doesn't seem to work as a Jieqi engine, check:

1. **FEN String Format**: Ensure you're using the correct Jieqi FEN format that includes rest piece information at the end

2. **UCI Interface**: Make sure your GUI supports Jieqi-specific FEN and moves

3. **Initial Position**: Verify the starting position has dark pieces and rest piece counts set correctly

4. **Move Format**: Jieqi moves may include flip information (e.g., a character indicating the revealed piece)

## Example Jieqi FEN

A Jieqi position FEN includes rest piece counts after the standard position:
```
rnbakabnr/9/1c5c1/p1p1p1p1p/9/9/P1P1P1P1P/1C5C1/9/RNBAKABNR w - - 0 1 R2A2C2P5K1N2B2r2a2c2p5k1n2b2
```
The trailing part `R2A2C2P5K1N2B2r2a2c2p5k1n2b2` indicates the count of each unrevealed piece type.

## References

- `jieqi_old` branch: Original Jieqi implementation (commit `23b9466c`)
- `jieqi` branch: Updated codebase with Jieqi features (commit `9b963f72`)
- Key files: `src/position.cpp`, `src/search.cpp`, `src/types.h`
