# 📈 Linda — Multi-TF Regime Dashboard

> **HMM-Powered Forex Scanner · XGBoost Signal Forecaster · Live Dashboard**

[![Python](https://img.shields.io/badge/python-3.10-blue)]()
[![XGBoost](https://img.shields.io/badge/XGBoost-3.x-orange)]()
[![Flask](https://img.shields.io/badge/Flask-3.x-lightgrey)]()
[![Status](https://img.shields.io/badge/status-production-green)]()

---

## ✨ What It Does

Linda is an **end-to-end market analysis system** that combines Hidden Markov Models with gradient-boosted machine learning to detect, classify, and predict market regimes across 15 forex/crypto/metal instruments in real time.

<p align="center">
  <img src="https://github.com/user-attachments/assets/7c1424eb-40ea-4e8e-b975-988caca40a7e" width="48%" />
  
  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
  
  <img src="https://github.com/user-attachments/assets/22781be5-a5b1-47dd-80d4-886358e82c6f" width="48%" />
</p>




**Key capabilities:**
- 🔍 **Real-time regime detection** — 8-state HMM on 5 timeframes (M5 → D1) for each symbol
- 🎯 **Scanner v5** — State machine (WATCHLIST → READY → EXECUTABLE → LATE → EXHAUSTED) with edge scoring
- 🤖 **Signal Forecaster** — XGBoost models (5 horizons per symbol) predicting direction 15min to 4h ahead  
- 📊 **Live Dashboard** — Dark-theme SPA with real-time updates, drag panels, trade decisions
- 💬 **LLM Chat** — Linda assistant (DeepSeek-powered) for market questions
- ⚡ **60+ trained models** — 15 symbols × 5 horizons, retrained on demand

---

<img width="1911" height="919" alt="image" src="https://github.com/user-attachments/assets/2d39e862-f709-4de2-bb1f-f225d1ce7e64" />

<img width="1161" height="641" alt="image" src="https://github.com/user-attachments/assets/e1277d4c-9f3a-44b2-969b-688b7fedaae8" />



## 🏆 Results

| Metric | Value |
|--------|-------|
| Models trained | **72** (15 symbols × 5 horizons) |
| Top CV accuracy | **99.2%** (EURUSD N=24), **100%** (USDJPY N=24) |
| Average CV (all horizons) | **53.7%** (vs 33% random) |
| Models with CV > 0.6 | **27%** (beats random by 2×) |
| Feature importance leader | `close_1` (18-38%), `hour` (15-26%), `maturity` (10-48%) |
| Historical bars | **183K** M5 bars, 12 symbols, 4 years |
| Data pipeline | **17K+ scan entries**, 16K labelled, 15 symbols |

---

<img width="386" height="456" alt="image" src="https://github.com/user-attachments/assets/3178a3fd-5be9-4ea5-9c61-4d1f4e2b571e" />

<img width="479" height="401" alt="image" src="https://github.com/user-attachments/assets/cf00964a-5847-46a1-a6b2-1db7072e4e71" />



## 🧠 Architecture

```
                    ┌─────────────────────────────────┐
                    │      MetaTrader 5 (Data Feed)   │
                    └──────────────┬──────────────────┘
                                   │ OHLCV bars
                    ┌──────────────▼──────────────────┐
                    │  Feature Engine (10D vector)    │
                    │  ret · mom · vol · rsi · bb · vs│
                    └──────────────┬──────────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                    ▼
    ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
    │   HMM Engine    │  │     Scanner v5  │  │ Signal Forecaster│
    │  8 states × 5 TF │  │  State Machine │  │ XGBoost × 5 hor. │
    │  HSMM · EM · BIC │  │  Edge Scoring  │  │ GridSearch · CV  │
    └────────┬────────┘  └────────┬────────┘  └────────┬────────┘
             │                    │                    │
             └────────────────────┼────────────────────┘
                                  │
                    ┌──────────────▼──────────────────┐
                    │       Flask Dashboard           │
                    │   HMR · Charts · Insights       │
                    │   Drag Panels · Chat · API      │
                    └─────────────────────────────────┘
```

---

## 🔬 HMM Engine (Core)

| Feature | Detail |
|---------|--------|
| **States** | 8 Gaussian states, named: IMP, REV, SURG, SQZ, TRD |
| **Training** | Baum-Welch EM, 3 restarts + best-LL selection |
| **HSMM** | Semi-Markov with logistic duration decay |
| **Covariance** | Shrinkage: 70% empirical + 30% diagonal prior |
| **Initialization** | K-means++ with 5 restarts |
| **Multi-TF** | 5 timeframes (M5/M15/H1/H4/D1) trained in parallel via ThreadPoolExecutor |
| **State selection** | AIC/AICc + majority vote across criteria |
| **Feature vector** | 10D: return, momentum, volatility, RSI, Bollinger squeeze, volume spike |

---

## 🎯 Scanner v5 — State Machine

States transition from early edge to exhaustion:

```
WATCHLIST ──► READY ──► EXECUTABLE ──► LATE ──► EXHAUSTED
    │            │            │           │           │
    │            │            │           │           └── IGNORE
    │            │            │           └────────────── IGNORE
    │            │            └────────────────────────── IGNORE
    │            └─────────────────────────────────────── IGNORE
    └──────────────────────────────────────────────────── IGNORE
```

- **Adaptive thresholds** per symbol class (forex/metal/crypto)
- **Edge scoring** via HMM transition matrix + multi-step absorption probability
- **Spike classification** — 5 subtypes: BREAKOUT / EXHAUSTION / LIQUIDATION / NEWS / ABSORPTION
- **Maturity tracking** — HSMM expected remaining duration
- **Exhaustion detection** — 6-signal composite (overstay, vol decay, entropy rise)

---

## 🤖 Signal Forecaster — XGBoost Pipeline

**13 features** per entry:

| Group | Features |
|-------|----------|
| **HMM-derived** (8) | `char_factor`, `edge`, `trans_prob`, `entropy`, `m15_aligned`, `h1_opposing`, `maturity`, `spike` |
| **Market-derived** (5) | `close_1` (T-5min), `close_5` (T-25min), `ATR`, `volume_ratio`, `hour_of_day` |

**Training pipeline:**
```
Scan ─► SQLite ─► Label (ATR-adaptive) ─► Train (GridSearch + TimeSeriesSplit CV) ─► Predict
```

- **Per-symbol, per-horizon** models (75 max)
- **ATR-adaptive thresholds** — label = UP/DOWN/FLAT relative to recent volatility
- **GridSearch** over `max_depth ∈ [3,5,7]` and `lr ∈ [0.03, 0.05, 0.1]`
- **Early stopping** (10 rounds patience) prevents overfitting
- **TimeSeriesSplit CV** (5 folds, temporal — no look-ahead)
- **Class remapping** for non-contiguous classes (XGBoost requirement)
- **Platt calibration** for well-calibrated confidence scores

---

## 📊 Dashboard

Single-page application — vanilla JS + CSS (dark theme).

| Panel | Purpose |
|-------|---------|
| 📋 **Instruments** | Symbol browser, categories, watchlist, history loader |
| 🔬 **Analysis** | Multi-TF regime view, transition matrix, state occupancy |
| 📡 **Scan Results** | Live scan table, sorting, exec/insight/chart buttons |
| 🎯 **Prospects** | Ranked by state priority, color by time-in-state |
| 💬 **Chat** | DeepSeek-powered trading assistant (Linda) |
| 📈 **Trade Decision** | Entry/stop/target with position sizing |
| 🔍 **Insight** | Professional multi-TF market context |
| 🛠️ **Tools** | HMM config, walk-forward validation, recalibrate |
| 🤖 **SF** | Signal Forecaster modal with 5-horizon predictions |

---

## 🚀 Quick Start

```powershell
# 1. Install dependencies
pip install flask waitress numpy pandas xgboost scikit-learn joblib

# 2. Start dashboard
cd C:\trader_viktor_best_17
python dashboard\main.py
# → Open http://127.0.0.1:5050

# 3. Load historical data (first time)
# Click "Load History" → loads EURUSD, GBPUSD, ... into SQLite

# 4. Scan symbols
# Click "Scan All" → HMM trains + scanner evaluates all symbols

# 5. Train Signal Forecaster (after 15+ min)
# Open SF → "Compute Labels" → "Train Models" → predictions available
```

---

## 🔌 API Endpoints

| Route | Method | Description |
|-------|--------|-------------|
| `/api/scan` | GET/POST | Scan symbols (POST: `{"symbols":["EURUSD"]}`) |
| `/api/analyze/<symbol>` | GET | Full HMM analysis, 5 TF regimes, charts |
| `/api/validate` | POST | Walk-forward validation per symbol |
| `/api/insight/<symbol>` | GET | Trading insight: direction, levels, ATR |
| `/api/execute/<symbol>` | GET | Trade decision with entry/stop/target |
| `/api/forecast/<symbol>` | GET | Signal Forecaster predictions (5 horizons) |
| `/api/forecast/train` | POST | Train all XGBoost models |
| `/api/forecast/collect` | POST | Compute labels for unlabelled entries |
| `/api/chat` | POST | Linda LLM chat |
| `/api/instruments` | GET | All available instruments with prices |
| `/api/status` | GET | Server / MT5 status |
| `/api/restart` | POST | Restart server |

---

## 🧪 Running Tests

```powershell
python tests\test_hmm.py           # 12 tests — HMM core
python tests\test_feature_engine.py # 9 tests — feature engineering
python tests\test_scanner.py        # 8 tests — scanner logic
python tests\test_validator.py      # 4 tests — walk-forward validation
```

---

## 📁 Project Structure

```
├── agent/
│   ├── regime_detector.py      # HMM core (HSMM, 8 states, parallel TF)
│   ├── scanner.py              # Scanner v5 (state machine, edge scoring)
│   ├── forecaster.py           # XGBoost Signal Forecaster
│   ├── signal_collector.py     # SQLite data pipeline
│   ├── label_computer.py       # ATR-adaptive label computation
│   ├── state_optimizer.py      # AIC/AICc + walk-forward K selection
│   ├── validator.py            # Walk-forward validation engine
│   ├── scanner_train.py        # XGBoost meta-model pipeline
│   ├── insight_engine.py       # Trading insight generator
│   ├── risk.py                 # Risk management
│   ├── symbol_scanner.py       # CLI scanner for data collection
│   └── models/                 # Trained XGBoost models
├── dashboard/
│   ├── main.py                 # Entry point
│   ├── server.py               # Global state, MT5, utilities
│   ├── config.py               # Constants
│   ├── routes/                 # API Blueprints
│   │   ├── scan.py             # /api/scan
│   │   ├── analyze.py          # /api/analyze
│   │   ├── validate.py         # /api/validate
│   │   └── misc.py             # /api/chat, /api/forecast, ...
│   ├── static/js/app.js        # Frontend (65+ functions)
│   └── templates/index.html    # SPA shell
├── data/
│   ├── feature_engine.py       # 10D feature computation
│   └── mt5_collector.py        # MetaTrader 5 bridge
├── memory/
│   ├── trades.db               # Historical OHLCV bars
│   └── signal_data.db          # Scan entries + labels
└── tests/
    ├── test_hmm.py, test_feature_engine.py, test_scanner.py, test_validator.py
```

---

## 🛠️ Tech Stack

| Component | Technology |
|-----------|------------|
| **Backend** | Python 3.10, Flask 3.x, Waitress |
| **ML** | NumPy, pandas, XGBoost 3.x, scikit-learn |
| **HMM** | Custom implementation (Baum-Welch, HSMM, k-means++) |
| **Data** | SQLite (WAL), MetaTrader 5 API |
| **Frontend** | Vanilla JS + CSS (dark theme), 84KB app.js |
| **LLM** | DeepSeek V4 API (Linda Chat) |

---

## 📜 License

MIT — built for personal trading research. Use at your own risk.

---

<div align="center">
  <sub>Built with ❤️ and a lot of ☕ · Viktor</sub>
</div>
