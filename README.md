# bist_algo_tracker
# 🚀 BIST-AlgoTracker (v1/v2)
**Enterprise-Grade Quantitative Trading & Anomaly Detection Terminal for Borsa Istanbul (BIST)**

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Version](https://img.shields.io/badge/version-2.0.0-green.svg)
![Python](https://img.shields.io/badge/python-3.10%2B-blue)
![React](https://img.shields.io/badge/react-18-cyan)

BIST-AlgoTracker is a high-performance, autonomous statistical arbitrage and anomaly detection engine designed for the Borsa Istanbul (BIST) equities market. Evolving from a simple bot into an institutional-grade trading terminal, it detects market manipulation (Spoofing, Iceberg orders, Layering) in real-time using Level-2 Order Book (Derinlik) data.

## 🧠 Core Philosophy
This project does **not** aim to compete in the microsecond High-Frequency Trading (HFT) latency race. Instead, it focuses on **Statistical Edge and Autonomous Intelligence**. It captures institutional footprints and order book imbalances (OBI) using quantitative models and executes trades safely via dynamic circuit breakers.

---

## 🔥 Key Features

### 1. Advanced Anomaly Detection (Analytics Engine)
* **Spoofing & Fake Liquidity (`SAHTE_LİKİDİTE`):** Detects massive order cancellations and manipulation using static OTR/OBI thresholds and dynamic Z-Scores.
* **Iceberg Order Detection:** Identifies hidden institutional accumulations by tracking repetitive, low-variance trades at specific price levels.
* **Layering Detection:** Catches multi-level simultaneous order cancellations.
* **Cross-Symbol Correlation:** Tracks institutional block buys (e.g., BofA, İş Yatırım) across multiple symbols simultaneously.

### 2. Institutional Trading Dashboard (React/TypeScript)
* **DOM Ladder:** Real-time, color-coded Depth of Market (DOM) ladder with click-to-trade functionality.
* **Network Health Telemetry:** Live `Msg/Sec` tracker monitoring WebSocket buffer health.
* **Throttle Protection:** UI is protected against WebSocket flooding via a strict 250ms batching/throttling layer, preventing browser memory leaks.
* **Zero-Hardcode Security (DevSecOps):** API keys (Broker & Market Data) are strictly isolated in `localStorage` via Zustand persist middleware. No credentials touch the codebase.

### 3. Execution & Resilience
* **Circuit Breakers:** Implements `pybreaker` to halt trading during API failures or excessive drawdown.
* **Backpressure Queue:** "Drop-oldest" queue management to ensure the engine only processes the freshest market data.
* **TWAP/VWAP Planning:** Smart order routing and execution simulation.

---

## 🏗️ System Architecture

| Component | Technology | Responsibility |
| :--- | :--- | :--- |
| **Kontrol Kulesi (Backend)** | FastAPI, Python | Core logic, Z-Score math, Risk Engine, WebSocket distribution. |
| **Message Bus** | NATS / Redis | High-throughput Pub/Sub messaging across symbol workers. |
| **Trader Terminal (Frontend)**| React, Tailwind, Zustand | UI/UX, DOM Ladder, Alert Feed, Secure API Key storage. |
| **Time-Series DB** | ClickHouse / SQLite | Storing historical OHLCV data for backtesting (V3 roadmap). |

---

## ⚙️ Quick Start & Installation

### Option 1: Docker (Recommended)
The easiest way to spin up the entire microservices stack (FastAPI, React, NATS, Redis, PostgreSQL).
```bash
git clone [https://github.com/YOUR_USERNAME/bist_algo_tracker.git](https://github.com/YOUR_USERNAME/bist_algo_tracker.git)
cd bist_algo_tracker
docker compose up --build

##   Option 2: Manual Setup (Local Development)
### 1. Start the Backend (API)

cd bist_algo_tracker
python3 -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
python -m uvicorn api.main_fastapi:app --reload

### 2. Start the Frontend (React Terminal)

cd dashboard-react
npm install
npm run dev
