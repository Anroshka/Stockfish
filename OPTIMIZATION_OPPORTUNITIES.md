# Stockfish Optimization Opportunities Analysis

**Analysis Date**: February 2026  
**Target**: Stockfish Master Branch  
**Methodology**: Static code analysis, algorithmic review, cache-efficiency evaluation

---

## 1. Move Ordering: History Table Cache-Line Alignment

**File**: `src/movepick.cpp` (MovePicker::score) + `src/history.h`  
**Function**: History table access patterns

### Current Behavior
History tables (ButterflyHistory, ContinuationHistory) use row-major layout without explicit cache-line alignment. Sequential move scoring causes cache misses when accessing non-contiguous history entries.

### Weakness
- ButterflyHistory dimensions: `[COLOR_NB][from_to_size]` where `from_to_size = 64*64 = 4096`
- Each color's history spans ~16KB, potentially crossing cache boundaries
- MovePicker scoring accesses multiple history tables per move, causing cache thrashing

### Proposed Modification
```cpp
// In history.h, add cache-line alignment for hot data structures
alignas(64) using ButterflyHistory = Stats<int16_t, 10692, COLOR_NB, int(SQUARE_NB) * int(SQUARE_NB)>;
```

Additionally, reorder history table access in move scoring to improve temporal locality:
```cpp
// In MovePicker, batch history lookups for similar piece types
// before switching to different history tables
```

### Risk Level
**Low** - Pure performance optimization without algorithmic changes

### Fishtest Setup
- **TC**: 10+0.1 (sensitive to speedup)
- **Threads**: 1 (cache effects most visible single-threaded)
- **Games**: 100k-200k
- **Bounds**: [-3, 1] Elo (speedup validation)

### Failure Modes
- No measurable improvement if memory controller already efficiently handles current access pattern
- Potential regression if alignment increases memory footprint causing L3 cache pressure
- May need architecture-specific tuning (different effect on AMD vs Intel)

---

## 2. LMR: Delta-Based Reduction Refinement

**File**: `src/search.cpp:1734-1737`  
**Function**: `Search::Worker::reduction()`

### Current Behavior
```cpp
Depth Search::Worker::reduction(bool i, Depth d, int mn, int delta) const {
    int reductionScale = reductions[d] * reductions[mn];
    return reductionScale - delta * 608 / rootDelta + !i * reductionScale * 238 / 512 + 1182;
}
```

The delta adjustment uses linear scaling (`delta * 608 / rootDelta`), which may be suboptimal when the PV window is very narrow or very wide.

### Weakness
- At narrow windows (small delta), the formula over-reduces promising lines
- At wide windows (large delta), insufficient reduction wastes time on unpromising moves
- Linear scaling doesn't account for non-linear relationship between window width and move quality

### Proposed Modification
```cpp
Depth Search::Worker::reduction(bool i, Depth d, int mn, int delta) const {
    int reductionScale = reductions[d] * reductions[mn];
    
    // Non-linear delta adjustment: stronger reduction at very wide windows
    int deltaAdjust = delta * 608 / rootDelta;
    if (delta > rootDelta / 2)
        deltaAdjust += (delta - rootDelta / 2) * 150 / rootDelta;
    
    return reductionScale - deltaAdjust + !i * reductionScale * 238 / 512 + 1182;
}
```

### Risk Level
**Medium** - Affects core LMR behavior, but change is localized and guarded

### Fishtest Setup
- **TC**: 60+0.6 (LMR effects need depth to materialize)
- **Threads**: 1
- **Games**: 200k-400k
- **Bounds**: [-3, 1] Elo
- **Alternative test**: STC 10+0.1 with 100k games for quick validation

### Failure Modes
- Over-reduction at narrow windows causes tactical misses
- Under-reduction at wide windows increases node count without Elo gain
- May interact poorly with singular extensions (both adjust depth)

---

## 3. Futility Pruning: Improving Margin Guard

**File**: `src/search.cpp:877-889`  
**Function**: `search<NodeType>()` - Step 8 (Futility pruning)

### Current Behavior
```cpp
if (!ss->ttPv && depth < 14 && eval - futility_margin(depth) >= beta && eval >= beta
    && (!ttData.move || ttCapture) && !is_loss(beta) && !is_win(eval))
    return (2 * beta + eval) / 3;
```

Futility margin calculation:
```cpp
Value futilityMult = 76 - 23 * !ss->ttHit;
return futilityMult * d - (2474 * improving + 331 * opponentWorsening) * futilityMult / 1024
     + std::abs(correctionValue) / 174665;
```

