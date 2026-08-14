# Spotify current-client compatibility audit

Audit date: 2026-08-14

This document records the compatibility review performed for Ticker's
production librespot fork. The comparison target was the signed Spotify for
Windows 1.2.96.518 client. Its installer matched the SHA-256 published in the
current WinGet manifest. Static protocol artifacts were inspected without
launching the client or authenticating an account.

The production acceptance test is not a successful `playing` event. A candidate
must authenticate, register as a Connect device, resolve metadata and storage,
obtain an audio key, fetch encrypted audio, decode it, and produce fresh PCM.

## Compatibility matrix

| Area | Current-client evidence | Fork status | Decision |
| --- | --- | --- | --- |
| Access-point discovery | `apresolve.spotify.com` still supplies access point, Dealer and spclient hosts. Live resolution returned the same service classes and ports used by librespot. | Current | No change. |
| AP authentication | Stored-credential authentication over the Shannon AP transport produced fresh PCM in the live canary. | Current for the Ticker account | Keep the proven path. External access-token login has a separate upstream regression in issue #1737 and is not Ticker's production credential path. |
| Login5 | The desktop contains both v3 and v4 schemas and endpoints. Librespot's v3 request remains live and authenticated successfully. | Current enough | Do not switch to v4 without implementing and testing its different challenges and interactions. |
| Client token | The desktop still contains `clienttoken/v1`; librespot requests and refreshes it. | Current | No change. Do not blindly replace the client ID or property set. |
| OAuth refresh | Spotify may omit a replacement refresh token when refreshing an access token. Unpatched librespot replaced the saved token with an empty string. | Fixed in this fork | Import upstream PR #1732 and retain the prior refresh token when the response omits one. |
| Dealer WebSocket | Dealer discovery and the access-token WebSocket remain current. Ticker also carries recovery for transient connection-ID failures and session loss. | Hardened | Continue soak testing. PR #1692 overlaps the carried session-recovery work, currently conflicts with `dev`, and previously needed account-handover fixes; do not stack it blindly. |
| Mercury / Connect state | Current protobufs retain the field numbers used by librespot. The desktop adds capabilities and fields, but the observed core changes are additive. | Compatible | Preserve unknown-field compatibility. Import individual fields only for reproduced failures. |
| Autoplay context | Endpoint-aware diagnostics isolated the recurring HTTP 400 to `POST /context-resolve/v1/autoplay`. The same requested track still produced fresh PCM, and upstream issue #1205 records this non-fatal response across several releases. | Optional and rejected for this client profile | Ticker owns its queue and Auto-DJ, so production now runs librespot with `--autoplay off`. Do not classify this 400 as a failed song start; PCM remains authoritative. |
| Metadata | The desktop contains both `extended-metadata/v0/extended-metadata` and a newer v3 batch-extension route. The v0 route works in production. | Current enough | Keep v0 until a missing entity or server retirement is reproduced; then implement a typed v3 fallback. |
| Storage resolve | The desktop uses `storage-resolve/v2/files/audio/interactive/{format}/{file_id}`; upstream librespot used the older `/storage-resolve/files/audio/interactive/{file_id}` route. | Updated in this fork | Pass the selected audio-format ID into storage resolution, try v2 first, and retain the legacy route as an automatic fallback. The PCM canary validates the returned protobuf and downstream CDN path end to end. |
| CDN edge selection | Spotify returns multiple signed CDN URLs using several token formats. Librespot understands `verify`, `__token__`, `Expires`, and timestamp query formats. | Current | Upstream PRs #1524, #1513 and #1722 already provide multi-edge and bad-status fallback. |
| CDN range fetching | Initial fetches fall through across edges and require HTTP 206. Later ranges reuse the selected signed URL. | Acceptable with a known edge | There is an upstream TODO to re-resolve a URL if its signature expires mid-stream. Ordinary bar tracks complete far inside the URL lifetime; implement only after a reproducible long-form failure. |
| Audio keys | The desktop includes newer PlayPlay/media-manifest machinery, while legacy Connect devices still use the AP/Hermes key paths. Ticker's live canary obtained keys for 10 consecutive unique tracks today. A separate 40-track, roughly one-change-per-second capacity probe eventually received service-unavailable responses from Spotify's key service. | Current for normal lossy Connect playback; externally rate-limited under synthetic abuse | Do not pace or restart the client in response to service throttling. Ticker keeps MPD audible, opens a one-minute circuit breaker, and races its independent safety source. Do not attempt to copy or circumvent PlayPlay DRM. |
| Audio formats | The desktop advertises newer FLAC, xHE-AAC, 24-bit and PHONO enum values. Librespot's decoder path supports the formats it requests today. | Intentionally limited | Do not advertise formats the fetch, key and decoder stack cannot safely play. 320 kbps Vorbis remains Ticker's selected bar profile. |
| Discovery | Zeroconf is healthy on Ticker's LAN. Open PR #1724 adds a dual-stack IPv6 bind fallback. | Not implicated | Defer unless the host enables problematic IPv6 discovery; the Web API/device-registration watchdog already verifies availability. |
| Audio backend | Ticker runs librespot's `subprocess` backend into a PCM bridge and the same X32 PipeWire sink used by MPD, not librespot's ALSA backend. This removed the exclusive ALSA lock that previously froze the bridge after exactly 63,744 bytes. | Current | Keep MPD and librespot on the shared PipeWire sink. ALSA-only PRs such as #1703 do not affect production. |

