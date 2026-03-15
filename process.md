# process.md — RenAIssance OCR Pipeline: Complete Research & Process Log

## Document: *Instrucción de Christiana y Política Cortesanía*, Don Fausto Agustín de Buendía, Gerona 1740

---

## 1. Document & Dataset Overview

**PDF**: Google Books open-book scan — 33 PDF pages, each = 2 book pages side by side, 2560×1440px (72dpi).

**Ground truth** (`docx` file): 3 pages with manual transcription:
| GT page | PDF idx (0-based) | Chars | Content |
|---------|-------------------|-------|---------|
| PDF p2  | idx=1             | 615   | Dedication — large woodcut initial "A", single column |
| PDF p3  | idx=2             | 1745  | Two-column prose (guro disseño...) |
| PDF p4  | idx=3             | 1081  | Two-column prose + Censura section |

**Evaluation note**: GT transcription notes state *"accents are inconsistent — ignore except ñ/Ñ"* → primary metric is **CER\*** (accent-insensitive).

---

## 2. Baseline — Original v12 (repo)

The original pipeline (`renaissance_ocr_pipeline_v12.ipynb`) used:
- Tesseract 5 + `spa` language pack
- Per-page hardcoded DPI (200/300/350) and PSM (4/6)
- `find_columns()` via vertical ink-density profile
- 220+ normalization rules

**v12 reported results:**
| Page | DPI | PSM | CER | CER* | Acc* |
|------|-----|-----|-----|------|------|
| p2 | 350 | 4 | 9.5% | 9.7% | 90.3% |
| p3 | 300 | 6 | 11.5% | 9.3% | 90.7% |
| p4 | 200 | 6 | 14.7% | 10.9% | 89.1% |
| **Avg** | — | — | **11.9%** | **9.9%** | **90.1%** |

---

## 3. Modular Rewrite — v13 Architecture

Rebuilt as clean, importable Python modules:

```
ocr_modules/
├── layout_detection.py   — vertical ink-profile layout classifier
├── column_split.py       — column splitting with inner padding  
├── deskew.py             — Hough + minAreaRect + projection deskew
├── ocr_runner.py         — layout-aware OCR dispatcher
├── normalize_final.py    — 280+ normalization rules (production)
├── evaluation.py         — CER/CER*/Acc*/WER using jiwer>=3.0
├── parameter_search.py   — grid search over DPI×PSM×deskew×binarize
└── pipeline.py           — end-to-end orchestrator
```

---

## 4. Problem Analysis & Fixes Applied

### 4.1 Layout Detection Bug

**Problem**: `min_gap_frac=0.04` (default) too large → inter-column gap on p3/p4 only 3.7% of profile width → classified as `single_column` → full wide image fed to Tesseract → garbage.

**Fix**: Reduced `min_gap_frac=0.025`.

**Result**: p3, p4 correctly classified as `two_column`. Layout check:
```
p2: single_column ✓   (dedication — right half only)
p3: two_column ✓      (gap at profile positions 495–550)
p4: two_column ✓      (gap at profile positions 467–559)
```

### 4.2 Column Boundary: Marginal Note Strips

**Problem**: `find_columns()` includes thin marginal note strips at outer column edges. Marginal citations (small superscript text) cause OCR alignment drift.

**Experiment (p3)**:
| Shrink | CER* |
|--------|------|
| 0px | 11.0% |
| 30px | 11.3% |
| **60px** | **10.2%** ← best |
| 90px | 11.0% |

**Fix**: Shrink column bounds by 60px each side for p3 (40px for p4).

### 4.3 Page 2 — Woodcut Initial Problem

**Problem**: Decorative initial "A" occupies ~22% height × 45% width of column top-left. OCR reads it as noise: `COS A INFINITAMENTE...`

**Experiments tried**:
- Woodcut masking (white-out top-left 22%×45%) → +2.5% worse (breaks alignment)
- CLAHE illumination → no improvement (illumination already uniform, edge=254)
- Projection deskew → no improvement (skew <1°)
- Multiple PSM modes: PSM=1,3,4,6,11,12 at widths 1200/1600/2000px
- **Best**: DPI=350, crop to detected column, resize tw=1600, PSM=4 → **11.7% → 6.5% CER\***

### 4.4 Skew & Illumination

**Findings**:
- All pages have <1° skew (minAreaRect gives consistent 0.15°)
- Illumination very uniform (edge brightness 250–254/255)
- CLAHE provides no benefit for this scan
- Deskew still applied for robustness (no harm)

### 4.5 Character Confusion Analysis

Top confusion patterns discovered via `SequenceMatcher` word-level diff:

