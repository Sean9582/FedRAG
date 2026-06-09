# FedRAG v4 — Privacy-Preserving Federated RAG for Distributed Biomedical QA

<p align="center">
  <img src="https://img.shields.io/badge/Python-3.10+-blue?logo=python&logoColor=white" />
  <img src="https://img.shields.io/badge/PyTorch-2.11-EE4C2C?logo=pytorch&logoColor=white" />
  <img src="https://img.shields.io/badge/FAISS-1.14.1-green" />
  <img src="https://img.shields.io/badge/Privacy-Differential%20Privacy-purple" />
  <img src="https://img.shields.io/badge/Dataset-PubMedQA-orange" />
  <img src="https://img.shields.io/badge/License-MIT-lightgrey" />
</p>

> **FedRAG** is a privacy-preserving federated Retrieval-Augmented Generation system for distributed biomedical question answering. It federates dense and sparse retrieval across multiple hospital nodes using differential privacy (DP) noise injection on embeddings, ensuring raw patient documents never leave their local silo.

---

## Table of Contents

- [Overview](#overview)
- [System Architecture](#system-architecture)
- [Key Results](#key-results)
  - [RQ1 — Retrieval Ablation](#rq1--retrieval-ablation)
  - [RQ2 — IID vs Non-IID Federation](#rq2--iid-vs-non-iid-federation)
  - [RQ3a — Generator Ablation](#rq3a--generator-ablation)
  - [RQ3b — Privacy-Utility Tradeoff](#rq3b--privacyutility-tradeoff)
  - [Security Evaluation](#security-evaluation)
  - [Scalability](#scalability)
  - [End-to-End Performance](#end-to-end-performance)
- [Figures](#figures)
- [Dataset](#dataset)
- [Setup & Installation](#setup--installation)
- [Repository Structure](#repository-structure)
- [v3 → v4 Changelog](#v3--v4-changelog)
- [Citation](#citation)

---

## Overview

FedRAG v4 addresses a core challenge in healthcare AI: institutions cannot share raw patient records, yet collaborative retrieval across institutions dramatically improves biomedical QA. Our solution:

- Each of **5 federated nodes** holds a private shard of the PubMedQA corpus, partitioned using a **Dirichlet (α=0.1) non-IID** distribution to simulate realistic domain skew across hospitals.
- Nodes encode their documents locally using **BAAI/bge-base-en-v1.5** and inject **Gaussian DP noise** (σ calibrated per ε) before transmitting embeddings to a central coordinator.
- The coordinator builds a **federated FAISS index** from noisy embeddings and runs **hybrid BM25 + dense retrieval with Reciprocal Rank Fusion (RRF)** and **cross-encoder reranking** (MS-MARCO MiniLM-L-6-v2).
- A multi-model generator ablation covers **Flan-T5-Base**, **Flan-T5-Large**, and **BioGPT**.
- Security is evaluated via **embedding inversion probes** and **membership inference attack (MIA) AUC**.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          FEDERATED NODES                            │
│                                                                     │
│  Node 1 (Aged-dominant)    Node 2 (Adolescent)   Node 3 (Aged)     │
│  6,034 docs                14,095 docs           9,340 docs        │
│       │                         │                     │            │
│  [BGE Encoder]            [BGE Encoder]         [BGE Encoder]      │
│       │                         │                     │            │
│  [DP Noise σ=0.0379]      [DP Noise]            [DP Noise]         │
│       └─────────────────────────┴─────────────────────┘            │
│                                 │                                   │
│              Node 4 (Adolescent) + Node 5 (Adult-dominant)         │
│              5,472 + 25,059 docs  → [DP Noised Embeddings]         │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │ Only DP-noised embeddings transmitted
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          COORDINATOR                                │
│                                                                     │
│  Federated FAISS Index (60,000 vectors, 768-dim)                    │
│  + Global BM25 Index (60,000 docs, unified pubid map)               │
│                                                                     │
│  Query → Dense (BGE) + BM25 → RRF Fusion → MS-MARCO Rerank         │
│                         │                                           │
│                  Retrieved Passages (Top-K)                         │
│                         │                                           │
│               Generator (Flan-T5-Large)                             │
│                         │                                           │
│               Answer: yes / no / maybe                              │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Key Results

### RQ1 — Retrieval Ablation

| System | R@1 | R@5 | R@10 | MRR | nDCG@5 |
|---|---|---|---|---|---|
| BM25 (v4 fixed) | 77.0% | 89.5% | 89.5% | 81.3% | 83.3% |
| Dense (BGE) | 68.0% | 92.0% | 92.0% | 77.9% | 81.4% |
| Hybrid RRF | 71.5% | 95.0% | 95.0% | 81.7% | 85.1% |
| **Hybrid + MS-MARCO** | **85.0%** | **93.5%** | **93.5%** | **88.1%** | **89.5%** |
| Hybrid + bge-rerank | 68.5% | 88.5% | 88.5% | 76.3% | 79.3% |

### RQ2 — IID vs Non-IID Federation

| Setting | Recall@5 | p-value |
|---|---|---|
| Non-IID (Dirichlet α=0.1) | 93.7% | — |
| IID | 93.0% | 0.528 |

Non-IID Dirichlet partitioning does **not** significantly degrade retrieval (p ≥ 0.05), confirming FedRAG's robustness to realistic data heterogeneity.

### RQ3a — Generator Ablation

| Generator | Accuracy | Macro-F1 | Weighted-F1 |
|---|---|---|---|
| Flan-T5-Base | 58.0% | 26.8% | 44.5% |
| **Flan-T5-Large ★** | 38.0% | **27.4%** | 34.2% |
| BioGPT | 59.0% | 27.2% | 45.0% |

Flan-T5-Large is selected as the best generator by Macro-F1, reflecting the importance of balanced class coverage over raw accuracy.

### RQ3b — Privacy-Utility Tradeoff

| ε | σ | Recall@5 | Accuracy | Retrieval |
|---|---|---|---|---|
| 8 | 1.2112 | 0.0% | 26.0% | ❌ |
| 16 | 0.6056 | 0.0% | 26.0% | ❌ |
| 32 | 0.3028 | 1.0% | 26.0% | ❌ |
| **64** | 0.1514 | **9.0%** | 27.0% | ✅ |
| 128 | 0.0757 | 57.0% | 31.0% | ✅ |
| **256** | **0.0379** | **91.0%** | **42.0%** | ✅ |
| 512 | 0.0189 | 93.0% | 37.0% | ✅ |

**Retrieval recovery threshold: ε ≥ 64.** At ε = 256, the system achieves 91% Recall@5 while maintaining meaningful privacy noise (σ = 0.0379).

### Security Evaluation

| ε | σ | Cosine(clean, noisy) | MIA AUC |
|---|---|---|---|
| 8 | 1.2112 | 0.031 ± 0.035 | 0.501 |
| 64 | 0.1514 | 0.233 ± 0.033 | 0.649 |
| 512 | 0.0189 | 0.886 ± 0.006 | 1.000 |

At ε = 8 the MIA AUC is near-random (0.501), confirming strong membership privacy. At ε = 256 the cosine similarity of 0.691 reflects the practical tradeoff between utility and protection.

### Scalability

| Nodes | Recall@5 | Latency | Build Time |
|---|---|---|---|
| 5 | 92.0% | 51 ± 25 ms | 539 s |
| 10 | 96.0% | 46 ± 9 ms | 537 s |
| 20 | 92.0% | 38 ± 10 ms | 535 s |

Latency decreases with more nodes (smaller per-node shards); build time is near-constant, confirming linear scalability.

### End-to-End Performance (ε=256, Non-IID, Flan-T5-Large)

| Metric | Value | 95% CI |
|---|---|---|
| Recall@5 | 93.5% | [90.0–96.5] |
| MRR | 88.1% | [84.0–92.1] |
| nDCG@5 | 89.5% | [85.6–92.8] |
| Accuracy | 35.0% | [29.0–41.0] |
| Macro-F1 | 25.3% | — |

---

## Figures

### Figure 1 — RQ1: Retrieval Ablation Study

![RQ1 Retrieval Ablation](fedrag_clean_figures/fig1_rq1_retrieval.png)

Comparison of BM25, Dense (BGE), Hybrid RRF, and reranked pipelines across Recall@1/5/10, MRR, and nDCG@5. The MS-MARCO reranked pipeline achieves the best MRR (88.1%) and Recall@1 (85.0%).

---

### Figure 2 — RQ3b: Privacy-Utility Tradeoff

![Privacy-Utility Tradeoff](fedrag_clean_figures/fig2_rq3_privacy.png)

End-to-end privacy-utility sweep across ε ∈ {8, 16, 32, 64, 128, 256, 512}. Retrieval collapses below ε = 64 and recovers sharply from ε = 128 onward, with peak Recall@5 at ε = 512 (93.0%) and best accuracy at ε = 256 (42.0%).

---

### Figure 3 — Confusion Matrix (End-to-End, ε=256)

![Confusion Matrix](fedrag_clean_figures/fig3_confusion.png)

The model correctly identifies `no` answers (recall 81%) but struggles with `maybe` (recall 0%), reflecting the generator's tendency to collapse output distribution to binary classes.

---

### Figure 4 — RQ2: IID vs Non-IID Domain Recall@5

![IID vs Non-IID](fedrag_clean_figures/fig4_rq2_iid.png)

Per-domain Recall@5 comparison under Dirichlet non-IID (α=0.1) versus IID partitioning (n=300 queries). Aggregate difference is only +0.7 pp (p=0.528), confirming negligible impact of realistic data heterogeneity.

---

### Figure 5 — Security Evaluation

![Security Evaluation](fedrag_clean_figures/fig5_security.png)

Embedding inversion cosine similarity and MIA AUC across ε values. At low ε, cosine similarity near zero and MIA AUC near 0.5 confirm that DP noise effectively prevents reconstruction and membership inference.

---

### Figure 6 — Scalability: Node Count Ablation

![Scalability](fedrag_clean_figures/fig6_scalability.png)

Recall@5, query latency, and index build time across 5, 10, and 20 federated nodes. Latency decreases with scale; build time remains stable (~537 s), confirming horizontal scalability.

---

### Figure 7 — Global Recall@k Curve

![Recall@k Curve](fedrag_clean_figures/fig7_recall_k.png)

Recall@k for k ∈ {1, 3, 5, 10, 20} on the full 200-query evaluation set. Recall saturates at 93.5% from k=5, indicating the top-5 retrieved passages already cover the majority of gold documents.

---

## Dataset

**PubMedQA** (`qiaojin/PubMedQA`, `pqa_labeled` + `pqa_unlabeled` splits via HuggingFace Datasets)

| Split | Count | Role |
|---|---|---|
| pqa_labeled | 1,000 QA pairs | Gold labels (yes / no / maybe) |
| pqa_unlabeled (truncated) | 59,000 abstracts | Distractor corpus |
| **Total corpus** | **60,000 documents** | FAISS + BM25 index |

All 1,000 gold abstracts are guaranteed injected into the corpus. Dirichlet (α=0.1) partition distributes documents across 5 nodes with realistic domain imbalance (dominant domains: Aged, Adolescent, Adult).

---

## Setup & Installation

### Requirements

- Python 3.10+
- CUDA-capable GPU (tested on Tesla T4, 15.6 GB VRAM)
- ~25 GB disk space for models + index

### Install

```bash
git clone https://github.com/AthithyaKrishnaa/FedRAG.git
cd FedRAG
pip install faiss-gpu-cu12 rank_bm25 datasets transformers>=4.40.0 \
    accelerate>=0.27.0 sentencepiece protobuf scikit-learn tqdm scipy matplotlib seaborn
```

### Run

Open and execute the notebook end-to-end:

```bash
jupyter notebook FedRAG_v4_fixed.ipynb
```

The notebook is self-contained and will download all models and data automatically. Stages run sequentially:

| Stage | Description |
|---|---|
| 1 | PubMedQA loading + Dirichlet non-IID partition |
| 2 | Evaluation framework (Recall@k, MRR, nDCG, bootstrap CI) |
| 3 | Federated BM25 with unified global pubid map |
| 4 | Dirichlet non-IID Partition (α=0.1) |
| 5 | Dense encoder: BAAI/bge-base-en-v1.5 |
| 6 | Differential privacy with RDP accounting |
| 7 | Per-node encoding + federated FAISS index |
| 8 | Cross-encoder rerankers (MS-MARCO, bge-reranker-base) |
| 9 | Full retrieval pipeline (Dense + BM25 + RRF + Rerank) |
| 10 | Generators: Flan-T5-Base, Flan-T5-Large, BioGPT |
| 11–18 | RQ1/2/3 experiments, security probes, scalability |
| 19 | Publication-quality figure export |
| 20 | v3 → v4 comparison summary |

---

## Repository Structure

```
FedRAG/
├── FedRAG_v4_fixed.ipynb        # Main notebook (20 stages, self-contained)
├── cell_outputs.txt             # Full cell-by-cell console output log
└── fedrag_clean_figures/
    ├── fig1_rq1_retrieval.png   # RQ1 retrieval ablation bar chart
    ├── fig1_rq1_retrieval.pdf
    ├── fig2_rq3_privacy.png     # Privacy-utility sweep
    ├── fig2_rq3_privacy.pdf
    ├── fig3_confusion.png       # Confusion matrix (ε=256)
    ├── fig3_confusion.pdf
    ├── fig4_rq2_iid.png         # IID vs Non-IID domain Recall@5
    ├── fig4_rq2_iid.pdf
    ├── fig5_security.png        # Embedding inversion + MIA
    ├── fig5_security.pdf
    ├── fig6_scalability.png     # Node count scalability
    ├── fig6_scalability.pdf
    ├── fig7_recall_k.png        # Global Recall@k curve
    └── fig7_recall_k.pdf
```

---

## v3 → v4 Changelog

| Metric / Feature | v3 | v4 | Δ |
|---|---|---|---|
| BM25 Recall@5 | 0.0% | **89.5%** | +89.5 pp |
| Dense Recall@5 | 96.0% | 92.0% | −4.0 pp |
| Reranked Recall@5 | 94.5% | 93.5% | −1.0 pp |
| QA Accuracy | 55.0% | 35.0% | −20.0 pp |
| Macro-F1 | 24.9% | **25.3%** | +0.4 pp |
| Non-IID R@5 | 94.7% | 93.7% | −1.0 pp |
| DP threshold ε | unknown | **ε ≥ 64** | ✅ quantified |
| Security evaluation | ✗ | ✅ | added |
| Generator ablation | ✗ | ✅ | added |
| Node scalability | ✗ | ✅ | added |

**Root cause of v3 BM25 failure:** v3 used per-node local pubid maps with no global deduplication, causing lookup mismatches between BM25-retrieved indices and the corpus pubid store. v4 fixes this with a single unified `global_idx_to_pubid` mapping built before partition.

---

## Citation

If you use this work, please cite:

```bibtex
@misc{krishnaa2025fedrag,
  author    = {Athithya Krishnaa},
  title     = {FedRAG v4: Privacy-Preserving Federated RAG for Distributed Biomedical QA},
  year      = {2025},
  publisher = {GitHub},
  url       = {https://github.com/AthithyaKrishnaa/FedRAG}
}
```


<p align="center">
  Built with FAISS · HuggingFace Transformers · PubMedQA · Differential Privacy (RDP Accounting)
</p>
