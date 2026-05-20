# nbai-bets

A daily-updating NBA betting dashboard powered by a 5-model stacked ensemble, live Polymarket odds, and a Raspberry Pi cron job. Brand name: **Hidden Lines**.

🔗 **Live site:** [paulkellar2023.github.io/nbai-bets](https://paulkellar2023.github.io/nbai-bets)

> ⚠️ **Disclaimer:** I am not a professional bettor. This project is a personal experiment in applied ML, prediction-market mechanics, and end-to-end system design. If you use these picks to inform real wagers, **please bet responsibly**. Past performance does not predict future results.

---

## What this is

A self-contained system that runs daily, end-to-end:

1. **Fetches** today's NBA schedule and the matching Polymarket prediction markets
2. **Predicts** each game using a 5-model stacked ensemble trained on 2019-2026 NBA data
3. **Identifies** pricing gaps between model probability and market probability
4. **Categorizes** each game as FAVORITE_BET, OPPORTUNITY, PASS, or AVOID
5. **Sizes** bets as a % of current bankroll with confidence-tier haircuts
6. **Generates** natural-language analysis per game via the Anthropic API (Claude)
7. **Renders** a static HTML dashboard and commits it to GitHub Pages
8. **Resolves** prior-day bets against actual scores from `nba_api`
9. **Tracks** bankroll and W-L history persistently in a JSON state file

The whole stack runs on a Raspberry Pi 4 with a cron job. No cloud bill, no servers.

---

## Architecture

```
                          ┌──────────────────────┐
                          │  Raspberry Pi 4      │
                          │  (cron, 3 PM ET)     │
                          └──────────┬───────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
              ▼                      ▼                      ▼
       ┌─────────────┐       ┌───────────────┐       ┌──────────────┐
       │  nba_api    │       │  Polymarket   │       │  Anthropic   │
       │  (schedule, │       │  (gamma +     │       │  (Claude     │
       │   results)  │       │   CLOB)       │       │   API)       │
       └──────┬──────┘       └───────┬───────┘       └──────┬───────┘
              │                      │                      │
              └──────────────────────┼──────────────────────┘
                                     │
                                     ▼
                          ┌──────────────────────┐
                          │  Inference notebook  │
                          │  (5-model ensemble)  │
                          │   ↓                  │
                          │  bet sizing          │
                          │   ↓                  │
                          │  state.json update   │
                          │   ↓                  │
                          │  HTML render         │
                          │   ↓                  │
                          │  git push (GH Pages) │
                          └──────────────────────┘
```

---

## Repo Structure

```
nbai-bets/                       # This repo (site)
├── index.html                   # Auto-generated daily by the inference notebook
└── README.md                    # You are here

nbai-bets-model/                 # Separate repo (model code)
├── nba_betting_model_v5.py      # Training notebook (model bundle creation)
├── nba_pi_inference_v9.ipynb    # Daily inference notebook
└── README.md
```

---

## Model

**5-model stacked ensemble:**

| Base Model | Accuracy | Log Loss | Brier |
|---|---|---|---|
| Logistic Regression | 65.6% | 0.621 | 0.216 |
| Random Forest | 65.7% | 0.625 | 0.217 |
| XGBoost | 65.0% | 0.627 | 0.219 |
| LightGBM | 65.0% | 0.632 | 0.221 |
| Extra Trees | 65.4% | 0.625 | 0.218 |

- **Meta-learner:** Logistic Regression on out-of-fold predictions (TimeSeriesSplit, 5 outer folds × 3 inner folds)
- **Training window:** 2019-20 → 2025-26 (regular season + playoffs, ~10k games)
- **Retrain cadence:** monthly during the regular season
- **Optimization target:** log loss / Brier score (for calibration), not accuracy

**Context for the Brier numbers:**
- Random coin flip: Brier = 0.250
- Sharp Vegas closing lines: Brier ≈ 0.18-0.20
- This model: Brier ≈ 0.218 → meaningfully better than random, not yet pro-sharp

**Features** (column-prefix conventions):
- `elo_*` — ELO ratings and gaps
- `home_*` / `away_*` — per-team rolling stats (offensive/defensive ratings, pace, availability %)
- `diff_*` — home-minus-away differentials
- `h2h_*` — head-to-head matchup history
- `rest_*` / `b2b_*` — rest days, back-to-back indicators
- `norm_prob_*` — normalized win probabilities from ELO
- `odds_spread` — market-derived spread

---

## Betting Strategy

Each prediction gets an **attractiveness score** (0-10) combining:
- Edge magnitude (model vs market probability gap)
- Market liquidity
- Money flow confirmation (order book imbalance)
- Price drift confirmation (24h)
- Calibration confidence (per-probability-bin error rate)

Bet sizing rule:

```python
base = (attractiveness / 10) * current_bankroll

if recommendation in ("OPPORTUNITY", "OPPORTUNITY_STRONG"):
    base *= 0.25   # 75% haircut for betting against market consensus
if attractiveness <= 3.3:
    base *= 0.50   # 50% haircut for low-confidence picks

bet_amount = min(base, 0.50 * current_bankroll)   # 50% per-bet cap
```

Examples on a $12.33 bankroll:
| Scenario | Attractiveness | Bet |
|---|---|---|
| High-conf favorite | 7.0/10 | $6.17 (capped at 50%) |
| High-conf underdog | 7.0/10 | $2.16 (×0.25 haircut) |
| Low-conf favorite | 3.0/10 | $1.85 (×0.50 haircut) |
| Low-conf underdog | 2.5/10 | $0.39 (×0.25 × 0.50, compound) |

---

## Daily Workflow

1. **3:00 PM ET cron** triggers `update_site.sh` on the Pi
2. Script loads `state.json` (persistent bankroll + history)
3. Resolves any prior-day bets (queries `nba_api` for final scores)
4. Fetches today's schedule, builds features, runs ensemble
5. For each game: fetches matching Polymarket market, computes edge, generates Claude analysis
6. Updates `state.json`, renders `index.html`, commits to this repo
7. GitHub Pages serves the new dashboard within ~60 seconds

---

## Local Setup (if you want to fork)

```bash
# Clone the model repo (separate from this site repo)
git clone https://github.com/<your-user>/nbai-bets-model.git
cd nbai-bets-model

# Install dependencies
pip install -r requirements.txt

# Set your API key
export ANTHROPIC_API_KEY="sk-ant-..."

# Train the model (one-time, takes ~30 min on a Pi)
python nba_betting_model_v5.py

# Run inference (predicts today + renders HTML)
jupyter nbconvert --to script nba_pi_inference_v9.ipynb
python nba_pi_inference_v9.py
```

You'll need to set up:
- A separate GitHub Pages repo (this one)
- An SSH deploy key on the Pi for that repo
- An `update_site.sh` wrapper script that runs the inference + git push

---

## Roadmap

See the [accompanying LinkedIn article](https://www.linkedin.com/in/paulkellar/) for the full write-up. Highlights:

- [ ] Blockchain-based auto-betting bot (Polymarket runs on Polygon)
- [ ] Apply same architecture to NFL and MLB
- [ ] Closing line value (CLV) tracking as a model-performance metric
- [ ] Player-level injury feature engineering (currently aggregated to team %)
- [ ] Regular-season vs playoff regime models
- [ ] Auto-A/B test framework on monthly retrains
- [ ] Multi-timeframe ensemble (early-season / mid-season / playoff)

---

## Credits

- [@zostaff on X](https://x.com/zostaff/status/2046228556234072547) for the initial idea on combining `nba_api` with the Polymarket API
- `swar/nba_api` — community NBA Stats wrapper
- Polymarket — prediction market with deep NBA liquidity
- Anthropic — Claude for natural-language analysis generation
- Brier (1950) — for proper scoring rules and the reason we don't just optimize accuracy

---

## License

MIT
