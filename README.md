# WiFi Robot Car — Cytron ROBO ESP32 + MicroBlocks

A classroom project. The board creates its own WiFi hotspot and serves a landscape
control page. Students connect a phone, open one address, and drive the car.

## Files

| File | Purpose |
|---|---|
| `KSU_RoBoESP32.ubl` | The library itself. Add it to any project with the **+** button under the block palette, then use the KSU blocks. |
| `robocar-STUDENT.ubp` | Handout project. One command is wired up, the rest sits in a toolbox for students to connect. |
| `robocar-TEACHER.ubp` | Answer key. Everything wired, ready to demonstrate. |
| `control-page.html` | The same page the board serves. Open it on a computer to preview the layout or to edit the design. |
| `worksheet.md` | Student worksheet, about 50 minutes. |

## Hardware

- Cytron ROBO ESP32 (Rev 1.0 or Rev 1.1 — the pin map is identical)
- Two DC gear motors into the **M1** and **M2** screw terminals
- Optional servo into header **D4**
- Optional HC-SR04 ultrasonic sensor: VCC → 5V, GND → GND, **trig → 33**, **echo → 32**
- One battery pack into the green power terminal, switch on

Nothing else to wire. The RGB LEDs and the buzzer are already on the board. The other
three servo headers (D5, D18, D19) and the eight GPIO LEDs are optional extras — leave
them unplugged and their blocks simply do nothing.

> **HC-SR04 pins.** 33 and 32 are also **LED number 8 and LED number 7**. Both LEDs blink
> along with the pings, which is harmless — but do not switch LED 7 or LED 8 on from the
> LIGHT tab while the sensor is plugged in, or the board and the sensor end up driving the
> same wire against each other.
>
> To move the sensor, use `KSU set ultrasonic trig [33] echo [32]` with the new numbers.
> Trig must sit on a pin that can be an **output**: GPIO 34, 35, 36 and 39 are input-only
> and have no output driver at all, so a trig on one of them can never be pulled high —
> the sensor never fires and `KSU distance cm` reports 0 forever. Echo can go anywhere.

## Software setup

1. MicroBlocks 2.0.120 or newer
2. Gear icon → **update firmware on board** → ESP32 (do this once per board)
3. Open `robocar-STUDENT.ubp`
4. Press **Start**
5. Wait for the RGB LEDs to glow dim green — the hotspot is up

## Driving it

1. On a phone, join the WiFi network **RoboCar**, password **robocar123**
2. Android will warn that the network has no internet. Choose **stay connected**, and turn off *switch to mobile data automatically* in the WiFi settings
3. Open a browser and go to **192.168.4.1**
4. Hold the phone sideways

To rename the network, edit the two text fields in the `start web control` block.
The password must be at least 8 characters.

## Control page layout

The page has three tabs. Only one panel is visible at a time; a small tab bar switches
between them.

```
+------------------------------------+
| [ DRIVE ]    LIGHT    MORE         |
+------------------+-----------------+
|        ▲         |  DIST 23 cm     |
|                   |  SPEED 70%      |
|    ◀   ■   ▶      |  ===o=======   |
|                   |  SERVO 90°      |
|        ▼          |  =====o=====   |
|                   |[☀ LIGHT][♪ HORN]|
+------------------+-----------------+
```

**DRIVE** is the tab above — the d-pad (hold to drive, release to stop), the live
ultrasonic distance, the SPEED and SERVO sliders, and the LIGHT/HORN buttons. Keyboard
arrows or WASD work here too, which is handy for testing from a laptop.

The `DIST` line refreshes about every 0.8 s and shows `--` when no echo comes back — no
sensor plugged in, or nothing in range. It never delays a driving command: the poll skips
its turn whenever a command is already on its way to the board.

**LIGHT** — eight colour swatches, a target selector (headlight 0, headlight 1, or
both), DIM +/- buttons, eight numbered LED toggles, and ALL ON / ALL OFF for the GPIO
LEDs.

**MORE** — four servo sliders (D4, D5, D18, D19), two motor sliders (M1, M2, spring
back to 0 on release), SONG and BEEP buttons, a tone slider with a PLAY button, and a
live readout of the two on-board buttons and the ultrasonic distance
(`D34 ● D35 ○ DIST 23 cm`, or `DIST -- cm` when nothing echoes back). This tab polls
twice as often as DRIVE, about every 0.5 s; the LIGHT tab does not poll at all.

## Commands sent by the page

The rule, no exceptions: **the command word comes first, then an underscore, then any
numbers it carries** — split the path at underscores and element 1 is always the word,
elements 2 and up are always numbers. A path with no underscore is a bare word.
`Light1` and `Light0` are deliberately whole words, not `Light` plus a number: the
worksheet teaches "different word, different action" before it teaches passing a value.

