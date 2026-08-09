# Agent Brief — Custom Grocery Image Labeler (dictionary-aligned, no object-detection training)

> Hand this document to an external cloud agent. It is self-contained: it
> explains the product, the exact contract, the fork quirks, and the
> definition of done. The agent works in its own sandbox; this file is the
> spec.

## 1. Mission

Build a **fully working on-device grocery item detector** for the ZikZak app
(`~/Developer/zik_zak`) using a **custom TFLite image-classification model
(image labeling)** — NOT a custom object-detection model. The detector's
output labels must align with the app's **embedded grocery dictionary** so
the existing half-gate (dictionary match → visible price flow / no match →
silent lookup) works unchanged.

Product decision already made: the **text-detection** price-compare mode is
LIVE; the produce (apple) and meat (beef) camera modes show a "Coming Soon"
gate in the UI. Your job is the engine behind the gate: when the gate is
lifted, tapping apple/beef must produce real grocery labels from the camera.

## 2. Background — how the feature works today

The scan page (`lib/src/presentation/pages/scan_barcode/`) has three camera
modes: barcode (live), text (live, dictionary-gated price compare), object
(gated behind Coming Soon UI).

The pipeline that already exists for object mode:

```
camera frame / single photo
  → MlKitObjectDetectionService (lib/src/data/services/mlkit_object_detection_service.dart)
      runs in PARALLEL:
        a) bundled ObjectDetector  (ObjectDetectorOptions, base model — 5 generic classes:
           food, goods, place, plant, animal) → gives bounding boxes + coarse class
        b) bundled ImageLabeler    (ImageLabelerOptions base — generic labels)
      fuses: _bestImageLabel (most confident ImageLabel) WINS over object label;
             bounding box taken from the object nearest frame center
  → ObjectDetectionResult(label, confidence, boundingBox)
  → GroceryObjectDetectUseCase (lib/src/domain/usecases/grocery_object_detect/)
      filters confidence <= 0.5, requires the candidate to be stable for ~1 s
      (debounce), suppresses repeats of the same label
  → ScanBarcodeController._resolveDetectedObject
      GroceryItemLookupUseCase.lookupByLabel(label)  →  case-insensitive EXACT match
        against item.detectionLabels
        matched    → visible flow: price overlay, or "choose exact item" chooser
                     when the item has subVarieties
        no match   → _silentPriceLookup: runs the price comparison in the
                     background, logs at fine level, touches NO UI state
```

`GroceryDemoMocks.enabled` (static bool in
`lib/src/data/services/grocery_demo_mocks.dart`, TEMP, default false in
release) replaces the **stream** path with deterministic mock detections so
the UI can be click-tested on an emulator whose virtual scene produces no
real frames. **Keep the mock switch working** — it is the regression harness.

## 3. The dictionary contract (the thing your labels must match)

Dictionary files: `assets/configs/grocery_items/{COUNTRY}.json` (US.json,
TR.json, `_default.json` fallback). Loaded per country from the app locale
by `EmbeddedGroceryItemDataSource`. Matching is done in
`lib/src/data/repositories/data_grocery_item_repository.dart`:

```dart
final normalizedLabel = label.toLowerCase();
for (final detectionLabel in item.detectionLabels) {
  if (detectionLabel.toLowerCase() == normalizedLabel) return item;
}
```

**Exact, case-insensitive string equality — no substring, no fuzzy, no
normalization beyond lowercase.** The model's top label must therefore be one
of these strings (spelling matters, including spaces).

### The US class list (train EXACTLY these 20 labels — primary detectionLabel per item)

| # | model label (exact string) | dictionary item | category | variants? |
|---|---|---|---|---|
| 1 | `tomato` | tomato | produce | 3 (organic/cherry/roma) |
| 2 | `banana` | banana | produce | 0 |
| 3 | `apple` | apple | produce | 1 (organic) |
| 4 | `cantaloupe` | cantaloupe | produce | 1 |
| 5 | `chicken breast` | chicken_breast | meat | 1 (organic) |
| 6 | `ground beef` | ground_beef | meat | 0 |
| 7 | `milk` | milk | dairy | 0 |
| 8 | `eggs` | eggs | dairy | 0 |
| 9 | `cheddar cheese` | cheddar_cheese | dairy | 0 |
| 10 | `bread` | bread | bakery | 0 |
| 11 | `rice` | rice | pantry | 0 |
| 12 | `pasta` | pasta | pantry | 0 |
| 13 | `salmon` | salmon | meat | 0 |
| 14 | `strawberries` | strawberries | produce | 1 |
| 15 | `avocado` | avocado | produce | 0 |
| 16 | `onions` | onions | produce | 0 |
| 17 | `potatoes` | potatoes | produce | 0 |
| 18 | `lettuce` | lettuce | produce | 0 |
| 19 | `carrots` | carrots | produce | 0 |
| 20 | `oranges` | oranges | produce | 0 |