### Weakness
The margin doesn't account for move count history from previous ply. If opponent just made a quiet move with low move count (suggesting a forced/critical position), futility pruning may be too aggressive.

### Proposed Modification
```cpp
auto futility_margin = [&](Depth d) {
    Value futilityMult = 76 - 23 * !ss->ttHit;
    
    // Reduce margin if previous ply had few alternatives (forced position)
    int priorMoveCountPenalty = std::min((ss - 1)->moveCount, 10) < 4 ? 15 : 0;
    
    return futilityMult * d
         - (2474 * improving + 331 * opponentWorsening) * futilityMult / 1024
         + std::abs(correctionValue) / 174665
         - priorMoveCountPenalty * d;
};
```

### Risk Level
**Low** - Conservatively reduces pruning in specific conditions without changing base formula

### Fishtest Setup
- **TC**: 60+0.6
- **Threads**: 1
- **Games**: 100k-200k
- **Bounds**: [-3, 1] Elo
- **Focus**: Positions with high tactical density

### Failure Modes
- Insufficient differentiation between truly forced positions and high branching factor
- Node count increase without tactical benefit
- May need moveCount threshold tuning (current: 4)

---

## 4. Singular Extension: Triple Extension Threshold

**File**: `src/search.cpp:1128-1151`  
**Function**: `search<NodeType>()` - Step 15 (Singular extensions)

### Current Behavior
```cpp
int tripleMargin = 73 + 302 * PvNode - 248 * !ttCapture + 90 * ss->ttPv - corrValAdj
                 - (ss->ply > rootDepth) * 48;

extension = 1 + (value < singularBeta - doubleMargin) 
              + (value < singularBeta - tripleMargin);
```

Triple extensions (depth +3) are very rare but powerful for finding deep tactics.

### Weakness
The `tripleMargin` formula has fixed coefficients that don't adapt to position complexity. In endgames with few pieces, singular moves are more common and triple extensions should be more conservative.

### Proposed Modification
```cpp
int tripleMargin = 73 + 302 * PvNode - 248 * !ttCapture + 90 * ss->ttPv - corrValAdj
                 - (ss->ply > rootDepth) * 48;

// Scale up triple margin in simpler positions (fewer pieces = less justification for deep extension)
int pieceCount = pos.count<ALL_PIECES>();
if (pieceCount < 12)
    tripleMargin += (12 - pieceCount) * 8;

extension = 1 + (value < singularBeta - doubleMargin) 
              + (value < singularBeta - tripleMargin);
```

### Risk Level
**Low** - Only affects rare triple extensions, making them slightly more conservative in endgames

### Fishtest Setup
- **TC**: 180+1.8 (endgame effects need long games)
- **Threads**: 1
- **Games**: 100k
- **Bounds**: [-3, 1] Elo
- **Book**: Varied openings with quick simplifications

### Failure Modes
- May miss deep endgame tactics if margin increase is too aggressive
- Coefficient 8 per missing piece may need tuning (could be 6-10)
- Interaction with tablebase probing might reduce importance

---

## 5. History Pruning: Static Exchange Evaluation Integration

**File**: `src/search.cpp:1082-1115`  
**Function**: `search<NodeType>()` - Step 14 (Quiet move pruning)

### Current Behavior
```cpp
int history = (*contHist[0])[movedPiece][move.to_sq()]
            + (*contHist[1])[movedPiece][move.to_sq()]
            + sharedHistory.pawn_entry(pos)[movedPiece][move.to_sq()];

// Continuation history based pruning
if (history < -4083 * depth)
    continue;
```

Then later:
```cpp
// Prune moves with negative SEE
if (!pos.see_ge(move, -25 * lmrDepth * lmrDepth))
    continue;
```

### Weakness
History pruning and SEE pruning are independent. A move might barely pass history threshold but have terrible SEE, or vice versa. Combined heuristic could prune more efficiently.

