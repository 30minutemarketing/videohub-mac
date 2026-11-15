# VideoHub for Mac — development roadmap

Target: Apple Silicon arm64 desktop `.app` and `.dmg`
Branch: `feature/macos-desktop`
Baseline: [MACOS-BASELINE.md](MACOS-BASELINE.md)

> GitHub Issues returned HTTP 410 ("Issues has been disabled in this repository") on 2026-11-15. This file tracks the work until the repository owner enables Issues. No issues were created.

## A0 — Baseline and hardware validation
- [ ] Record `uname -m`, `sw_vers`, `node --version`, `git rev-parse HEAD`, remotes
- [ ] Run `npm ci`, `npm run check`, `npm test`, `bash setup.sh --dry-run`
- [ ] Verify sample film startup and live preview on Apple Silicon
- [ ] Verify notes, full MP4 with sound, partial render, VideoToolbox detection
- [ ] Attach real test outcomes and blockers to the baseline document

**Done when:** real-machine results and any reproducible failures are documented. Not yet done.

## A1 — Electron shell
- [ ] Implement separate backend startup, readiness probes and graceful shutdown
- [ ] Add macOS app window and single-instance handling
- [ ] Isolate UI with sandbox/context isolation, restricted navigation and validated IPC
- [ ] Move all mutable state out of the signed application bundle
- [ ] Add `dev:mac` command and automated lifecycle tests

**Done when:** VideoHub UI opens in a Mac application window while retaining project/review functions.

## A2 — Self-contained video rendering
- [ ] Audit Puppeteer/Chromium and Node child-process requirements
- [ ] Bundle and validate compatible arm64 browser runtime
- [ ] Bundle arm64 FFmpeg and ffprobe with VideoToolbox; include required licenses/notices
- [ ] Resolve packaged runtime paths safely, including `app.asar` boundaries
- [ ] Test full/partial video renders, audio, cancellation and cleanup

**Done when:** no user-installed Homebrew, Node or Chrome is necessary for rendering.

## A3 — MCP and installer
- [ ] Add safe, opt-in Claude Code/OpenCode MCP registration and an independent launcher
- [ ] Verify `hub_*` tools when UI closed
- [ ] Build arm64 `.app` and `.dmg` using electron-builder
- [ ] Configure Developer ID signing, hardened runtime, notarization and stapling
- [ ] Test fresh install/update preserving all projects, jobs, notes and exports

**Done when:** macOS accepts a signed, notarized installer and the complete VideoHub workflow is operational.

## Development safeguards
- Main stays untouched until reviewable pull requests are merged.
- Preserve upstream MIT license and attribution.
- Never overwrite existing agent configuration or user video projects.
- Every claim of a passing macOS test must reference an actual Mac run or CI job.