| Path | Word | Numbers | Effect |
|---|---|---|---|
| `/Forward` `/Backward` `/Left` `/Right` `/Stop` | as-is | — | drive |
| `/Light1` `/Light0` | as-is | — | white headlights on / off |
| `/Horn` | Horn | — | two-tone beep |
| `/Speed_70` | Speed | 70 | motor power % |
| `/Servo_120` | Servo | 120 | arm servo (D4), 0-180 |
| `/ServoB_120` `/ServoC_120` `/ServoD_120` | ServoB / ServoC / ServoD | angle | servos D5 / D18 / D19, 0-180 |
| `/Rgb_2_255_120_0` | Rgb | which, r, g, b | which: 0, 1, or 2 = all |
| `/Dim_25` `/Dim_-25` | Dim | delta % | change RGB brightness |
| `/Led_3_1` | Led | index 1-8, 0/1 | one GPIO LED |
| `/Leds_1` `/Leds_0` | Leds | 0/1 | all 8 GPIO LEDs |
| `/Motor_1_80` `/Motor_2_-80` | Motor | which 1/2, power -100 to 100 | one motor, raw, trim applied |
| `/Tone_440_300` | Tone | hz, ms | buzzer |
| `/Song` | Song | — | Happy Birthday |
| `/Beep` | Beep | — | one short beep |
| `/Status` | — | — | handled inside `KSU web command`, never reaches your program |

`/Status` is answered directly by the board — plain text `b1,b2,speed,arm,cm`, for
example `1,0,70,90,23` (b1 = button D34 pressed 1/0, b2 = D35, cm = ultrasonic distance,
0 when nothing echoes back). It never shows up as a command word, so you cannot write an
`if` against it; the MORE tab's button and distance readout uses it behind the scenes.

Every one of these can be typed straight into a browser address bar, which makes a
good demonstration that the page is nothing but a set of shortcuts.

Capital letters matter. Every command word starts with a capital letter and the rest
of the word is lower case.

## The KSU_RoBoESP32 library

Everything the car needs lives in one library, so a project only holds the part a
student writes. Add it to a new project with the **+** button, or open one of the
`.ubp` files where it is already included.

```
-- web --
KSU start web control name [RoboCar] password [robocar123]
KSU web command                                    (reporter)
KSU command value                                  (reporter)
KSU command value # [1]                            (reporter)
KSU do command [ ]
KSU IP address                                      (reporter)

-- driving --
KSU forward · KSU backward · KSU turn left · KSU turn right · KSU stop
KSU set speed [70] %
KSU drive [forward] for [500] ms
KSU steer [0] (-100 left to 100 right)
KSU set motor [M1] power [70] (-100 to 100)
KSU set motor trim M1 [-1] M2 [-1]

-- servos --
KSU set arm angle [90] degrees (0 to 180)
KSU set servo [D4] to [90] degrees (0 to 180)

-- lights --
KSU light [on]
KSU set RGB [all] color [ ]
KSU set RGB [all] red [255] green [240] blue [200]
KSU change RGB brightness by [25] %
KSU LED [D16] [on]
KSU LED number [1] (1 to 8) [on]

-- sound --
KSU horn · KSU beep
KSU play tone [440] Hz for [300] ms
KSU play note [C] octave [0] for [400] ms
KSU play Happy Birthday

-- input --
KSU button [D34] is pressed                        (reporter)
KSU read analog pin [ ]                            (reporter)
KSU distance cm                                    (reporter)
KSU set ultrasonic trig [33] echo [32]
```

`KSU distance cm` pings the HC-SR04 and reports whole centimetres, or 0 when no echo
comes back within about 4 metres — nothing plugged in, nothing in range, or a surface
too soft or slanted to bounce the ping back. It waits for the echo, so it takes as long
as the ping does (up to 25 ms with no answer); call it once per turn of the `forever`
loop, not several times in a row. `KSU start web control` sets trig 33 / echo 32, so
`KSU set ultrasonic` is only needed to move the sensor to other pins — see the pin note
under **Hardware** before picking them.

```
forever
  set command to (KSU web command)
  KSU do command (command)
  set cm to (KSU distance cm)
  if (cm > 0) and (cm < 20)
    KSU stop
    KSU beep
  wait 10 millisecs
```

The library pulls in Robo ESP32, NeoPixel, WiFi, HTTP server and Tone by itself.

`KSU web command` is the block that hides the difficult part. Each time it runs it checks
for an incoming request, serves the control page when a browser asks for it, replies
to the browser, and reports the command as a plain word such as `Forward`. Students only
write "if this word arrives, do this".

Commands that carry numbers (`Speed`, `Servo`, `Rgb`, `Motor`, `Tone`, and the rest)
arrive with the word reported by `KSU web command` and the number(s) stored alongside
it. `KSU command value` reports the first number; for commands that carry more than one
(`Rgb`, `Motor`, `Tone`), use `KSU command value # _` to read the 2nd, 3rd, and so on:

