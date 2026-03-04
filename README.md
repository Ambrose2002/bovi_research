Proposed DB: **MongoDB (local / edge-first)**

## Why MongoDB (vs Cosmos DB)

- MongoDB runs as the real database engine locally, providing full parity with production environments, whereas Cosmos DB is cloud-native and relies on an emulator for local development.
- Cosmos emulator has platform and development constraints, with limitations on Linux and macOS; MongoDB runs natively on macOS, Linux, and Windows.
- MongoDB’s local behavior closely matches production environments, offering better feature parity; Cosmos emulator has limitations and does not fully reflect all cloud modes.
- MongoDB offers better offline-first sync options that can be optionally used.

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
- How long do we keep files (video, images, etc)?
- What does sensor data look like for different modalities (consider different entities also)
- For videos, what features do we care about?
- For activities, what features do we care about?
- Realistically, what are our options for getting these features (CV for video?)?
- Will embeddings upload to cloud be scheduled or on-demand?

## Pipeline

1. Raw arrives -> Edge API stores raw batches in `observations` (and media pointers in `assets`).
2. Processing worker selects a window (per cow, per time range) and queries raw docs that overlap the window.
3. Worker builds an in-memory tensor of numeric features:
   - Per cow
   - Many cows
4. Worker computes derived artifacts:
   - features (interpretable summaries)
   - embeddings
5. Worker stores outputs in `derived` with `t_start/t_end` and `input_refs`.
6. Uploader worker batches new `derived` records and POSTs them to the cloud uplink endpoint.

## End-to-end flow (ASCII)
```text
Sensors / Farm Systems
  |  (batched data; variable arrival)
  v
+---------------------+        +------------------+
| Edge Ingest API     |        | Local Storage    |
|  /api/ingest/*      |------->| MongoDB          |
+---------------------+        |  - observations  |
                               |  - assets        |
                               |  - streams       |
                               +------------------+
                                        |
                                        | query (cow_id, t_start..t_end)
                                        v
                              +------------------+
                              | Processing Worker|
                              |  build tensor X  |
                              |  encode -> z     |
                              +------------------+
                                        |
                                        | write
                                        v
                              +------------------+
                              | MongoDB          |
                              |  - derived       |
                              +------------------+
                                        |
                                        | batch + POST
                                        v
                              +--------------------------+
                              | Uploader Worker          |
                              |  /cloud/v1/uplink/...    |
                              +--------------------------+
                                        |
                                        v
                              +--------------------------+
                              | Cloud                    |
                              | training / aggregation   |
                              +--------------------------+
```
---

## Minimal Edge API (proof-of-concept)

This is a proof-of-concept API contract (local only, subject to change).

### Endpoints

1. `POST /api/ingest/batch`
2. `POST /api/ingest/asset`
3. `POST /api/process/enqueue`
4. `GET /api/cows/{cow_id}/observations`
5. `GET /api/cows/{cow_id}/derived`

---

### 1. `POST /api/ingest/batch`

**Purpose:** Ingest a batch of raw observations (activity, milk, etc.) for a stream.

**Request body schema:**

- `farm_id`
- `stream_id`
- `entity`
- `modality`
- `t_start`, `t_end`
- `payload_type` (`inline_points` or `asset_ref`)
- `payload` (modality-specific)

**Example: Activity batch**

```json
{
  "farm_id": "farm1",
  "stream_id": "stream123",
  "entity": { "kind": "cow", "id": "COW001" },
  "modality": "activity",
  "t_start": 1718000000,
  "t_end": 1718000600,
  "payload_type": "inline_points",
  "payload": {
    "points": [
      { "t": 1718000000, "steps": 10, "accel_mag": 0.98 },
      { "t": 1718000300, "steps": 15, "posture": "standing" }
    ]
  }
}
```

**Example: Milk session summary**

```json
{
  "farm_id": "farm1",
  "stream_id": "stream456",
  "entity": { "kind": "cow", "id": "COW001" },
  "modality": "milk",
  "t_start": 1718001000,
  "t_end": 1718001100,
  "payload_type": "inline_points",
  "payload": {
    "session_id": "sess-789",
    "measurements": { "milk_yield": 34.2, "temp": 38.1 }
  }
}
```

**Example: Video observation (asset_ref)**

```json
{
  "farm_id": "farm1",
  "stream_id": "stream789",
  "entity": { "kind": "cow", "id": "COW001" },
  "modality": "video",
  "t_start": 1718002000,
  "t_end": 1718002060,
  "payload_type": "asset_ref",
  "payload": {
    "asset_id": "asset_abc123",
    "frame_rate": 24
  }
}
```

