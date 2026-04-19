# F1 2026 — Live Standings PWA + ML Tire Strategy

A Progressive Web App that installs on your iPhone home screen, tracks the 2026 F1 season in real time, and runs a machine learning model to predict tire degradation curves and recommend pit windows.

Live demo: [f1app-seven.vercel.app](https://f1app-seven.vercel.app)

---

## What it does

**Live season tracking** — driver and constructor standings, race results, and the full 2026 calendar, synced automatically from the Jolpica F1 API every 10 minutes. Push notifications fire when new race results drop.

**ML tire strategy** — a trained LSTM model predicts how each tire compound will degrade over a stint at any track, then surfaces an optimal pit window for Soft, Medium, and Hard tires. Powered by a FastAPI backend deployed on Railway.

---

## ML Model

The strategy engine is built on a two-layer LSTM trained on 36 races across the 2022 and 2023 seasons using FastF1 lap telemetry data.

### Architecture

```
10-lap sliding window (per-lap features)
        ↓
LSTM (2 layers, hidden_size=128, dropout=0.3)
        ↓
Final hidden state (128 values)
        +
Track embedding (21 tracks → 8-dim learned vectors)
        ↓
Feedforward head (136 → 64 → 32 → 1)
        ↓
Predicted lap time delta (seconds vs. fresh tire)
```

The model takes the last 10 laps of a stint as input and predicts the degradation delta for the next lap. Track identity is encoded via a learned embedding layer — the model discovers that Bahrain and Abu Dhabi have similar degradation profiles without being told.

### Features

| Feature | Description |
|---|---|
| `TyreLife` | Laps on current tire set |
| `StintLap` | Lap number within the stint |
| `TrackTemp / AirTemp` | Surface and air temperature |
| `FuelProxy` | Estimated fuel remaining (race lap / total laps) |
| `Compound` | SOFT / MEDIUM / HARD (one-hot encoded) |
| `PositionNorm` | On-track position normalized by field size |
| `StintProgress` | How far through the stint (0–1) |
| `Sector1/2/3 Time` | Sector breakdown in seconds |

**Target:** `LapTimeDelta` — seconds slower than the first clean lap of the stint. Isolates tire degradation from car pace and driver differences.

### Training results

| Phase | Model | RMSE |
|---|---|---|
| Phase 1 | Feedforward (single lap) | 1.33s |
| Phase 2 | LSTM, window=5 | 1.01s |
| Phase 3 | LSTM, window=10, track embeddings, 36 races | 0.59s |

**Final metrics:** RMSE 0.588s · MAE 0.402s · 90th percentile error 0.881s · 55.8% improvement over baseline

In F1 terms: predictions are within ±0.4s per lap on average. Over a 20-lap stint that's ±8s total — accurate enough to inform meaningful pit stop decisions.

### Why the improvement across phases

Phase 1 used a feedforward network that saw one lap at a time with no memory of what came before. It learned to predict near the mean and failed completely on warm-up laps (the cold tire dip at the start of a stint). Phase 2 introduced sequence context with a 5-lap sliding window — the model could now see the degradation trend before predicting. Phase 3 scaled to 36 races, widened the window to 10 laps, added track embeddings, and applied outlier filtering to remove anomalous stints that were inflating the loss.

---

## Strategy API

The ML model runs as a FastAPI service deployed on Railway. The PWA calls it from the Strategy tab.

**Base URL:** `https://your-api.railway.app`

### Endpoints

`GET /health` — API status and model loaded confirmation

`GET /tracks` — List of 21 tracks the model was trained on

`POST /strategy` — Simulate one compound stint and return degradation curve + pit window

```json
{
  "track": "Bahrain",
  "compound": "SOFT",
  "total_race_laps": 57,
  "current_race_lap": 1,
  "tyre_age_at_start": 0,
  "track_temp": 38,
  "air_temp": 28,
  "driver_position": 1
}
```

`POST /compare` — Simulate all three compounds and return a side-by-side comparison. This is what the Strategy tab calls.

### Pit window logic

The API computes a 3-lap rolling degradation rate from the predicted delta curve. When that rate exceeds 0.08s/lap — meaning the tire is losing 0.08 seconds of performance every lap — it flags that lap as the start of the pit window. The optimal lap is the first lap that crosses the threshold; the window spans ±2–3 laps around it.

---

## App features

- **Drivers tab** — full championship standings with team colors and win counts
- **Teams tab** — constructor standings
- **Results tab** — race-by-race results with gap times, fastest lap badge, and round navigation
- **Calendar tab** — full 2026 schedule with race status badges
- **Strategy tab** — ML pit window analysis per compound for any known track

---

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | Vanilla JS, HTML/CSS, PWA (service worker + manifest) |
| Standings API | Jolpica F1 API (free, no key required) |
| ML model | PyTorch · LSTM · FastF1 |
| Strategy API | FastAPI · Uvicorn |
| Frontend hosting | Vercel |
| API hosting | Railway |

---

## Running locally

**Frontend**

No build step needed. Open `index.html` directly or serve with any static server:

```bash
npx serve .
```

Point your browser to `localhost:3000`.

**Strategy API**

```bash
pip install fastapi uvicorn torch scikit-learn pandas fastf1 numpy pydantic
python main.py
```

The server starts on `http://localhost:8000`. On first run it downloads and caches all 36 training races via FastF1 — this takes a few minutes. Subsequent starts are instant.

You'll need `best_model_phase3.pt` in the same directory. To train it from scratch, run the Phase 3 training script.

**Connecting frontend to local API**

In `index.html`, change:
```javascript
const ML_API = "https://your-api.railway.app";
```
to:
```javascript
const ML_API = "http://localhost:8000";
```

---

## Installing on iPhone

1. Open the Vercel URL in **Safari** (not Chrome)
2. Tap the Share button → **Add to Home Screen**
3. Tap **Add** — the F1 icon appears on your home screen

Push notifications require iOS 16.4+ and the app must be opened from the home screen icon, not from Safari directly.

---

## Files

| File | Purpose |
|---|---|
| `index.html` | Full PWA — HTML, CSS, and JS in one file |
| `main.py` | FastAPI strategy server wrapping the LSTM predictor |
| `requirements.txt` | Python dependencies for the API |
| `sw.js` | Service worker — caching, background sync, push notifications |
| `manifest.json` | PWA metadata — name, icon, theme color |
| `vercel.json` | Hosting headers required for service worker scope |
| `Procfile` | Railway start command |

---
