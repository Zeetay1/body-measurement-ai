# Building a Self-Improving Multimodal AI System for Real-Time Body Measurement Estimation

## The Problem

The system was designed to estimate full-body anthropometric measurements using a combination of **user-provided media inputs and manual structured inputs**, with the goal of enabling reliable body measurement extraction for downstream personalization and recommendation use cases.

The core challenge was not just prediction accuracy, but producing **physically consistent, reliable, and usable measurement outputs** under real-world conditions.

In production, the main issues were:

* High dropout during scanning sessions due to confusing guidance
* Noisy or incomplete user inputs (pose variation, lighting, occlusion)
* Edge-case failures on underrepresented body types and environments
* Silent degradation of model performance on rare scenarios

---

## System Architecture Overview

The system is a **multimodal ML + agentic feedback architecture** composed of three tightly coupled layers:

1. **Multimodal inference pipeline (vision + structured inputs)**
2. **Real-time guidance agent (RAG + state-aware prompting)**
3. **Self-improving feedback loop (evaluation + retraining + prompt iteration)**

All three layers share a unified event and logging schema, allowing downstream learning from both model and user interaction signals.

---

## 1. Multimodal Input and Inference Layer

### Inputs

The system uses:

* **User media inputs** (mobile camera images during scan session)
* **Structured inputs** (partial measurements and demographic attributes)

These are fused into a shared representation space used for inference.

---

### Pose and Landmark Extraction

A dual-model approach is used for pose estimation:

* A lightweight on-device pose estimator for real-time responsiveness
* A higher-accuracy cloud-based model for refinement on selected frames

Outputs are combined using a **confidence-weighted selection strategy**, where frame-level confidence scores determine final landmark selection.

---

### Landmark Refinement Model

A CNN-based refinement model improves raw pose outputs by correcting:

* Occluded joints
* Motion blur artifacts
* Unstable keypoints in low-light or angled captures

Each landmark is assigned a confidence score, which is used downstream by both the measurement engine and the guidance agent.

---

### Depth-Based Measurement Estimation

Body measurements are derived using:

* Landmark geometry
* Monocular depth estimation
* Camera calibration priors (device-specific approximations)

Final outputs are normalized to ensure consistent proportional relationships across measurements.

---

## 2. Real-Time Guidance Agent (RAG System)

### Purpose

The agent improves scan completion rates by providing **adaptive, state-aware instructions during the scanning session**.

---

### Retrieval Layer

Each user session maintains a lightweight memory store containing:

* Past scan summaries
* Common failure patterns
* Historical measurement trends
* Previous guidance interactions

At each step, the system retrieves top-k relevant memory items based on:

* Current scan state embedding
* Recent user actions
* Detected failure signals

---

### Context Assembly

Retrieved memory is filtered using:

* Recency weighting
* Relevance scoring
* Token budget constraints

Final prompt structure:

* System instructions
* Current scan state
* Retrieved user memory
* Live inference signals (confidence scores, anomalies)

---

### Generation and Tooling

The LLM agent can:

* Generate real-time guidance messages
* Query live inference signals
* Log interaction events
* Trigger fallback flows when user confusion is detected

---

### Memory Design (Key Decision)

A naive “store everything” approach caused:

* Context dilution
* Hallucinated personalization
* Degraded guidance relevance

This was replaced with a **tiered memory system**:

* **Hot memory:** last 3 sessions (fully structured, verbatim)
* **Warm memory:** sessions 4–12 (compressed behavioral patterns)
* **Cold memory:** aggregated historical summaries, only retained if statistically significant or anomalous

This significantly improved personalization consistency and reduced context drift.

---

## 3. Self-Improving Feedback Loop

The system continuously learns from production data through a structured pipeline.

---

### Data Logging

Each session generates structured logs including:

* Pose sequences
* Landmark confidence scores
* Model outputs
* User interactions (retry, skip, dropout)
* Agent guidance history
* Final measurement outputs

---

### Session Classification

Sessions are classified into:

* **Clean:** high-confidence, consistent outputs, no anomaly flags
* **Review:** ambiguous or edge-case sessions requiring inspection
* **Exclude:** corrupted inputs, extreme noise, or invalid capture conditions

A distributional shift check is applied to incoming batches to detect dataset drift before inclusion in training.

---

### Model Retraining

The CNN landmark refinement model is retrained on a scheduled cycle using only **clean data**.

Every candidate model must pass:

1. A fixed golden dataset
2. A recent production sample

Metrics include:

* Landmark-level error
* Measurement RMSE
* Consistency validation across correlated measurements
* Anomaly detection precision/recall

A model is rejected if it regresses on either dataset slice.

---

### Agent Learning Loop

Agent improvement is driven by interaction mining:

Key signals:

* Guidance ignored or immediately contradicted
* High dropout immediately following specific instructions
* Retrieval-context mismatches
* Repeated user confusion patterns

These are converted into:

* Updated golden datasets
* Prompt refinements
* Retrieval tuning improvements

Over time, this built a growing dataset of real-world guidance failures used for regression testing.

---

## Evaluation Infrastructure

Evaluation is treated as a first-class system component.

---

### Inference Model Evaluation

* MAE / RMSE on measurements
* Per-landmark accuracy
* Consistency constraints across body dimensions
* Anomaly detection performance

---

### Agent Evaluation (LLM-as-Judge)

Scored across independent dimensions:

* Groundedness (state alignment)
* Relevance (task appropriateness)
* Clarity (instruction usability)
* Personalization (use of memory context)

No composite score is used to avoid masking failure modes.

---

### Regression Control

Every change to:

* model weights
* prompts
* retrieval logic
* thresholds

must pass full eval suite before deployment.

This prevented multiple production regressions during iterative development.

---

## Key Engineering Lessons

### 1. Monitoring must precede learning loops

Without robust monitoring, silent drift could not be detected reliably.

---

### 2. Data contracts are critical in production ML

Schema evolution in logging pipelines caused ingestion instability. Formalizing data contracts early would have prevented multiple downstream failures.

---

### 3. Evaluation design matters more than model iteration

Many performance gains came from better evaluation design and data filtering rather than architectural changes.

---

### 4. Multimodal systems fail at the edges, not the center

Most production issues emerged from edge cases: lighting, pose variation, and device heterogeneity rather than average-case performance.

---

## Summary

The system demonstrates how multimodal perception, agentic reasoning, and feedback-driven learning can be combined into a production-grade self-improving architecture.

The key insight throughout the work is that performance in real-world AI systems is determined less by model selection and more by:

* data quality
* evaluation rigor
* system design
* feedback loop structure
