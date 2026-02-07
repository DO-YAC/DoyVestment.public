# DoyVestment

**An end-to-end algorithmic trading platform combining live market data, AI-powered signal generation, strategy optimization, and automated broker execution.**

---

## About

DoyVestment is a full-stack quantitative trading system built to ingest real-time market data and generate trade signals through an AI and rule-based strategies, and execute orders with a live broker - all with xx ms latency.

The platform spans multiple repositories covering every stage of the trading lifecycle like data collection, model training, strategy optimization, signal generation, and order execution.

---

## Architecture

DRAWIO DIAGRAMM

---

## Platform Components

### SignalHandler

The SignalHandler is the core engine that orchestrates the entire trading pipeline. It Receives candle data via memory-mapped files (MT5, SignalMemoryReader), runs it through the strategy engine (DoyLib), and returns a trade decision (BUY, SELL, HOLD).

- ASP.NET Core 9.0 API with MongoDB persistence and Redis caching
- Memory-mapped file protocol for zero-copy IPC with MetaTrader 5
- Multi-symbol, multi-timeframe support (H1, D1, etc.)
- Hot-swappable strategy modules at runtime

### DoyLib - Multi-Strategy Trade Decision Engine

DoyLib is the shared core library that powers trade decisions across the entire platform. It takes in candle data enriched with technical features and runs it through a modular strategy engine where multiple decision modules - AI-based, rule-based, or both - each independently evaluate whether to buy, sell, or hold.

The engine supports two voting modes: consensus, where all modules must agree before a trade is placed, and first-match, where the first clear signal wins. Modules that crash or return garbage are silently filtered out, so a single bad module never takes down the pipeline.

The AI module runs ONNX inference with GPU acceleration. It only acts when it's confident enough.

New strategy modules can be registered without restructuring the whole System. Everything is configured through environment variables, so swapping between strategies, toggling the AI on or off, or adjusting confidence thresholds is a container restart away without code changes or redeployment.

### MetaTrader 5

The MetaTrader 5 instance runs fully containerized with Docker. The container is built on a Debian base image with KasmVNC, Wine, and all required dependencies pre-installed, so the entire broker terminal launches automatically with `docker compose up` -- no manual Windows setup needed. The DLLs and Expert Advisors are mounted into the container's MQL5 Libraries directory via volume binds, and a shared tmpfs volume connects MT5 to the SignalHandler for memory-mapped file communication. The terminal is accessible remotely through a browser-based VNC session routed through Traefik.

### Memory Bridge (C++ DLLs)

#### ChartMemoryPrinter

Every time a new tick comes in on MetaTrader 5, an Expert Advisor collects the current candle's OHLCV data, formats it as JSON, and passes it to the ChartMemoryPrinter DLL. The DLL is a small C++ library with a single exported function that writes the JSON into a shared memory file using an atomic write pattern. The Data goes to a temporary file first, then gets swapped into place with a single rename operation. This means the SignalHandler on the other side never reads a half-written candle. The whole path from tick to readable shared memory takes microseconds.

#### SignalMemoryReader

The SignalMemoryReader is the other half of the Memory Bridge and handles the reverse direction. Once the SignalHandler has made a trade decision, it writes the result as JSON into a shared memory file with a marker byte set to 1. A Expert Advisor inside MetaTrader 5 polls this file every 5 milliseconds through the SignalMemoryReader DLL. The DLL reads the marker byte, and if new data is present, it converts it to JSON, returns it to the Expert Advisor, and flips the marker back to 0 so the same signal is never read twice. The Expert Advisor then parses the action and ticker from the JSON and executes the buy, sell, or close order on the broker.

### DoyAi -- Neural Network Training Pipeline

A PyTorch-based pipeline that trains LSTM models on historical market data fetched directly from MongoDB, then exports trained models to ONNX format for real-time inference.

- LSTM sequence-to-prediction architecture with configurable depth and hidden dimensions
- Full data pipeline: MongoDB ingestion, normalization (Min-Max / Standard), configurable lookback windows, 70/15/15 train/val/test splits
- Experiment tracking with Weights & Biases
- Hydra-based configuration management for reproducible experiments
- Multi-GPU support via CUDA