Notes:
- `subVarieties` (organic/cherry/roma …) are NOT model classes — the
  "choose exact item" chooser UI handles them after a base label matches.
- `localizedName` per language is display-only; ignore for the model.
- The dictionary may grow; the model's `labels.txt` must be ordered to match
  the .tflite output indices and contain exactly these 20 strings.

## 4. The two-folded approach (why image labeling, not object detection)

1. The **bundled object detector** stays as-is: it provides (a) the
   bounding box for the HUD and (b) a coarse class (food/goods/place/plant/
   animal) with `classifyObjects: true`.
2. The **custom image labeler** replaces the bundled labeler: it provides
   the fine-grained grocery label (tomato, banana, …) that feeds the
   dictionary.

Fold rule to implement: **only run the custom labeler when the bundled
detector reports a food/plant/animal class** (i.e. something grocery-like);
otherwise skip labeling that frame (saves battery and avoids nonsense
labels on hands, shelves, logos). The labeler labels the whole frame (or the
detected box crop — your choice, document it; whole-frame is acceptable for
phase 1 since the camera centers one item). Whatever you choose, the final
`ObjectDetectionResult.label` must be the custom labeler's top label, and
`confidence` its confidence. Keep the existing `_combineResults` /
`_bestImageLabel` structure — swap the labeler source, don't rewire the
fusion.

## 5. The fork and its API (READ THIS — it differs from upstream docs)

All ML Kit packages come from a FORK via git dependency (do not use
pub.dev versions):

```yaml
google_mlkit_barcode_scanning:
  git: { url: https://github.com/arrrrny/zikzak_ml_kit_flutter.git,
         path: packages/google_mlkit_barcode_scanning, ref: spm-fixed }
google_mlkit_text_recognition:   { same, path: packages/google_mlkit_text_recognition }
google_mlkit_object_detection:   { same, path: packages/google_mlkit_object_detection }
google_mlkit_image_labeling:     { same, path: packages/google_mlkit_image_labeling }
google_mlkit_commons:            { same, path: packages/google_mlkit_commons }
```

Custom image-labeler API in this fork
(`packages/google_mlkit_image_labeling/lib/src/image_labeler.dart`):

```dart
class LocalLabelerOptions extends ImageLabelerOptions {
  final String modelPath;        // ⚠ ABSOLUTE FILE PATH — not an asset path
  final int maxCount;            // default 10
  final double confidenceThreshold; // default 0.5
  ImageLabeler({required LocalLabelerOptions options});
}
```

**Critical fork quirk:** the Android native code does
`LocalModel.Builder().setAbsoluteFilePath(path)` — there is NO asset-path
support. You must **copy the bundled .tflite from Flutter assets to a
writable absolute path** (e.g. `getApplicationSupportDirectory()` from
`path_provider`) once at startup, and pass that path to
`LocalLabelerOptions(modelPath: ...)`. (iOS side of the fork: verify the
same — handle both platforms the same way; copying to a temp dir works for
both.)

## 6. What to build

### 6a. Train the model

- Use **TFLite Model Maker** (Python) with an ImageClassifier (e.g.
  EfficientNet-Lite0 or MobileNetV3-Small, input 224×224).
- Classes: the exact 20 strings from the table above.
- Dataset: source per class from public datasets (Fruits-360, Food-101,
  Open Images v7 `food`/`produce`/`meat` subsets, Kaggle grocery sets).
  Target ≥ 30–50 usable images per class after augmentation (flips,
  slight rotation/brightness). Real-store shelf lighting beats studio
  shots — prefer images of loose/unpackaged items.
- Train/val split ~85/15, early-stop on val accuracy. Export **float16 or
  int8-quantized** .tflite (target ≤ ~5 MB; the app ships it embedded).
- Deliverable: `grocery_labeler.tflite` + `grocery_labeler_labels.txt`
  (20 lines, EXACT strings, index-ordered) + a short training report:
  per-class accuracy, top-1/top-5, val confusion notes, model size, and
  inference latency estimate.

### 6b. Wire it into the app

- Bundle: `assets/models/grocery_labeler.tflite` (the repo registers
  `assets/` wholesale — no pubspec change needed; verify).
- New small service/setup step: on first use, copy the asset to an
  absolute temp path (guard against multiple copies; cache the path).
