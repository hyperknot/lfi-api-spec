# Live Flight Ingest API (LFIA)

Push-only, server-to-server ingest for live tracking of paragliders and hang gliders. This document uses RFC 2119 keywords (MUST/SHOULD/MAY).

## What this is

- LFIA lets a manufacturer's server (**Publisher**) push live tracker points and task metadata to a tracking server (**Ingestor**).
- **Push-only**: the **Ingestor** never calls the **Publisher**. There are no pull, webhook, or callback flows.
- The **Publisher** sends:
  - Raw, unsorted GPS points in small batches, as they are produced.
  - Periodic metadata saying which trackers belong to which groups/competitions and the active task definition.
- The **Ingestor** takes responsibility for:
  - Sorting and deduplicating points per tracker.
  - Mapping tracker IDs into groups/competitions and their tasks for scoring/visualization.

## Roles

- **Publisher** (aka Manufacturer): You operate the devices and a server that pushes data. You know your manufacturer ID (`manufacturer_id`).
- **Ingestor**: Receives data, validates, stores, deduplicates, and associates it to groups/tasks. The **Ingestor** runs the endpoint URLs and issues tokens.

## Scope

- **Direction**: **Publisher** → **Ingestor** only.
- **Format**: JSON over HTTPS; UTF-8; gzip-compressed request bodies.
- **Coordinate system**: WGS84.
- **Authentication**: Bearer token REQUIRED.
- **Endpoints**:
  - `POST /raw-data`
  - `POST /metadata`

---

## Transport and envelope

- HTTPS only.
- **Headers**:
  - `Content-Type: application/json`
  - `Content-Encoding: gzip` (REQUIRED)
  - `Authorization: Bearer <token>` (REQUIRED)
- **Character encoding**: UTF-8
- **Base URL**: Chosen by the **Ingestor** (may include a version path like `/ingest/v1`). All endpoints are relative to this base URL.

---

## Authentication (Bearer tokens)

- Required on every request: `Authorization: Bearer <token>`
- **Token format**:
  - **Characters**: uppercase letters, lowercase letters, digits.
  - **Length**: Tokens SHOULD be at least 32 characters long.
  - **Example**: `8Kp1xA3dE6fQ9LmN2rT5vW8yZ1bC4hJ`
- **Provisioning** (what this means in practice):
  - **Out-of-band**: Tokens are exchanged outside this API (e.g., in person or email).
  - Each token is assigned to exactly one `manufacturer_id` by the **Ingestor**.
  - The **Ingestor** MAY keep multiple active tokens for the same `manufacturer_id` to allow rotation (issue new token, switch sender, then revoke old).
  - Treat tokens as secrets; do not log or expose them.
- **Response policy for auth errors**:
  - Errors are always returned as JSON (see "Responses" section).

---

## Common data rules

- **Coordinates**: `lat`/`lon` in decimal degrees (WGS84). **Publishers** SHOULD send 6 decimal places.
- **Altitude**: `alt` in integer meters above the WGS84 ellipsoid (GPS altitude).
- **Optional static pressure**: `p` in integer pascals (if available).
- **Timestamps**:
  - Raw points: UTC, integer seconds since Unix epoch.
  - XCTSK task times: passed through as given by the XCTSK format (see link below).
- **Unknown fields**: **Ingestors** ignore unknown JSON fields (forward compatibility).
- **Payload size**: The **Ingestor** MAY reject very large requests; a soft limit of 10,000 raw points per request is RECOMMENDED.
- **Identifier rules** (`group_id` and tracker IDs):
  - `manufacturer_id` (like "flymaster"): string matching regex `[a-z0-9_]+`
  - **Tracker IDs**: `tid` in `/raw-data` and keys in `tracker_id_map`. Any non-empty ASCII string, case-sensitive. **Publishers** SHOULD keep tracker IDs stable over time.
    - `tid` stands for "tracker ID."
  - `group_id`: any non-empty ASCII string, case-sensitive. This is the stable identifier for a logical group/competition across renames.
  - `group_name`: human-friendly display name for the group; MAY change at any time. Not used for identity or matching.
  - **Reserved `group_id`**: `__FREEFLYERS__`
    - Use this to group free-flying trackers that are not in a competition task.
- **Labels**:
  - The `label` field under `tracker_id_map` values is a free-form display label for a tracker within a group. It MAY be any UTF-8 string (e.g., "Bus 1", "Spare 3"), is not used for identity or scoring, and MAY change at any time.

---

## Endpoint: `/raw-data`

### Purpose

Push live raw GPS/baro points from any number of trackers. Batches may be out-of-order and contain duplicates or backfilled points.

### Request body (schema)

