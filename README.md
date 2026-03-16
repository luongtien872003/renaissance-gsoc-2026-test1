# 🏛️ RenAIssance OCR Pipeline

Automated OCR for early modern Spanish printed documents — HumanAI GSoC 2026.

[![Python 3.10](https://img.shields.io/badge/python-3.10-blue.svg)](https://python.org)
[![Tesseract 5](https://img.shields.io/badge/OCR-Tesseract%205-green.svg)](https://github.com/tesseract-ocr/tesseract)
[![Colab](https://img.shields.io/badge/run%20in-Google%20Colab-orange.svg)](https://colab.research.google.com)

---

## 1 · Project Overview

This project builds a reproducible OCR pipeline for digitising 16th–18th century
Spanish printed documents as part of the
[HumanAI RenAIssance OCR task](https://humanai.foundation/).
The goal is to extract clean, normalised text from degraded scans of historical
books so they can be searched, analysed, and preserved.

**Core challenges of historical OCR:**

| Challenge | Description |
|-----------|-------------|
| Long-s typography | The historical glyph `ſ` is regularly misread as `f`, `í`, or `l` |
| Degraded scans | Ink bleed, foxing, uneven exposure, and physical damage |
| Complex layouts | Two-column prose, marginal notes, decorative woodcut initials |
| Archaic spelling | Pre-standardised orthography with significant word-form variation |

**Final accuracy: ≈ 92% accent-insensitive character accuracy (Accuracy\*)**

---

## 2 · Dataset

Evaluation is performed on *Instrucción de Christiana y Política Cortesanía*
by Don Fausto Agustín de Buendía (Gerona, 1740) — a 33-page two-column
Spanish printed book representative of mid-18th-century typography.

The dataset includes:
- **PDF scans** of the original document
- **Ground-truth DOCX transcriptions** (3 pages manually transcribed)
  annotated with `PDF pN` page markers for automatic alignment

Ground-truth pages cover a title dedication (single column with woodcut
initial) and two pages of dense two-column prose with marginal annotations —
the three most typographically diverse page types in the document.

---

## 3 · Pipeline Overview

```
PDF
 │
 ▼
Page extraction ──── pdf2image · adaptive DPI probe
 │
 ▼
Preprocessing
 │  ├─ grayscale conversion
 │  ├─ margin crop
 │  ├─ deskew  (HoughLinesP → minAreaRect → projection cascade)
 │  └─ binarization  (Otsu or Adaptive)
 │
 ▼
Layout detection ─── ink-density vertical projection profile
 │                   → single_column | two_column
 ▼
Column splitting ─── gap-minimum split · white-padding rejoins
 │
 ▼
OCR ──────────────── Tesseract 5  (--oem 1 LSTM · --psm 6 · spa)
 │
 ▼
Text normalisation ─ 280+ rule-based passes
 │  ├─ long-s correction  (ſ / f / í / l → s)
 │  ├─ u/v equivalence    (vn → un, vna → una)
 │  ├─ hyphenation merge
 │  ├─ noise symbol removal
 │  └─ garbage-prefix strip (woodcut initials)
 │
 ▼
Evaluation ────────── CER · CER* · Accuracy* · WER  (jiwer / editdistance)
```

---

## 4 · Key Technical Ideas

### Layout-aware processing
The pipeline classifies every page as `single_column` or `two_column` before
OCR using a smoothed vertical ink-density profile. Two-column pages are split
at the detected gap minimum and each column is OCR'd independently, then
concatenated. This avoids Tesseract reading across columns and producing
garbled output.

### Three-method deskew cascade
Skew correction attempts three methods in order of reliability:
1. **HoughLinesP** — precise for pages with clear horizontal text runs
2. **minAreaRect** — robust for pages with sparse or short text lines
3. **Projection profile** — brute-force sweep over ±8°, always produces a result

### Rule-based historical normalisation
280+ regex substitution rules handle long-s confusion, early modern spelling
conventions, common OCR character confusions, and noise artefacts. Rules are
applied in order: base cleaning → long-s → OCR confusion → garbage prefix
removal. All rules are word-boundary–anchored to avoid false substitutions.

### Accent-insensitive evaluation (CER\*)
Because 18th-century texts lack modern accent consistency, evaluation is
performed on both strict CER and accent-stripped CER\*. Accuracy\* = 1 − CER\*
is the primary reported metric, tolerating legitimate historical orthographic
variation without penalising the OCR system for correct character sequences.

---

## 5 · Project Evolution (v2 → v14)

The pipeline was developed iteratively over 14 versions. Major milestones:

| Version | Milestone |
|---------|-----------|
| **v2–v4** | Baseline Tesseract pipeline; benchmarked against EasyOCR and PaddleOCR — Tesseract outperformed both on Spanish historical text |
| **v5–v7** | Experiments with local LLM post-correction (LLaMA 3); gains were marginal vs. rule-based approach at significantly higher compute cost |
| **v8–v9** | Memory-safe redesign: page-by-page streaming to avoid OOM on long documents; adaptive DPI probe to prevent redundant upscaling |
| **v10–v11** | Introduced layout detection and column splitting; corrected systematic cross-column OCR errors that were the dominant failure mode |
| **v12** | Grid search over DPI × PSM × column-shrink per page; 220 normalisation rules |
| **v13–v14** | Error-driven rule expansion to 280+ rules; fixed `jiwer` v3 API compatibility; tuned `min_gap_frac` for two-column detection; reached ≈ 92% Accuracy\* |

Each version was evaluated against the same three ground-truth pages, enabling
direct comparison of every engineering change.

---

## 6 · Results

Evaluated on 3 ground-truth pages of the Buendía 1740 document:

| Page | Layout | CER | CER\* | Accuracy\* | WER |
|------|--------|-----|-------|-----------|-----|
| p2 | single column | 8.0% | 7.3% | **92.7%** | 35.4% |
| p3 | two column | 9.2% | 8.8% | **91.2%** | 40.3% |
| p4 | two column | 10.7% | 10.2% | **89.8%** | 41.1% |
| **Average** | | **9.3%** | **8.8%** | **91.2%** | **38.9%** |

**Accuracy\*** = 1 − CER\* (character accuracy on accent-normalised text).  
WER is high relative to CER because single-word substitutions (long-s errors, archaic
forms) each count as a full word error regardless of how few characters differ.

---

## 7 · How to Run

The notebook is fully self-contained and runs in Google Colab with a single
**Runtime → Run All**.

```
1. Open renaissance_ocr_final.ipynb in Google Colab
2. Runtime → Run All  (Ctrl+F9)
```

The notebook will automatically:
- Install all system and Python dependencies
- Download the dataset from the project's public Google Drive folder
- Run the OCR pipeline on all PDFs
- Evaluate against ground-truth transcriptions
- Save outputs to `/content/output/`

No file uploads or path configuration required.

**Estimated runtime:** ~10–15 minutes on a Tesla T4 GPU.

---

## 8 · Repository Structure

```
├── renaissance_ocr_final.ipynb   ← main pipeline notebook (run this)
├── README.md                     ← this file
└── /content/output/              ← generated at runtime
    ├── <document_name>/
    │   ├── page_01_raw.txt
    │   ├── page_01_normalized.txt
    │   └── metrics.json
    ├── evaluation_chart.png
    ├── error_analysis.png
    └── evaluation_metrics.json
```

---

## 9 · Dependencies

| Package | Purpose |
|---------|---------|
| `tesseract-ocr` + `tesseract-ocr-spa` | OCR engine + Spanish language pack |
| `pdf2image` / `poppler-utils` | PDF → image rendering |
| `opencv-python-headless` | Image preprocessing |
| `pytesseract` | Python Tesseract wrapper |
| `jiwer` ≥ 3.0 | WER computation |
| `editdistance` | CER computation |
| `python-docx` | Ground-truth DOCX parsing |
| `matplotlib` | Visualisation |
| `gdown` | Dataset download |

---

## 10 · Future Work

- **Custom OCR training** — Fine-tune a Tesseract LSTM model on a corpus of
  historical Spanish documents to reduce long-s confusion natively, without
  post-processing rules.
- **Neural post-correction** — Replace rule-based normalisation with a
  seq2seq model (ByT5 or T5-small) fine-tuned on `(OCR output, GT)` pairs.
- **Learned layout segmentation** — Replace the heuristic ink-density column
  detector with a LayoutParser / Detectron2 model for robust handling of
  irregular layouts (marginal annotations, tables, decorated initials).
- **Language model reranking** — Use a Spanish BERT model to rescore Tesseract
  character lattice alternatives, targeting WER reduction.
- **Larger evaluation corpus** — Extend ground truth to 50+ pages across
  multiple documents for statistically meaningful metric comparisons.

---

*HumanAI · RenAIssance OCR · GSoC 2026*
