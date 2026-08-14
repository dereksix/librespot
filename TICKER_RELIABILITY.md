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