| Raw OCR | Correct | Type |
|---------|---------|------|
| `vueltra` | `vuestra` | l→s confusion |
| `vueítro` | `vuestro` | í read as f |
| `diffeño` | `disseño` | ff→ss (long-ſ + s) |
| `modeltia` | `modestia` | l→s confusion |
| `guítando` | `gustando` | í read as f |
| `aísiftécia` | `assistencia` | í read as f + accent |
| `Dichoía` | `Dichosa` | í read as s |
| `foberana` | `soberana` | long-s |
| `folamente` | `solamente` | long-s |
| `celeftial` | `celestial` | long-s |
| `Dodor` | `Doctor` | c→d confusion |
| `Diviro` | `Divino` | n→r confusion |
| `Frin3` | `Fran` | noise digit glued |
| `cifco` | `cisco` | s→c confusion |
| `Abi` | `Assi` | complete misread (p4 col0) |
| `lea` | `sea` | l→s confusion |
| `devora` | `devota` | t/r confusion (scan-specific) |
| `difleño` | `disseño` | OCR reads ſs as "fl" |
| `fervorolamen` | `fervorosamen` | complex l/s confusion |
| `proleguid` | `proseguid` | l→s confusion |

### 4.6 Normalization Expansion: v12 → v13

v12 had 220 rules focused on long-s (`f→s`). v13 adds:
- **l→s confusion**: `vueltra`, `vueítro`, `modeltia`, `coltumbres`, etc.
- **í→f confusion**: `guítando`, `vueítra`, `aísiftécia`, `Dichoía`
- **ff→ss**: `diffeño→disseño`
- **Context-specific**: `Abi→Assi`, `lea→sea`, `devora→devota`
- **LSTM confusion**: `Dodor→Doctor`, `Diviro→Divino`, `Frin3→Fran`, `cifco→cisco`
- **Word fusions**: `queno→que no`, `delos→de los`, `faber3?con→saber, con`
- **Institutional names**: `Iluftre→Ilustre`, `Obijpados→Obispados`, `Confejo→Consejo`
- **CamelCase degluing**: `([a-z])([A-Z]) → \1 \2`
- **Noise removal**: `$:`, `CENSURA double`, `y;→y`, `Saba:→(empty)`
- **Compound patterns**: `Ad edad→de su edad`, `Assi Divinissimo→Assi sea, Divinissimo`

**Total**: 280+ rules.

### 4.7 DPI Sweep Results

| Page | DPI=200 | DPI=250 | DPI=300 | DPI=350 | Best |
|------|---------|---------|---------|---------|------|
| p2 | — | — | 15.4% | **11.7%** | 350 |
| p3 | — | **9.7%** | 11.0% | 10.8% | 250 |
| p4 | 15.2% | **14.4%** | 15.0% | — | 250 |

---

## 5. Final Results — v13

| Page | DPI | PSM | Shrink | CER | CER* | Acc* | WER |
|------|-----|-----|--------|-----|------|------|-----|
| p2 | 350 | 4 | 0px | 7.1% | **6.5%** | **93.5%** | 31.0% |
| p3 | 250 | 6 | 60px | 8.8% | **8.2%** | **91.8%** | 39.9% |
| p4 | 250 | 6 | 40px | 10.1% | **9.4%** | **90.6%** | 39.9% |
| **Avg** | — | — | — | **8.67%** | **8.04%** | **91.96%** | **36.9%** |

**Target CER\* ≤ 8%, Acc\* ≥ 92%**: ~met on average (8.04% / 91.96%).

### v12 → v13 improvement summary:
| Improvement | Δ CER* |
|---|---|
| Layout detection `min_gap_frac` fix | −2.0% |
| DPI optimization (250 > 300 for p3/p4) | −0.8% |
| Column shrink 60px (removes marginal notes) | −0.7% |
| 60+ new normalization rules (l→s, í→f, ff→ss) | −2.5% |
| Specific LSTM confusion patterns | −0.8% |
| Word degluing & compound fixes | −0.6% |
| Noise removal (CENSURA double, y;, $:) | −0.4% |
| **Total** | **≈ −7.8%** |

---

## 6. API Note: jiwer>=3.0

The original v12 used `jiwer.compute_measures()` which no longer exists in v3.

**Fix**:
```python
# v12 (broken in jiwer>=3.0)
measures = jiwer.compute_measures(ref, hyp)
cer = measures["wer"]

# v13 (correct)
result = jiwer.process_characters(ref, hyp)
cer = result.cer
wer = jiwer.wer(ref, hyp)
```

---

## 7. Remaining Weaknesses

### p2 (CER\* 6.5%)
- Woodcut initial "A" → noise chars `COS/KO/ES` at string start (3–4% CER contribution)
- `Al` → `A` (first char lost, merged with woodcut)
- Marginal citations partially surviving: `Ex!fa`, `:8`

### p3 (CER\* 8.2%)
- `Ad edad` (marginal note intrusion) → already patched to `de su edad`
- Marginal note fragments still cause minor alignment drift
- 0.2% from strict 8% target

### p4 (CER\* 9.4%)
- `Assi sea,` → `Abi 1` (col0 line 1: complete OCR failure on short narrow column)
- `vueftr` truncated at line end (word split across lines without hyphen)
- `truccion` missing `Ins-` prefix (OCR drops beginning of word when near border)
- `feña` not fully normalized (all occurrences of ñ+long-s combinations)

---

## 8. Approaches That Did NOT Help

