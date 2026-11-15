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

---

# A0 run 1: real Apple Silicon results (2026-11-15)

Run locally on the owner's Mac by Claude Code (desktop app, local session). Logs were kept outside the repository.

## Environment
| Item | Value |
|---|---|
| `uname -m` | `arm64` |
| macOS | 26.6.2 (build 25G83) |
| Node / npm | v26.5.0 (Homebrew, native arm64) / 11.17.0 |
| FFmpeg / ffprobe | 8.1.2 Homebrew, native arm64; `h264_videotoolbox`, `hevc_videotoolbox`, `prores_videotoolbox` present |
| Browser | Google Chrome at `/Applications/Google Chrome.app` (no Edge or Chromium) |
| Git HEAD | `89c9c384371c122fae507a6f1626ebc6fd95113a` (`feature/macos-desktop`) |
| `main` / `origin/main` | `2302ba83581a114f2f41ef094914897fd654c35f` (matches preliminary snapshot) |
| Remotes | `origin` only, `https://github.com/30minutemarketing/videohub-mac.git`; no `upstream` configured |

## Automated results
| Command | Result |
|---|---|
| `npm ci` | **Pass.** 119 packages. `npm audit`: 2 moderate (transitive `hono`, `@hono/node-server` via the MCP SDK); `npm audit fix` available, not applied. |
| `npm run check` | **Pass.** |
| `npm test` | **340 pass, 1 fail, 1 skipped** of 342 (about 1.2 s). After removing `open.vbs`/`open.ps1`: **341 pass, 0 fail, 1 skipped**. |
| `bash setup.sh --dry-run` | **Pass**, run with `HOME`, `XDG_*` and `VIDEOHUB_DATA_DIR` pointed at a throwaway folder; it planned the workspace, config, Claude Code MCP registration, skill, rule, `Desktop/VideoHub.command` and example film, and wrote nothing. |
| `node scripts/init-config.mjs --detect` (isolated) | **Pass.** `{"available":["videotoolbox","x264"],"best":"videotoolbox","gpus":[]}` |
| `platform.findBrowser()` / `findTool()` | Chrome, `/opt/homebrew/bin/ffmpeg` and `/opt/homebrew/bin/ffprobe` resolved correctly. |

All macOS-specific suites passed, including `server/platform.darwin.test.js`.

### Failure: `scripts/windows-safety.test.js`, "Windows clickable lifecycle entry points invoke Node directly without policy overrides"
The test requires `open.vbs` and `open.ps1` to be absent. Both are still tracked (last touched by browser uploads `3e9b9cd` and `192239f`). This is a repository-content problem, not an Apple Silicon problem: these legacy Windows launchers were retired upstream but the upload never deleted them. **Fixed** on `feature/macos-desktop` by removing both files (approved by the owner). `open.vbs` also contained a forbidden `-ExecutionPolicy Bypass`.

### Skipped
"Windows agent batch shims preserve special path characters..." is Windows-only (`# Windows batch execution`). Expected on macOS.

### Other observations
- **No `.gitignore` or `.github/`.** Browser upload can't carry dot-prefixed files, so `node_modules/` and `.test-tmp/` show as untracked after testing. `scripts/upload-rules.test.js` also fails if any extra dot-folder exists in the checkout. Add a `.gitignore` through git, not browser upload.
- `gpus` is empty on Apple Silicon (expected: no discrete GPU list). VideoToolbox detection doesn't depend on it.
- `claude` and `opencode` CLIs are not on `PATH` (Claude Code runs inside the desktop app). A real setup would fall back to editing `~/.claude.json`. Settle this before A3.

## A0 run 2: live hub, preview and notes (2026-11-15)
Isolated test hub: `VIDEOHUB_PORT=7799`, data, workspace and a copy of `hello-film` under `.test-tmp/a0-live`, `VIDEOHUB_GUARD=off`, `VIDEOHUB_JOBS=off`. The test hub's config listed only the scratch films folder, so real Desktop/Documents folders were not scanned.