- In `MlKitObjectDetectionService`: build the `ImageLabeler` with
  `LocalLabelerOptions(modelPath: <absolute>, confidenceThreshold: 0.5)`
  instead of the bundled options. Implement the fold gate from §4 in both
  the stream path and `detectFromSingleFrame`.
- Keep `GroceryDemoMocks.enabled` behavior byte-for-byte (mock replaces the
  whole detection when enabled — do not let the real pipeline run under it).
- Add a **debug-only override** (e.g. `static bool debugForceRealObject =
  false;` next to the mock switch, TEMP-commented) that bypasses the mock
  so the real pipeline can be exercised on a device without editing code.
- Do NOT touch: text recognition path, barcode path, dictionary files,
  lookup repository, controller half-gate, price overlay UI, Coming Soon
  gate.

## 7. Verification (must all pass before you report done)

1. `flutter analyze` → no NEW issues (12 pre-existing infos exist: dio
   test import, rebate_alert formals, app_notification context, scripts
   lints — leave them alone).
2. Mock regression: with `GroceryDemoMocks.enabled = true`, the emulator
   click-flow still works: object mode emits the cycle (tomato → chooser,
   avocado → direct prices, "unrecognized object" → silent, camera stays
   clean).
3. Real pipeline (debug override on): with the custom model loaded, point
   the camera at a real grocery item (or a printed photo of one):
   - label comes from the custom model and matches the dictionary
     (e.g. a tomato photo → `tomato` → price overlay / chooser);
   - non-grocery scene → no label emission (fold gate works);
   - confidence flows through (ObjectDetectionResult.confidence is the
     labeler's, > 0.5 filter + 1 s debounce still apply).
4. Model sanity on the emulator if possible; **real-device verification is
   the acceptance target** — if you cannot reach a device, state it clearly
   in the report (see §8 emulator caveat).
5. Latency: single-frame detection (shutter) should return in well under
   1.5 s on a mid-range device; stream path throttled ~2 fps as today.

## 8. Environment facts

- Repo: `github.com/arrrrny/zik_zak` (private), branch to work on:
  `060-photo-grocery-price-compare` (create your own branch from it, e.g.
  `060-grocery-image-labeler`). Read `AGENTS.md` at the repo root first.
- App is Flutter (stable channel, Dart SDK current), `get_it` DI, zuraffa
  framework conventions (usecases/presenters/services layout under
  `lib/src/`).
- Existing files you will touch:
  - `lib/src/data/services/mlkit_object_detection_service.dart` (the only
    ML Kit service for object mode; all ML Kit types stay confined here)
  - `lib/src/data/services/grocery_demo_mocks.dart` (add the debug flag
    next to `enabled`)
  - `assets/models/…` (new)
- Known emulator caveat: the REMOTE Android emulator used for UI testing
  (API 36, x86_64) **SIGSEGVs inside the native ML Kit pipeline when the
  bundled object detector runs in streaming mode** (crash in
  libmlkitcommonpipeline). Text recognition streams fine. Mitigation:
  verify the real pipeline via the single-frame path on the emulator, and
  treat streaming verification as real-device-only. Do not "fix" this with
  hacks — just document it.
- Reference for a minimal real-detection harness: the standalone probe
  `~/Developer/mlkit_probe` (Flutter, forks only, compute isolate) already
  proved real tomato photos → "Food 71% / Fruit / Vegetable" and price-tag
  text ("PARSLEY / 99¢ / PER EACH"). You can replicate that harness
  pattern inside the app or a scratch folder; it is not shipped.

## 9. Hard constraints

- No new pub.dev ML Kit packages — only the fork above.
- No changes to dictionary schema/labels, lookup matching, half-gate,
  text/barcode paths, or the price overlay UI.
- Keep the diff minimal and readable; follow the file's existing style
  (Loggable mixin, logger, existing comments).
- Do not remove `GroceryDemoMocks` — it is the emulator regression harness.
- No training data committed to the app repo unless it is small and
  licensed; ship the trained .tflite + labels + report, not datasets.
- Commit convention: conventional commits (`feat(scan): …`), one logical
  commit per step, push your branch, open a PR against
  `060-photo-grocery-price-compare` (or deliver a git patch if you cannot
  push).

## 10. Definition of done

- [ ] 20-class custom labeler model bundled + loaded from absolute path
- [ ] Fold gate implemented (labeler only when detector says food-ish)
- [ ] Stream + single-frame paths use the custom labeler; fusion unchanged
- [ ] Mock regression passes; debug override added
- [ ] `flutter analyze` clean (no new issues)
- [ ] Real-device verification performed OR explicitly documented as pending
- [ ] PR/patch with training report, model size, per-class quality
