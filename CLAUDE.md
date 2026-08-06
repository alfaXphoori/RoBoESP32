# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A classroom project for MicroBlocks (block-based visual programming, similar to Scratch,
that compiles to run on microcontrollers). A Cytron ROBO ESP32 robot car creates its own
WiFi hotspot, serves an HTML control page, and drives two DC motors plus an optional servo,
an optional HC-SR04 ultrasonic sensor, RGB LEDs, and a buzzer based on commands sent from
a phone browser.

There is no build/lint/test tooling — this is not a conventional software project. The
"source" is a MicroBlocks library and one MicroBlocks project file, edited either in the
MicroBlocks IDE (block editor) or as plain text here. Alongside them is a set of Thai
teaching materials published as a GitHub Pages site.

## Files

| File | Purpose |
|---|---|
| `KSU_RoBoESP32.ubl` | The library: all reusable blocks students call from their project. |
| `RoboCar.ubp` | The one project file — answer key. The `forever` loop uses `KSU do command`; the ten-command long-form `if` chain sits beside it as an unattached script (so the class can see the shortcut block is not magic), along with click-to-run demo blocks. Embeds its own full copy of the library (see below). |
| `RoboCar-Control.html` | A standalone, human-editable copy of the HTML/CSS/JS control page. This is NOT what the board serves — it exists only so the page can be previewed/edited in a normal browser/editor. Changes here must be manually re-split and copied into the two `.ubp`/`.ubl` files (see below). |
| `README.md` | Full reference: hardware wiring, pin map, command list, library API, troubleshooting. Read this first for anything not covered below. |
| `lesson-m3-th.md`, `slides-m3-th.html`, `worksheet-m3-th.md` | The current Thai teaching set for Grade 9 (ม.3): teacher lesson plan, reveal-style slide deck, student worksheet. These are what the classroom actually uses. |
| `index.html` | GitHub Pages landing page linking the Thai materials. Uses screenshots in `images/`. |
| `worksheet.md` | The older English worksheet (~50 min). **Stale** — still references `robocar-STUDENT.ubp`, which no longer exists (recoverable via `git show 40df6c7:robocar-STUDENT.ubp`). |
| `backup_20260729/` | Pre-consolidation copies of the old STUDENT/TEACHER projects. Reference only; do not edit. |

## File format

`.ubl`/`.ubp` files are plain-text MicroBlocks/GP source, not binary. Structure:

- `module <name> <category>` starts a module; a `.ubp` project file typically embeds
  `module main` (the project's own script) followed by every library module it depends on
  (`KSU_RoBoESP32`, `NeoPixel`, `Robo ESP32`, `Servo`, `WiFi`, `Tone`, `HTTP server`), each
  in full. This means `RoboCar.ubp` contains its own embedded copy of `KSU_RoBoESP32` —
  editing `KSU_RoBoESP32.ubl` does **not** change behavior of the already-exported `.ubp`;
  edits must be re-imported/re-exported through the MicroBlocks IDE, or applied by hand to
  both files if you're editing text directly. `RoboCar.ubp` is what students flash, so an
  edit that lands only in the `.ubl` changes nothing they will see.
- **The two copies use different identifiers, and that is expected.** They are the same
  code with different names, so never copy a block body verbatim between them:

  | | `KSU_RoBoESP32.ubl` | embedded copy in `RoboCar.ubp` |
  |---|---|---|
  | function names | `KSU_RoBoESP32_forward` | `KSU_ROBOESP32_forward` |
  | block labels | `KSU_RoBoESP32 forward` | `KSU forward` |
  | variables | `_KSU_RoBoESP32_spd` | `_ksu_spd` |
  | `choices` menus | `KSU_RoBoESP32onoff` | `ksuonoff` |

  The `_p1`..`_p8` page chunks are the one thing that must be byte-identical in both. The
  rest of this document names blocks in the `KSU_ROBOESP32_*` / `_ksu_*` form as shorthand
  for "this block, in whichever file you are editing".
- `spec ...` lines declare a block's palette signature (name, label text, argument types/defaults).
- `to <name> <args> { ... }` defines a block's implementation (GP language: C-like braces,
  `=` for assignment, `[module:primitive]` calls into MicroBlocks/ESP32 primitives).
