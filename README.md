# PRISM
### Post-quantum Resilient Image Steganography with Multi-domain Embedding

[![Python](https://img.shields.io/badge/Python-3.8%2B-blue?logo=python)](https://www.python.org/)
[![PyTorch](https://img.shields.io/badge/PyTorch-2.x-EE4C2C?logo=pytorch)](https://pytorch.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Published: COMSIA 2025](https://img.shields.io/badge/Related%20Work-COMSIA%202025-orange)]()
[![Platform: Google Colab](https://img.shields.io/badge/Platform-Google%20Colab-F9AB00?logo=googlecolab)](https://colab.research.google.com/)

> **PRISM** is a cryptographically secure, visually imperceptible image-in-image steganography framework that combines post-quantum encryption, deep learning–based texture analysis, and adaptive multi-domain embedding.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Architecture](#architecture)
- [Pipeline Stages](#pipeline-stages)
- [Results](#results)
- [Installation](#installation)
- [Usage](#usage)
- [Project Structure](#project-structure)
- [Technical Specifications](#technical-specifications)
- [Dependencies](#dependencies)
- [Citation](#citation)
- [Author](#author)

---

## Overview

Conventional steganography frameworks embed data uniformly across image regions, ignoring the spatial and frequency characteristics of the cover image. This leads to detectable statistical artifacts and offers no resistance against quantum adversaries.

**PRISM** addresses these limitations by building a fully adaptive pipeline that:

1. **Encrypts** the secret image payload using AES-256 + Kyber768 (post-quantum KEM) before a single bit touches the cover image.
2. **Analyses** the cover image using a pretrained CNN (EfficientNet-B0) to classify every 8×8 block as smooth, textured, or edge.
3. **Generates** cryptographically seeded, per-block per-channel embedding weights from the Kyber-derived shared secret.
4. **Selects** the optimal embedding domain (DCT / Spatial LSB / DWT-Haar) for each block based on its texture class.
5. **Embeds** the encrypted bitstream through a centralized `AdaptiveStegoEmbedder` controller, producing a stego image with a **1:1 payload-to-cover ratio**.

Evaluated on **5,000 images from Tiny ImageNet**, PRISM achieves a mean **PSNR of 51.47 dB** and mean **SSIM of 0.99935**, both well above standard imperceptibility thresholds.

---

## Key Features

| Feature | Detail |
|---|---|
| Post-Quantum Security | Kyber768 KEM + AES-256-CBC hybrid encryption |
| Deep Texture Analysis | EfficientNet-B0 pretrained on ImageNet; smooth / textured / edge classification |
| Adaptive Multi-Domain Embedding | DCT (smooth), Spatial LSB (textured), DWT-Haar (edge) |
| HVS-Aware Channel Bias | Higher embedding capacity assigned to the blue channel |
| Seeded Weight Generator | PRNG seeded with Kyber shared secret hash — key-dependent, unpredictable |
| 1:1 Payload Ratio | A full secret image is hidden inside a same-sized cover image |
| Mean PSNR 51.47 dB | Significantly exceeds the widely accepted 40 dB imperceptibility threshold |
| Mean SSIM 0.99935 | Near-perfect structural similarity between cover and stego images |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                         PRISM Pipeline                               │
│                                                                      │
│  Secret Image                                                        │
│      │                                                               │
│      ▼                                                               │
│  ┌──────────────────────────────────┐                                │
│  │  1. Hybrid Payload Encryption    │                                │
│  │  AES-256 → Kyber768 KEM          │                                │
│  │  SHA-256 hash of shared secret   │──────────────► PRNG Seed      │
│  └──────────────────────────────────┘                    │           │
│                     │ Encrypted Bitstream                 │           │
│                     │                                     │           │
│  Cover Image        │          ┌────────────────────────────────┐   │
│      │              │          │  3. Adaptive Weight Generation  │   │
│      ▼              │          │  Seeded PRNG → Per-Block &      │◄──┘
│  ┌──────────────────────────┐  │  Per-Channel Weights            │   │
│  │ 2. Texture Classification│  │  (blue ↑ var ↑)                 │   │
│  │ & Channel Profiling      │──┤                                 │   │
│  │ 8×8 blocks → EfficientNet│  └────────────────────────────────┘   │
│  │ smooth / textured / edge │              │                         │
│  └──────────────────────────┘              │                         │
│           │ texture label                  │ weights                 │
│           ▼                                │                         │
│  ┌─────────────────────────┐               │                         │
│  │ 4. Domain Selection      │               │                         │
│  │  Smooth  → DCT mid-freq  │◄──────────────┘                        │
│  │  Textured→ Spatial LSB   │                                        │
│  │  Edge    → DWT-Haar      │                                        │
│  └─────────────────────────┘                                         │
│           │ domain op                                                 │
│           ▼                                                           │
│  ┌────────────────────────────────────────┐                          │
│  │  5. AdaptiveStegoEmbedder Controller   │                          │
│  │  Processes blocks sequentially,        │──► Stego Image           │
│  │  inserts encrypted bits until done     │                          │
│  └────────────────────────────────────────┘                          │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Pipeline Stages

### Stage 1 — Hybrid Payload Encryption

The secret image is serialized to bytes and encrypted using **AES-256-CBC**, ensuring payload confidentiality even if the stego image is intercepted. The AES session key is encapsulated using **Kyber768** (a NIST-standardized post-quantum KEM), protecting against both classical and quantum adversaries. The Kyber-derived shared secret is hashed with **SHA-256**; this hash doubles as a cryptographic seed for the PRNG used in weight generation (Stage 3).

### Stage 2 — Texture Classification & Channel Profiling

The cover image is partitioned into non-overlapping **8×8 pixel blocks**. Each block is passed through **EfficientNet-B0** (pretrained on ImageNet) to extract deep feature representations, which are then classified by a lightweight head into one of three texture classes:

| Class | Description | Embedding Sensitivity |
|---|---|---|
| Smooth | Uniform, low-variation regions (skies, walls) | High — changes are visible |
| Textured | Complex, noisy regions (grass, fabric) | Low — changes are camouflaged |
| Edge | Sharp boundary regions (object outlines) | Medium — contour-sensitive |

Simultaneously, **per-channel RGB energy** is computed per block to estimate local channel dominance for subsequent payload allocation.

### Stage 3 — Adaptive Embedding Weight Generation

Rather than distributing payload uniformly, PRISM uses the **Kyber-derived shared secret hash as a PRNG seed** to generate unique per-block, per-channel weights. Blocks with higher spatial variance and the blue channel (due to lower human-visual sensitivity) receive a higher baseline allocation. This makes the spatial distribution of hidden bits **key-dependent and unpredictable** without the private key.

### Stage 4 — Texture-Driven Embedding Domain Selection

Each block is routed to the most appropriate embedding domain:

- **Smooth → DCT mid-frequency**: Modifications to mid-frequency DCT coefficients are perceptually invisible in flat regions and resist low-pass filtering.
- **Textured → Spatial LSB**: Flipping the LSB of pixels in noisy regions is visually indistinguishable; the surrounding texture acts as natural camouflage.
- **Edge → DWT-Haar**: Embedding within Haar wavelet coefficients preserves edge contours while still accommodating payload bits. Blue-channel blocks receive higher-capacity variants of each method.

### Stage 5 — AdaptiveStegoEmbedder Controller

A centralized controller (`AdaptiveStegoEmbedder`) orchestrates the end-to-end process. It:
1. Retrieves the predicted texture label for each block (Stage 2).
2. Fetches embedding weights for that block and channel (Stage 3).
3. Looks up the correct embedding operator (Stage 4).
4. Invokes the operator and inserts payload bits until the full encrypted bitstream is embedded.

The final output is the **stego image** — visually indistinguishable from the original cover image.

---

## Results

All experiments were conducted on a **5,000-image subset of Tiny ImageNet**.

### Imperceptibility (PSNR & SSIM)

| Metric | Mean | Std Dev | Minimum | Maximum |
|---|---|---|---|---|
| PSNR (dB) | **51.40 ± 0.24** | — | 50.71 | 55.17 |
| SSIM | **0.99935 ± 0.00072** | — | 0.98412 | 0.99998 |

> The mean PSNR of **51.40 dB** significantly exceeds the commonly accepted **40 dB** imperceptibility threshold.

### Per-Channel PSNR & SSIM

| Channel | PSNR Mean (dB) | PSNR Std | SSIM Mean | SSIM Std |
|---|---|---|---|---|
| Red | 51.34 | 0.28 | 0.99938 | 0.00082 |
| Green | 51.38 | 0.27 | 0.99935 | 0.00078 |
| **Blue** | **51.71** | **0.44** | **0.99932** | 0.00076 |
| Overall | 51.47 | 0.27 | 0.99935 | 0.00075 |

The **blue channel achieves the highest PSNR** despite carrying the maximum hidden payload, validating the HVS-informed channel bias strategy.

### Embedding Metadata (Full Dataset — 320,000 blocks)

| Metric | Value |
|---|---|
| Total Images | 5,000 |
| Total Blocks | 320,000 |
| Mean Bits / Block | 26.27 |
| Median Bits / Block | 29.00 |
| Std Dev Bits / Block | 8.16 |
| Smooth Blocks | 197,645 (61.8%) |
| Edge Blocks | 122,355 (38.2%) |

### Statistical Security

Per-channel histogram correlation between cover and stego images remains **close to 1.0 across all channels**, confirming that the embedding introduces no detectable changes to pixel intensity distributions — confirming first-order statistical steganalysis resistance.

---

## Installation

```bash
# Clone the repository
git clone https://github.com/<your-username>/PRISM.git
cd PRISM

# (Recommended) Create a virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Running on Google Colab

The primary implementation notebook is designed for Google Colab. Upload `PRISM_code.ipynb` to your Drive and open it in Colab. All dependencies are installed within the notebook.

---

## Usage

### Basic Embedding (Colab Notebook)

Open `PRISM.ipynb` and run all cells in order. The notebook walks through:

1. Dependency installation and imports
2. Loading the cover image and secret image
3. Running hybrid payload encryption (AES-256 + Kyber768)
4. Texture classification and channel profiling via EfficientNet-B0
5. Adaptive weight generation from the PRNG seed
6. Domain selection and embedding
7. Saving and evaluating the stego image (PSNR, SSIM)

### Extraction

The extraction pipeline reverses the process:
1. Provide the stego image and the **Kyber private key**
2. The PRNG is re-seeded using the shared secret hash to recreate the exact embedding map
3. Bits are extracted in the same block-channel-domain order
4. The bitstream is decrypted (Kyber768 unwrap → AES-256 decrypt) to recover the original secret image

> ⚠️ **Extraction requires the Kyber private key.** Without it, the PRNG seed cannot be reproduced and the embedded bits remain unintelligible.

---

## Technical Specifications

| Component | Specification |
|---|---|
| Secret Payload | Any image (serialized to bytes) |
| Payload-to-Cover Ratio | 1:1 (full image hidden in same-size cover) |
| Encryption | AES-256-CBC + Kyber768 KEM (post-quantum) |
| Key Hashing | SHA-256 |
| Texture Classifier | EfficientNet-B0, pretrained on ImageNet |
| Block Size | 8×8 pixels (non-overlapping) |
| Embedding Domains | DCT (smooth), Spatial LSB (textured), DWT-Haar (edge) |
| HVS Bias | Blue channel receives higher embedding capacity |
| Training Dataset | Tiny ImageNet |
| Evaluation Dataset | 5,000-image subset of Tiny ImageNet |
| Mean PSNR | 51.40 ± 0.24 dB |
| Mean SSIM | 0.99935 ± 0.00072 |

---

## Dependencies

```
torch>=2.0
torchvision>=0.15
efficientnet-pytorch
numpy
opencv-python
Pillow
pywavelets
scipy
scikit-image
liboqs-python          # Kyber768 (Open Quantum Safe)
pycryptodome           # AES-256
matplotlib
seaborn
tqdm
```

Install all at once:

```bash
pip install -r requirements.txt
```

> **Note on liboqs:** `liboqs-python` requires the Open Quantum Safe `liboqs` C library. See the [OQS installation guide](https://github.com/open-quantum-safe/liboqs-python) for platform-specific setup.

---

## Related Work

This repository is the implementation for the B.Tech major project **PRISM**. A related, lighter framework — **KEAS** (Kyber-Enhanced Adaptive Steganography) — was published at **COMSIA 2025**. KEAS focuses on single-domain adaptive LSB with Kyber-1024 encryption and achieves 100% extraction accuracy with a PSNR of 75.50 dB on a 10k-image benchmark.

---

## Citation

If you use PRISM or this codebase in your research, please cite:

```bibtex
@misc{prism2026,
  author       = {Vaishnavi},
  title        = {PRISM: Post-quantum Resilient Image Steganography with Multi-domain Embedding},
  year         = {2026},
  institution  = {Indira Gandhi Delhi Technical University for Women (IGDTUW)},
  note         = {B.Tech Major Project, Department of Computer Science (AI Specialization)}
}
```

---

## Author

**Vaishnavi**
B.Tech Computer Science (AI Specialization) — IGDTUW, 2022–2026
BS Data Science — IIT Madras, 2023–2026
Generation Google Scholar 2024 (APAC)

[![GitHub](https://img.shields.io/badge/GitHub-Profile-black?logo=github)](https://github.com/<your-username>)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue?logo=linkedin)](https://linkedin.com/in/<your-profile>)

---

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.
