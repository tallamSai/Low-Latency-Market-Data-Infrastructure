# Low-Latency Market Data Infrastructure

[![C++20](https://img.shields.io/badge/C%2B%2B-20-blue.svg)](https://en.cppreference.com/w/cpp/20)
[![CMake](https://img.shields.io/badge/build-CMake-064F8C.svg)](https://cmake.org/)
[![Docker](https://img.shields.io/badge/runtime-Docker-2496ED.svg)](https://www.docker.com/)
[![Prometheus](https://img.shields.io/badge/metrics-Prometheus-E6522C.svg)](https://prometheus.io/)

A high-performance market data processing system built in modern C++20, focused on asynchronous ingestion, concurrent event processing, durable recording, deterministic replay, backpressure management, and real-time observability.

The system is designed as market-data infrastructure rather than a trading-strategy application, with an emphasis on predictable behavior, bounded resource usage, replayability, and operational visibility.

---

## Overview

Market-data systems must continuously process high-volume event streams while preserving ordering, controlling resource usage, and maintaining enough state to reproduce historical behavior.

MarketPulse addresses these requirements through an asynchronous and concurrent processing pipeline with dedicated ingestion, normalization, publishing, recording, replay, and observability components.

### Core capabilities

* Asynchronous network ingestion
* Concurrent event processing
* Per-symbol ordering guarantees
* Backpressure-aware pipelines
* Memory-mapped market-data recording
* Indexed historical replay
* CRC-protected binary frames
* Deterministic playback
* Configurable replay speed
* Prometheus metrics and Grafana dashboards
* Containerized deployment

---

## Architecture

```text
                   Market / Mock Feed
                           │
                           ▼
                 ┌──────────────────┐
                 │  Boost.Asio I/O  │
                 │     Event Loop   │
                 └────────┬─────────┘
                          │
                          ▼
                 ┌──────────────────┐
                 │ Ingestion Queue  │
                 └────────┬─────────┘
                          │
                          ▼
                 ┌──────────────────┐
                 │ Normalizer       │
                 │ Worker Pool      │
                 └────────┬─────────┘
                          │
                 ┌────────┴────────┐
                 │                 │
                 ▼                 ▼
          ┌──────────────┐   ┌──────────────┐
          │  Publisher   │   │   Recorder   │
          └──────┬───────┘   └──────┬───────┘
                 │                  │
                 ▼                  ▼
          ┌──────────────┐   ┌──────────────┐
          │  WebSocket   │   │ MDF / IDX    │
          │   Clients    │   │    Files     │
          └──────────────┘   └──────┬───────┘
                                    │
                                    ▼
                            ┌──────────────┐
                            │ Replay Engine│
                            └──────────────┘

                       ┌─────────────────────┐
                       │ Prometheus / Grafana│
                       └─────────────────────┘
```

The pipeline separates network I/O from downstream processing so that ingestion, normalization, publication, persistence, and replay can evolve independently.

---

## Key Components

### Asynchronous Ingestion

MarketPulse uses **Boost.Asio** for asynchronous network I/O and event-driven ingestion.

The ingestion layer is responsible for accepting incoming market events and transferring them into the processing pipeline without coupling network activity to downstream consumers.

### Concurrent Processing

Pipeline stages communicate through concurrent queues to allow ingestion and processing work to proceed independently.

Per-symbol ordering is preserved while allowing unrelated symbols to be processed concurrently.

### Backpressure

The processing pipeline uses bounded queues and producer throttling to control behavior when downstream consumers temporarily fall behind.

This provides:

* bounded memory usage
* controlled burst handling
* predictable queue growth
* reduced risk of cascading overload

Backpressure is treated as part of the system design rather than as an exceptional failure condition.

---

## Recording & Replay

MarketPulse provides a persistent event-recording layer for reproducing market-data streams.

### Memory-Mapped Storage

Market events are recorded using memory-mapped files.

This approach provides:

* efficient sequential writes
* reduced system-call overhead
* OS-managed page caching
* efficient access during replay

The implementation also explicitly considers durability and crash-recovery tradeoffs associated with mapped storage.

### Indexed Replay

Recorded data can be replayed from indexed files without requiring the original live feed.

Replay supports configurable playback rates, allowing the same dataset to be used for:

* development
* debugging
* performance testing
* deterministic system validation
* downstream strategy research

### Binary Frame Format

Recorded events use a structured binary frame:

```cpp
struct FrameHeader {
    uint32_t magic;
    uint16_t version;
    uint16_t msg_type;
    uint32_t body_len;
    uint32_t crc32;
};
```

The framing layer provides:

* message identification
* versioning
* payload length validation
* corruption detection
* safer replay of partially damaged files

---

## Concurrency Model

The system separates independent pipeline stages and uses concurrent queues for event transport.

```text
Network I/O
     │
     ▼
Ingestion
     │
     ▼
Queue
     │
     ▼
Normalization Workers
     │
 ┌───┴───────────┐
 ▼               ▼
Publisher     Recorder
```

This design avoids coupling ingestion throughput directly to the execution speed of downstream components.

The implementation also considers synchronization overhead, queue contention, cache behavior, and ordering requirements when scaling the pipeline.

---

## Why Boost.Asio?

Several networking approaches were considered, including blocking sockets with thread pools and Linux-specific interfaces.

Boost.Asio was selected because it provides:

* asynchronous I/O
* event-driven networking
* portable abstractions
* integration with modern C++
* explicit control over connection and event handling

The abstraction keeps the networking layer maintainable while still allowing performance-sensitive behavior to be controlled where required.

---

## Why Concurrent Queues?

Pipeline stages use `moodycamel::ConcurrentQueue` for concurrent event transfer.

Compared with a simple mutex-protected queue, concurrent queues reduce explicit lock contention between producers and consumers and allow multiple pipeline stages to progress independently.

The queue layer also provides a natural boundary for:

* burst absorption
* backpressure
* pipeline isolation
* instrumentation

---

## Backpressure Strategy

A market-data consumer cannot always be assumed to process events as quickly as they arrive.

MarketPulse therefore treats queue saturation as a first-class operational state.

```text
Normal Load
     │
     ▼
Producer → Queue → Consumer
                │
                ▼
          Queue Saturation
                │
                ▼
        Producer Throttling
```

The bounded design prevents unbounded queue growth and makes resource usage easier to reason about during traffic bursts.

---

## Observability

The system exposes runtime metrics through **Prometheus** and provides dashboards through **Grafana**.

### Monitored signals

* ingestion rate
* publication rate
* replay rate
* queue depth
* producer throttling
* backpressure state
* processing latency
* dropped frames
* pipeline health

Latency instrumentation includes percentile-based measurements such as P50 and P99 to make tail behavior visible rather than relying only on average latency.

---

## Performance

A dedicated benchmark harness is included to evaluate the complete processing pipeline under concurrent workloads.

### Benchmark environment

* C++20
* Boost.Asio
* ConcurrentQueue
* multi-threaded processing
* containerized runtime

A representative benchmark run processed **500K+ messages** with observed throughput in the **40K+ messages/sec range** without dropped frames.

Benchmark results can vary with CPU architecture, workload configuration, thread scheduling, compiler settings, and container environment. Results should therefore be interpreted together with the benchmark configuration rather than as a universal throughput limit.

---

## Deployment

The complete environment can be started with Docker Compose.

Services include:

| Service          | Purpose                           |
| ---------------- | --------------------------------- |
| Market Data Core | Ingestion and processing pipeline |
| Control API      | Runtime control and configuration |
| Web Dashboard    | Monitoring interface              |
| Prometheus       | Metrics collection                |
| Grafana          | Metrics visualization             |

The containerized setup provides reproducible local environments and simplifies development across the different system components.

---

## Project Structure

```text
MarketPulse/
├── src/
│   ├── common/
│   ├── feed/
│   ├── normalize/
│   ├── publisher/
│   ├── recorder/
│   ├── replay/
│   ├── ctrl/
│   └── main_core.cpp
│
├── benchmarks/
│
├── ui/
│
├── infra/
│   ├── prometheus/
│   └── grafana/
│
├── docs/
│
├── docker-compose.yml
├── CMakeLists.txt
└── config.json
```

---

## Quick Start

### Docker

```bash
git clone https://github.com/tallamSai/Low-Latency-Market-Data-Infrastructure.git
cd Low-Latency-Market-Data-Infrastructure

docker compose up -d
```

### Local Build

```bash
mkdir build
cd build

cmake ..
cmake --build . --config Release
```

### Run

```bash
./md_core_main ../config.json
```

### Local Services

| Service     | URL                   |
| ----------- | --------------------- |
| Dashboard   | http://localhost:3000 |
| Prometheus  | http://localhost:9090 |
| Grafana     | http://localhost:3001 |
| Control API | http://localhost:8080 |

---

## Design Principles

### Predictable Resource Usage

Bounded queues and controlled producer behavior keep memory usage predictable during bursty workloads.

### Separation of Concerns

Networking, normalization, publication, persistence, replay, and monitoring are isolated into independent components.

### Deterministic Replay

Persisted market events can be replayed under controlled conditions, making debugging and system validation reproducible.

### Observability by Design

Throughput, queue behavior, latency, and backpressure are exposed as runtime signals rather than treated as post-deployment diagnostics.

### Performance with Correctness

Performance improvements are evaluated alongside ordering, data integrity, recovery behavior, and system stability.

---

## Engineering Considerations

The project explores several systems-level tradeoffs:

* asynchronous I/O versus blocking network models
* concurrent queues versus lock-based synchronization
* throughput versus tail latency
* bounded buffering versus burst tolerance
* memory-mapped storage versus durability requirements
* concurrency versus ordering guarantees
* performance optimization versus implementation complexity

These tradeoffs are evaluated through profiling, benchmarks, and observable runtime behavior.

---

## Future Work

Planned areas of development include:

* exchange-specific market-data connectors
* UDP multicast ingestion
* packet-gap detection and recovery
* zero-copy parsing
* CPU affinity and cache-aware optimization
* stronger persistence and crash recovery
* expanded latency instrumentation
* distributed replay
* additional market-data sources

---

## Engineering Focus

MarketPulse is intended as a systems-engineering exploration of the infrastructure behind real-time financial data.

The project focuses on:

```text
Asynchronous Networking
        +
Concurrent Processing
        +
Bounded Pipelines
        +
Persistent Storage
        +
Deterministic Replay
        +
Runtime Observability
```

The result is a modular C++ platform for experimenting with the performance, correctness, and operational characteristics of real-time market-data systems.