- `manufacturer_id` (string; required): Manufacturer ID, regex `[a-z0-9_]+`
- `raw_data` (array; required): Each element is a point object:
  - `tid` (string; required): Manufacturer's tracker identifier (tracker ID); any ASCII string (case-sensitive)
  - `t` (integer; required): UTC seconds since epoch
  - `lat` (number; required): latitude in degrees
  - `lon` (number; required): longitude in degrees
  - `alt` (integer; required): meters above WGS84 ellipsoid
  - `p` (integer; optional): static pressure in pascals

### Processing rules (Ingestor)

- **Time window and skew**:
  - SHOULD drop points with `t` > `now_utc` + 5 seconds (future beyond small clock skew).
  - SHOULD drop points with `t` < `now_utc` − 24 hours.
- **Deduplication**:
  - SHOULD deduplicate within the same `manufacturer_id` on (`tid`, `t`).
- **Ordering**:
  - Batches MAY be unsorted; the **Ingestor** is responsible for ordering as needed.
- **Size**:
  - The **Ingestor** MAY reject a request deemed too large (recommended soft limit: 10,000 points).

### Responses

- **Success**: `200` with JSON: `{"status":"ok"}`
- **Failure**: always JSON with an error message, for example:
  - `{"error":{"message":"Body could not be parsed as JSON"}}`

### Example request body

```json
{
  "manufacturer_id": "flymaster",
  "raw_data": [
    { "tid": "f2d24a", "t": 1718130000, "lat": 47.123456, "lon": 12.654321, "alt": 1832, "p": 970123 },
    { "tid": "f2d24a", "t": 1718130060, "lat": 47.123555, "lon": 12.654222, "alt": 1841 },
    { "tid": "g9x88z", "t": 1718130030, "lat": 47.001122, "lon": 12.331144, "alt": 1599 }
  ]
}
```

---

## Endpoint: `/metadata`

### Purpose

Provide mappings from manufacturer tracker IDs (`tid`) to pilot/competitor IDs and the current competition tasks. This lets the **Ingestor** associate tracks with pilots and groups/competitions. Identity is based on `group_id` (stable); `group_name` is display-only and may change at any time.

### Cadence and window

- **Publishers** SHOULD send metadata approximately every 10 minutes.
- Include ALL groups/tasks that are active now OR were active within the last 24 hours.
  - This endpoint is snapshot-style, not incremental: send the full current set each time (for the last 24h window).
  - The latest valid push overwrites earlier state for the same `manufacturer_id` and `group_id` (last-write-wins).
  - A tracker may appear in multiple groups (e.g., a competition and `__FREEFLYERS__`) at the same time.

### Request body (schema)

- `manufacturer_id` (string; required)
- `tasks` (array; required): each task object contains:
  - `group_id` (string; required)
    - Any non-empty ASCII string (case-sensitive). Stable identifier for this logical group across renames.
    - **Reserved `group_id`**: `__FREEFLYERS__`
      - Use this to group free-flying trackers that are not in a competition task.
  - `group_name` (string; optional)
    - Human-readable label for display. MAY change between pushes without changing identity.
  - `tracker_id_map` (object; required)
    - **Keys**: tracker `tid`s (ASCII strings; case-sensitive).
    - **Values**: mapping objects describing pilot/competitor identifiers ("target IDs") and optional display metadata:
      - Recognized optional keys include:
        - `civl_id`: integer (FAI CIVL ID). Should be provided as an integer, but if supplied as a string, the **Ingestor** MUST automatically convert it to integer.
        - `dhv_id`/`ffvl_id`/`appi_id` etc.: alternative local competition IDs. Treated as opaque string by the **Ingestor**.
        - `label`: string. Free-form display label for this tracker within this group (e.g., "Bus 1", "Spare 3"). Not used for identity or scoring.
  - `task_details` (object; optional; omit for `__FREEFLYERS__`)
    - When present, MUST contain an `xctsk` object (see next section). The **Ingestor** passes it through without modification.

### Processing rules (Ingestor)

- Treat each push as the latest snapshot.
- Last-write-wins by (`manufacturer_id`, `group_id`). **Publishers** MAY change `group_name` without changing identity.
- Allow a tracker to exist in multiple groups simultaneously, including `__FREEFLYERS__`.

### Responses

- **Success**: `200` with JSON: `{"status":"ok"}`
- **Failure**: always JSON with an error message, for example:
  - `{"error":{"message":"Missing manufacturer_id"}}`

### Example request body