```
if (command) = 'Speed'   KSU set speed (KSU command value) %
if (command) = 'Servo'   KSU set arm angle (KSU command value) degrees
```

The servo starts centred at 90 degrees.

### KSU do command

`KSU do command _` takes the word from `KSU web command` and runs the matching block
itself — forward, speed, RGB colour, LED, tone, all of it. A complete driving program
becomes four blocks:

```
KSU start web control name [RoboCar] password [robocar123]
forever
  set command to (KSU web command)
  KSU do command (command)
  wait 10 millisecs
```

This is what powers the LIGHT and MORE tabs: every button on those tabs sends a
command word that only `KSU do command` understands, so there is nothing to wire by
hand for them. The worksheet's Part 2 still wires the ten core commands one `if` at a
time, so students see how each block works before trusting `KSU do command` to do it
for them.

**Internal**

`_ksu send control page` serves the HTML; `_ksu send status` answers `/Status` for the
MORE tab's live button readout. The page itself lives in `_KSU_ROBOESP32_p1` through `_KSU_ROBOESP32_pN` —
a MicroBlocks script cannot compile to more than 1000 bytes, so the page is split into
several small blocks and glued back together with `join`. Keep each block well under
800 bytes when editing.

## Pin map

| Function | GPIO |
|---|---|
| M1A / M1B | 12 / 13 |
| M2A / M2B | 14 / 27 |
| RGB (2 NeoPixels) | 15 |
| Buzzer | 23 |
| Servo headers | D4, D5, D18, D19 |
| GPIO LEDs | 16, 17, 21, 22, 25, 26, 32, 33 |
| User buttons | D34, D35 |
| HC-SR04 trig / echo | 33 / 32 (shared with LED 8 and LED 7) |

## Library versions embedded in the files

| Library | Version |
|---|---|
| Robo ESP32 (Cytron) | 1.1 |
| NeoPixel | 1.15 |
| Servo | 1.4 |
| Tone | 1.11 |
| WiFi | 1.10 |
| HTTP server | 1.3 |

These are the newest versions published at the time of writing, so MicroBlocks should
open the files without asking to update anything. If a future release does ask,
answering YES and saving the file again is safe and stops the question returning.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Forward drives backward **and** left turns right | Both motors are wired in reverse. The library already starts with both trims at `-1` for this. |
| Pressing forward spins the car in place | Only one motor is reversed. Use `KSU set motor trim M1 [1] M2 [-1]`, and if that does not help try `M1 [-1] M2 [1]`. |
| Forward and back are correct but left and right are swapped | The motor leads are in the wrong terminals. Swap the M1 and M2 plugs. |
| Phone cannot open the page | Check that the phone is on the RoboCar network, not mobile data. The address is `192.168.4.1` with no `https`. |
| Board resets when the servo moves | The servo is drawing too much current. Use a fresh battery or a separate supply for the servo. |
| A servo on D5, D18, or D19 does not move | Nothing is plugged into that header. The block still runs and reports success either way — the board has no way to tell whether a servo is actually connected. |
| `KSU distance cm` always reports 0 | Check the sensor has 5V, not 3.3V, and that trig and echo are not swapped (trig 33, echo 32). If the trig wire has been moved, make sure it is not on 34, 35, 36 or 39 — those are input-only and cannot fire the sensor — and that `KSU set ultrasonic` was told the new pins. |
| LED 7 or LED 8 will not switch, or the distance goes wrong when they do | Those two LEDs are on pins 32 and 33, the same pins as echo and trig. Leave them alone while the sensor is plugged in, or move the sensor to other pins with `KSU set ultrasonic`. |
| The distance jumps around or reads 0 every other time | Ping no faster than about once every 60 ms; back-to-back pings hear the echo of the previous one. Soft, thin or slanted surfaces reflect the ping away from the sensor, and below ~2 cm the sensor cannot hear its own echo at all. |
| The page feels sluggish | The DRIVE and MORE tabs poll `/Status` (every 800 ms and 500 ms) over the same single connection the drive commands use, and each poll pings the ultrasonic sensor, which blocks the loop for up to 25 ms when nothing answers. The LIGHT tab does not poll at all — switch to it if you want the board to yourself. |
| `Script too large; over 1000 bytes!` | A block holds too much text. Split it into two blocks and join them. |

## Switching to a home router instead of a hotspot

Open the `KSU start web control` block and replace `wifi create hotspot` with
`wifi connect to _ password _` from the WiFi library, fill in the
network name and password, and press Start. The IP address then appears in the speech
bubble — `KSU IP address` reports it too — and it changes from network to network — which is exactly why the hotspot is
the better choice in a classroom.
