# cmdr-unoq-ir-speaker

**Point a remote at the board and it says the button name out loud.**

A [commander](https://github.com/gbryant/commander) consumer for the **Arduino Uno Q** —
and the clearest demonstration of why that board is interesting. The Uno Q has two
brains: an **STM32U585 (M33)** running commander under Zephyr, and a **QRB2210**
running Debian. This project puts each one where it belongs.

- The **M33** decodes IR. It's a hard-real-time job — edge timing at microsecond
  resolution — and it's all the MCU does.
- The **Debian side** matches the press against a button map and speaks it through a
  warm neural TTS voice. That's a job wanting a filesystem, Python, and a few hundred
  MB of RAM.
- The **channel bus** joins them: the M33 publishes presses on channel 1, and a Linux
  process subscribes. Neither side blocks the other, and the human console stays free
  on channel 0 the whole time.

**There is no custom firmware.** `src/main.cpp` is the stock template — a
`commander_config()` and a `commander_setup()` that registers whatever `cmdr` enabled.
The device was composed, not coded:

```bash
cmdr init unoq cmdr-unoq-ir-speaker
cmdr module enable ir          # NEC/Sony decode on D5, published on channel 1
cmdr autostart add "ir recv"   # stream from boot — no command ever sent
```

That's the whole firmware story. The interesting logic lives on the Linux side, in
Python, where it's easy to change.

## What you'll hear

Press a button on a mapped remote and the board announces it — "volume up", "play",
"power". Unmapped remotes still print the raw protocol/address/command, which is how
you build a new map (see below).

`ir_speak.py` handles held buttons the way you'd want: a held button announces **once**
and re-announces only after you release it. Switching remotes mid-stream doesn't
announce the button you just left — a frame has to be the newest for two consecutive
reads before it counts, so a lone trailing frame gets superseded and dropped.

Seven remote maps ship in `maps/` (Sony DVD, MiniDisc, CD, audio system, digital
video; Hisense Roku; Vizio sound bar).

---

## Setup

Prereqs on your machine: a Zephyr/`west` checkout (`~/zephyrproject`), the Arm GNU
Toolchain, and `adb`. Override `ZEPHYR_BASE` / `GNUARMEMB_TOOLCHAIN_PATH` / `GDB` if
yours live elsewhere.

### 1. Build (also fetches the framework)

```bash
./build      # west build; FetchContent pulls commander into build-unoq/, including the broker
./flash      # openocd-over-adb gdb load (west flash isn't working for this board upstream)
./monitor    # ch0 console over the USB-CDC gadget (tio); type `help`  (after ./install-broker)
./bum        # build + flash + monitor
```

The Zephyr SDK has no Intel-Mac toolchain, which is why `./build` sets `gnuarmemb`.

### 2. One-time board setup (reversible, needs board sudo)

Run these **after a first `./build`** — `install-broker` pushes the broker from the
fetched commander source, so the framework has to be downloaded first.

```bash
./enable-flash-boot   # M33 boots from flash (STM32 option bytes)
./install-broker      # broker service owns /dev/ttyHS1; bridges ch0 → USB serial, chN → sockets
```

`enable-flash-boot` is not optional and not obvious: **the Uno Q ships booting its
STM32 ROM bootloader, not flash.** Skip it and your firmware never runs — the link is
dead silent, which looks exactly like a wiring fault. Both scripts have reverts
(`./restore-arduino`).

### 3. Install the voice (one time, on the board)

From the [unoq-tools](https://github.com/gbryant/unoq-tools) repo:

```bash
./setup-tts.py            # installs Piper + a voice into a venv on the board
./tts.py daemon install   # systemd --user unit; keeps the voice warm
```

The daemon takes fire-and-forget lines on a FIFO at `/run/user/<uid>/tts.fifo`, so
speaking a button costs no model load. Without it `ir_speak.py` falls back to
`espeak-ng`, and without that it prints only — the fallback is per-utterance, so a
daemon restart mid-demo just means a few robotic announcements rather than silence.

For audio out, `unoq-tools` also has `setup-bt-audio.py` and `bt.py` for pairing a
Bluetooth speaker headlessly (`docs/unoq-bluetooth-audio.md` covers the details —
notably that the stack is PipeWire, and that WirePlumber's bluez seat-monitoring has
to be disabled for headless operation).

### 4. Run it

```bash
./deploy-sbc                                            # push bin/ tools + seed maps/ to the board
adb shell "cd /home/arduino && python3 ir_speak.py"     # press a remote → hear the button name
```

That's the demo. The board is already streaming IR (autostart), so `ir_speak.py` starts
nothing — it's a pure subscriber to `ch1.sock`.

## The other tools

All in `bin/`, all pure subscribers, all run on the SBC next to the broker:

```bash
adb shell "cd /home/arduino && python3 ir_lookup.py"           # identify presses, print only
adb shell "cd /home/arduino && python3 ir_map.py -o sony.json" # build a map for a new remote
adb pull /home/arduino/sony.json maps/                         # keep new maps in version control
```

`ir_map.py` is press-driven: press a button, name it, repeat, Ctrl-C to save. It resumes
from an existing map, so you can build one over several sittings, and `--selects` handles
remotes with a multi-position source switch. The JSON format is identical to the serial
boards' maps, so they're interchangeable.

To watch the raw stream: `adb shell "socat - UNIX-CONNECT:/tmp/commander/ch1.sock"`.

To toggle the stream by hand, open a command session on `ch2` and run `ir recv` — the
consuming tools never touch it, and neither does the human console on ch0. That
separation is the channel bus's whole point: several processes can talk to the board at
once without stepping on each other.

## Revert to stock Arduino

```bash
./restore-arduino    # disable the broker, restore the bridge + router, optionally revert boot bytes
```

## Background

In the commander repo: `docs/getting-started-unoq.md` (the board track),
`docs/zephyr-hal-spike.md` (boot fix + flash path), `docs/commander-channels-bringup.md`
(the bus), `docs/unoq-access.md` (access map), and `dev/unoq/` (the service units).
