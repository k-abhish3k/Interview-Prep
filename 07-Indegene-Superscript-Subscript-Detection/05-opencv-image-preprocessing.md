# 05 — OpenCV Image Preprocessing

## Why this chapter matters

Everything in Chapters 01–04 assumes the pipeline receives a reasonably clean image. Real pharma
documents rarely arrive that way — scanned journal reprints have skew and noise, slide-deck exports
have inconsistent contrast and compression artifacts, and photographed pages have perspective
distortion. This was true in production for both Eli Lilly's and AstraZeneca's content: neither
client's source material arrived as clean, uniform input, so this preprocessing stage had to hold up
across genuinely varied real-world document quality, not just a curated test set. **OpenCV** is the tool most likely used to close that gap before OCR ever runs, and it's
explicitly called out in the candidate's skill list for this project. Interviewers ask about
preprocessing because it's the part of a CV pipeline most junior candidates skip over — being able
to explain *why* a specific preprocessing step improves downstream accuracy (not just that you
"used OpenCV") is a strong signal of hands-on experience.

## Why Preprocessing Quality Is an Upstream Multiplier, Not a Nice-to-Have

Chapter 04 established that this pipeline's errors compound stage to stage. Preprocessing sits
*before* Stage 1 (OCR), which makes it the earliest possible point to introduce — or prevent — error
that every later stage inherits. A skewed page shifts every line's baseline; a low-contrast scan
depresses OCR confidence broadly; noise and speckle can be mistaken for small glyphs (which is
especially costly here, since Stage 2 is specifically hunting for small, elevated regions — image
noise is exactly the kind of artifact that looks like a false-positive superscript candidate before
it even gets there). Getting preprocessing right is one of the highest-leverage, lowest-glamour parts
of this system.

## Binarization / Thresholding

Converting a grayscale (or color) image to pure black-and-white is often the first real
preprocessing step, because most classical OCR segmentation logic (and many detection pipelines)
work more reliably on high-contrast binary images than on raw grayscale.

```python
import cv2

gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)

# Global threshold -- fine for uniformly-lit, high-contrast scans
_, binary = cv2.threshold(gray, 0, 255, cv2.THRESH_BINARY + cv2.THRESH_OTSU)

# Adaptive threshold -- better for scans with uneven lighting/shadowing,
# which is common on photographed or poorly-scanned pharma reprints
adaptive = cv2.adaptiveThreshold(
    gray, 255, cv2.ADAPTIVE_THRESH_GAUSSIAN_C, cv2.THRESH_BINARY, 31, 11
)
```

**Otsu's method** automatically picks a single global threshold by finding the value that best
separates the image's pixel-intensity histogram into two classes (foreground/background) —
appropriate when lighting is fairly even across the page. **Adaptive thresholding** instead computes
a local threshold per neighborhood, which matters for scanned or photographed documents where
lighting isn't uniform (a shadow across one corner of a scanned page would defeat a single global
threshold, silently losing text in that region — and losing text is a Stage 1 recall problem that,
per Chapter 04, nothing downstream can recover from).

## Deskewing

Scanned or photographed pages are rarely perfectly aligned — even a 1-2 degree rotation measurably
hurts OCR line-segmentation, and it's a particularly acute problem for this project because
superscript detection depends on comparing a glyph's vertical position to its line's baseline
(Chapter 01); a skewed page makes "baseline" itself an unstable, sloped reference.

A standard OpenCV approach: find the dominant text-line angle and rotate the image to correct it.

```python
import cv2
import numpy as np

def deskew(binary_img):
    coords = np.column_stack(np.where(binary_img > 0))
    angle = cv2.minAreaRect(coords)[-1]
    # cv2.minAreaRect returns angles in a range that needs normalizing to a
    # signed rotation between -45 and +45 degrees
    angle = -(90 + angle) if angle < -45 else -angle

    (h, w) = binary_img.shape[:2]
    center = (w // 2, h // 2)
    M = cv2.getRotationMatrix2D(center, angle, 1.0)
    return cv2.warpAffine(
        binary_img, M, (w, h),
        flags=cv2.INTER_CUBIC, borderMode=cv2.BORDER_REPLICATE,
    )
```