## What “CDN compatibility” means here

Librespot does not contain a permanent Spotify media hostname. It asks
`storage-resolve` for a protobuf containing short-lived, signed edge URLs, then
tries the returned edges and requests byte ranges. The reversed-engineered part
is the storage-resolve request/response contract, token-expiry recognition,
encrypted range mapping and audio-key flow—not a hard-coded list of CDN hosts.

The 2026 audit found one real route drift and fixed it: the selected audio format
is now included in Spotify's v2 storage route. The old storage route remains a
fallback. Multi-edge fallback, signed-token parsing and non-206 failover are
already present. The accepted candidate then traversed storage resolution, the
returned CDN edge, audio-key retrieval, decryption and decoding to fresh PCM;
that is stronger evidence than merely observing a successful HTTP response.

Safe diagnostics log only the HTTP method and route path on failure. Query
strings, signed CDN URLs, authorization headers and response bodies are never
logged, because those can carry bearer-like media tokens.

## Protocol drift

The official client yielded 708 protobuf descriptors versus 478 in the
librespot 1.2.52.442 import. That number does not mean 230 playback messages are
missing: the desktop bundles UI, telemetry, social, offline, video, audiobook,
experimentation and platform services that Ticker never invokes.

The relevant Connect/player messages preserve the identifiers and field numbers
used by librespot. New capabilities include ping, playlist mixing, remote audio
quality, Zephyr, gapless playback and crossfade. Advertising those flags without
their matching behavior would be less compatible, not more.

## Production validation

- Production: `librespot 0.8.0 7a31b92`, SHA-256
  `c86384abfd099f0e8d3b332f6c1552294844a47daa4da4b52da852e022432c6a`.
- The final release gate on 2026-08-14 passed 10/10 consecutive unique-track
  PCM starts in 504–1,104 ms with zero retries. The complete Ticker gate passed
  all five stages and restored queue, radio, wall and X32 state.
- A real `/wall` plus compact-jukebox guest journey passed 16/16 searches and
  2/2 Play taps. The remote Spotify handoff was verified audible in 2,015 ms
  with 0 ms measured dead air because MPD remained hot until fresh PCM existed.
- The endpoint-aware run proved the repeated 400 was the optional autoplay
  request, not storage, CDN, audio-key, metadata, authentication or Dealer.
- Production now disables librespot autoplay. A subsequent canary produced PCM
  without another `/context-resolve/v1/autoplay` request.
- The accepted binary is also retained as
  `/usr/local/bin/librespot-7a31b92` for an exact rollback/install source.

## Dependency security audit

RustSec now runs on every compatibility pull request as well as on its daily
schedule. The 2026-08-14 review updated the compatible patched releases of
`bytes`, `quick-xml`, `anyhow`, `event-listener`, `rand` and the production
`rustls-webpki` line. The `quick-xml` migration explicitly selects XML 1.0 for
Spotify product-info parsing.

Six advisories remain explicitly ignored, rather than silently disappearing:

- `RUSTSEC-2023-0071`: librespot constructs an `RsaPublicKey` and verifies the
  Spotify access-point signature. It never performs the private-key operation
  affected by the Marvin timing attack, and the `rsa` crate has no patched
  release.
- `RUSTSEC-2026-0009`: the patch begins at `time` 0.3.47, which requires Rust
  1.88 and would raise librespot's supported Rust 1.85 baseline. Ticker parses
  bounded Spotify-controlled dates, not attacker-supplied RFC 2822 input.
- `RUSTSEC-2026-0049`, `RUSTSEC-2026-0098`, `RUSTSEC-2026-0099` and
  `RUSTSEC-2026-0104`: these remain only in `rustls-webpki` 0.102 through
  `hyper-proxy2`'s optional rustls feature. Ticker's production build uses
  native TLS, and the active rustls 0.103 line is updated to 0.103.13.

Review these ignores whenever the minimum Rust version, TLS backend or proxy
stack changes. New advisories still fail CI.

## Promotion rule

Every network compatibility change must pass unit tests, the silent production
canary, and an attended live soak. A candidate is accepted only on fresh PCM;
HTTP success, a Connect `playing` state, or process liveness alone are
insufficient. Keep the previous binary beside the candidate for immediate
rollback.
