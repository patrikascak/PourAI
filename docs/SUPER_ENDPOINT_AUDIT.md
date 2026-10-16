# Audit: current `super-endpoint`

This review is based on the uploaded Supabase Edge Function source (`index.ts`) and the current `main/index.html` frontend.

## Executive summary

The current endpoint cannot realistically meet the target of <=5% measurement error across real bar conditions.

The biggest reason is architectural: in the current frontend/endpoint combination, the pixel liquid-level detector is not called at all. The frontend sends `image`, while the endpoint calls `detectLiquidLevel()` only when `binary_image` exists. Therefore `remainingPercent` is normally estimated directly by a multimodal LLM from a single photo.

Then the endpoint computes:

```ts
remainingMl = totalVolume * finalPercent / 100
```

This assumes fill percentage is linearly proportional to volume, which is false for many bottle shapes.

## P0 – accuracy blockers

### 1. Pixel detector is currently disconnected

Endpoint:

```ts
const colorImage = body.color_image || body.image;
const binaryImage = body.binary_image;
...
if (binaryImage) {
  const px = await detectLiquidLevel(binaryImage);
}
```

Current frontend sends only:

```js
body: JSON.stringify({
  image: base64Data,
  inventory_id: currentInventory.id,
  image_hash: window.currentImageHash
})
```

Result: `binaryImage` is undefined and pixel detection is skipped.

### 2. LLM is being used as the primary measuring instrument

The endpoint asks the multimodal LLM to infer:

- brand,
- bottle capacity,
- remaining percentage,
- EAN,
- confidence.

The LLM is useful for semantic recognition, but it should not be the primary metric measurement engine for a <=5% product requirement.

Recommended use of LLM/vision-language model:

- product/brand candidate generation,
- OCR interpretation,
- fallback visual reasoning,
- anomaly explanation.

Do not use it as the source of truth for liquid volume.

### 3. Volume calculation is linear

Current code:

```ts
const remainingMl = Math.round(totalVolume * finalPercent / 100);
```

This silently assumes equal volume per unit bottle height.

For shaped bottles, shoulders, tapered bases and necks, the correct model is variant-specific:

```text
V = V_variant(h)
```

where `h` is normalized liquid level.

### 4. No bottle variant model exists

The system recognizes a product/brand but does not identify a physical bottle geometry version.

This is a critical issue for products whose bottle shape changes over time.

Required entity split:

```text
Product
  Jameson Irish Whiskey 700 ml

BottleVariant
  Jameson-700-v2016
  Jameson-700-v2020
  Jameson-700-v2024
```

Each variant needs its own calibration curve.

### 5. Bad scans are never rejected

There is no quality gate for:

- blur,
- cropped bottle,
- bottle tilt,
- camera roll,
- glare,
- multiple bottles,
- label occlusion,
- invisible liquid boundary,
- unknown bottle variant.

The endpoint always tries to produce a number.

For a measurement product, the correct behavior is often `RESCAN`, not a guessed value.

## P0 – failure modes in current pixel detector

Even if `binary_image` is wired in, the current detector is too fragile for production.

### Fixed image-space ROI

```ts
const xCenter = Math.floor(bitmap.width / 2);
const scanWidth = Math.floor(bitmap.width * 0.2);
const bottom = Math.floor(bitmap.height * 0.88);
const top = Math.floor(bitmap.height * 0.12);
```

This assumes the bottle is centered and occupies approximately the same image region every time.

There is no bottle mask or bounding box.

### It scans 40% of the whole image width

The loop runs from `xCenter - scanWidth` to `xCenter + scanWidth`.

Because `scanWidth = 20%` of the image width, the detector analyzes 40% of the image width. This can include:

- labels,
- shelf/background edges,
- reflections,
- bottle contours.

### It uses only one color channel

```ts
const diff = Math.abs(data[i] - data[above]);
```

`data[i]` is effectively the red channel. This is not luminance and is highly sensitive to glass/liquid color.

### It accepts the first strong edge from the bottom

```ts
if (ratio > 0.15 && ratio > bestScore) {
  bestScore = ratio;
  liquidSurfaceY = y;
  break;
}
```

Because `bestScore` starts at zero and the function immediately breaks, it does not actually find the best edge. It finds the first edge above threshold.

Common false positives:

- bottle bottom,
- bottom of label,
- embossing,
- shelf edge,
- reflection,
- shadow.

### Pixel fill percentage is also linear image height

```ts
const fill = (bottom - liquidSurfaceY) / (bottom - top);
```

