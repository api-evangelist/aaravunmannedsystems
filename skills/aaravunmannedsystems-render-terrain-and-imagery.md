---
name: Render Aereo Cloud terrain and imagery tiles
description: >-
  Load an Aereo Cloud drone survey into a 3D map client — terrain metadata first, then quantized-mesh
  terrain, orthomosaic raster and vector tiles — and handle the undeclared bearer auth correctly.
api: openapi/aaravunmannedsystems-tile-server-openapi-original.json
base_url: https://tiles.aereo.io
operations:
  - ping_ping_get
  - terrain_metadata_view_terrain_layer_json_get
  - cesium_terrain_metadata_view_cesium_terrain_layer_json_get
  - terrain_tile_view_terrain__z___x___y__terrain_get
  - cesium_terrain_tile_view_cesium_terrain__z___x___y__terrain_get
  - cog_tile_view_ortho__z___x___y__png_get
  - vector_tile_view_vector__z___x___y__pbf_get
  - cog_histogram_tile_view_histogram__z___x___y__png_get
generated: '2026-09-05'
method: generated
source: Derived from the published Tile Server OpenAPI plus probe-verified runtime behaviour.
---

# Render Aereo Cloud terrain and imagery tiles

Aereo (formerly Aarav Unmanned Systems) serves the map layers behind Aereo Cloud from a Tile Server
at `https://tiles.aereo.io`. Every operation below is a `GET` that exists verbatim in the published
OpenAPI.

## Before you start — read this, the spec will mislead you

The published contract declares **no `securitySchemes` and no `servers[]` block**, but the service
enforces auth anyway. Verified by probe:

- `GET /ping` → `200 {"data":"pong"}` with no credential.
- Every other operation → `401` without a credential.

So: send `Authorization: Bearer <token>` on everything except `/ping`. Aereo does not publicly
document how to obtain that token — it is issued to Aereo Cloud tenants. If you have no token, stop
here rather than retrying; the 401 is not transient.

The base URL is `https://tiles.aereo.io`. It is absent from the spec, so a generated client will have
no host configured. Set it explicitly.

## Steps

1. **Confirm the service is up.** Call `ping_ping_get` (`GET /ping`). Expect
   `{"meta": {"success": true, "status_code": 200, ...}, "data": "pong"}`. This needs no token, so it
   is the one call that distinguishes "service down" from "my token is bad".

2. **Fetch the terrain metadata before any terrain tile.** Call
   `terrain_metadata_view_terrain_layer_json_get` (`GET /terrain/layer.json`), or
   `cesium_terrain_metadata_view_cesium_terrain_layer_json_get` (`GET /cesium-terrain/layer.json`)
   when you are driving CesiumJS. The returned `layer.json` tells you the tileset's extent and which
   zoom levels actually exist. Do not guess a `z` — requesting a level the tileset does not carry
   wastes calls and returns an error, not an empty tile.

3. **Request terrain tiles within the advertised levels.** Call
   `terrain_tile_view_terrain__z___x___y__terrain_get` (`GET /terrain/{z}/{x}/{y}.terrain`) or the
   Cesium variant `cesium_terrain_tile_view_cesium_terrain__z___x___y__terrain_get`. The response is
   a quantized-mesh binary. Note the spec types every 200 as `application/json`, which is wrong —
   read the bytes, do not JSON-parse them.

4. **Drape imagery over the terrain.** Call `cog_tile_view_ortho__z___x___y__png_get`
   (`GET /ortho/{z}/{x}/{y}.png`) for the orthomosaic raster at the same `z/x/y` address. Use
   `cog_histogram_tile_view_histogram__z___x___y__png_get` (`GET /histogram/{z}/{x}/{y}.png`) instead
   when you want the histogram-stretched rendering for visual analysis.

5. **Add vector features.** Call `vector_tile_view_vector__z___x___y__pbf_get`
   (`GET /vector/{z}/{x}/{y}.pbf`) for Mapbox Vector Tiles on the same tile addresses.

All five layer types share one XYZ addressing scheme, so a single `z/x/y` walk drives the whole
render.

## Error handling

Errors come back in Aereo's house envelope, not RFC 9457:

```json
{"meta": {"success": false, "status_code": 401, "message": "", "type": "HTTPException", "slug": "", "details": {}}, "data": {}}
```

`meta.slug` is the machine-readable error-code slot and was **empty on every response observed**, so
branch on the HTTP status:

- `401` — missing or invalid bearer token. Not retryable; get a valid token.
- `404` — unknown path, or a tile address outside the tileset. Re-read `layer.json`.
- `405` — wrong method. Every operation here is `GET`; `HEAD` is rejected.
- `422` — request validation failure. This is the only error the spec declares; the body is a FastAPI
  `HTTPValidationError` with `detail[].loc` naming the offending parameter.

No `RateLimit-*` or `Retry-After` headers are returned and no limits are published, so use
conservative concurrency of your own and back off on any 5xx.

## Safety

Every operation is read-only. There is no write, no idempotency concern and nothing to reverse.
