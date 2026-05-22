# manifest_service

Sinatra microservice with two responsibilities:
1. **DRM license proxy** — intercepts player license requests, generates KeyOS auth tokens, proxies to DRM servers
2. **Manifest proxy** — fetches HLS/DASH manifests from origin, rewrites CDN URLs, injects DRM license endpoints

No database. All data sourced from Redis (streams, video info, DRM keys).

## Key files

| File | Purpose |
|------|---------|
| `src/controllers/ManifestController.rb` | HLS/DASH manifest endpoint |
| `src/controllers/LicenseProxyController.rb` | DRM license proxy |
| `src/services/LicenseProxyService.rb` | KeyOS token generation + DRM routing |
| `src/services/HlsService.rb` | HLS manifest generation with CDN URL rewrite |
| `src/services/DashService.rb` | DASH/MPD manifest generation |
| `src/services/StreamService.rb` | Redis stream metadata fetch |
| `src/services/AuthenticationService.rb` | Timestamp + signature validation |
| `src/services/CdnTokenizationService.rb` | Signed CDN URL generation |
| `src/initializers/redis.rb` | Redis connections (streams DB + manifest cache DB) |
| `config/config.yaml` | CDN domains, DRM URLs, KeyOS keys, auth config |

## Manifest route pattern

```
GET /v1/:video_id/:cdn/:cdn_domain/:format/:type/:src/:profile/:resolution/:stream_region/:drm_format/manifest.:ext
```

- CDNs: `akamai`, `limelight`, `cloudfront`, `google`, `fastly`, `cdn_test`
- Formats: `m3u8` (HLS), `mpd` (DASH)
- DRM formats: `na` (no DRM), `dt1` (FairPlay), `dt2` (PlayReady), `dt3` (Widevine), `dt2_dt3` (multi)

## DRM flow

```
POST /v1/license/*
  → validate params (video_id, dt, app_id, user_id, device_id)
  → fetch key_id / container_id from Redis
  → generate KeyOS XML auth token (RSA SHA1 signed, 2048-bit)
  → POST to remote DRM server (dt1=FairPlay, dt2=PlayReady, dt3=Widevine)
  → return binary license to player
```

Multikey DRM: different keys per track type (AUDIO / VIDEO_SD / VIDEO_HD). HD gets higher security level (L4).

FairPlay (`dt1`) transforms request/response format (`spc` + `assetId` → JSON wrapper).

## Manifest flow

```
GET /v1/...
  → Redis: r:{video_id}:playback_streams → stream_id → r:{stream_id}
  → fetch M3U8 / MPD from origin (cached TTL 86400s)
  → rewrite variant URLs to CDN host
  → inject DRM license server URLs (if DRM-enabled)
  → apply CDN tokenization (Cloudfront, Edgio, etc.)
  → return manifest to player
```

## Tests

```bash
make rspec
bundle exec rspec spec/path/to/file_spec.rb
```

Framework: RSpec + Rack::Test. Redis mocked via `STREAM_REDIS.flushall`. Config stubbed via `CONFIG[...]`.
