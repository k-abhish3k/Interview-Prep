# 01 — OCR Fundamentals

## Why this chapter matters

Every stage of this pipeline depends on the OCR stage producing more than just text — it needs to
produce *text with geometry*. If an interviewer asks "why didn't you just use OCR output directly to
find superscripts," the answer lives entirely in this chapter: plain OCR text throws away exactly
the signal (size, position, baseline) that defines what a superscript *is*. Understanding OCR at the
level of "what does the engine actually output, and what does it discard" is what separates a
credible answer from a hand-wavy one.

## Text Detection vs. Text Recognition

"OCR" is really two separate sub-problems bundled under one name:

1. **Text detection** — given an image, find the regions that contain text (as bounding boxes or
   polygons), without necessarily knowing what the text says yet. This is a localization/object
   detection problem, conceptually similar to what Chapter 02's YOLO stage does for superscripts,
   just applied to whole words or lines instead of individual small glyphs.
2. **Text recognition** — given a cropped region known to contain text, decode the actual
   characters. This is a sequence-modeling problem: classical engines use per-character
   segmentation and classification, while deep-learning engines typically use CNN feature
   extraction feeding an RNN/Transformer decoder trained with CTC (Connectionist Temporal
   Classification) loss, which lets the model learn character sequences without needing
   perfectly-segmented single-character crops.

Most modern OCR systems (both classical and deep-learning) are architected as detection followed by
recognition, run per line or per word. This decomposition matters for this project because the
**detection half is reusable** — the same word-level bounding boxes that a good OCR engine already
produces are the geometric anchor that Stage 2 (YOLO) uses to know *where* to even look for a
superscript candidate. You don't run YOLO blindly over the whole image; you run it near text regions
OCR has already located.

## Classical OCR (Tesseract) vs. Deep-Learning OCR

**Tesseract** (the long-standing open-source engine, now on its LSTM-based v4/v5) is a reasonable
default for a first pass: it's free, runs on CPU, and for clean, high-contrast, reading-order text
it's fast and accurate. Its recognition core is itself now an LSTM, but its page/line segmentation
still leans on classical heuristics (connected-component analysis, projection profiles) to find text
lines before recognition runs.

**Deep-learning end-to-end OCR engines** (e.g., PaddleOCR, EasyOCR, or fully learned detector+
recognizer stacks) replace those classical segmentation heuristics with learned detectors (often a
CNN-based text-region proposal network) feeding a learned recognizer. The practical trade-off:

| | Tesseract (classical + LSTM recognizer) | Deep-learning OCR stack |
|---|---|---|
| Setup cost | Low — pip/binary install, no training | Higher — larger models, sometimes needs GPU for speed |
| Robustness to skew/noise/unusual layout | Weaker — segmentation heuristics assume fairly clean pages | Stronger — learned detection generalizes better to messy real-world images |
| Speed on CPU | Fast | Slower, unless a lightweight model is chosen |
| Handling of small/irregular text (e.g., superscripts) | Poor out of the box | Still imperfect, but generally better feature representations |
| Managed/cloud alternative | — | **AWS Textract** — no infra to manage, strong general-purpose accuracy, returns word/line geometry and confidence scores natively |

For a pharma-document pipeline dealing with slide-deck exports, scans of printed reprints, and
inconsistent source quality — across the real Eli Lilly and AstraZeneca content this system
processed in production — this project's OCR stage plausibly evaluated both a self-hosted engine
(Tesseract, for cost/control) and a managed option like **AWS Textract** (for robustness and less
operational overhead) — the two are not mutually exclusive; many production pipelines use
Textract (or similar) as the primary path and keep a Tesseract fallback for cost-sensitive or
offline batch jobs. What Textract adds specifically for this use case is that it returns structured
`WORD` and `LINE` blocks with bounding-box geometry and confidence scores directly in its API
response, which is exactly the geometric metadata the downstream YOLO stage needs — you don't have
to parse it out of raw pixel analysis yourself.