**Success response**

```json
{
  "obs_id": "obs_1234567890"
}
```

**Error response**

```json
{
  "error": "invalid stream_id"
}
```

---

### 2. `POST /api/ingest/asset`

**Purpose:** Register a new asset (e.g., video, audio) and create a corresponding observation.

**Request body schema:**

- `farm_id`
- `device_id`
- `entity`
- `t_start`, `t_end`
- `uri` (local file path or object-store key)
- `format` (mp4, wav, etc.)

**Example: MP4 video asset**

```json
{
  "farm_id": "farm1",
  "device_id": "dev_cam01",
  "entity": { "kind": "cow", "id": "COW001" },
  "t_start": 1718002000,
  "t_end": 1718002060,
  "uri": "/data/assets/cow001_vid1.mp4",
  "format": "mp4"
}
```

**Success response**

```json
{
  "asset_id": "asset_abc123",
  "obs_id": "obs_987654321",
  "observation": {
    "payload_type": "asset_ref",
    "payload": {
      "asset_id": "asset_abc123"
    }
  }
}
```

**Error response**

```json
{
  "error": "invalid uri"
}
```

---

### 3. `POST /api/process/enqueue`

**Purpose:** Request background processing for a given observation/asset.

**Request body schema:**

- `obs_id` or `asset_id`
- `process_type` (e.g., "embedding", "feature_extraction")

**Example**

```json
{
  "obs_id": "obs_1234567890",
  "process_type": "embedding"
}
```

**Success response**

```json
{
  "status": "enqueued"
}
```

**Error response**

```json
{
  "error": "not found"
}
```

---

### 4. `GET /api/cows/{cow_id}/observations`

**Purpose:** Retrieve observations for a cow.

**Query params:** `t_start`, `t_end`, `modality`, `limit`

**Example response**

```json
[
  {
    "obs_id": "obs_123",
    "modality": "activity",
    "t_start": 1718000000,
    "t_end": 1718000600,
    "payload_type": "inline_points",
    "payload": {
      "points": [{ "t": 1718000000, "steps": 10 }]
    }
  },
  {
    "obs_id": "obs_124",
    "modality": "milk",
    "t_start": 1718001000,
    "t_end": 1718001100,
    "payload_type": "inline_points",
    "payload": {
      "session_id": "sess-789",
      "measurements": { "milk_yield": 34.2 }
    }
  }
]
```

---

### 5. `GET /api/cows/{cow_id}/derived`

**Purpose:** Retrieve derived features/embeddings for a cow.

**Query params:** `t_start`, `t_end`, `modality`, `limit`

**Example response**

```json
[
  {
    "derived_id": "d_001",
    "derived_type": "embedding",
    "t_start": 1718000000,
    "t_end": 1718000600,
    "value": [0.12, 0.98, -0.34],
    "input_refs": ["obs_123"]
  },
  {
    "derived_id": "d_002",
    "derived_type": "score",
    "t_start": 1718001000,
    "t_end": 1718001100,
    "value": 0.87,
    "input_refs": ["obs_124"]
  }
]
```


---

## Cloud Uplink API (derived-only, v0)

The cloud interface is separate from the local GET endpoints; the default policy is to upload derived artifacts (embeddings/features/scores) rather than raw data.

### `POST /cloud/v1/uplink/derived/batch`

**Purpose:** Upload a batch of derived artifacts to the cloud.

**Request body schema:**

- `farm_id`
- `sent_at`
- `items` (array), each item includes:
  - `derived_id`
  - `entity`
  - `modality`
  - `t_start`
  - `t_end`
  - `derived_type`
  - `value`
  - `model_ref` (optional)

**Example request body**

```json
{
  "farm_id": "farm1",
  "sent_at": 1718003000,
  "items": [
    {
      "derived_id": "d_003",
      "entity": { "kind": "cow", "id": "COW001" },
      "modality": "activity",
      "t_start": 1718000000,
      "t_end": 1718000600,
      "derived_type": "embedding",
      "value": [0.25, -0.11, 0.42]
    },
    {
      "derived_id": "d_004",
      "entity": { "kind": "cow", "id": "COW001" },
      "modality": "milk",
      "t_start": 1718001000,
      "t_end": 1718001100,
      "derived_type": "features",
      "value": { "fat_content": 3.8, "protein": 3.2 }
    }
  ]
}
```

**Success response**

```json
{
  "accepted_ids": ["d_003", "d_004"],
  "rejected": []
}
```

**Error response example**

```json
{
  "accepted_ids": ["d_003"],
  "rejected": [
    { "derived_id": "d_004", "reason": "invalid value format" }
  ]
}
```