### DoyStratOptimizer -- Genetic Algorithm Optimization

A genetic algorithm suite that searches the parameter space of any trading strategy to find optimal configurations.

- Multi-generational genetic algorithm with elitism (top-10 carry-over) and configurable mutation
- Parallel fitness evaluation across all CPU cores
- Loads compiled strategy DLLs at runtime via reflection
- WPF desktop UI for real-time generation monitoring and parameter visualization
- Console mode for headless batch optimization

### MT5 Simulator

A cross-platform MAUI desktop app that simulates market tick generation, enabling full pipeline testing without a live broker connection.

- Cross-platform (macOS + Windows) with .NET 10.0 and MAUI
- Configurable tick generation rate and simulation parameters
- Outputs to shared memory-mapped files, identical to the live pipeline
- Session-based state management with persistent configuration

### Memory Bridge (C++ DLLs)

Two low-level C++ libraries handling the memory-mapped file protocol between MetaTrader 5 and the rest of the platform.

- **SignalMemoryReader**: Reads trade signals from shared memory and forwards them via HTTP to the Signal Handler API
- **ChartMemoryPrinter**: Serializes chart/candle data from MT5 into shared memory with atomic write operations
- UTF-16/UTF-8 encoding conversion for cross-platform compatibility
- Built-in logging and error tracking

---

## Feature Engineering

The platform computes a 16-dimensional feature vector for each candle, including:

| Category       | Indicators                           |
| -------------- | ------------------------------------ |
| **Trend**      | SMA, EMA, MACD                       |
| **Momentum**   | RSI                                  |
| **Volatility** | Bollinger Bands, ATR                 |
| **Volume**     | Volume analysis                      |
| **Temporal**   | Intraday cycle, day-of-week encoding |

These features feed both the AI module (as ONNX model inputs) and rule-based strategy modules.

---

## Infrastructure

| Component           | Technology                            |
| ------------------- | ------------------------------------- |
| Backend API         | ASP.NET Core 9.0 (C#)                 |
| ML Training         | PyTorch (Python)                      |
| ML Inference        | ONNX Runtime 1.23.2                   |
| Data Storage        | MongoDB                               |
| Caching             | Redis                                 |
| Broker Integration  | MetaTrader 5 (MQL5)                   |
| IPC                 | Memory-Mapped Files                   |
| Containerization    | Docker & Docker Compose               |
| Reverse Proxy       | Traefik                               |
| Authentication      | Authelia                              |
| Desktop UI          | WPF / .NET MAUI                       |
| Experiment Tracking | Weights & Biases                      |
| Configuration       | Hydra (Python), Environment Variables |
| Indicators Library  | Skender Stock Indicators              |

---

## Key Technical Highlights

- **Hybrid AI + Rule-Based Ensemble**: Combines neural network inference with classical strategy modules through a consensus voting system, balancing adaptability with interpretability.
- **Sub-Second Latency Pipeline**: Memory-mapped files provide zero-copy IPC between the broker and the signal engine, avoiding network overhead entirely.
- **Full ML Lifecycle**: Training in PyTorch, export to ONNX, inference with GPU acceleration in production -- a complete MLOps pipeline.
- **Automated Parameter Search**: Genetic algorithm optimization discovers optimal strategy parameters without manual tuning, evaluating thousands of configurations in parallel.
- **Production-Grade Deployment**: Dockerized microservice architecture with Traefik routing, Authelia authentication, and persistent data stores.
- **Cross-Platform**: Runs on macOS, Windows, and Linux. MT5 integration on non-Windows hosts achieved via Docker + Wine.

---

## Technology Stack

Later Automated with GH WORKFLOW

**Languages**: C#, Python, C++, MQL5
**Frameworks**: ASP.NET Core, PyTorch, .NET MAUI, WPF
**Databases**: MongoDB, Redis
**ML/AI**: ONNX Runtime, LSTM Networks, Genetic Algorithms
**DevOps**: Docker, Docker Compose, Traefik, Authelia
**Tools**: Weights & Biases, Hydra, Skender Stock Indicators
