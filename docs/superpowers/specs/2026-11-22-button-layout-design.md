# uConsole button layout — design

The uConsole (hack2you 9900 trackpad keyboard, ZMK) is typed with both thumbs, mostly into a
terminal: kitty on the uConsole, ssh to the Mac (`Host mac`), tmux there, Claude Code inside
tmux. Neither tmux nor Claude Code runs on the uConsole. The gamepad buttons (A B X Y), the
spare right mouse button (R), Start and Select are all reachable without shifting grip
(Gati, 22-11-2026). This design gives them terminal jobs.

## Goals

- Left and right click stay single presses.
- One-press access to: the next agent waiting on Gati, previous/next tab at both levels
  (kitty tabs, tmux windows), the tmux prefix, and a screenshot.
- Every button means the same everywhere (the Codex Micro's rule: "a cap is printed once").

Non-goals: a joystick report (ZMK has none); voice input (waits on the microphone).

## Approaches considered

| Approach                                          | Trade                                                            | Verdict  |
| ------------------------------------------------- | ---------------------------------------------------------------- | -------- |
| B held as a shoulder: a tab layer under the D-pad | Previous and next at both levels; B's tap fires on release       | **Chosen** |
| Flat: one verb per button, no holds               | Simpler; no "previous", no prefix button                         | Passed on |
| Agent grammar (A = Enter, B = Esc, R = left click) | Duplicates Enter/Esc; loses the two-button mouse                | Passed on |

## The layout

```
Trackpad press  left click
R        right click
A        next waiting agent
B  tap   tmux prefix, Ctrl+Space
   hold  tab layer:  D-pad ← →  tmux previous / next window   (prefix, p / n)
                     D-pad ↑ ↓  kitty previous / next tab     (ctrl+shift+← / →, kitty's defaults)
X  tap   screenshot into the Mac's Claude
   hold  screenshot to the uConsole's clipboard
Y        reserved for hold-to-talk (sends F17 = XF86Launch8, unbound)
Start    Super (hold, e.g. Super+Return opens a terminal); play/pause moves to LFn+Start
Select   Caps Lock, unchanged: the trackpad scrolls while it is on
Shift    both one-shot (`&sk`): a tap capitalises the next key, a hold is a normal Shift
LFn+Space   backlight on/off (unchanged)
RFn+Space   backlight brightness, cycling in steps of 10
```

The left mouse button stays `&none`: the front plate holds this unit's left mouse button
closed (Upgrade Path U14). The trackpad press gets left click back on trial (Gati,
22-11-2026): both positions were nulled without proving which switch jams. If the post-flash
sweep shows left click held, flash back the previous firmware. `&bootloader` on RFn+\ and
RFn+LFn reset stay.

## How each press reaches its target

```
button ─▶ ZMK
            ├─ B layer, Start, Shift, backlight: plain keys, done in firmware
            ├─ A        ─▶ F14 = XF86Launch5 ─▶ kitty mapping sends tmux's "next waiting" key ─▶ tmux (Mac)
            └─ R tap    ─▶ F15 = XF86Launch6 ─▶ labwc ─▶ script: slurp → grim → scp to mac → wtype the path
               R hold   ─▶ F16 = XF86Launch7 ─▶ labwc ─▶ slurp → grim → wl-copy

F14–F16 because xkb's evdev rules (`symbols/inet`) turn them into launch keys nothing binds,
while F21–F23 become `XF86TouchpadToggle/On/Off`, which a desktop may act on.
```

- **A does nothing until the Mac's tmux has a "next waiting" binding.** That binding, and the
  Claude Code hooks that flag a waiting window, belong to the tmux work (the Scratchpad's tmux
  prompt, item 3). Only then does kitty get its `map f21`. No stopgap job in between.
- **R tap:** an image cannot cross ssh's clipboard (OSC 52 carries text), so the shot is copied
  to the Mac at `/tmp/uconsole-shots/` and its path is typed at the cursor; Claude Code reads an
  image path as an image. Esc in slurp cancels and types nothing. Needs `wtype` (apt).
- The script lives in `uconsole-repo/bin/` beside `console`, deployed to `~/.local/bin`.

## Verification

After flashing, a key sweep (`keysweep.py`) shows every remapped key's code and nothing held;
then each job is exercised once on the device: both tab levels, the prefix, Super+Return, a
one-shot capital, backlight steps, both screenshot paths ending in Claude Code and a paste.
Volume on LFn+↑/↓ is checked against labwc's `XF86AudioRaiseVolume`/`LowerVolume` in the same
pass.

## Left to the builder

- Hold-tap timing for B and R (start from ZMK's `hold-preferred`, 200 ms).
- Whether this fork's backlight behaviour supports `BL_CYCLE` or needs `BL_INC`.
- Confirm with `wev` that A and R arrive as `XF86Launch5`–`7` before binding them.

## Revision — 22-11-2026, after the first flash

The trackpad press had never clicked: its switch seated on first use, stuck closed once, then
clicked cleanly. Gati chose it as left click, with R back as right click, so the face buttons
carry the jobs: X takes the screenshot, Y is reserved. A stuck pad has no backup left click;
a firm press on the pad frees it.
