# Cardinality Waterfall — Iterative Conversion Engine

## Technical Documentation for Investment Team

**Tab:** Iterative Conversion  
**Purpose:** Determine how each preferred tranche (A-1 and B-series) is treated at exit — liquidation preference or conversion to common — across 36 enterprise value scenarios ($50M–$400M), and calculate BHC's MOIC warrant entitlement.

---

## 1. Overview: Why "Iterative"?

Each preferred tranche faces a binary choice at exit: **take its liquidation preference** (guaranteed $ amount) or **convert to common shares** (participate in residual equity). The conversion decision for one tranche affects all others because:

- A tranche that takes its liq pref **reduces the equity pool** available to remaining holders
- A tranche that converts **increases the share count**, diluting per-share value for everyone

This creates interdependency. The engine resolves it by testing tranches **sequentially** — each tranche's decision is made in the context of all prior decisions — producing a stable equilibrium for each EV scenario.

---

## 2. Static Inputs (Rows 4–25)

All inputs are linked to source tabs (no hardcoded values):

| Input | Value | Source |
|---|---|---|
| Common + Options + Pre-MOIC Warrants ("Base FD") | 832,222 shares | Post-Exchange Cap Structure |
| Transaction Fee Rate | 3.0% of EV | Inputs!C9 |
| BHC Debt Principal | $10,000,000 | BHC Senior Debt |
| BHC Debt Interest (14% simple) | $6,650,000 | BHC Senior Debt |
| BHC Basis (for MOIC calc) | $10,000,000 | Inputs!C44 |
| PIK Factor | 1.1515x | Inputs!C58 |
| Liquidation Preference Multiple | 2.0x | Inputs!C20 |
| Target MOIC | 3.0x | Inputs!C43 |
| BHC Ownership Cap | 12.5% of total FD | MOIC Ratchet!C50 |

**Tranche Data (Rows 18–25):**

Each preferred tranche has: share count, Original Issue Price (OIP), and a calculated Liquidation Preference per share = OIP × 2.0x (liq pref multiple) × 1.1515 (PIK factor).

| Tranche | Shares | OIP | Liq Pref/Share | Total Liq Pref |
|---|---|---|---|---|
| A-1 (BHC) | 130,984 | $26.68 | $61.45 | $8,048,426 |
| B-1 | 200,000 | $15.00 | $34.55 | $6,909,206 |
| B-2 | 50,000 | $20.00 | $46.06 | $2,303,069 |
| B-6 | 69,262 | $37.53 | $86.43 | $5,986,408 |
| B-3 | 26,250 | $40.00 | $92.12 | $2,418,222 |
| B-4 | 74,730 | $40.79 | $93.94 | $7,020,298 |
| B-5 | 18,125 | $68.21 | $157.09 | $2,847,298 |

---

## 3. Scenario Setup (Rows 28–32)

For each EV scenario, compute:

```
Equity Available = EV − Transaction Fees (3%) − BHC Debt Principal ($10M) − CEO Phantom Equity
```

The CEO Phantom Equity is a tiered payout looked up from the CEO SVPT Incentive tab based on EV.

**Example at $90M EV:**
- Fees: $2,700,000
- CEO Phantom: $1,350,000
- Equity Available: $90M − $2.7M − $10M − $1.35M = **$75,950,000**

Total fully-diluted shares pre-warrant = Base FD (832,222) + All Preferred (569,351) = **1,401,573**

---

## 4. Phase 1 — A-1 Pre-Warrant Conversion Test (Rows 34–38)

**Purpose:** Preliminary test of whether BHC's A-1 preferred would convert *before* considering the MOIC warrant. This estimate feeds into the MOIC warrant calculation (Phase 2).

**Logic:**
1. Calculate hypothetical per-share price if A-1 converts: `Equity Available / (Base FD + A-1 Shares)`
2. A-1 As-Converted Value = A-1 Shares × Per-Share Price
3. **Converts if:** As-Converted Value > A-1 Total Liq Pref ($8,048,426)

**Important:** This is a *preliminary* test only. The **final** A-1 conversion decision happens in Phase 2b after the warrant is determined.

| EV | Per-Share | As-Conv Value | Liq Pref | Phase 1 Result |
|---|---|---|---|---|
| $70M | $59.10 | $7,741,770 | $8,048,426 | ❌ Take liq pref |
| $80M | $69.01 | $9,039,091 | $8,048,426 | ✅ Convert |
| $90M | $78.85 | $10,328,253 | $8,048,426 | ✅ Convert |