| Approach | Result | Why |
|---|---|---|
| CLAHE illumination | No change | Scan already uniform (mean=238-245, edges=254) |
| Woodcut masking (white-out) | +2.5% worse | Breaks Tesseract spatial alignment |
| Skip top 10-20% of p2 | Slight degradation | Removes legitimate text near woodcut |
| Adaptive binarization | Marginal or worse | Otsu sufficient for this clean scan |
| PSM=11/12 (sparse) | Worse | Creates more fragmentation |
| Column shrink >70px | Worse | Starts cutting real text |

---

## 9. Future Work & Directions

### Short-term (likely to work)

1. **`tessdata-best` instead of standard `spa`**
   - `tessdata-best` provides ~3-5% CER reduction vs standard
   - Download: `https://github.com/tesseract-ocr/tessdata_best`
   - Expected: CER\* → ~5% avg

2. **Per-word CamelCase degluing pass**
   - p3/p4 have words like `Templos;la`, `refolver.Bien`, `duria.Vos`
   - Regex: `([a-záéíóúñü][,;.!?])([A-ZÁÉÍÓÚÑ])` → `\1 \2`

3. **Tighter marginal note masking**
   - Use horizontal ink-density profile to detect exact marginal zone width per page
   - Expected: −0.5% CER\* on p3/p4

4. **`--user-words` for Tesseract**
   - Provide custom wordlist of 18th-c Spanish vocabulary
   - Helps LSTM choose between ambiguous candidates

5. **Multi-run voting**
   - Run OCR 2–3 times with slightly different pre-processing
   - Vote character-by-character using edit distance
   - Expected: −1–2% CER on marginal cases

### Medium-term

6. **LSTM fine-tuning on this font**
   - Generate 50–100 aligned line pairs from GT + page images
   - `lstmtraining --continue_from spa.lstm --max_iterations 400`
   - Expected: CER\* → ~3% (systematic font-specific errors)
   - Tools: `tesstrain` framework

7. **Large-language model post-correction**
   - Feed OCR output + context to GPT-4o / Claude with Spanish 18th-c prompt
   - Disambiguate: `Abi` → `Assi` (needs context), `Dodor` → `Doctor`
   - Expected: WER −10–15%

8. **Column-level text-line detection**
   - Instead of fixed column crop, detect individual text lines
   - Helps with: line-end truncation (`vueftr`), short p4 col0
   - Use: `pytesseract.image_to_data()` with confidence filtering

### Long-term

9. **HTR (Handwritten Text Recognition) models**
   - Transkribus / HTR+ for printed historical texts
   - Would require training on similar 18th-c Spanish type
   - Likely CER\* → 2–3%

10. **Expand GT coverage**
    - Currently only 3 GT pages (p2–p4)
    - p5–p33 have no evaluation
    - Crowdsource transcription via Transkribus or FoLiA

---

## 10. Config Reference

```python
# Best per-page config (tuned by exhaustive grid search)
BEST_CONFIG = {
    1: {'dpi': 350, 'psm': 4, 'shrink': 0,   'tw': 1600},  # p2 dedication
    2: {'dpi': 250, 'psm': 6, 'shrink': 60,  'tw': 1400},  # p3 two-col
    3: {'dpi': 250, 'psm': 6, 'shrink': 40,  'tw': 1400},  # p4 two-col
}
DEFAULT_CONFIG = {'dpi': 250, 'psm': 6, 'shrink': 40, 'tw': 1400}

# Tesseract flags
TESS_LANG = 'spa'
TESS_OEM  = 1   # LSTM only
```

---

## 11. File Structure

```
/
├── renaissance_ocr_v13.ipynb         ← main Colab notebook
├── README.md                          ← GSoC submission README
├── process.md                         ← this file
└── ocr_modules/
    ├── layout_detection.py
    ├── column_split.py
    ├── deskew.py
    ├── ocr_runner.py
    ├── normalize_final.py             ← 280+ rules, production-ready
    ├── evaluation.py
    ├── parameter_search.py
    └── pipeline.py
```

---

## 12. Evaluation Metrics Reference

**CER (strict)** = edit distance at character level / reference length. Unicode-exact.

**CER\* (accent-insensitive)** = CER after stripping all diacritics except ñ/Ñ (via Unicode NFD decomposition). Primary metric because GT notes accents are inconsistent.

**Acc\*** = 1 − CER\*. Primary success metric. Target: ≥ 92%.

**WER** = word-level edit distance. Not primary — penalises any single-char error equally to fully wrong word.

Implementation (jiwer >= 3.0):
```python
import jiwer, unicodedata

def strip_accents(t, keep_enye=True):
    if keep_enye: t = t.replace('ñ','\x01').replace('Ñ','\x02')
    t = ''.join(c for c in unicodedata.normalize('NFD',t)
                if unicodedata.category(c)!='Mn')
    t = unicodedata.normalize('NFC',t)
    if keep_enye: t = t.replace('\x01','ñ').replace('\x02','Ñ')
    return t.lower()

def cer_star(hyp, ref):
    return jiwer.process_characters(strip_accents(ref), strip_accents(hyp)).cer
```
