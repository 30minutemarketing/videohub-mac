# macOS Apple Silicon baseline — preliminary source audit

Date: 2026-11-15
Repository: https://github.com/30minutemarketing/videohub-mac
Development branch: `feature/macos-desktop`
Source snapshot: latest commit observed in GitHub commit search, `2302ba83581a114f2f41ef094914897fd654c35f`. Verify against `git rev-parse main` before treating it as the definitive upstream baseline.
Status: **Source inspection only; not a passed build or hardware qualification.**

## Goal
Ship an installable Apple Silicon (`arm64`) macOS `.app` and `.dmg` based on the existing VideoHub engine. Preserve live previews, annotation/review notes, render queue, partial re-renders, and Claude Code/OpenCode MCP workflows. Keep original MIT copyright and license notices.

## Inspected source
- `package.json`: ES modules; Node >=22; `npm run check` (syntax), `npm test` (Node test runner); `puppeteer-core`, MCP SDK, Zod dependencies.
- `package-lock.json`: lockfile v3, available for reproducible `npm ci`.
- `scripts/test.mjs`: launches suites in isolated `.test-tmp` with `VIDEOHUB_BASE_DIR` and `VIDEOHUB_DATA_DIR` overrides; tests were **not executed** during this GitHub inspection.
- `server/hub.js`: existing loopback HTTP/SSE server and UI; imports platform, workspace, rendering, and review modules.
- `server/platform.js`, `docs/PLATFORMS.md`: macOS paths, dialogs, process handling, VideoToolbox encoder, and browser discovery are implemented but explicitly await real-machine acceptance.
- `server/util.js`, `server/config.js`: default mutable `data/` resides under install root unless `VIDEOHUB_DATA_DIR` is set. Packaged app must redirect writable data outside its signed/read-only bundle.
- `server/jobs.js`: render queue, subprocess commands, custom render mode with explicit approval path; audit process cleanup and command execution before bundling.
- `server/mcp.js`: stdio MCP server backed by `server/tools.js`; must be launchable outside the UI window and preserve existing user registrations.
- `server/encoders.js`: probes supported encoders, with VideoToolbox candidates on macOS.
- `examples/hello-film/render.js`: Puppeteer-controlled headless browser and FFmpeg; rendering requires executable resolution and an example-film acceptance test.
- `docs/ARCHITECTURE.md`, `docs/PROJECT-CONTRACT.md`: existing API and project conventions that the desktop packaging must preserve.
- `LICENSE`: MIT license requiring preservation of license and copyright notice.

## Preliminary findings / blockers
1. **Hardware unverified.** Upstream documentation explicitly says macOS and Linux have not been tested on real machines. Neither Mac GUI nor rendering success is yet established.
2. **Installed-app write paths.** Default `HUB_DIR/data` will not be suitable for signed `.app` resources; adopt `~/Library/Application Support/VideoHub` for mutable app state, retain user-selected project folders, and verify all path consumers.
3. **Browser and FFmpeg distribution.** Current rendering assumes discoverable local browser/FFmpeg executables. Assess whether Electron's Chromium can satisfy Puppeteer rendering; if not, ship a compatible headless browser with required notices. Package `arm64` FFmpeg and `ffprobe` and review their license obligations.
4. **Node execution inside Electron.** Existing render scripts spawn Node and resolve modules using environment paths; app resources, `app.asar`, Electron's Node mode, executable permissions and child-process behavior require an explicit design and test.
5. **MCP lifetime.** Stdio agent tools must still bootstrap/reach the hub when the desktop window is closed, without destructive edits to Claude Code configuration.
6. **Security.** Preserve loopback binding and host/origin protections, renderer isolation, restricted navigation, explicit custom-command approval, project path containment, and safe handling of untrusted film code.
7. **Compatibility.** Existing Windows/Linux installation behavior and project metadata must not be silently changed by Mac packaging.
8. **Provenance.** Verify whether this repository is a native GitHub fork with upstream association or a repository copy; configure `upstream` remote locally in either case.

## Proposed acceptance sequence
- **A0 — Baseline on real Apple Silicon:** `uname -m` returns `arm64`; record macOS and Node versions; `npm ci`; `npm run check`; `npm test`; `bash setup.sh --dry-run`; separately run the example-film preview and sample MP4 render in a temporary workspace. Save command output and failures in an audit report.
- **A1 — Desktop shell:** create an Electron main process that starts/stops the existing backend, waits for HTTP readiness, opens a restricted window and reports diagnostic failures. No frontend rewrite.
- **A2 — Self-contained rendering:** resolve writable data, browser, Node runtime, FFmpeg/ffprobe and child-process paths from packaged assets. Verify rendered MP4 with audio and partial re-render.
- **A3 — Agent integration:** opt-in reversible MCP registration; check `hub_*` tool discovery and calls with UI closed.
- **A4 — Release:** build ARM64 `.app` and `.dmg`; test on a clean Mac account, then sign, notarize, staple, and document update/data retention.

## Local audit commands (run on Apple Silicon, not yet run)
```bash
git status --short
git rev-parse HEAD
git remote -v
uname -m
sw_vers
node --version
npm ci
npm run check
npm test
bash setup.sh --dry-run
```

## Change discipline
Work on `feature/macos-desktop` and use pull requests into `main`. This first commit is documentation only. Do not run setup/install scripts or edit agent configuration until reviewed. Preserve originals and all generated user video assets.
