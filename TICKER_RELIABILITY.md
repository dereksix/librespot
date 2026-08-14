# Ticker reliability fork

This fork is the Spotify transport used by the Ticker bar jukebox. It stays
close to `librespot-org/librespot:dev`; Ticker's Node audio supervisor remains
the product-level authority for retries, verified PCM, make-before-break
handoffs, local safety copies, and rollback.

## Patch policy

- Carry only failures reproduced by Ticker's automated jukebox soak or small,
  independently reviewable upstream recovery fixes.
- Preserve the upstream author and commit when importing an existing pull
  request.
- Keep speculative recovery proposals on separate candidate branches.
- Never promote a binary because it merely emits `playing`; fresh PCM and room
  continuity are the acceptance signals.

Current carried changes:

- Upstream PR #1716: keep SPIRC alive after a transient connection-ID update
  failure instead of leaving a healthy-looking but undiscoverable process.
- Tagged transfer/context/connect-state diagnostics for correlating Spotify
  control-plane failures with Ticker's PCM timeline.
- A startup breadcrumb records the exact emulated desktop, numeric protocol and
  SPIRC versions so a production trace can always be tied to its wire profile.
- Upstream PR #1732 preserves an existing OAuth refresh token when Spotify's
  refresh response omits a replacement.
- Audio storage resolution follows the current desktop client's versioned v2
  route first and automatically falls back to the proven legacy interactive
  route.
- Spclient failures identify the HTTP method and route path without exposing
  signed CDN query parameters, authorization headers or response bodies. This
  isolated Ticker's recurring 400 to the optional autoplay-context request;
  production disables librespot autoplay because Ticker owns that queue.

## Current-client drift audit (2026-08-14)

The official Spotify 1.2.96.518 Windows x64 installer was downloaded from
Spotify's CDN using the current WinGet manifest and matched its published
SHA-256. The same `arkadiyt/protodump` process cited in upstream PR #1424 was
run against the signed `Spotify.dll` without launching or authenticating the
client.

- The current client yielded 708 protobuf definitions; librespot's
  1.2.52.442 import contains 478.
- The Connect and player messages retain the field numbers librespot uses. The
  observed changes are predominantly additive, so unknown-field compatibility
  protects the current playback path.
- New Connect capabilities include ping, playlist mixing, remote audio quality,
  Zephyr, gapless playback and crossfade. Do not advertise these until the
  matching behaviors exist.
- Spotify now ships a larger playback-context stack, which warrants continued
  black-box recovery testing even though it does not justify a blind wholesale
  protobuf replacement.
- Do not bump only the emulated version or property-set identifier: those are
  coupled to the matching client behavior and schema set.

The generated dump remains an audit artifact. Import only the smallest schema
or behavior required by a reproduced failure and promote it through Ticker's
PCM-gated compatibility canary and attended live soak.

See [CURRENT_CLIENT_AUDIT.md](CURRENT_CLIENT_AUDIT.md) for the endpoint,
authentication, Dealer, metadata, storage/CDN, audio-key, format, discovery and
audio-backend decision matrix.

## Promotion gate

Build the Linux release binary, record its Git SHA, and install it beside—not
over—the production binary. Then run from the Ticker checkout:

```bash
node scripts/jukebox-soak.js --profile=live --allow-live-audio --low-volume=.18 --rapid --no-wall
```

Promotion requires repeated cold Spotify fixtures, zero backend timeouts, no
audio-engine gap over two seconds, correct last-request-wins behavior, and no
regression in first-track time to verified PCM. Restore the previous binary
immediately if any gate fails.

## Upstream sync

```bash
git fetch upstream
git rebase upstream/dev
```

Resolve and test locally, push the fork branch, and let GitHub Actions finish
before a candidate reaches the MasterServer.
