# Worksheet — WiFi Robot Car

**Name** ............................................ **Group** ..............

## Goals

1. Connect blocks so the car responds to every button on the control page
2. Explain how a command travels from a phone screen to a motor
3. Add a new button of your own and make the car react to it

---

## Part 1 — Try it first (10 minutes)

1. Open `robocar-STUDENT.ubp` in MicroBlocks
2. Plug in the USB cable, choose the board, press **Start**
3. Wait until the RGB LEDs glow dim green and the servo arm centres itself at 90 degrees
4. On your phone, join the WiFi network **RoboCar**, password **robocar123**
5. Open a browser and go to `192.168.4.1`, then hold the phone sideways

**Q1.** Which controls work already, and which ones do nothing?

....................................................................................

**Q2.** Type `192.168.4.1/Forward` straight into the address bar. What happens, and what
does that tell you about the buttons?

....................................................................................

---

## Part 2 — Wire up the commands (20 minutes)

The loop already does two things:

```
set [command] to (KSU web command)      <- did anyone press a button?
if ((command) = 'Forward')  KSU forward   <- if that word arrived, do this
```

Drag blocks from the **toolbox** on the right into the loop until all ten commands
work. Tick each one after you test it on the phone.

| Command word | Block to use | Done |
|---|---|:--:|
| `Forward` | KSU forward | example |
| `Backward` | | ☐ |
| `Left` | | ☐ |
| `Right` | | ☐ |
| `Stop` | | ☐ |
| `Light1` | | ☐ |
| `Light0` | | ☐ |
| `Horn` | | ☐ |
| `Speed` | KSU set speed (KSU command value) | ☐ |
| `Servo` | KSU set arm angle (KSU command value) | ☐ |

**Careful:** capital letters matter. `Forward` is not the same word as `front`.

**Q3.** Delete the `if (command = 'Stop') KSU stop` block, then press and release an arrow
button. What happens and why?

....................................................................................

**Q4.** `Speed` and `Servo` need one extra block that the other eight do not. What is
it, and why do those two commands need it?

....................................................................................

---

## Part 3 — Add your own button (20 minutes)

The page lives inside the library, in a run of text blocks named `_KSU_ROBOESP32_p1`, `_KSU_ROBOESP32_p2`, and
so on up to `_KSU_ROBOESP32_pN`. Find the block containing this line:

```
<div id=r><button id=W>&#9728; LIGHT</button><button id=H>&#9834; HORN</button></div>
```

**Mission:** add a SPIN button that turns the car on the spot for one second.

1. Add `<button id=N>SPIN</button>` after the HORN button
2. Find the block containing `E("H").onpointerdown` and add this line below it:

```
E("N").onpointerdown=function(v){v.preventDefault();s("Spin")};
```

3. Make a new command block called **spin around** using `KSU turn left`,
   `wait 1000 millisecs` and `KSU stop`
4. Add `if (command = 'Spin')` to the loop and drop your new block inside
5. Press Start, then reload the page on the phone

**Rules:** never put an apostrophe `'` inside the HTML text, and keep every block under
1000 bytes. Right click a script and choose **show compiled bytes** to check.

**Q5.** Why is the page stored in several small blocks instead of one big one?

....................................................................................

---

## Part 4 — Challenges

- [ ] Add three speed buttons — slow, medium, fast — without using the slider
- [ ] Make the RGB LEDs blink while the car reverses, like a real reversing light
- [ ] Add a button that sweeps the servo arm left and right three times
- [ ] Rename the WiFi network to your group name
- [ ] Stop the car automatically if no command arrives for two seconds

---

## How a command travels

```
finger presses a button
  -> browser sends  /Forward
  -> ESP32 receives it over WiFi
  -> the KSU web command block reports the word Forward
  -> your if block matches the word
  -> the KSU forward block sets motor power
  -> wheels turn
```

**Q6.** If the phone leaves the WiFi network while the car is moving, what does the car
do? Which challenge above fixes that problem?

....................................................................................

---

## Going further

Reload the page and look at the tab bar above the d-pad: DRIVE, LIGHT, MORE. LIGHT and
MORE send many more command words than the ten from Part 2 — RGB colours, individual
GPIO LEDs, extra servos, raw motor power, tones, a song, and the two on-board buttons.

Wiring each one by hand with its own `if` block would take an hour. Instead, one block
handles all of them:

```
set command to (KSU web command)
KSU do command (command)
```

`KSU do command` already knows every word the page can send and runs the matching
block itself. Drag it into your loop below the `if` chain from Part 2, and everything
on the LIGHT and MORE tabs starts working immediately, with nothing hand-wired.

Now try a few of these, using the new blocks directly — no page required:

- [ ] Flash the 8 GPIO LEDs in sequence, one at a time, using `KSU LED number _ (1 to 8) _` in a loop
- [ ] Make the car reverse for exactly one second using a single `KSU drive _ for _ ms` block
- [ ] Use `KSU button D34 is pressed` to start the car driving forward without touching the phone at all
- [ ] Play a short tune with three or four `KSU play note _ octave _ for _ ms` blocks in a row
- [ ] Combine `KSU set RGB _ color _` with `KSU change RGB brightness by _ %` to make a slow colour fade
- [ ] Read `KSU read analog pin _` on a spare GPIO pin and report the value out loud