| Check | Result |
|---|---|
| Hub startup | **Pass.** Listening on `127.0.0.1:7799` only. |
| Foreign `Host` header | **Pass.** `403`. |
| `hub_workspace`, `hub_project` | **Pass.** Example found as chunked (partial re-renders available), 30 fps, 10 s. |
| `hub_live` opens window | **Pass.** VideoHub opened a Chrome `--app` window on macOS; film page, `film.js` and `audio/mix.wav` (882 KB) served `200`; Chrome connected. |
| Owner watched preview with audio | **Pass** (owner reviewed and left notes). |
| Frame notes | **Pass.** Owner left 3 notes (frames 129, 129, 244); saved with frame, crop and annotation JPEGs in the film's `.videohub/`; read back through `hub_notes`. |
| Fix code, live reload, `hub_note_update` | **Pass** for code edit and status updates (all 3 marked fixed with replies). Owner confirmed the open preview reloaded by itself. |

### Bug: Windows path separators in tool text on macOS
`hub_workspace` prints `films\<film-name>` and `templates\new-film` on macOS (`server/tools.js`, hard-coded `\\`). Display text only; real paths are correct. Fix: use `path.join` or `path.sep`.

## A0 run 3: full render (2026-11-15)
Test hub restarted with the job queue on (`VIDEOHUB_JOBS` unset), still `VIDEOHUB_GUARD=off` and scratch data. Owner requested the render after approving the preview.

| Check | Result |
|---|---|
| `hub_render` mode `full` through the queue | **Pass.** 5/5 chunks, 300 frames in 4.9 s (60.9 fps); job done in 6 s. |
| Encoder | **Pass.** Render log `encoder: videotoolbox (h264_videotoolbox)`; MP4 stream tag `h264_videotoolbox`. |
| Video | **Pass.** H.264 High, 1920x1080, 30 fps, 300 frames, 10.000 s, 1.5 MB. |
| Audio | **Pass.** AAC LC stereo from `audio/mix.wav` (22,050 Hz, same as source), muxed with `-shortest` and `faststart`. |
| Fixes present in render | **Pass.** Frames at 5.0 s and 8.2 s show the rectangle, red triangle and "Fix It In Code!". |
| Outputs folder | **Pass.** Hard-linked into `workspace/outputs/2026-11-15/`. |
| Process cleanup | **Pass.** No headless Chrome or FFmpeg left from this render (other headless browsers on the machine belong to unrelated tools and predate the run). |
| RAM advisory | Works on macOS: reported Chrome using 18.9 GB as a suggestion, closed nothing. |

## A0 run 4: partial re-render, cancel, render protection (2026-11-15)

### Partial re-render: **Pass**
A test change to the outro subtitle (scratch copy only), then `hub_render` mode `partial` with seconds `[[7.8,10]]`:
- Only chunks 3 and 4 (6–10 s) were re-rendered, 120 frames in 1.9 s with VideoToolbox; job took 3 s.
- `hello-film_v2.mp4` is 300 frames, 10.000 s. Per-frame MD5 (`framemd5`): frames 0–179 are identical to v1; the first changed frame is 237 (7.9 s, where the subtitle fades in).
- Decoded audio MD5 is identical between v1 and v2, so the splice keeps audio in sync.
- A frame at 8.8 s shows the new subtitle.

### Cancel mid-render: **Failed on macOS (bug), fixed in run 5**
Cancelling (`POST /api/jobs/:id/cancel`) marks the job cancelled and no version is created, but the render does not stop at once:
1. `killTree` (`server/platform.js`, Unix) sends SIGTERM to the render's process group and each known descendant, waits 3 s, then sends SIGKILL to survivors from the **original** snapshot.
2. `render.js` uses `puppeteer.launch()` with the default `handleSIGTERM: true`. Puppeteer's handler (`@puppeteer/browsers/lib/launch.js`, `#onDriverProcessSignal`) closes the browser on SIGTERM but **does not exit**; installing the listener also overrides Node's default exit on SIGTERM.
3. So `render.js` survives SIGTERM, sees FFmpeg die (code 255), retries the chunk, **launches a new Chrome**, and keeps rendering for about 3 s (2–4 more chunks were written after the cancel).
4. SIGKILL then kills `render.js`, but the new Chrome isn't in the original snapshot, and Puppeteer starts it in its own process group, so it is **orphaned** (reparented to PID 1, still running at +10 s). Reproduced with a 60 s render; the orphan was stopped by hand.