---

## 5. Phase 2 — MOIC Warrant Calculation (Rows 40–49)

**Purpose:** Determine how many warrant shares BHC receives to help achieve a 3.0x MOIC on its $10M debt investment.

### Step 1: Estimate Pre-Warrant MOIC

BHC's pre-warrant proceeds are calculated **conservatively** using `MIN(Phase 1 payout, A-1 Liq Pref)` — this ensures the warrant calculation doesn't overestimate BHC's position:

```
BHC Pre-Warrant Proceeds = Debt Principal ($10M) + Debt Interest ($6.65M) + MIN(A-1 Pre-Warrant Payout, A-1 Liq Pref)
```

At all EVs: MIN caps the A-1 component at $8,048,426 (the liq pref), so pre-warrant proceeds = $24,698,426 and **Pre-Warrant MOIC = 2.47x** (below 3.0x target).

### Step 2: Calculate Warrant Shares

Two constraints determine warrant shares:

1. **12.5% Ownership Cap:** Maximum shares such that BHC's total FD ownership (existing preferred + warrant) ≤ 12.5%
   - Formula: `(Cap% × Pre-Warrant FD − Existing Pref Shares) / (1 − Cap%)`
   - Result: **50,529 shares** (constant across EVs since pre-warrant FD is constant)

2. **Shortfall Shares:** Shares needed to close the gap to 3.0x MOIC
   - Shortfall = $30,000,000 − $24,698,426 = $5,301,574
   - Shares = Shortfall / (OIP × Liq Pref Multiple) = $5,301,574 / $53.36 = 99,355 shares

**Warrant Shares = MIN(Cap Shares, Shortfall Shares)** — in practice, the cap (50,529) is always binding at lower EVs.

If MOIC ≥ 3.0x before warrant → Warrant Shares = 0.

### Warrant Liquidation Preference

Warrant shares receive a liq pref = Shares × OIP × 2.0x (same multiple, **no PIK accrual**):
- 50,529 × $26.68 × 2.0 = **$2,696,227**

---

## 6. Phase 2b — Final A-1 + Warrant Conversion Test (Rows 51–57)

**Purpose:** Now that warrant shares are determined, re-test the A-1 conversion decision with A-1 and warrant shares **bundled together**. This is the **final, authoritative** conversion decision used by all downstream sheets.

**Why bundle?** BHC controls both the A-1 preferred and the MOIC warrant. The optimal decision considers them as a package — both convert or both take liq pref.

**Logic:**
1. Combined Shares = A-1 (130,984) + Warrant (50,529) = **181,513**
2. Combined Liq Pref = A-1 Liq Pref ($8,048,426) + Warrant Liq Pref ($2,696,227) = **$10,744,653**
3. Post-Warrant FD = Pre-Warrant FD + Warrant Shares = 1,401,573 + 50,529 = **1,452,102**
4. Post-Warrant Per-Share = Equity Available / Post-Warrant FD
5. A-1+W As-Converted Value = Combined Shares × Post-Warrant Per-Share
6. **Converts if:** As-Converted Value > Combined Liq Pref

| EV | Per-Share | A-1+W As-Conv | Combined Liq Pref | Final Decision |
|---|---|---|---|---|
| $90M | $52.30 | $9,493,763 | $10,744,653 | ❌ Take liq pref |
| $100M | $58.78 | $10,668,765 | $10,744,653 | ❌ Take liq pref |
| $110M | $65.11 | $11,818,267 | $10,744,653 | ✅ Convert |

**Key insight:** Phase 1 shows A-1 converting at $80M+, but the final decision (Phase 2b) doesn't convert until **$110M** because the warrant's additional liq pref ($2.7M) raises the hurdle from $8.0M to $10.7M. The bundled test is more conservative.

---

## 7. Phase 3 — B-Series Sequential Conversion (Rows 61–102)

**Purpose:** Test each B-series tranche individually, in ascending OIP order, with each decision updating the available equity and share count for the next tranche.

**Processing Order:** B-1 ($15) → B-2 ($20) → B-6 ($37.53) → B-3 ($40) → B-4 ($40.79) → B-5 ($68.21)

**For each tranche, the test is:**

1. Start with remaining equity and running FD share count (after A-1/warrant decision and all prior B tranches)
2. As-Converted Value = `Tranche Shares / (Running FD + Tranche Shares) × Remaining Equity`
3. **Converts if:** As-Converted Value > Tranche Total Liq Pref