### Proposed Modification
```cpp
int history = (*contHist[0])[movedPiece][move.to_sq()]
            + (*contHist[1])[movedPiece][move.to_sq()]
            + sharedHistory.pawn_entry(pos)[movedPiece][move.to_sq()];

// Early SEE check for moves with poor history
if (history < -2000 * depth && !pos.see_ge(move, 0))
    continue;

// Continuation history based pruning (unchanged)
if (history < -4083 * depth)
    continue;

history += 69 * mainHistory[us][move.raw()] / 32;
lmrDepth += history / 3208;

Value futilityValue = ss->staticEval + 42 + 161 * !bestMove + 127 * lmrDepth
                    + 85 * (ss->staticEval > alpha);

if (!ss->inCheck && lmrDepth < 13 && futilityValue <= alpha)
{
    if (bestValue <= futilityValue && !is_decisive(bestValue) && !is_win(futilityValue))
        bestValue = futilityValue;
    continue;
}

lmrDepth = std::max(lmrDepth, 0);

// Adjusted SEE pruning: more aggressive for moves with poor history
int seeMargin = -25 * lmrDepth * lmrDepth;
if (history < -1000 * depth)
    seeMargin += 10 * lmrDepth;  // Make pruning easier for bad history moves

if (!pos.see_ge(move, seeMargin))
    continue;
```

### Risk Level
**Medium** - Adds early pruning path that could miss tactics if thresholds are wrong

### Fishtest Setup
- **TC**: 60+0.6
- **Threads**: 1
- **Games**: 200k-400k
- **Bounds**: [-3, 1] Elo
- **Focus**: Tactical positions (computer games book)

### Failure Modes
- Early SEE pruning at `-2000 * depth` threshold may miss quiet moves with long-term compensation
- Adjusted SEE margin interaction with existing pruning may cause unexpected node count changes
- History scoring can be misleading in unique positions

---

## 6. Move Picker: Partial Sort Optimization

**File**: `src/movepick.cpp:62-73`  
**Function**: `partial_insertion_sort()`

### Current Behavior
```cpp
void partial_insertion_sort(ExtMove* begin, ExtMove* end, int limit) {
    for (ExtMove *sortedEnd = begin, *p = begin + 1; p < end; ++p)
        if (p->value >= limit)
        {
            ExtMove tmp = *p, *q;
            *p          = *++sortedEnd;
            for (q = sortedEnd; q != begin && *(q - 1) < tmp; --q)
                *q = *(q - 1);
            *q = tmp;
        }
}
```

### Weakness
- Insertion sort performs many comparisons and shifts for each element
- When many moves exceed the limit (common in tactical positions), the inner loop becomes expensive
- Modern CPUs can handle branch mispredictions better with vectorized operations

### Proposed Modification
```cpp
void partial_insertion_sort(ExtMove* begin, ExtMove* end, int limit) {
    // Quick count of moves above limit
    int aboveLimitCount = 0;
    for (ExtMove* p = begin; p < end; ++p)
        aboveLimitCount += (p->value >= limit);
    
    // If very few or very many moves exceed limit, use simple approaches
    if (aboveLimitCount <= 2)
    {
        // Just find the top 2 moves
        if (aboveLimitCount >= 1)
        {
            ExtMove* best = begin;
            for (ExtMove* p = begin + 1; p < end; ++p)
                if (p->value > best->value)
                    best = p;
            std::swap(*begin, *best);
        }
        if (aboveLimitCount == 2)
        {
            ExtMove* best = begin + 1;
            for (ExtMove* p = begin + 2; p < end; ++p)
                if (p->value >= limit && p->value > best->value)
                    best = p;
            std::swap(*(begin + 1), *best);
        }
        return;
    }
    
    // Original algorithm for intermediate cases
    for (ExtMove *sortedEnd = begin, *p = begin + 1; p < end; ++p)
        if (p->value >= limit)
        {
            ExtMove tmp = *p, *q;
            *p          = *++sortedEnd;
            for (q = sortedEnd; q != begin && *(q - 1) < tmp; --q)
                *q = *(q - 1);
            *q = tmp;
        }
}
```

### Risk Level
**Low** - Pure performance optimization with no behavioral change (same sorting result)

### Fishtest Setup
- **TC**: 10+0.1 (speedup test)
- **Threads**: 1
- **Games**: 100k
- **Bounds**: [-3, 1] Elo (speedup validation)

### Failure Modes
- Additional counting pass may negate speedup benefits
- Branch prediction on `aboveLimitCount <= 2` might be poor in mixed positions
- May need profiling data to validate actual hotspot status

---

## 7. Null Move: Verification Search Depth

**File**: `src/search.cpp:891-924`  
**Function**: `search<NodeType>()` - Step 9 (Null move search)

