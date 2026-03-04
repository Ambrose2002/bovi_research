
Proposed DB: **MongoDB (local / edge-first)**

## Purpose
We need a **local edge database** that can:
- Persist **raw multimodal data** (system of record) **per cow over time**
- Support **batched ingestion** with **variable arrival time** (offline / delayed uploads)
- Enable later computation of **tensors/embeddings** and other derived artifacts **locally**

This matches the project direction: keep **raw data on-farm** and treat **embeddings/aggregates** as the main artifacts to send to the cloud later.

---

## Data ownership model
All records are linked to an `entity`:

`entity = { kind: "cow" | "farm" | "pen" | "device", id: string }`

- Most sensor streams will be `kind="cow"`.
- Some modalities are naturally farm/pen-level (e.g., environment, cameras).

---

## Collections (v0)

### 1) `cows`
One document per cow.

**Fields**
- `_id` (cow_id)
- `farm_id`
- `external_ids` (optional; collar id, management system id)
- `meta` (optional; breed, birthdate, parity, etc.)

---

### 2) `devices`
One document per physical sensor/camera/gateway.

**Fields**
- `_id` (device_id)
- `farm_id`
- `device_type` (e.g., `milk_meter`, `camera`, `activity_collar`)
- `attached_to` (optional cow_id)

---

### 3) `streams`
Defines a logical stream: “cow 183 activity collar” or “stall camera 2”.

**Fields**
- `_id` (stream_id)
- `farm_id`
- `entity` (see ownership model)
- `modality` (e.g., `milk`, `activity`, `video`, …)
- `device_id`

---

### 4) `observations` (raw batches, append-only)
**Main raw store**. Each document represents a **batch** that arrived.

**Common fields**
- `_id` (obs_id)
- `farm_id`
- `stream_id`
- `entity`
- `modality`
- `t_start`, `t_end` (time covered by the batch)
- `received_at` (when the batch arrived at the edge service)
- `payload_type` (e.g., `inline_points` | `file_ref` | `asset_ref`)
- `payload` (modality-specific)

**Modality payload examples**

**Activity (time-series, per cow)**
- `payload_type: inline_points`
- `payload.points: [{t, steps, accel_mag?, posture?}, ...]`

**Milk (milking session / event-bounded)**
- `payload_type: inline_points`
- `payload.session_id`
- `payload.measurements: { milk_yield, temp?, flow_rate?, conductivity? }`

**Video (store pointer, not bytes)**
- `payload_type: asset_ref`
- `payload.asset_id`
- Optional `payload.frame_rate`, `payload.resolution`

---

### 5) `assets` (video/audio/image pointers)
Stores metadata + **local URI/path**, not the blob itself.

**Fields**
- `_id` (asset_id)
- `farm_id`
- `entity`
- `device_id`
- `t_start`, `t_end`
- `uri` (local path or local object-store key)
- `format` (mp4, wav, jpg, etc.)

---

### 7) `derived` (local embeddings/features/scores)
Stores outputs of local processing with traceability back to raw.

**Fields**
- `_id` (derived_id)
- `farm_id`
- `entity`
- `t_start`, `t_end`
- `derived_type` (`embedding` | `features` | `score`)
- `value` (vector for embedding, scalar for score)
- `input_refs` (list of `obs_id` and/or `asset_id` used)

---

## Notes / open questions
- Can we settle on some modalities?
- What does data for each modality look like?
- How long do we keep files?
- What does sensor data look like for different modalities (consider different entities also)
- For videos, what features do we care about?
- For activities, what features do we care about?
- Realistically, what are our options for getting these features?

## Pipeline
1. Raw arrives -> store in observations / assets
2. Worker builds features tensor for a window. X (C, T, F)
3. Pass X into an encoder. encoder(X) -> z
4. Store z in derived