`cv2.minAreaRect` fits the smallest rotated rectangle around all foreground (text) pixels; its angle
is a proxy for the page's overall skew, which is then corrected with an affine rotation. This is a
minimal-viable approach — production systems often instead detect skew per text-line via Hough-line
detection on line contours, since a single global angle can be wrong if a document contains multiple
skewed regions (e.g., a rotated inset image within an otherwise straight slide).

## Contour Detection

Contours — continuous curves joining points of similar intensity along a boundary — are how OpenCV
finds discrete connected shapes in a binary image, and they're a common building block both for
general layout analysis (finding text blocks, tables, figure regions to exclude before OCR) and as a
cheap classical pre-filter ahead of Stage 2's YOLO model.

```python
contours, _ = cv2.findContours(binary, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)

small_elevated_candidates = []
for c in contours:
    x, y, w, h = cv2.boundingRect(c)
    if w < 12 and h < 12:  # illustrative size threshold, tuned per document DPI
        small_elevated_candidates.append((x, y, w, h))
```

It's worth being explicit in an interview that a contour-based size filter like this is a *much*
cruder tool than Stage 2's learned YOLO detector — it has no notion of context, baseline, or visual
appearance beyond raw connected-component size — but it's a legitimate, cheap way to do initial
noise reduction or region-of-interest proposal ahead of the more expensive learned stages, and it's
a natural thing to reach for first when prototyping before a YOLO model is trained and available.

## Cropping Regions of Interest

Once text lines/blocks are located (via OCR's own layout output, or via contour/projection-profile
analysis done in OpenCV), cropping to just those regions — rather than always running downstream
models over the full page — is both an accuracy and a cost optimization:

```python
def crop_line_regions(image, line_boxes, pad=4):
    crops = []
    h, w = image.shape[:2]
    for (x, y, bw, bh) in line_boxes:
        x0, y0 = max(0, x - pad), max(0, y - pad)
        x1, y1 = min(w, x + bw + pad), min(h, y + bh + pad)
        crops.append(image[y0:y1, x0:x1])
    return crops
```

This connects directly to Chapter 02's small-object-detection discussion: running YOLO on a
full-page image makes a superscript character an extremely small fraction of the frame, which hurts
detection performance. Cropping to individual text-line regions first — with a small padding margin,
as above — makes the same physical-size superscript a much larger fraction of the crop it's actually
detected within, which is one of the concrete, practical ways this pipeline compensates for YOLO's
known weakness on tiny objects.

## How Preprocessing Quality Impacts Everything Downstream

Tying the specific techniques together against the pipeline as a whole:

| Preprocessing step | Downstream effect if skipped or done poorly |
|---|---|
| Binarization/adaptive threshold | Uneven lighting causes OCR to miss text in shadowed regions (Stage 1 recall loss — unrecoverable per Chapter 04) |
| Deskewing | Sloped baselines make superscript vertical-offset detection unreliable across the whole page (corrupts the core signal Stage 2 relies on) |
| Contour-based noise filtering | Scan speckle/artifacts get treated as superscript candidates, inflating Stage 2 false positives that Stage 3 then has to absorb |
| ROI cropping to text lines | Without it, superscripts are a vanishingly small fraction of a full-page detection frame, worsening Stage 2's small-object recall |

## Tying It Back

OpenCV's role in this project isn't a separate "step" so much as it's the discipline of making sure
the image the OCR engine (Chapter 01) actually receives is as clean, upright, and well-cropped as
the pipeline's downstream assumptions require. Given how directly Chapter 04 shows early-stage
errors propagating and compounding, time spent here — binarization tuned to the actual scan quality
of the document corpus, deskewing, and sensible ROI cropping — is arguably one of the more
cost-effective places to invest engineering effort in this whole system, and it's a strong, concrete
answer to "what's the first thing you'd check if accuracy dropped on a new batch of documents."
