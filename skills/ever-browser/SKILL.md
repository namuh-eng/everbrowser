---
name: ever-browser
description: Use the Ever CLI to automate the active browser session, including ever exec page/browser globals, navigation, snapshots, clicks, input, screenshots, video recording, tabs, and raw browser actions.
---

# Ever Browser

Use the Ever CLI to automate the active browser session.

## Prerequisites (check first)

Ever drives the user's real Chrome via the Ever extension. Before automating, the user must have:

1. The **Ever Chrome extension** installed — https://chromewebstore.google.com/detail/ever/codfpjgkcdackkjhjhfefmfbckijjjnf
2. Signed in (click the Ever toolbar icon → side panel → sign in with Google)

If any command reports the browser is **not connected** / no browser available, do not retry blindly — tell the user to install the extension from the Chrome Web Store link above and sign in, then retry. Verify connectivity any time with `ever doctor`.

## `ever exec` globals

`ever exec` injects two globals:

- `page`: Playwright-style facade for common page actions. Prefer this for portable scripts.
- `browser`: Ever-native lower-level API and escape hatch for tab/session helpers or unsupported actions.

Example:

```js
await page.goto('https://example.com');
const snapshot = await browser.snapshot();
await page.click(12); // positive integer snapshot id, not a CSS selector
await page.fill(13, 'hello'); // replaces/clears first
await page.keyboard.press('Enter');
```

## `page` facade

Supported methods:

- `page.goto(url, opts?)` routes to Ever navigation.
- `page.click(target)` clicks a positive integer numeric snapshot id.
- `page.fill(target, text, opts?)` replaces/clears an input value by default for a positive integer numeric snapshot id.
- `page.evaluate(fnOrString)` evaluates JavaScript in the page context.
- `page.mouse.click(x, y, opts?)` clicks viewport coordinates; supports `{ button, clickCount }`.
- `page.keyboard.press(keys)` sends keyboard keys.
- `page.waitForTimeout(ms)` waits the requested milliseconds.
- `page.screenshot(options?)` captures a viewport screenshot. The result is Ever's action result; screenshot bytes are in `result.extracted_content`.

Selector support is intentionally not implemented yet. For `page.click` and `page.fill`, pass a numeric snapshot id from `browser.snapshot()`. String selector targets throw an error like: `selector targets are not supported yet; use a snapshot numeric ID`.

`page.evaluate(fnOrString)` serialization semantics:

- Strings are passed as JavaScript expressions/statements to Ever page evaluation.
- Functions are stringified and invoked in the page context as zero-argument functions.
- Functions do not capture closures from the `ever exec` script context.
- Additional arguments and functions with declared parameters are unsupported and throw a clear error.

## `browser` Ever-native API

Use `browser` when you need Ever-specific behavior:

- `browser.snapshot(opts?)`
- `browser.tabs()`
- `browser.switchTab(tabId)`
- `browser.closeTab(tabId)`
- `browser.wait(seconds)`
- `browser.screenshot(options?)`
- `browser.action(name, params)` raw action escape hatch

Existing helpers such as `browser.navigate`, `browser.click`, `browser.input`, `browser.sendKeys`, session helpers, sheets helpers, and filesystem helpers remain available.

## Tab groups & session health

Ever organizes its tabs into browser tab groups. These helpers inspect group/tab
health and reclaim stale groups or excess tabs. Cleanup/prune default to a
**dry run** — pass an apply/`dryRun: false` flag to actually close anything.

### `ever exec` API

```js
await browser.groupHealth(); // list groups + per-group cleanup status
await browser.tabHealth();   // health for the active session's tabs

// Close stale Ever-owned groups (dry run unless dryRun: false)
await browser.cleanup({
  staleOnly: true,        // only target stale groups
  dryRun: true,
  maxAgeMs,               // staleness threshold
  includeActive: false,   // allow closing the active group
  includeAdoptedGroups: false, // allow user-created/adopted groups
});

// Drop excess tabs from one group (dry run unless dryRun: false)
await browser.prune({
  session: true,          // or groupId: <id>
  maxTabs,
  dryRun: true,
  includeActive: false,
  includeAdoptedGroups: false,
});

// Reuse an existing tab for url, or open one (navigate new_tab=true)
await browser.findOrOpenTab({ url, newTab: true });
```