This uses image coordinates, not bottle-body coordinates, and again assumes linear height-to-volume conversion.

## P0 – dangerous default behavior

### API/model failure can be stored as a valid empty 700 ml bottle

The code does not verify the Anthropic HTTP status before parsing.

If the response is missing/invalid:

```ts
const parsed = {};
const aiPercent = Number(parsed.remainingPercent) || 0;
const totalVolume = Number(parsed.totalVolumeMl) || 700;
```

This can produce:

```text
brand = Unknown
capacity = 700 ml
remaining = 0%
remainingMl = 0 ml
```

and the scan is still inserted into the database.

This must become a hard error / rescan path.

## P1 – confidence is not meaningful

Current confidence:

```ts
confidence = (pixelConfidence + aiConfidence) / 2
```

Problems:

- without `binary_image`, pixel confidence is zero, so even AI confidence 90 becomes 45,
- edge density is not a calibrated probability,
- LLM self-reported confidence is not a calibrated measurement uncertainty,
- the confidence value is not used to reject bad scans,
- confidence is not persisted in the current `scans` insert.

Recommended output should separate:

```text
product_confidence
variant_confidence
level_confidence
quality_score
estimated_error_ml
```

## P1 – prompt encourages hallucination for impossible cases

The prompt says:

```text
Opaque/frosted bottles: look for weight distribution or any visible boundary
```

A photo cannot infer hidden liquid quantity from weight distribution in an opaque bottle.

Correct behavior for visually opaque bottles should be:

```text
UNMEASURABLE_BY_VISION
```

with fallback to manual input or weight-based measurement.

## P1 – contradictory measurement instructions

The prompt says both:

- account for bottle shape,
- and `fill% = liquid_height / body_height`.

Those are not equivalent for non-cylindrical bottles.

## P1 – product capacity identification needs deterministic checks

Current capacity comes from the LLM with a silent fallback to 700 ml.

Capacity should instead come from a product database after deterministic identification, preferably using:

1. barcode/EAN scanner + checksum,
2. product catalog lookup,
3. OCR/vision as supporting evidence.

Never silently default unknown capacity to 700 ml for a production inventory measurement.

## P1 – duplicate-image hash is inconsistent and weak

Frontend computes SHA-256 over the original image and sends `image_hash`.

The endpoint ignores that value and recomputes a hash from only the first 1000 Base64 characters:

```ts
base64.substring(0, 1000)
```

This should be replaced with a full-content hash or a canonical server-side content hash.

## P1 – service-role authorization issue

The Edge Function creates Supabase using `SUPABASE_SERVICE_ROLE_KEY`, which bypasses Row Level Security.

The user token is authenticated, but `inventory_id` is accepted directly from the request and is not verified to belong to the authenticated user/venue before inserting the scan.

Required server-side check:

```text
inventory_id belongs to user's organization/venue
AND inventory status == open
```

before any scan insert.

## P2 – concurrency / quota issue

The scan counter is implemented as read-then-write:

```text
read scans_this_month
write scans_this_month + 1
```

Parallel requests can race and lose increments. Use an atomic DB function/transaction instead.

## Recommended target architecture

```text
Photo / short burst
  -> quality gate
  -> bottle detection/segmentation
  -> product identification
  -> bottle-variant retrieval
  -> perspective/tilt normalization
  -> liquid-level detector
  -> second independent CV signal
  -> variant calibration V(h)
  -> uncertainty estimator
  -> ACCEPT / RESCAN / MANUAL
```

## Recommended role of the current LLM

Keep the multimodal LLM, but demote it from measurement authority to assistant signal.

Good uses:

- read label text,
- propose brand/product candidates,
- explain why a scan may have failed,
- help bootstrap annotations,
- resolve ambiguous catalog entries.

Do not let it independently set the final `remaining_ml`.

## Immediate next implementation milestone

Before rebuilding the mobile UI, create a Python `vision-lab` that can evaluate labeled bottle images offline.

Minimum first milestone:

```text
input: image + known BottleVariant
output:
  bottle bbox/mask
  normalized liquid level h
  calibrated ml
  quality metrics
  diagnostic overlay image
```

Then test against a ground-truth dataset and report:

- MAE (ml),
- median absolute error,
- P95 absolute error,
- rejection rate,
- accepted-scan P95 error.

Target for supported bottle variants:

```text
P95 accepted-scan error <= 5% nominal bottle capacity
```

For a 700 ml bottle this means <=35 ml.
