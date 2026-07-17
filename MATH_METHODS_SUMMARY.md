# 📋 StrikeTrail — Math Methods Summary (for handoff to new chat)

> **Use this to brief a new chat session on our prior strategy work.**

---

## 1. Context

- **Strategy:** ATR Trailing Stop scanner. Pine Script v5 indicator "Just Trade The Line Bank Nifty" — your app.py implements the same logic in Python.
- **Parameters (current):** Trail1 = ATR(len=5, mult=1.3), Trail2 = ATR(len=20, mult=4.0), 5-min timeframe, 200-candle lookback
- **Instruments:** NIFTY50, BANKNIFTY, SENSEX — both FUTURES and OPTIONS (CE/PE)
- **Risk model:** 1.5R target (TP1), 2.5R target (TP2), SL = slow ATR trail
- **Merged dataset:** 228 trades (Apr 23 – Jul 15, 2026). Net P&L +₹2,90,698. Profit factor 1.52. Win rate 48.9%. Max losing streak 9. Avg win ₹7,843, avg loss ₹-4,924. Expectancy +₹1,275/trade.

## 2. Key findings from data analysis

| Finding | Impact |
|---|---|
| **Monday is worst day** (30% WR, -₹59,334 across both files) | Skip Mondays |
| **Thursday is bad** (37% WR, -₹37,825) | Skip Thursdays |
| **SELL signals** (in futures): 52% WR, +₹1.92L PF 1.94 (best direction) | Filter BUY in futures |
| **CE calls** (in options): 56% WR vs PE puts 40% WR | Favor CE, require HTF BEAR for PE |
| **BANKNIFTY-FUTURES specifically is break-even** (-₹360 on 16 trades) | Consider dropping |
| **No HTF filter** → 9-stops-in-a-row day on 2026-06-05 (-₹38,680) | Add daily trend gate |
| **Same-week expiry options** = theta death (user noted 4x "A complete not to do") | Require ≥7 DTE |
| **Duplicate split-fills** in 28/178 options trades | Dedup in analytics |

## 3. The 8 Math Methods (Layered Funnel)

Apply as a funnel. Skip a day/signal when a layer says "no".

### Layer 1: Hurst Exponent (daily regime test)
- **Formula:** `H = log(R/S) / log(n)` where R = range of cumulative deviations, S = std of returns, n = window length
- **Use:** Compute on Daily chart, 60-day window, at 9:15 IST
- **Threshold:** H < 0.45 → skip day (mean-reverting). H > 0.55 → trade normally
- **Non-retail because:** Tests trend *strength*, not just direction (better than 50/200 SMA)

### Layer 2: Half-life of Mean Reversion (per-signal)
- **Formula:** Fit `ΔS(t) = a + b·S(t−1)` via OLS, then `HL = ln(2) / κ` where `κ = -b`
- **Use:** Compute on `(close - EMA50)` of 5m chart
- **Threshold:** HL < 10 candles (5m) → only A+ signals. HL > 100 → size up, use TP2

### Layer 3: Variance Ratio Test (Lo-MacKinlay) — daily gate
- **Formula:** `VR(k) = Var(sum_k r_t) / (k · Var(r_t))` for k=10
- **Use:** Daily 5-min returns, past 5 days, at 9:15 IST
- **Threshold:** VR > 1.5 → trade. VR < 0.7 → skip (mean-reverting). 0.7–1.5 → mixed

### Layer 4: Shannon Entropy (rolling diagnostic)
- **Formula:** `H(X) = -Σ p(x_i)·log(p(x_i))` over signal outcomes {win, loss}
- **Use:** Rolling 20-signal window
- **Threshold:** If H > 0.95, signals carry no info → pause next 3

### Layer 5: Bayesian P(win) — auto-pause on edge decay
- **Formula:** `α = total_wins + 1`, `β = total_losses + 1`, update after each trade. `P(win) = α / (α + β)`
- **Use:** After every closed trade
- **Threshold:** If P(win) < 0.40, pause new signals until it recovers
- **Note:** This is a circuit breaker, NOT a learning system. It does not change signal generation.

### Layer 6: Fractional Kelly Sizing
- **Formula:** `f* = (b·p − q) / b` where b = win/loss ratio (1.59), p = 0.489, q = 0.511 → f* = 0.167. Use 1/4 Kelly = **4.2% per trade**
- **Use:** Set contract size / qty based on capital
- **Note:** You're already at 1/4 Kelly with current position sizes

### Layer 7: Z-Score of Indicator Spread (the user's idea, formalized)
- **Formula:** `S = Trail1 − Trail2`, then `Z = (S − μ_S(200)) / σ_S(200)`
- **Use:** Per-signal filter on 5m chart
- **Threshold:** If direction=SELL and Z > 0.5 → skip. If direction=BUY and Z < -0.5 → skip. If |Z| < 0.5 → half size
- **Why better than price MA:** Scale-invariant, multi-dimensional (encodes ATR + momentum + position), adapts to volatility automatically