**If converts:** Shares join the common pool, equity is unchanged (they participate in residual)  
**If takes liq pref:** Payout = MIN(Liq Pref, Remaining Equity), equity reduced by payout amount

This sequential approach means **earlier tranches (lower OIP) convert first**, which is economically correct — lower-OIP tranches have smaller liq prefs relative to their share counts, making conversion attractive at lower EVs.

**Conversion thresholds (approximate):**

| Tranche | First Converts At | OIP |
|---|---|---|
| B-1 | $60M | $15.00 |
| B-2 | $80M | $20.00 |
| B-6 | $150M | $37.53 |
| B-3 | $150M | $40.00 |
| B-4 | $170M | $40.79 |
| B-5 | $250M | $68.21 |

---

## 8. Residual Equity & Final Payouts (Rows 105–127)

After all preferred tranches are resolved:

- **Residual Equity** = whatever remains after all liq prefs are paid
- **Residual FD Shares** = Base FD + all converted preferred shares (+ warrant shares if A-1+W converted)
- **Residual Per-Share** = Residual Equity / Residual FD Shares

**Final payouts (Rows 119–127):**
- **If tranche converted:** Payout = Tranche Shares × Residual Per-Share (pro-rata with common)
- **If tranche took liq pref:** Payout = the liq pref amount already deducted in Phase 3
- **Common equity:** Base FD × Residual Per-Share

---

## 9. 3x MOIC Cap Adjustment (Rows 152–167)

**Purpose:** After running the full waterfall with cap-level warrant shares, check whether BHC's actual MOIC exceeds 3.0x. If so, reduce warrant shares so BHC achieves exactly 3.0x — excess equity is redistributed to common shareholders.

**Logic:**
1. **Preliminary BHC Total** = Debt Principal + Interest + A-1 Payout + Warrant Payout
2. **Preliminary MOIC** = Total / $10M Basis
3. If MOIC > 3.0x:
   - Target BHC Equity (A-1+W combined) = 3.0x × $10M − $10M − $6.65M = **$13,350,000**
   - Adjusted Warrant Shares = (Target Equity / Residual Per-Share) − A-1 Shares
   - Final Warrant Shares = MIN(Original Warrant Shares, Adjusted Warrant Shares)
4. Any excess warrant payout is redirected to common shareholders (Row 165–166)

**Result across EVs:**

| EV Range | Warrant Shares | BHC MOIC | Binding Constraint |
|---|---|---|---|
| $50M–$100M | 50,529 | 2.74x | 12.5% ownership cap (below 3x) |
| $110M–$120M | 50,529 | 2.76x–2.89x | Cap binding, approaching 3x |
| $130M–$170M | 47,359 → 1,404 | **3.00x** | 3x MOIC cap binding |
| $180M+ | 0 | 3.06x+ | Naturally exceeds 3x, no warrant needed |

BHC's returns are **monotonically increasing** across all 36 scenarios — the 3x cap prevents the "valley" that would otherwise occur as warrant shares decline.

---

## 10. Waterfall Balance Check (Row 167)

Every EV scenario is verified to balance to the penny:

```
Final Waterfall Total = Debt Principal + Fees + CEO Phantom + A-1 Payout + Warrant Payout + Σ(B-Series Payouts) + Common Equity = EV
```

All 36 scenarios balance exactly.

---

## 11. Summary: BHC Total Proceeds Composition

BHC receives four components at exit:

| Component | Description | Priority |
|---|---|---|
| 1. Debt Principal | $10,000,000 senior secured | 1st (after fees + CEO) |
| 2. Debt Interest | $6,650,000 (14% simple interest) | Paid with principal |
| 3. A-1 Preferred | Liq pref OR conversion to common | 3rd (after debt) |
| 4. MOIC Warrant | Additional shares to reach 3.0x target | Participates with A-1 |

**BHC MOIC** = (Debt Principal + Interest + A-1 Payout + Warrant Payout) / $10M Basis

---

## 12. Key Downstream References

| Consuming Sheet | What It Pulls | IC Source Row |
|---|---|---|
| Investor Returns | A-1+W conversion flag | Row 57 |
| Investor Returns | Final A-1 Payout | Row 161 |
| Investor Returns | Final Warrant Payout | Row 162 |
| Investor Returns | B-series payouts | Rows 121–126 |
| Investor Returns | Common equity | Row 166 |
| Super Summary | A-1 liq pref (when not converting) | Row 57 + F19 |
| Super Summary | Converted preferred shares | Row 131 |
| MOIC Ratchet | Warrant shares, ownership calc | Rows 48–49 |