## Common Failure Modes With Superscripts and Subscripts

This is the crux of why OCR alone can't solve the problem. Superscript/subscript text breaks OCR in
a few specific, predictable ways:

- **Font-size sensitivity.** OCR line-segmentation algorithms typically expect all glyphs on a
  detected "line" to be roughly the same size. A superscript character is commonly 50–70% of the
  surrounding font size. Depending on the engine and its confidence thresholds, this can cause the
  engine to (a) merge the superscript into the preceding word as if it were just a smaller version
  of the same character stream, (b) discard it entirely as noise below a minimum glyph-size
  threshold, or (c) recognize it but report it with low confidence and no marker that it's
  structurally different from body text.
- **Positional ambiguity.** A superscript sits above the baseline; a subscript sits below it. Most
  OCR output formats (plain text, and even many "structured" outputs) don't preserve this
  vertical offset as a distinct field — you get a flat character string. Two documents that read
  identically as plain text — `"reduced onset by 40%1"` vs `"reduced onset by 40%¹"` — can be
  visually completely different (one is a mid-sentence numeral, the other is a citation marker),
  and naive OCR text output collapses that distinction.
- **Character confusability at small size.** At superscript scale, digits and punctuation are more
  easily confused (a small `1` vs a stray mark vs part of a symbol like `†`), so recognition
  confidence for these tiny glyphs tends to be inherently lower than for full-size body text.
- **Baseline detection failure.** Text-line baseline estimation (the geometric line that most
  characters' bottoms sit on) is itself sensitive to including superscripts in the line — a
  handful of small, elevated glyphs can skew a naive baseline-fitting algorithm, which then
  propagates a small geometric error into every character's reported bounding box on that line.

## Why Bounding-Box and Baseline Metadata Matters Downstream

The single most important thing a good OCR engine gives this pipeline isn't the transcribed text —
it's the **geometry**: per-word (ideally per-character) bounding boxes, plus, where available, the
line's baseline y-coordinate and the dominant/median font size for that line. That metadata is what
turns "detect a superscript" from an ill-posed image classification problem (is this crop of pixels
a superscript?) into a well-posed relative-geometry problem: *is this glyph's bounding box small
relative to its line's median glyph size, and is its vertical position offset above the line's
baseline by more than a threshold?*

```python
# Illustrative — shape of what an OCR engine's structured output looks like
# (based on the kind of block Tesseract's TSV output or AWS Textract's WORD blocks return)
ocr_words = [
    {"text": "onset", "bbox": (120, 200, 168, 216), "line_baseline_y": 216, "line_font_px": 16},
    {"text": "by",    "bbox": (172, 200, 188, 216), "line_baseline_y": 216, "line_font_px": 16},
    {"text": "40%",   "bbox": (192, 200, 224, 216), "line_baseline_y": 216, "line_font_px": 16},
    {"text": "1",     "bbox": (226, 190, 232, 200), "line_baseline_y": 216, "line_font_px": 16},
    # ^ this glyph is 10px tall vs. the line's 16px, and its top/bottom sit ~10-16px
    #   above the line baseline -- exactly the signal Stage 2 (YOLO) is trained to pick up on.
]
```

This is precisely the input contract between Stage 1 and Stage 2 of the pipeline: OCR doesn't try to
decide what's a superscript — it just reliably reports where every glyph is and how big it is. Stage
2 (Chapter 02) takes that geometry and the raw image crop and does the actual candidate detection.

## Tying It Back

When an interviewer asks "why not just detect superscripts from OCR output directly," the answer is
this chapter, condensed: OCR text is a lossy 1-D projection of a 2-D layout problem. The engine you
choose (Tesseract for a cheap self-hosted baseline, a deep-learning stack or **AWS Textract** for
more robust geometry and confidence scores on messy real pharma documents) matters less than making
sure whichever engine you pick surfaces bounding-box and baseline metadata — because that metadata,
not the text string, is the actual input the rest of the pipeline is built on.