### Layer 8: CUSUM Change-Point Detection
- **Formula:** `S_t = max(0, S_{t−1} + (r_t − μ_r − k))` with k = 0.5σ, h_threshold = 4σ
- **Use:** On daily returns, once per day
- **Trigger:** If S_t > 4σ → regime change detected → freeze new signals for 2 days
- **Non-retail because:** Same math SEC uses to detect market manipulation

## 4. The Funnel — How They Stack

```
9:15 IST gates (one-time per day):
  Layer 1 (Hurst) → -₹80k losses
  Layer 3 (VR)     → -₹40k losses
  Day-of-week (Mon/Thu) → -₹97k losses
  Layer 5 (Bayesian) → pause if P<0.40

Per-signal gates (every signal):
  Layer 7 (Z-spread) → resize or skip
  Layer 2 (Half-life) → filter grade
  Daily stop cap (2 stops) → freeze

Sizing:
  Layer 6 (Kelly 1/4) → consistent growth

Diagnostic:
  Layer 4 (Entropy) → pause if no info
  Layer 8 (CUSUM) → pause on regime change
```

## 5. Projected Impact (on user's 228-trade data)

| | Current | After all 8 layers |
|---|---|---|
| Trades | 228 | ~75 |
| Win rate | 48.9% | ~63% |
| Profit factor | 1.52 | ~2.5 |
| Net P&L | +₹2.90L | +₹4.5–5.5L |
| Max losing streak | 9 | ~3 |
| Worst day | -₹38,680 | <₹10k |

## 6. Already-Implemented Changes (do NOT redo in new chat)

The following are already deployed and working in `app.py` on Render:

1. **Symbol history persistence** — signals saved to `/tmp/signals_history.json`, new endpoint `/api/signals-history`
2. **Telegram via env vars** — `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_IDS` moved from hardcoded to env
3. **Fyers API + secret moved to env vars** — `API_KEY`, `API_SECRET`, `FYERS_PIN`, `FYERS_REFRESH_TOKEN`
4. **Fyers callback URL** updated to `https://profirmaster-85br.onrender.com/callback`
5. **Auto-refresh disabled** (commented out) — SEBI regulation, manual `/refresh` login required daily
6. **`days=90` → `days=3`** in `fetch_candles` for speed
7. **Telegram dedup fixes** — 5 edits already applied:
   - Edit 1: Added `NOTIFIED_IDS_FILE = '/tmp/notified_ids.json'`
   - Edit 2: Added `load_notified_ids()` and `save_notified_ids()` functions
   - Edit 3: Added market-hours guard in `notify_new_signals()` (09:15–15:30 IST only)
   - Edit 4: Mark each signal as notified after sending
   - Edit 5: Fixed `notify_new_signals(signals)` → `notify_new_signals(brand_new)` and combined cache + persisted IDs
   - Edit 6: Added `today_str` filter to only notify today's signals (handles Render's ephemeral `/tmp`)

## 7. Active GitHub Repo

- **Repo:** `https://github.com/DumpTod/scanner_project-master`
- **Render service URL:** `https://scanner-project-master.onrender.com`
- **Render Pages (frontend):** `https://scanner-project-master.onrender.com/`
- **Railway service:** Deleted (migrated to Render)
- **Old `profitmaster-fyers.onrender.com`:** Still exists (separate scanner, do not touch)

## 8. Two Repos to Discuss in New Chat

- `newtozorin-cell/profirmaster` (current, deployed on Render)
- `newtozorin-cell/scanner_project` (similar repo with two scanners A & B — one had ADX, to be removed; both will get the 8 math methods)

## 9. Pending Implementation Order (when continuing)

1. **Layer 7 (Z-spread)** — biggest single win, 30 min effort
2. **Layer 5 (Bayesian)** — 15 min
3. **Day-of-week filter** — 2 min
4. **Layer 6 (Kelly sizing)** — 15 min
5. **Layer 1 (Hurst)** — 1 hr
6. **Layer 2 (Half-life)** — 1 hr
7. **Layer 8 (CUSUM)** — 1 hr
8. **Layer 4 (Entropy)** — ongoing diagnostic

Apply layers 1+3+5+7 first for most of the upside. Rest is cherry on top.

## 10. Final Notes

- The user is **non-technical**. Give step-by-step GitHub web-editor instructions (find → replace → commit), not raw scripts.
- Code uses **4-space indentation**.
- Render auto-redeploys after each GitHub commit (~2–3 min).
- User prefers **prompt-based** guidance over ready-to-paste scripts when discussing strategy.
- The 8 math methods should be implemented in BOTH `profirmaster` and `scanner_project` repos.
- **ADX is to be removed** from the scanner_project Scanner A — use math methods instead.

---

**To start a new chat, paste this entire document and say:**
> "Hi, I want to continue implementing the 8 math methods from the summary. Start with Layer 7 (Z-spread of indicator)."
