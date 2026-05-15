# API

Your API must expose exactly the endpoints listed below. Traffic always arrives through the load balancer on port **8080**, distributed in round-robin across the 3+ instances.

Every response, on any endpoint, **must** include the `X-Instance-Id` header with the identifier of the instance that responded. The load balancer distribution verifier depends on it.

## `GET /healthz`

Liveness probe. Your API must respond with `HTTP 200` while the process is alive. It **must not** query storage — this endpoint only measures whether the process is up.

## `GET /readyz`

Readiness probe. Responds `HTTP 200` when the service is ready to receive traffic (storage connection established, pool mounted) and `HTTP 503` while it is not. Storage must be one of the four engines accepted by the challenge: `redis`, `postgres`, `mariadb`, or `mysql`. The first countable benchmark request is the first one after `readyz` returns `200`.

## `GET /metrics`

Exposes metrics in Prometheus format (`text/plain; version=0.0.4`). Must respond `HTTP 200`.

## `POST /devices/{id}/telemetry`

Ingests a single telemetry point from a device. The `id` in the path identifies the device and must match the pattern `^[a-zA-Z0-9_-]{1,64}$`.

Payload format example:

```json
{
  "ts": 1715800000000,
  "lat": -23.5505,
  "lon": -46.6333,
  "battery": 0.82,
  "ax": 0.11,
  "ay": -0.04,
  "az": 9.81
}
```

### Request fields

| Field | Type | Description |
| --- | --- | --- |
| `ts` | integer (int64) | Point timestamp in epoch millis. Must be positive. |
| `lat` | number | Latitude, in the range `[-90, 90]`. |
| `lon` | number | Longitude, in the range `[-180, 180]`. |
| `battery` | number | Battery level, in the range `[0, 1]`. |
| `ax` | number | Acceleration on the X axis. Must be finite. |
| `ay` | number | Acceleration on the Y axis. Must be finite. |
| `az` | number | Acceleration on the Z axis. Must be finite. |

Required fields: `ts`, `lat`, `lon`, `ax`, `ay`, `az`. The `battery` field is optional.

### Response

Your API must respond with `HTTP 202` and an empty body on success. Persistence may be synchronous or asynchronous — the contract only guarantees the `202`.

| Status | When |
| --- | --- |
| `202` | Point accepted. |
| `400` | Invalid payload (malformed JSON, missing required field, value out of range). |
| `413` | Payload exceeds the size limit. |
| `429` | Rate limited (optional, at the submission's discretion). |

## `POST /devices/{id}/telemetry/batch`

Ingests 1 to 100 telemetry points in a single request. Same `id` pattern as the single endpoint.

Payload format example:

```json
{
  "points": [
    {
      "ts": 1715800000000,
      "lat": -23.5505,
      "lon": -46.6333,
      "battery": 0.82,
      "ax": 0.11,
      "ay": -0.04,
      "az": 9.81
    },
    {
      "ts": 1715800000100,
      "lat": -23.5506,
      "lon": -46.6334,
      "battery": 0.81,
      "ax": 0.09,
      "ay": -0.02,
      "az": 9.79
    }
  ]
}
```

### Request fields

| Field | Type | Description |
| --- | --- | --- |
| `points` | array | List of telemetry points. Minimum 1, maximum 100 items. |
| `points[].*` | — | Each item follows exactly the schema of `POST /devices/{id}/telemetry`. |

### Response

Your API must respond in the following format:

```json
{
  "accepted": 2
}
```

| Status | When |
| --- | --- |
| `202` | Batch accepted. `accepted` carries the number of points persisted. |
| `400` | Invalid batch (empty, malformed JSON, any invalid point). |
| `413` | Batch exceeds 100 points. |

## `GET /devices/{id}/telemetry`

Queries the points of a device within a time window, with cursor-based pagination.

### Query parameters

| Parameter | Type | Description |
| --- | --- | --- |
| `from` | integer (int64) | **Required.** Start of the window, epoch millis (inclusive). |
| `to` | integer (int64) | **Required.** End of the window, epoch millis (inclusive). |
| `limit` | integer | Maximum points per page. Default `100`, minimum `1`, maximum `500`. |
| `cursor` | string | Opaque cursor for the next page. Absent on the first call. |

`from` must be less than or equal to `to`.

### Response

Your API must respond in the following format:

```json
{
  "points": [
    {
      "ts": 1715800000000,
      "lat": -23.5505,
      "lon": -46.6333,
      "battery": 0.82,
      "ax": 0.11,
      "ay": -0.04,
      "az": 9.81
    }
  ],
  "next_cursor": "1715800000000"
}
```

| Field | Type | Description |
| --- | --- | --- |
| `points` | array | Points in the window, ordered by `ts` ascending. Never larger than `limit`. |
| `next_cursor` | string \| `null` | Cursor for the next page, or `null` when there is no more data. |

| Status | When |
| --- | --- |
| `200` | Query successful (even if `points` is empty). |
| `400` | Invalid parameters (`from`/`to` missing, `from > to`, `limit` out of range, invalid cursor). |
| `404` | Device has no data (at the submission's discretion; `200` with empty list is also accepted). |

## `GET /devices/{id}/anomaly`

Computes the z-score of the acceleration magnitude over the last **256** points of the device.

The magnitude of each point is `sqrt(ax² + ay² + az²)`. The z-score is that of the **most recent** point against the mean and standard deviation of that 256-point window. **Caching is not allowed** — the calculation must be performed on every call.

### Response

Your API must respond in the following format:

```json
{
  "z_score": 4.21,
  "samples": 256,
  "anomalous": true,
  "mean": 9.78,
  "stddev": 0.34
}
```

| Field | Type | Description |
| --- | --- | --- |
| `z_score` | number | Z-score of the most recent point's magnitude. |
| `samples` | integer | Number of points used in the calculation (`0`–`256`). |
| `anomalous` | boolean | `true` if `z_score > 3`. |
| `mean` | number | Mean of the magnitudes in the window. |
| `stddev` | number | Standard deviation of the magnitudes in the window. |

Required response fields: `z_score`, `samples`, `anomalous`.

| Status | When |
| --- | --- |
| `200` | Calculation performed. |
| `404` | Device has fewer than 8 points (insufficient samples). |

---

## General rules

- The device `id`, on any endpoint, must match `^[a-zA-Z0-9_-]{1,64}$`. Outside the pattern → `HTTP 400`.
- Every request body is `application/json`.
- Every response includes the `X-Instance-Id` header.
- Graceful shutdown is required: upon receiving `SIGTERM`, the application must drain in-flight requests within 10 seconds.