- `script <x> <y> { ... }` in `module main` is one visual script positioned on the canvas;
  `RoboCar.ubp` deliberately keeps unattached scripts (the long-form `if` chain and the
  demo blocks) parked to the right of the running loop.
- `comment '...'` blocks are the sticky-note text visible in the MicroBlocks editor —
  treat them as user-facing documentation, not code comments; keep their tone matching
  the rest of the file (plain, classroom-friendly).

## Architecture

**Everything runs from one polling loop.** `KSU_ROBOESP32_command` (in `KSU_RoBoESP32.ubl`) is called
once per iteration of the student's `forever` loop. Each call:
1. Polls `[net:httpServerGetRequest]` for a pending HTTP request; returns `''` if none.
2. On first contact (empty path) calls `_KSU_ROBOESP32_sendPage` to serve the HTML control page.
3. Answers a `/Status` poll itself (`b1,b2,speed,value,cm` — two button states, speed, the
   first command arg, and the ultrasonic distance) and returns `''` — that word never
   reaches the student's program. Note this makes a status poll ping the HC-SR04, which
   blocks the loop for up to 25 ms when nothing echoes back.
4. Otherwise immediately responds `200 OK` to the browser, then returns the command word
   parsed from the request path (e.g. `Forward`, `Light1`), stripped of its leading `/`.

**Wire protocol: split the path on underscores.** Element 1 is the command word, elements
2..n are numbers, parsed into the args list (`_ksu_args` / `_KSU_RoBoESP32_args`). So `/Speed_70` → word `Speed`, args
`[70]`; `/Rgb_2_255_120_0` → word `Rgb`, args `[2 255 120 0]`. A path with no underscore
is a bare word — which is why `Light1`/`Light0` stay whole words, deliberately, so the
worksheet can teach "different word → different action" before it teaches passing values.
Read the numbers with `KSU_ROBOESP32_value` (first) or `KSU_ROBOESP32_valueN` (i-th) in the *same* iteration.
`KSU_ROBOESP32_value` reports 90 when the last command carried no number.

Two student-facing patterns in `module main`. The long form the worksheet teaches:
```
set command to (KSU web command)
if (command) = 'Forward' : KSU forward
if (command) = 'Speed'   : KSU set speed (KSU command value) %
...
```
and the one-block form, which handles every word the page can send:
```
set command to (KSU web command)
KSU do command (command)
```
`KSU_ROBOESP32_do` covers the ten DRIVE-tab words and falls through to `_KSU_ROBOESP32_do2` for the other
eleven — the split exists purely to stay under the 1000-byte compiled script limit.
Keep it that way when adding commands: add to `_KSU_ROBOESP32_do2`, not to `KSU_ROBOESP32_do`.

**Board primitives.** The library covers the whole board: driving (`KSU_ROBOESP32_forward`…`KSU_ROBOESP32_stop`,
`KSU_ROBOESP32_speed`, `KSU_ROBOESP32_drive`, `KSU_ROBOESP32_steer`, `KSU_ROBOESP32_motor`, `KSU_ROBOESP32_trim`), all four servos
(`KSU_ROBOESP32_servo`, with `KSU_ROBOESP32_arm` as the D4 shorthand), lights (`KSU_ROBOESP32_light`, `KSU_ROBOESP32_rgb`,
`KSU_ROBOESP32_rgbraw`, `KSU_ROBOESP32_dim`, `KSU_ROBOESP32_led`, `KSU_ROBOESP32_lednum`), sound (`KSU_ROBOESP32_horn`, `KSU_ROBOESP32_beep`,
`KSU_ROBOESP32_tone`, `KSU_ROBOESP32_note`, `KSU_ROBOESP32_song`) and input (`KSU_ROBOESP32_button`, `KSU_ROBOESP32_read`,
`KSU_ROBOESP32_distance`, `KSU_ROBOESP32_ultrasonic`). Most call into the `Robo ESP32`, `NeoPixel`, and
`Tone` library modules (also embedded in the `.ubp`).