### CLI commands

```bash
ever tab-health                         # tab health for the active session group
ever cleanup --dry-run                  # preview stale-group cleanup (default)
ever cleanup --apply                    # close stale Ever-owned groups
#   flags: --stale (default) --include-active --include-adopted-groups
ever prune --session --dry-run          # preview pruning excess tabs (default)
ever prune --group <id> --apply         # close excess tabs in a group
#   flags: --include-active --include-adopted-groups
```

### Session reuse & teardown

- `ever start --new` forces a new session even if a matching one exists;
  `ever start --purpose <key>` sets an explicit purpose key for reuse matching.
  Via exec: `browser.start({ new: true, purposeKey })`.
- `ever stop --keep-tabs` ends the session but preserves its visible tabs.
  Via exec: `browser.stop({ session: true, keepTabs: true })`.

## Video recording

Ever CLI can record the active CLI session to a local WebM file. Use this for
demo capture, bug reproduction evidence, or visual QA artifacts.

### CLI commands

Typical live smoke:

```bash
ever start --url https://example.com
ever video-start demo.webm --size 800x600
ever exec "await page.goto('https://example.com'); await page.waitForTimeout(1000)"
ever video-stop --json
```

Useful commands:

- `ever video-start [output.webm]` starts recording the active session.
- `ever video-start demo.webm --size 800x600 --quality 75 --fps 10`
- `ever video-status` reports active recording status, frame count, drops, and output path.
- `ever video-stop` stops recording and finalizes the local WebM.
- `ever video-stop --json` returns structured artifact metadata.

Recording writes the artifact on the local daemon machine, not the API server.
By default, output is saved under `$EVER_HOME/recordings/` (e.g. `~/.ever/recordings/`):
a bare filename like `demo.webm` lands there, while an absolute path is used as-is.
Local `ffmpeg` is required; missing or failing ffmpeg should be treated as a
real setup error, not silently ignored.

### `ever exec` video API

`ever exec` exposes video controls on both facades:

```js
await browser.video.start('demo.webm', {
  size: { width: 800, height: 600 },
  quality: 75,
  fps: 10,
});
await page.goto('https://example.com');
await page.waitForTimeout(1000);
const result = await page.video.stop();
console.log(result.outputPath);
```

Supported helpers:

- `browser.video.start(outputPathOrOptions?, options?)`
- `browser.video.stop()`
- `browser.video.status()`
- `page.video.start(outputPathOrOptions?, options?)`
- `page.video.stop()`
- `page.video.status()`

### Validation recipe

After recording, verify the local artifact instead of assuming success:

```bash
file demo.webm
ffprobe -hide_banner -loglevel error \
  -select_streams v:0 \
  -show_entries stream=codec_name,width,height,avg_frame_rate,pix_fmt \
  -show_entries format=duration \
  -of json demo.webm
```

Expected shape: `file` reports `WebM`; `ffprobe` reports a video stream such as
VP8/VP9 with non-zero dimensions. Recordings are **variable frame rate** and
preserve real timing, so `format.duration` should match the actual wall-clock
recording length (not `frameCount / fps`). Also check
`video-status`/`video-stop --json` for `droppedFrames` when diagnosing choppy
captures.

### Timing model

- Recordings use **true-to-life timing**: each frame is stamped with its real
  capture time (CDP `Page.screencastFrame` `metadata.timestamp`) and encoded as
  variable frame rate, so playback duration and pacing match reality.
- CDP emits frames on visual change, so idle periods stay compact and the file
  is not padded to a fixed rate.
- `--fps` is deprecated and no longer forces a constant rate; throttle capture
  with `--every-nth-frame` instead.

### Lifecycle notes

- Recording pins the tab active at `video-start`; later tab switching does not
  retarget the recording.
- `ever stop --session`, daemon shutdown, tab removal, debugger detach, and frame
  stream abort should all stop/cleanup recording.
- The API coordinates CDP `Page.startScreencast` frames and acknowledges them;
  the CLI daemon paces the frame stream to real capture timing and encodes
  locally via ffmpeg.