```json
{
  "manufacturer_id": "flymaster",
  "tasks": [
    {
      "group_id": "europeans_2024",
      "group_name": "PG Europeans Pegalajar — Task 1",
      "tracker_id_map": {
        "tx123": { "civl_id": 12001 },
        "tx124": { "civl_id": 12001 },
        "tx125": { "civl_id": 12003 },
        "tx201": { "label": "Bus 1" }
      },
      "task_details": {
        "xctsk": {
          "taskType": "CLASSIC",
          "version": 1,
          "turnpoints": [
            { "type": "TAKEOFF", "radius": 200, "waypoint": { "name": "TO", "lat": 46.0, "lon": 13.0, "altSmoothed": 980 } },
            { "type": "SSS",     "radius": 1000, "waypoint": { "name": "Start", "lat": 46.05, "lon": 13.1, "altSmoothed": 1400 } },
            { "type": "ESS",     "radius": 400,  "waypoint": { "name": "Goal", "lat": 46.1, "lon": 13.2, "altSmoothed": 500 } }
          ]
        }
      }
    },
    {
      "group_id": "__FREEFLYERS__",
      "group_name": "Free Flyers",
      "tracker_id_map": {
        "tx202": { "ffvl_id": "ABC-123" }
      }
    }
  ]
}
```

---

## XCTSK task details

- When `task_details` is present, it MUST contain an `xctsk` object following the XCTSK specification **version 1**.
- The **Ingestor** treats `xctsk` as a pass-through blob; it does not reinterpret its times or geometry.
- **Specification reference**: [XCTrack Competition Interfaces](https://xctrack.org/Competition_Interfaces.html)

### Simplified sample

```json
{
  "taskType": "CLASSIC",
  "version": 1,
  "earthModel": "WGS84",
  "turnpoints": [
    {
      "type": "TAKEOFF",
      "radius": 200,
      "waypoint": { "name": "TO", "lat": 46.000000, "lon": 13.000000, "altSmoothed": 980 }
    },
    {
      "type": "SSS",
      "radius": 1000,
      "waypoint": { "name": "Start", "lat": 46.050000, "lon": 13.100000, "altSmoothed": 1400 }
    },
    {
      "type": "ESS",
      "radius": 400,
      "waypoint": { "name": "Goal", "lat": 46.100000, "lon": 13.200000, "altSmoothed": 500 }
    }
  ],
  "takeoff": { "timeOpen": "09:30:00Z", "timeClose": "11:00:00Z" },
  "sss": { "type": "ELAPSED-TIME", "timeGates": ["11:00:00Z","11:15:00Z","11:30:00Z"] },
  "goal": { "type": "CYLINDER", "deadline": "17:00:00Z" }
}
```

---

## Responses (uniform JSON)

- **Success** (both endpoints):
  - **Status**: `200`
  - **Body**: `{"status":"ok"}`
- **Failure** (both endpoints):
  - **Status**: non-`200` (implementation-defined)
  - **Body** (always JSON):
    ```json
    {
      "error": { "message": "Human readable explanation" }
    }
    ```

### Examples of possible error messages (non-exhaustive)

- "Authorization token missing"
- "Authorization token invalid"
- "Token not authorized for manufacturer_id"
- "Content-Encoding gzip required"
- "Body could not be parsed as JSON"
- "Missing required field: manufacturer_id"
- "raw_data must be a non-empty array"
- "Too many points in one request"

---

## Validation summary (who does what)

### Publisher MUST

- Use HTTPS.
- Gzip-compress all request bodies.
- Include `Authorization: Bearer <token>` on every request.
- Send well-formed JSON with UTF-8 encoding.
- Send metadata about every 10 minutes and include ALL groups/tasks active now or within the last 24 hours.
- Keep `group_id` stable across any renames of `group_name`.

### Ingestor SHOULD

- Verify the Bearer token and its association to the provided `manufacturer_id`.
- Ignore unknown JSON fields.
- For `/raw-data`:
  - Drop points with `t` > `now_utc` + 5 seconds.
  - Drop points with `t` < `now_utc` − 24 hours.
  - Deduplicate on (`manufacturer_id`, `tid`, `t`).
  - Accept out-of-order and mixed-tracker batches.
  - Optionally reject very large batches (soft limit ~10,000 points).
- For `/metadata`:
  - Treat each push as the latest snapshot for the last 24 hours.
  - Use (`manufacturer_id`, `group_id`) as the identity for last-write-wins.
  - Allow a tracker to exist in multiple groups simultaneously, including `__FREEFLYERS__`.
- Always respond with JSON per "Responses" above.

---

## Security considerations

- Always use HTTPS; never send tokens over plaintext.
- Do not log `Authorization` headers.

---

## Versioning and compatibility

- The base URL may include a version component chosen by the **Ingestor** (e.g., `/ingest/v1`).
- Unknown fields are ignored to allow forward-compatible extensions.
