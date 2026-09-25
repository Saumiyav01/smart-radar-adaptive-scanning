# Adaptive Radar Scan Strategy

An intelligent adaptive radar scanning framework for Electronic Warfare that combines temporal pulse prediction, Bayesian state estimation, rendezvous prediction, and adaptive scan scheduling.

## Overview

Electronic Warfare receivers must observe a wide frequency spectrum while operating with limited instantaneous bandwidth and finite observation time. A fixed scanning strategy can waste dwell time on inactive or predictable emitters while missing important short-duration or time-sensitive signals.

This project develops a closed-loop adaptive scanning framework that uses observed Pulse Descriptor Words (PDWs) to predict future emitter activity and dynamically prioritize which emitter or frequency region should be observed next.

The system continuously follows:

**Observe → Predict → Estimate → Score → Schedule → Scan → Learn**

---

## System Architecture

```text
                  RAW INTERLEAVED PDWs
                           │
                           ▼
                ┌──────────────────────┐
                │ PDW Association /    │
                │ Emitter Tracking     │
                └──────────┬───────────┘
                           │
                           ▼
                ┌──────────────────────┐
                │ Temporal Feature     │
                │ Engineering          │
                └──────────┬───────────┘
                           │
                           ▼
                ┌──────────────────────┐
                │ Timing GRU           │
                │ Next ΔToA Prediction │
                └──────────┬───────────┘
                           │
                           ▼
                ┌──────────────────────┐
                │ Bayesian State       │
                │ Estimation           │
                └──────────┬───────────┘
                           │
                           ▼
                ┌──────────────────────┐
                │ Priority / Risk      │
                │ Scoring              │
                └──────────┬───────────┘
                           │
                           ▼
                ┌──────────────────────┐
                │ Rendezvous Prediction│
                │ + Observation Window │
                └──────────┬───────────┘
                           │
                           ▼
                ┌──────────────────────┐
                │ Thompson Sampling    │
                │ Adaptive Scheduler   │
                └──────────┬───────────┘
                           │
                           ▼
                       SCAN ACTION
                           │
                           ▼
                       HIT / MISS
                       ↙       ↘
                     HIT       MISS
                      │           │
                   α ← α+1     β ← β+1
                       \         /
                        \       /
                         ▼     ▼
                       Updated
                    Bayesian State
                           │
                           └──────────► Feedback Loop
