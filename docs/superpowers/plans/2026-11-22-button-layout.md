# uConsole button layout — implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give the uConsole's A, B, R, Start and Shift keys their terminal jobs, and add a
two-mode screenshot, in one firmware flash.

**Architecture:** The ZMK keymap in this fork sends plain keys, tmux/kitty chords, or neutral
launch keys (F14–F16). labwc binds the launch keys from R to a screenshot script in the device
repo. A's launch key stays unbound until the Mac's tmux has a "next waiting" binding.

**Tech Stack:** ZMK (ZitaoTech/zmk `bbkeyboard_tp`, built by this fork's GitHub Actions), labwc
`rc.xml`, POSIX sh, grim/slurp/wl-copy/wtype, ssh/scp to `Host mac`.

**Spec:** `docs/superpowers/specs/2026-11-22-button-layout-design.md` (this repo).

## Global Constraints

- Y = `&mkp LCLK`, X = `&mkp RCLK`, single presses, unchanged.
- The left mouse button position stays `&none` in every layer.
- `&bootloader` stays on RFn+\ ; `&sys_reset` stays on RFn+LFn.
- Launch keys: A = `F14` (XF86Launch5), R tap = `F15` (XF86Launch6), R hold = `F16` (XF86Launch7). Never F21–F23.
- Screenshots land on the Mac in `/tmp/uconsole-shots/`.
- labwc `Execute` commands use absolute paths (`/home/gati3478/.local/bin/...`): labwc's `/bin/sh` never has `~/.local/bin` on PATH.
- Commits and pushes happen only with Gati's go-ahead for this plan.
- A timed physical capture starts only after an AskUserQuestion answer.

## Review Focus

1. B pressed, then a D-pad key within 200 ms (a rolled chord) → the tab-layer action, never Ctrl+Space plus an arrow. Pinned in Task 3, step 4.
2. Esc in slurp → exit 0, nothing typed, no file left on either machine. Pinned in Task 2, step 1.
3. The Mac unreachable on an R tap → a dunst notification and nothing typed, never a path that points to no file. Pinned in Task 2, step 1.
4. Holding Shift while clicking or arrowing (shift-select) → behaves as a normal held Shift despite `&sk`. Pinned in Task 3, step 4.
5. The trackpad press turns out to be the jammed switch → left click shows held right after the flash, and the fallback firmware goes back on. Pinned in Task 3, step 3.

---

### Task 1: Keymap

**Files:**
- Modify: `config/bb9900.keymap`

**Interfaces:**
- Produces: firmware artifact `firmware` from the push's Actions run, holding the `.uf2` for Task 3.

- [ ] **Step 1: Add the behaviors** in the existing `behaviors` block:
  - `lt_hp`: `zmk,behavior-hold-tap`, `#binding-cells = <2>`, `flavor = "hold-preferred"`, `tapping-term-ms = <200>`, `bindings = <&mo>, <&kp>`. Comment: hold-preferred so B rolled into a D-pad key is always the layer.
  - `shot`: `zmk,behavior-hold-tap`, `#binding-cells = <2>`, `flavor = "hold-preferred"`, `tapping-term-ms = <300>`, `bindings = <&kp>, <&kp>`.
  - `ZMK_MACRO(tmux_prev, wait-ms = <0>; tap-ms = <10>; bindings = <&macro_tap &kp LC(SPACE) &kp P>;)` and `tmux_next` with `&kp N`.
  - `&sk { release-after-ms = <3000>; };` (the old QMK firmware's one-shot timeout).

- [ ] **Step 2: Edit the layers.** `#define TABS 3`. Base layer:
  - row 0: `&kp LEFT  &none  &mkp LCLK  &mkp LCLK  &mkp RCLK` (the trackpad press back on trial; rewrite the U14 comment above it to say so)
  - row 1: `&kp UP  &kp DOWN  &kp RIGHT  &shot F16 F15  &lt_hp TABS LC(SPACE)  &kp F14`, with one comment: F14–F16 are launch keys under xkb, F21–F23 are touchpad keys
  - Start (row 2, third binding): `&kp C_PLAY` → `&kp LGUI`
  - Shifts: `&kp LSHFT` → `&sk LSHFT`, `&kp RSHFT` → `&sk RSHFT`
  - SYM layer, Start position: `&mkp C_PAUSE` → `&kp C_PLAY_PAUSE`
  - UPPER layer, Space position: `&bl BL_TOG` → `&bl BL_CYCLE`
  - New `tabs_layer` after `upper_layer`: `&trans` everywhere (74 positions, the same row shape as the others) except LEFT → `&tmux_prev`, RIGHT → `&tmux_next`, UP → `&kp LC(LS(LEFT))`, DOWN → `&kp LC(LS(RIGHT))`.

- [ ] **Step 3: Check the counts before pushing.**
  Run: `awk '/bindings = </,/>;/' config/bb9900.keymap | grep -o '&[a-z_]*' | wc -l`, then compare each layer's binding count by eye with the base layer's.
  Expected: four layers with the same count.

- [ ] **Step 4: Commit, push, and watch the build.**
  ```bash
  git add config/bb9900.keymap docs/superpowers
  git commit -m "keymap: terminal jobs for A, B, R, Start; one-shot Shift; tab layer"
  git push origin V2.0
  gh run watch -R gati3478/uc9900 $(gh run list -R gati3478/uc9900 --branch V2.0 --limit 1 --json databaseId --jq '.[0].databaseId')
  ```
  Expected: the run concludes `success`. A devicetree error names the line; fix it and push again.

- [ ] **Step 5: Fetch both firmwares** into the scratchpad: the new run's `firmware` artifact, and run `45042766422` (backlight on, Y/X clicks) as the fallback.
  Run: `gh run download <run-id> -R gati3478/uc9900 -n firmware -D <scratchpad>/fw-new` and the same for `45042766422` into `fw-fallback`.
  Expected: one `.uf2` in each.

### Task 2: Screenshot script and labwc bindings

**Files:**
- Create: `~/Development/Devices/uconsole-repo/bin/uconsole-shot`
- Modify: `~/Development/Devices/uconsole-repo/config/labwc/rc.xml` (the `<keyboard>` block)
- Modify: `~/Development/Devices/uconsole-repo/README.md` (the `chmod 755` lines near 226; `wtype` beside the other required tools)

**Interfaces:**
- Produces: `uconsole-shot mac|clip`, exit 0 on success or cancel, 1 on failure. Environment seams for tests: `SHOT_GEOMETRY` (skips slurp), `SHOT_MAC_HOST` (default `mac`), `SHOT_TYPE_CMD` (default `wtype`).

- [ ] **Step 1: Write the failing checks**, run on the device with `WAYLAND_DISPLAY=wayland-0 XDG_RUNTIME_DIR=/run/user/$(id -u)`:
  ```sh
  # mac: the path is "typed" and the file exists on the Mac
  p=$(SHOT_GEOMETRY="0,0 64x64" SHOT_TYPE_CMD=echo uconsole-shot mac)
  case "$p" in /tmp/uconsole-shots/shot-*.png) ;; *) echo FAIL path;; esac
  ssh mac test -s "$p" || echo FAIL remote
  # clip: an image is on the clipboard
  SHOT_GEOMETRY="0,0 64x64" uconsole-shot clip && wl-paste --list-types | grep -qx image/png || echo FAIL clip
  # unreachable Mac: exit 1, nothing typed
  out=$(SHOT_GEOMETRY="0,0 64x64" SHOT_MAC_HOST=nonexistent.invalid SHOT_TYPE_CMD=echo uconsole-shot mac); [ $? -eq 1 ] && [ -z "$out" ] || echo FAIL unreachable
  # cancelled slurp (Esc): exit 0, nothing typed
  out=$(PATH="$HOME/.cache/shot-test:$PATH" SHOT_TYPE_CMD=echo uconsole-shot mac); [ $? -eq 0 ] && [ -z "$out" ] || echo FAIL cancel
  ```
  where `~/.cache/shot-test/slurp` is a stub that `exit 1`s, as slurp does on Esc.

- [ ] **Step 2: Run them before the script exists.**
  Expected: every line prints `FAIL`.

- [ ] **Step 3: Write `uconsole-shot`** (POSIX sh, `set -eu`). `mac`: region from `SHOT_GEOMETRY` or `slurp` (non-zero → exit 0 silently), `grim -g` into a local temp file named `shot-YYYYmmdd-HHMMSS.png`, `ssh "$SHOT_MAC_HOST" mkdir -p /tmp/uconsole-shots`, `scp` it there, delete the local file, then `$SHOT_TYPE_CMD` the remote path. On any ssh/scp failure: `dunstify "Screenshot not sent: <host> unreachable"`, delete the local file, exit 1. `clip`: same region step, `grim -g … - | wl-copy --type image/png`.

- [ ] **Step 4: Deploy and run the checks.**
  ```bash
  ssh uconsole 'sudo apt-get install -y wtype'
  scp bin/uconsole-shot uconsole:.local/bin/ && ssh uconsole 'chmod 755 ~/.local/bin/uconsole-shot'
  ```
  Expected: the step 1 checks print nothing.

- [ ] **Step 5: Bind the keys** in `rc.xml`, after the brightness bindings:
  ```xml
  <keybind key="XF86Launch6"><action name="Execute" command="/home/gati3478/.local/bin/uconsole-shot mac"/></keybind>
  <keybind key="XF86Launch7"><action name="Execute" command="/home/gati3478/.local/bin/uconsole-shot clip"/></keybind>
  ```
  Deploy: `scp config/labwc/rc.xml uconsole:.config/labwc/rc.xml && ssh uconsole 'labwc --reconfigure'`.
  Expected: `ssh uconsole 'cat ~/.config/labwc/rc.xml' | diff - config/labwc/rc.xml` prints nothing. The keys can't be pressed until Task 3's flash.

- [ ] **Step 6: Commit** in `uconsole-repo`.
  ```bash
  git add bin/uconsole-shot config/labwc/rc.xml README.md
  git commit -m "uconsole-shot: region screenshot into the Mac's Claude, or to the clipboard"
  ```

### Task 3: Flash and verify on the device (Gati's hands)

**Interfaces:**
- Consumes: Task 1's `fw-new` and `fw-fallback` `.uf2`; Task 2's deployed bindings; `/tmp/keysweep.py` and `/tmp/uf2wait.sh` on the device (copy them back from the scratchpad if a reboot cleared `/tmp`).

- [ ] **Step 1: Gate.** AskUserQuestion: ready to press RFn+\ for the flash.

- [ ] **Step 2: Flash.** Run `uf2wait.sh`, then copy `fw-new`'s `.uf2` onto ADM840BOOT.
  Expected: the drive unmounts itself and the keyboard re-enumerates (`ls /dev/input/by-id | grep -i 9900`).

- [ ] **Step 3: Held-key check, immediately.** Run `python3 /tmp/keysweep.py 5` with no keys touched.
  Expected: "held now" is empty. If `BTN_LEFT` is held, the trackpad press is the jammed switch: flash `fw-fallback`, revert row 0 col 2 to `&none` in a follow-up commit, and record it in step 6.

- [ ] **Step 4: Gated sweep** (AskUserQuestion first). Run `python3 /tmp/keysweep.py 60` while Gati presses, in order: trackpad press, Y, X, A, B tap, B+each D-pad direction (one rolled fast), R tap (then Esc in slurp), R hold (then Esc), Start, LFn+Start, Shift tap then `a`, Shift held + `a`, RFn+Space ×3, LFn+↑, LFn+↓.
  Expected: BTN_LEFT (trackpad, Y), BTN_RIGHT (X), KEY_F14 (A), Ctrl+Space (B tap), Ctrl+Space+p / n and Ctrl+Shift+Left / Right (B layer, including the rolled one), KEY_F15 and KEY_F16, KEY_LEFTMETA, KEY_PLAYPAUSE, a Shift press around the letter in both Shift cases, KEY_VOLUMEUP / KEY_VOLUMEDOWN, nothing left held.

- [ ] **Step 5: The jobs, by Gati.** In a kitty tab running `ssh mac` + tmux: B+←/→ moves windows, B+↑/↓ moves kitty tabs, B tap then `c` opens a window, Start+Enter opens a terminal, R tap → region → the path appears in Claude Code's prompt and Claude reads the image, R hold → region → pastes as an image into a uConsole app, RFn+Space steps the backlight brightness.
  Expected: each works; anything that doesn't is fixed before step 6.

- [ ] **Step 6: Records.**
  - Vault `01 Projects/uConsole Upgrade Path.md`: the U14 row gets the trackpad-press result.
  - Vault `00 Inbox/Scratchpad.md`, sitting B: the Super, one-shot Shift, Select, gamepad and volume items answered (22-11-2026); A's kitty mapping is added to the tmux prompt's item 3.
  - Memory `project_uconsole_upgrade.md`: the U14 ruling line updated to the new layout.