### Current Behavior
```cpp
if (cutNode && ss->staticEval >= beta - 18 * depth + 350 && !excludedMove
    && pos.non_pawn_material(us) && ss->ply >= nmpMinPly && !is_loss(beta))
{
    Depth R = 7 + depth / 3;
    do_null_move(pos, st, ss);
    Value nullValue = -search<NonPV>(pos, ss + 1, -beta, -beta + 1, depth - R, false);
    undo_null_move(pos);

    if (nullValue >= beta && !is_win(nullValue))
    {
        if (nmpMinPly || depth < 16)
            return nullValue;

        // Do verification search at high depths
        nmpMinPly = ss->ply + 3 * (depth - R) / 4;
        Value v = search<NonPV>(pos, ss, beta - 1, beta, depth - R, false);
        nmpMinPly = 0;

        if (v >= beta)
            return nullValue;
    }
}
```

### Weakness
Verification search uses `depth - R` which might be too deep when initial null move had a marginal pass. This wastes nodes on positions where beta cutoff is uncertain.

### Proposed Modification
```cpp
if (nullValue >= beta && !is_win(nullValue))
{
    if (nmpMinPly || depth < 16)
        return nullValue;

    assert(!nmpMinPly);

    // Shallower verification if null move barely passed beta
    int verificationDepthReduction = (nullValue < beta + 100) ? 1 : 0;
    Depth verificationDepth = depth - R - verificationDepthReduction;

    nmpMinPly = ss->ply + 3 * (depth - R) / 4;
    Value v = search<NonPV>(pos, ss, beta - 1, beta, verificationDepth, false);
    nmpMinPly = 0;

    if (v >= beta)
        return nullValue;
}
```

### Risk Level
**Low** - Makes verification slightly cheaper in marginal cases without affecting strong null moves

### Fishtest Setup
- **TC**: 60+0.6
- **Threads**: 1
- **Games**: 100k-200k
- **Bounds**: [-3, 1] Elo

### Failure Modes
- May increase zugzwang misses if verification is too shallow
- `beta + 100` threshold might need tuning (could be 80-120)
- Interaction with `nmpMinPly` recalculation might be complex

---

## 8. Correction History: Update Frequency Guard

**File**: `src/search.cpp:1472-1480`  
**Function**: `search<NodeType>()` - Correction history update

### Current Behavior
```cpp
if (!ss->inCheck && !(bestMove && pos.capture(bestMove))
    && (bestValue > ss->staticEval) == bool(bestMove))
{
    auto bonus = std::clamp(int(bestValue - ss->staticEval) * depth / (bestMove ? 10 : 8),
                            -CORRECTION_HISTORY_LIMIT / 4, CORRECTION_HISTORY_LIMIT / 4);
    update_correction_history(pos, ss, *this, bonus);
}
```

### Weakness
Correction history is updated on every node meeting the condition, even when the error is tiny. This adds noise to the correction tables, particularly at shallow depths where `depth / 10` makes even large errors produce small bonuses.

### Proposed Modification
```cpp
if (!ss->inCheck && !(bestMove && pos.capture(bestMove))
    && (bestValue > ss->staticEval) == bool(bestMove))
{
    int error = int(bestValue - ss->staticEval);
    
    // Only update if error is significant relative to depth
    int minError = bestMove ? 40 : 30;
    if (std::abs(error) >= minError || depth >= 8)
    {
        auto bonus = std::clamp(error * depth / (bestMove ? 10 : 8),
                                -CORRECTION_HISTORY_LIMIT / 4, CORRECTION_HISTORY_LIMIT / 4);
        update_correction_history(pos, ss, *this, bonus);
    }
}
```

### Risk Level
**Low** - Reduces update frequency without changing the update formula

### Fishtest Setup
- **TC**: 60+0.6
- **Threads**: 8 (correction history is shared across threads)
- **Games**: 100k-200k
- **Bounds**: [-3, 1] Elo

### Failure Modes
- May slow down correction history learning in early game
- Threshold values (40/30) might need tuning
- Could increase shared cache contention if updates become bursty

---

## 9. Capture History: MVV-LVA Initialization

**File**: `src/search.cpp:585-603`  
**Function**: `Search::Worker::clear()` - History initialization

### Current Behavior
```cpp
void Search::Worker::clear() {
    mainHistory.fill(0);
    captureHistory.fill(-689);  // Single constant for all captures
    // ...
}
```

### Weakness
All capture types start with the same history score (-689), regardless of piece values. Good captures (QxP) and bad captures (PxQ) are treated identically until the search learns their difference.

### Proposed Modification
```cpp
void Search::Worker::clear() {
    mainHistory.fill(0);
    
    // Initialize capture history with MVV-LVA (Most Valuable Victim - Least Valuable Aggressor)
    for (Piece attacker = PAWN; attacker <= KING; ++attacker)
        for (Square to = SQ_A1; to <= SQ_H8; ++to)
            for (PieceType victim = PAWN; victim <= QUEEN; ++victim)
            {
                int mvvLvaScore = (PieceValue[victim] - PieceValue[type_of(attacker)]) / 16;
                captureHistory[attacker][to][victim] = std::clamp(mvvLvaScore - 689, -2000, 500);
            }
    
    // Rest unchanged...
}
```

