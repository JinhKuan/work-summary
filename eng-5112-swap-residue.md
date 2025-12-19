# Swap Residue Approach: Mathematical Analysis

## Goal

Preserve the elegant invariant formula while forcing swap_out to realize $0 PnL by introducing a `swap_residue` term.

## Current Formulas (from verify-formulas.md)

```
realized_pnl = sold_usd - bought_usd + fees + avg_cost * balance
unrealized_pnl = valuation - avg_cost * balance - fees

Invariant: realized + unrealized = valuation + sold - bought
```

## Proposed Formulas with swap_residue

```
realized_pnl = sold_usd - bought_usd + fees + avg_cost * balance + swap_residue
unrealized_pnl = valuation - avg_cost * balance - fees

Modified Invariant: realized + unrealized = valuation + sold - bought + sum(swap_residue)
```

## Derivation of swap_residue for swap_out

When a swap_out occurs on a position:

- `sold_usd` increases by S (the swap-out value)
- `balance` decreases by T tokens
- `fees` change by F
- `avg_cost` remains constant (no new cost basis)

Natural change in realized PnL:

```
delta_realized = ΔS + ΔF + avg_cost * (-T)
               = S + F - avg_cost * T
```

To force `delta_realized = 0` for swaps:

```
swap_residue = -(S + F - avg_cost * T)
             = avg_cost * T - S - F
             = cost_basis - proceeds - fees
```

**Interpretation**: swap_residue exactly cancels the natural PnL that would occur from the "sale"

## Verification: Modified Invariant Still Holds

```
realized + unrealized
= (sold - bought + fees + avg_cost * balance + swap_residue) + (valuation - avg_cost * balance - fees)
= sold - bought + valuation + swap_residue
✓ This equals: valuation + sold - bought + swap_residue
```

The invariant is preserved! The `swap_residue` term explicitly represents the artificial constraint we're imposing for product reasons.

## Example: Swap A → B

### Token A (swap_out)

- Swapping out $100 worth
- avg_cost = $10/token
- Swapping 10 tokens
- fees = -$2

```
swap_residue_A = (10 * 10) - 100 - (-2)
               = 100 - 100 + 2
               = $2
```

Changes to Token A:

- sold_usd increases by $100
- balance decreases by 10
- fees decrease by $2
- swap_residue = $2

Net effect on realized PnL:

```
delta_realized_A = +100 (sold) + (-2) (fees) + 10*(-10) (balance) + 2 (residue)
                 = 100 - 2 - 100 + 2
                 = 0 ✓
```

### Token B (swap_in)

- Receiving $100 worth (treated as normal buy)
- No swap_residue needed
- bought_usd increases by $100

## Edge Cases

### 1. Swap Chain (A→B→C→D)

Each swap_out generates its own swap_residue that cancels its PnL. The chain doesn't accumulate PnL because each link has delta_realized = 0.

### 2. Swap_out then Spot Sell

1. Swap_out: Creates swap_residue to cancel PnL
2. Later spot sell: Realizes PnL naturally (no swap_residue)

The spot sell realizes PnL based on current avg_cost, which was affected by the swap_out reducing balance.

### 3. Full Position Swap_out

- sold_usd increases by full valuation
- balance goes to 0
- swap_residue = avg_cost \* initial_balance - valuation - fees
- Final realized PnL = 0 (as desired)

## Implementation Benefits

1. **Elegant**: Preserves the invariant structure with explicit accounting for swap constraints
2. **Auditable**: swap_residue explicitly shows deviation from natural accounting
3. **Flexible**: Can query positions with/without swap_residue to see "natural" vs "product" PnL
4. **Debuggable**: sum(swap_residue) across all positions should equal the total PnL "suppressed" by swaps

## Storage

Add to `order_aggregate_results` table:

```sql
swap_residue NUMERIC DEFAULT 0
```

For each swap_out order:

```
swap_residue = avg_cost * tokens_swapped - swap_out_value - fees
```

For all other orders (including swap_in):

```
swap_residue = 0
```

## Formula Update

```typescript
// In position-metrics.ts
function calculateRealizedPnl(position: Position): number {
  const {
    totalSoldUsd,
    totalBoughtUsd,
    totalFees,
    avgCost,
    balance,
    totalSwapResidue, // NEW: sum of swap_residue from order_aggregate_results
  } = position;

  return (
    totalSoldUsd -
    totalBoughtUsd +
    totalFees +
    avgCost * balance +
    totalSwapResidue
  );
}

// Unrealized PnL unchanged
function calculateUnrealizedPnl(position: Position): number {
  const { valuation, avgCost, balance, totalFees } = position;
  return valuation - avgCost * balance - totalFees;
}
```

## Product Requirements Validation

| Aspect                  | Spot Trades                | Swap Trades                         | swap_residue Satisfies?                                            |
| ----------------------- | -------------------------- | ----------------------------------- | ------------------------------------------------------------------ |
| **Bought Amount**       | ✅ Contributes to bought $ | ✅ Contributes to bought $          | ✅ YES - swap_in treated as normal buy                             |
| **Sold Amount**         | ✅ Contributes to sold $   | ✅ Contributes to sold $            | ✅ YES - swap_out increases sold_usd                               |
| **PnL at Transaction**  | Realizes PnL on sell       | No PnL on swap_in/out. Ignore drift | ✅ YES - swap_residue forces delta_realized = 0, cancels any drift |
| **Cost Accounting**     | Use purchase cost basis    | Use purchase cost basis             | ✅ YES - swap_in updates avg_cost, later sales use it              |
| **Inventory Type**      | Standard inventory         | Standard (no virtual)               | ✅ YES - no virtual inventory needed with this approach            |
| **FIFO Priority**       | Standard FIFO              | Standard FIFO                       | ✅ YES - all tokens tracked uniformly                              |
| **Shows in Trades Tab** | ✅ Yes                     | ✅ Yes (closed, $0 PnL)             | ✅ YES - swap_out creates closed position with realized_pnl = 0    |
| **Buy/Sell Counter**    | Counts towards B/S         | Counts towards B/S                  | ✅ YES - swap_in/out count as buy/sell                             |
| **Average Exit**        | Weighs in on avg exit      | Weighs in on avg exit               | ✅ YES - swap_out contributes to sold_usd                          |

### Key Validations

1. **"Ignore drift PnL" for swaps**: The swap_residue cancels the _entire_ natural delta_realized (which includes any drift component), effectively ignoring drift for swaps while regular sells can still use drift.

2. **Closed position with $0 PnL**: When swap_out occurs, the position shows tokens were sold (sold_usd increases), but realized_pnl change = 0 due to swap_residue.

3. **No virtual inventory**: All tokens tracked in single balance. No need for separate swapped inventory or FIFO priority logic.

4. **Average exit pricing**: Since swap_out increases sold_usd, it correctly weighs into average exit calculations.

## Conclusion

The swap_residue approach:

- ✅ Preserves mathematical elegance of invariant
- ✅ Forces swap_out to realize $0 PnL
- ✅ Allows swap_out to contribute to sold_usd (product requirement)
- ✅ Makes the constraint explicit and auditable
- ✅ Requires only one new field in order_aggregate_results
- ✅ **Satisfies ALL product requirements in the table**
- ✅ Eliminates need for virtual inventory system
- ✅ Allows drift PnL to be used for regular sells while ignoring it for swaps

This approach is mathematically sound and maintains the elegance you're looking for!