The Cytron `Robo ESP32` v1.1 module embedded in the `.ubp` file is the complete list of
what the board can do — read it before adding a block that claims a new capability.
Three things live in the KSU layer and exist nowhere else: servo angles are 0–180
everywhere in KSU but the Cytron block wants −90–90, `KSU_ROBOESP32_lednum` maps 1–8 onto pins
16 17 21 22 25 26 32 33, and the HC-SR04 is driven directly with `digitalWriteOp` /
`microsOp` / `waitUntil` because Cytron's module has no ultrasonic block at all.

**The HC-SR04 is bit-banged, and its pins are shared with two LEDs.** `KSU_ROBOESP32_distance`
pulses trig for 10 µs, times the echo pulse against a 25000 µs timeout, and returns
`µs / 58` centimetres — 0 on timeout, which also covers "no sensor plugged in". Pins live
in `_ksu_trig` / `_ksu_echo`, set to **trig 33 / echo 32** by `KSU_ROBOESP32_start`. Those are
also GPIO LED 8 and LED 7, so `Led_7_1` / `Led_8_1` from the LIGHT tab drive the same
wires as the sensor — a documented "leave those two alone", not something the code
prevents. Two constraints govern any repinning: **GPIO 34, 35, 36 and 39 are input-only**
so trig can never live there (the block would report 0 forever), and the block blocks the
loop while it waits, so it belongs once per `forever` iteration, never several times in a
row. Both caveats belong in any doc that names the pins.

Motor direction is controlled by two trim variables, `_ksu_rev1`/`_ksu_rev2` (`KSU set motor
trim`), both defaulting to `-1` to match how the classroom cars are wired — see the
Troubleshooting table in README.md before "fixing" direction bugs elsewhere.

**The board does whole-number arithmetic.** Multiply before you divide. `KSU_ROBOESP32_steer` has a
comment about this because the obvious form of its formula silently collapses to three
steering positions.

**The control page is embedded as string-literal blocks** (`_KSU_ROBOESP32_p1` … `_KSU_ROBOESP32_pN`, currently
8) because a single MicroBlocks script cannot compile to more than 1000 bytes, and the
whole page is ~5KB. `_KSU_ROBOESP32_sendPage` reassembles them with `[data:join]` and serves them as
one HTTP response. `RoboCar-Control.html` is the same content in one readable file for editing —
**when changing the page, edit `RoboCar-Control.html`, then re-split the result into
`_KSU_ROBOESP32_p1`..`_KSU_ROBOESP32_pN` and copy those into both `KSU_RoBoESP32.ubl` and the embedded
copy inside `RoboCar.ubp`.**

The exact relationship, which any re-split must reproduce: the chunks concatenated in order
equal `RoboCar-Control.html` with every newline removed and nothing else changed. Verify it
with a script rather than by eye — for example:

```bash
grep -A1 "^to _KSU_RoBoESP32_p[0-9]* {" KSU_RoBoESP32.ubl | grep "^  return '" \
  | sed "s/^  return '//; s/'$//" | tr -d '\n' > /tmp/a
tr -d '\n' < RoboCar-Control.html > /tmp/b
cmp /tmp/a /tmp/b     # and again with _KSU_ROBOESP32_p against RoboCar.ubp
```

Constraints on the page itself:

- Never an apostrophe `'` anywhere in it — GP string literals use `'` as the delimiter and
  there is no escape for it.
- ASCII only. Use JS escapes (`\u00b0`, `\u25cf`) or HTML entities (`&deg;`, `&#9650;`),
  never the literal character. Chunks are split by character count, and a split landing
  inside a multi-byte character corrupts the page.
- No `//` line comments in the JS, and no statement that depends on a newline to terminate —
  the newlines are gone by the time the board serves it.
- Each chunk ≤ 700 characters, split only at safe boundaries: never inside an HTML entity,
  a `\uXXXX` escape, a quoted JS string, an identifier, or a number.