Effects: orphaned headless Chrome processes pile up after each cancel; chunk files in `render/chunks` are rewritten after the user cancelled. Windows is unaffected (`taskkill /T /F` force-kills at once).

Chunks are written to `c_NNN.mp4.tmp.mp4` and renamed when complete, so a cancel never leaves a half-written chunk.

### Render protection (`VIDEOHUB_GUARD` on): **Pass**
Test hub restarted with `HOME` set to a throwaway folder (protection settings live in `~/Library/Application Support/VideoHub` on macOS), protection on, a 30 s render:
- `render.js`, its Chrome, and FFmpeg processes started later in the job all ran at nice 10 (low priority) throughout.
- The job finished normally in 12 s; nothing was written to the fake home (defaults used).
- macOS has no CPU affinity, as documented; priority and background QoS (`taskpolicy`) are the controls.

## A0 run 5: cancel fix (2026-11-15)
Changes:
- `examples/hello-film/render.js` and `templates/new-film/render.js` (still identical): `puppeteer.launch` gets `handleSIGTERM: false, handleSIGHUP: false`, so a stop signal ends the script at once.
- `server/platform.js` Unix `killTree`: before the SIGKILL pass, also kills survivors' descendants that started after the first snapshot. This covers films that keep an older copy of `render.js`.
- `server/platform.darwin.test.js`: new test for a child started after SIGTERM (fails without the fix, passes with it).

Results: `npm test` 342 pass, 0 fail, 1 skipped (Windows-only).

| Real cancel test (60 s render, cancel after 4 s) | Before | After |
|---|---|---|
| Old `render.js` (existing films) | Ran on ~3 s, new Chrome orphaned | Ran on ~3 s (until SIGKILL); late Chrome killed with it, nothing left |
| New `render.js` | n/a | All processes gone within 1 s, no retry, no new Chrome |

A normal full render with the new `render.js` still works.

### Note: length inferred from chunks when no length is stored
A scratch copy made with `cp -R` lacks `.videohub/project.json` (the installer creates it from `videohub-project.json`). `jobs.recipe()` then infers `DUR` from the chunk files on disk, so after a cancel left chunks 0–3, the next full render silently produced 8 s instead of 10 s. With `.videohub/project.json` present, as a real install has, a full render after a cancel resumed correctly (300 frames, 10.000 s). Projects without a stored length (older projects) can still be shortened this way after a cancel; this is a platform-independent edge case to consider upstream.

## A0 summary
Apple Silicon qualifies for A1. The macOS render-cancel bug found in run 4 is fixed (run 5). Also open: Windows-style path text in `hub_workspace`, 2 moderate `npm audit` advisories, no `.gitignore`.

## Next: live preview and render acceptance plan (isolated)
All steps use a scratch copy and test hub, never a real install or `~/Documents/VideoHub`:
```bash
export VH=$PWD/.test-tmp/a0-live
mkdir -p "$VH" && cp -R examples/hello-film "$VH/films-hello-film"
export VIDEOHUB_PORT=7799 VIDEOHUB_DATA_DIR="$VH/data" VIDEOHUB_BASE_DIR="$VH" VIDEOHUB_GUARD=off
```
1. **Hub startup.** `npm start` with the vars above (jobs on, no window). Confirm `http://127.0.0.1:7799` responds, loopback only, and `node server/cli.js hub_workspace` (same port) reports the scratch workspace.
2. **Live preview.** Open the example through `hub_live` in Chrome. Owner watches: canvas animation, timing, audio playback in the preview.
3. **Frame notes.** Owner leaves notes on two frames; confirm they persist in `$VH/data` and come back through `hub_project`/notes tools.
4. **Example audio.** Confirm `examples/hello-film/make-audio.js` output and the `audio/` assets load in preview.
5. **Full render (only after the owner approves the preview).** Queue one render through the hub. Confirm the MP4 plays, has an AAC audio track (`ffprobe`), and that the job log shows `h264_videotoolbox`. Record render time and compare with `x264`.
6. **Partial re-render.** Change one scene, re-render only that range, confirm the splice is frame-accurate and audio stays in sync.
7. **Cancel and cleanup.** Cancel a render mid-way; confirm no orphan Chrome or FFmpeg processes and that temp files are cleaned up.
8. Record results here as "A0 run 2".
