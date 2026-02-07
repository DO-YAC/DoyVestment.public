# DoyVestment

**An end-to-end algorithmic trading platform combining live market data, AI-powered signal generation, strategy optimization, and automated broker execution.**

---

## About

DoyVestment is a full-stack quantitative trading system built to ingest real-time market data and generate trade signals through an AI and rule-based strategies, and execute orders with a live broker - all with xx ms latency.

The platform spans multiple repositories covering every stage of the trading lifecycle like data collection, model training, strategy optimization, signal generation, and order execution.

---

## Architecture

![Architecture Diagram](draw.io/Doyvestment.svg)

---

## Platform Components

### Signalhandler

The SignalHandler is the core engine that orchestrates the entire trading pipeline. It continuously reads chart data from a shared memory filesystem via our Memory Communication Service (MCS), enriches it with technical analysis indicators and feeds the result into DoyLib for evaluation. Once DoyLib returns a decision (BUY, SELL, or HOLD) the SignalHandler writes it back through the Memory Communication Service (MCS) for MetaTrader 5 to execute.

The SignalHandler also exposes a private REST API for manual control. Trades can be opened or closed via HTTP, and chart data can be pushed directly to MongoDB through a dedicated endpoint. This is how we save historical candle data for AI model training.

### DoyLib - Multi-Strategy Trade Decision Engine

DoyLib is the shared core library that powers trade decisions across the entire platform. It takes in candle data enriched with technical features and runs it through a modular strategy engine where multiple decision modules - AI-based, rule-based, or both - each independently evaluate whether to buy, sell, or hold.

The engine supports two voting modes: consensus, where all modules must agree before a trade is placed, and first-match, where the first clear signal wins. Modules that crash or return garbage are silently filtered out, so a single bad module never takes down the pipeline.

The AI module runs ONNX inference with GPU acceleration. It only acts when it's confident enough.

New strategy modules can be registered without restructuring the whole System. Everything is configured through environment variables, so swapping between strategies, toggling the AI on or off, or adjusting confidence thresholds is a container restart away without code changes or redeployment.

### MetaTrader 5

The MetaTrader 5 instance runs fully containerized with Docker. The container is built on a Debian base image with KasmVNC, Wine, and all required dependencies pre-installed, so the entire broker terminal launches automatically with `docker compose up` -- no manual Windows setup needed. The DLLs and Expert Advisors are mounted into the container's MQL5 Libraries directory via volume binds, and a shared tmpfs volume connects MT5 to the SignalHandler for memory-mapped file communication. The terminal is accessible remotely through a browser-based VNC session routed through Traefik.

### Memory Bridge (C++ DLLs)

Two C++ libraries are handling the memory-mapped file protocol for MetaTrader 5.

#### ChartMemoryPrinter (CMP)

Every time a new tick comes in on MetaTrader 5, an Expert Advisor collects the current candle's OHLCV data, formats it as JSON, and passes it to the ChartMemoryPrinter DLL. The DLL is a small C++ library with a single exported function that writes the JSON into a shared memory file using an atomic write pattern. The Data goes to a temporary file first, then gets swapped into place with a single rename operation. This means the SignalHandler on the other side never reads a half-written candle. The whole path from tick to readable shared memory takes microseconds.

#### SignalMemoryReader (SMR)

The SignalMemoryReader is the other half of the Memory Bridge and handles the reverse direction. Once the SignalHandler has made a trade decision, it writes the result as JSON into a shared memory file with a marker byte set to 1. A Expert Advisor inside MetaTrader 5 polls this file every 5 milliseconds through the SignalMemoryReader DLL. The DLL reads the marker byte, and if new data is present, it converts it to JSON, returns it to the Expert Advisor, and flips the marker back to 0 so the same signal is never read twice. The Expert Advisor then parses the action and ticker from the JSON and executes the buy, sell, or close order on the broker.

### DoyAi -- Neural Network Training Pipeline --> TODO

### DoyStratOptimizer -- Genetic Algorithm Optimization --> TODO

### MT5 Simulator --> TODO

---

## Technology Stack --> TODO

Later Automated with GH WORKFLOW