**Client-side page logic** (`RoboCar-Control.html`'s `<script>`) throttles outgoing requests:
only one `fetch` is in flight at a time (`b` flag); a newly queued command (`q`) overwrites
whatever was queued but not yet sent, so the car always acts on the latest input rather than
queuing up stale button presses. Arrow buttons/WASD/on-screen buttons all funnel through the
same `s(word)` function that sets `q` and triggers `g()` to send it. **Every new control must
go through `s()`** — a direct `fetch` would bypass the throttle and can stall driving.

The page has three tabs, built by JS loops rather than repeated markup to stay inside the
size budget: DRIVE (the original screen — d-pad, speed, servo, light, horn, live distance
readout, plus keyboard WASD), LIGHT (RGB swatches, 0/1/all target, dimmer, 8 GPIO LED
toggles) and MORE (four servo sliders, two motor sliders, tone/song/beep, live button +
distance readout).

The one deliberate exception to the `s()` rule is the `/Status` poll, which needs the
response body. It is **one perpetual chain** started once at load — `N()` reschedules
itself and nothing else ever calls it, which is what keeps a second chain from doubling
the poll rate. It stays out of driving's way by checking two things before each fetch: `Z`
(tab index — it skips the fetch entirely on LIGHT) and `b` (the command-in-flight flag), 
retrying in 400 ms rather than competing. Cadence comes from the same `Z`: 500 ms on MORE,
800 ms on DRIVE. If you add a caller of `N()`, you have created a second chain. Its handler
splits the body on commas — read fields by index, and if you add a field to
`_KSU_ROBOESP32_status`, append it rather than reordering.

The DRIVE distance line is created in JS (`sd`, inserted at the top of `#g`) rather than in
the static markup. That is deliberate: `_p1`..`_p7` are at 686–700 characters with no room
to grow, so new page content goes into the JS tail in `_p8`, which has headroom. Editing an
existing chunk in place is fine as long as it stays ≤ 700 — the invariant is the
concatenation, not the chunk boundaries, so a surgical edit beats a full re-split.

## Working on this repo

- There's no compiler/linter to run here; "testing" means opening `RoboCar.ubp` in the
  MicroBlocks IDE, flashing an ESP32, and driving the car — or previewing
  `RoboCar-Control.html` directly in a browser for layout/JS changes (note: the fetch calls
  will fail with no board to answer them, so it's useful only for visual/layout iteration).
  State plainly in the summary that board behavior was not verified.
- Command words sent by the page are case-sensitive: capitalized first letter, lowercase rest
  (`Forward`, not `forward` or `FORWARD`). Keep new commands consistent with this convention.
- A library change is only finished when it has landed in **both** `KSU_RoBoESP32.ubl` and the
  embedded module in `RoboCar.ubp`, translated to that file's naming (see the table above),
  with `README.md` updated to match. The Thai materials (`lesson-m3-th.md`,
  `slides-m3-th.html`, `worksheet-m3-th.md`, `index.html`) are lesson-paced, not API
  reference — update them only when asked, or when a change breaks a step they describe.
- The Thai worksheet's Part 2 teaches exactly ten core command words (Forward, Backward,
  Left, Right, Stop, Light1, Light0, Horn, Speed, Servo). **Do not add new commands to
  Part 2** — extra commands belong to the LIGHT/MORE tabs and are reached with the single
  `KSU do command` block, which is what the "Going further" section is for. The README's
  "Commands sent by the page" table is the one place that lists all of them.
- Adding a command means touching four things at once: the page must send it, `_KSU_ROBOESP32_do2` must
  handle it, the README table must list it, and — if it needs a new block — the library needs
  the `spec` line. A command the page sends but `_KSU_ROBOESP32_do2` ignores fails silently.
- A *sensor* is not a command. Readings reach the page through the `/Status` body, not through
  a new command word — that is how the ultrasonic distance gets onto the MORE tab, and it
  keeps the `if (command) = ...` teaching model untouched.
- Before wiring anything to a GPIO, check it can do the job: 34, 35, 36, 39 are input-only;
  12/13/14/27 are the motors, 15 the RGB strip, 23 the buzzer, 4/5/18/19 the servo headers,
  16/17/21/22/25/26/32/33 the eight LEDs, and 33/32 double as the ultrasonic trig/echo. A
  "free" pin usually means a servo header or an LED pin whose device is unplugged — say so
  in the docs when you claim one, because nothing on this board is truly spare.
