# ADR-0003: Adopt the keypoint frame record as the central motion-data contract

- Status: Proposed (needs team sign-off; does not pre-empt the extractor comparison)
- Date: 2026-08-01

## Context

ADR-0002 rejected the first derived-feature schema (`torso_angle_deg`/`center_y`) as premature, and `CONTEXT.md` lists "a fixed JSON schema before model and classifier comparison" as a non-goal. The feasibility protocol (`.scratch/feasibility/`) requires comparing MediaPipe Pose Landmarker (33 keypoints) and MoveNet SinglePose Lightning (17 keypoints) on the same footage before choosing an extractor.

Since then, the first real extraction exists: a 79-second, 2370-frame MoveNet recording (`examples/keypoints/movenet17-v0-experiment.jsonl`, schema id `movenet-17/v0-experiment`) emitting one JSON object per frame with a schema tag, monotonic timestamps, a detection flag, and named keypoints with per-point scores.

Three consumers need exactly this data now, in one stable shape:

1. **The extractor comparison itself.** Experiment B is only meaningful if both candidates dump into one diffable, replayable record on the same video.
2. **The privacy view.** A skeleton/silhouette rendering needs real keypoints; derived scalars cannot draw one.
3. **Downstream development.** The classification layer, MiMo flow, and both demo UIs need injectable fixtures from hour zero, without betting on the extractor winner.

What ADR-0002 refused to freeze — features, thresholds, keypoint count, extractor choice — stays unfrozen. The record *shape* is a different object: every candidate emits "per-frame keypoints plus per-point confidence"; only the topology (count, names, edges) differs. The shape can be frozen without prejudging the comparison by making topology a registry entry rather than part of the schema.

## Decision

Adopt the de-facto record as contract v1: freeze the shape, register the topologies, defer everything else.

### 1. Frame record

One JSON object per frame, ordered by `timestamp_ms`. JSONL on disk; the same object per message on the live transport (decision D-02).

```json
{"schema": "movenet-17/v0-experiment",
 "frame_index": 0,
 "timestamp_ms": 33.333,
 "torso_detected": true,
 "keypoints": [{"name": "nose", "x_norm": 0.4989, "y_norm": 0.409892, "score": 0.551507}, ...]}
```

| Field | Type | Rule |
|---|---|---|
| `schema` | string | Topology-and-version id; MUST resolve to a registry entry. |
| `frame_index` | int ≥ 0 | Source frame counter; diagnostic only. |
| `timestamp_ms` | number ≥ 0 | Milliseconds since sequence start, monotonic non-decreasing. Never wall-clock time — the demo runs on an accelerated virtual clock. |
| `torso_detected` | bool | Extractor-level "a usable person is present". When `true`, `keypoints` MUST contain exactly the registry entry's points in registry order. When `false`, `keypoints` MAY be empty or carry the extractor's raw low-score output; consumers MUST treat such frames as evidence toward `unknown`, never force-classify them. |
| `keypoints` | array of `{name, x_norm, y_norm, score}` | See `torso_detected` rule for cardinality. |
| `src` | string, optional | `"video"` (offline extraction, default), `"live"`, `"script"`, `"replay"` — the offline path plus the three demo modes. |
| `ext` | object, optional | Extractor-specific extras (e.g. MediaPipe world landmarks, segmentation). Consumers MUST NOT depend on it. |

Coordinate and score rules:

- `x_norm`, `y_norm` are normalized to frame width/height, origin top-left, y pointing down (both candidates' convention).
- Values slightly outside `[0,1]` are legal — extractors estimate off-frame landmarks. Consumers clamp for display; validators MUST NOT reject such frames.
- `score` is the extractor's own confidence in `[0,1]`. Its meaning is extractor-relative: thresholds tuned for one schema id do not transfer to another. The first recording's scores span 0.08–0.97, so consumers must expect and handle low-score points.
- Consumers MUST ignore unknown extra fields (forward compatibility); producers MUST NOT require consumers to read them.

### 2. Topology registry

A shared pure-data table, keyed by schema id, holding the ordered keypoint names and the skeleton edge list (index pairs) used for rendering:

- `movenet-17/v0-experiment` — the 17 COCO-style points of the existing recording;
- `mediapipe-pose-33/v0-experiment` — reserved for the comparison run.

Renderers and feature code read names and edges from the registry, never hardcode them. Switching the production extractor after Experiment B changes the emitted `schema` string and nothing else. Adding a registry entry is not a schema change; changing the frame-record shape is, and bumps the version suffix.

### 3. Explicitly not frozen by this ADR

Per the feasibility gates, all of the following stay open until Experiments B/C/D report:

- the production extractor (MoveNet vs MediaPipe) and keypoint count;
- derived features (torso direction, bbox ratios, velocities) — internal to the classification layer;
- posture and transition labels (working sets only) and every threshold;
- the structured result MiMo receives (decided after the classification gate, within the v3.0 contract fields).

### 4. Privacy boundary (restates ADR-0001)

The frame record and data derived from it are the only motion artifacts that may leave the perception component. Raw frames never enter the record, the transport, fixtures, or MiMo requests. The privacy view renders from this record alone.

## Why this does not violate the "no fixed JSON schema" non-goal

The non-goal targets committing to an extractor, landmark set, features, or thresholds before comparison. This ADR commits to none of those: both candidates are registry entries, and the frozen part is only the observation container the comparison itself needs in order to be replayable — a container the repository is already producing. If a gate surfaces something the record cannot express, the version suffix exists to absorb it.

## Consequences

- Experiment B gains a common measurement container; runs become diffable and replayable frame-by-frame.
- Downstream work starts immediately against `examples/keypoints/movenet17-v0-experiment.jsonl` plus scripted fixtures; nobody waits for the extractor verdict.
- The stickman / privacy view is built once against the registry and survives an extractor swap.
- Detection failure is first-class (`torso_detected` plus scores), which coverage metrics and `unknown` handling both need.
- Accepted v1 limits: single person; no z or world coordinates in the contract (`ext` only); scores not comparable across schema ids; named-object keypoints cost more bytes than flat arrays (~1.5 KB/frame — irrelevant at demo scale, and self-describing wins for debuggability).
- If skeleton replay stays in scope (decision D-10), stored JSONL doubles as the replay format; otherwise storage remains fixture-only.

## Relation to ADR-0002

The rejection stands. This ADR freezes the layer *beneath* the rejected schema: the raw observation record from which such features would be derived. `MotionObservation`, if it survives at all, becomes an internal type of the classification layer, not a cross-team contract.

## Adoption

Requires sign-off from all four members at the contract-freeze meeting, together with D-02 (live transport) and D-06 (ownership boundary: which side consumes this record and produces the v3.0 event fields). Producers must document in the registry entry which `torso_detected=false` payload variant they emit (empty vs raw low-score points).