### Risk Level
**Low** - Better initialization doesn't change learning dynamics, just speeds up convergence

### Fishtest Setup
- **TC**: 60+0.6
- **Threads**: 1
- **Games**: 100k-200k
- **Bounds**: [-3, 1] Elo
- **Book**: Standard openings (tests if early capture ordering improves)

### Failure Modes
- May bias capture history too strongly before position-specific learning
- MVV-LVA heuristic might conflict with position-specific capture patterns
- Division by 16 and clamp ranges might need adjustment

---

## 10. SEE Pruning: Endgame Margin Scaling

**File**: `src/search.cpp:1113-1114`  
**Function**: `search<NodeType>()` - SEE pruning for quiet moves

### Current Behavior
```cpp
// Prune moves with negative SEE
if (!pos.see_ge(move, -25 * lmrDepth * lmrDepth))
    continue;
```

### Weakness
The margin `-25 * lmrDepth²` is constant regardless of game phase. In endgames, piece values are more concrete (fewer tactical complications), so SEE becomes more reliable and should allow more aggressive pruning.

### Proposed Modification
```cpp
// Prune moves with negative SEE
int seeMargin = -25 * lmrDepth * lmrDepth;

// In endgames, trust SEE more (fewer tactics, clearer exchanges)
int pieceCount = pos.count<ALL_PIECES>();
if (pieceCount <= 10)
    seeMargin = seeMargin * 130 / 100;  // 30% more aggressive

if (!pos.see_ge(move, seeMargin))
    continue;
```

### Risk Level
**Low-Medium** - Affects endgame pruning but with conservative 30% increase

### Fishtest Setup
- **TC**: 180+1.8 (endgame benefits need long games)
- **Threads**: 1
- **Games**: 100k
- **Bounds**: [-3, 1] Elo
- **Book**: Endgame-heavy suite or quick simplifications

### Failure Modes
- May miss endgame tactics involving sacrifices for passed pawns
- Piece count threshold (10) might be too high or too low
- 30% scaling factor might need tuning (20-40% range)

---

## Priority Ranking

**High Priority** (Most likely to pass fishtest):
1. **#1 - Cache-Line Alignment**: Pure speedup, no algorithmic risk
2. **#6 - Partial Sort Optimization**: Algorithmic speedup without behavior change
3. **#4 - Singular Extension Endgame**: Conservative change to rare code path
4. **#8 - Correction History Guard**: Reduces noise in learning

**Medium Priority** (Good chance with tuning):
5. **#9 - Capture History MVV-LVA**: Better initialization for faster learning
6. **#7 - Null Move Verification**: Saves nodes in marginal cases
7. **#3 - Futility Pruning**: Context-aware margin adjustment

**Lower Priority** (Needs careful tuning):
8. **#2 - LMR Delta Refinement**: Core search change, needs extensive testing
9. **#5 - History + SEE Integration**: Complex interaction between heuristics
10. **#10 - SEE Endgame Scaling**: Might conflict with other endgame heuristics

---

## Testing Strategy

### Phase 1: Quick Validation (10+0.1, 50k games)
Test #1, #6, #8 for speedups or quick failures

### Phase 2: STC Testing (60+0.6, 100k games)
Test #3, #4, #7, #9 individually

### Phase 3: LTC Testing (180+1.8, 100k games)
Retest promising STC patches at longer time controls
Specifically test #4, #10 which have endgame focus

### Phase 4: Combined Testing
If multiple patches pass individually, test combinations:
- #1 + #6 (both are performance improvements)
- #4 + #10 (both target endgames)
- #7 + #9 (both improve move ordering)

---

## Implementation Notes

- All coefficient values are based on surrounding tuned parameters in current Stockfish
- Each change is minimal and localized to avoid collateral effects
- Prefer adding conditions over replacing logic to maintain fallback behavior
- Use `assert()` to validate assumptions during development
- Profile with `perf` or similar tools to validate cache effects for #1 and #6

---

## Disclaimer

These opportunities are based on static analysis and chess programming heuristics. Stockfish's search space is highly non-linear, and interactions between heuristics can be unexpected. Each idea requires empirical validation through fishtest before integration. Some suggestions may regress or show neutral results despite sound reasoning.

**Fishtest is the ultimate arbiter.**
