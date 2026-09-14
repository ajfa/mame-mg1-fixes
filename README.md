# MAME fixes found on the Whitechapel MG-1

Two fixes for MAME, found while bringing up 42nix 2.6 on the emulated
Whitechapel Computer Works MG-1 (the `mg1` driver: NS32016, 1984). Both are in
shared devices rather than in the driver, so they reach beyond this one machine.
They apply to MAME 0.289.

## 1. uPD7261 hard disk controller: step rate ten times too slow

`src/devices/machine/upd7261.cpp`

The datasheet gives the ST506 stepping rate as

    (16 - stp) * 2110 * tCY

and the code used `21100`, with a comment wondering whether the datasheet was
wrong by a factor of ten. It is not. With `stp` at 15 and a 10 MHz clock, the
larger constant works out at 2.11 ms per cylinder, so a full stroke across a
918 cylinder drive would take almost two seconds.

Measured on the MG-1 running 42nix: each 300 cylinder seek cost 0.633 seconds
of emulated time. The boot time `fsck` seeks on nearly every metadata read, so
reaching the login prompt took about fifteen minutes. With the datasheet value
it takes about ninety seconds.

## 2. MC6801 SCI: the receiver never resynchronises to the start bit

`src/devices/cpu/m6800/m6801.cpp`, `src/devices/cpu/m6800/m6801.h`

`sci_clock_internal()` calls `sci_tick()` once every `divider` clocks, and each
tick samples one bit. There is no oversampling and no start bit
resynchronisation, so the sampling point keeps whatever phase the free running
prescaler happened to hold when the frame arrived. A real SCI oversamples the
line, restarts its bit period on the falling edge of the start bit, and samples
near the middle of every bit cell.

The symptom is a serial link that delivers most bytes correctly and mangles one
every so often. On the MG-1 that link is the keyboard, and the driver already
carried the note `keyboard intermittently outputs garbage`: typing `root` came
out as `obot`, `echo` as `ELCHO`, and a phantom Caps Lock dropped the console
into the old LCASE mode.

Measured: the keyboard MCU bit-bangs its output at 837.5 us per bit, obtained by
logging the transitions of P1.1 on its 8039, while the IOP's SCI expects
833.3 us. Half a percent of difference is enough for the sample point to land on
a bit edge sooner or later.

The fix tracks the previous state of the receive line and, on the falling edge
of a start bit, sets the prescaler so the next tick lands in the middle of that
start bit. With it, four test lines of 35 characters each arrived byte perfect,
where before roughly one character in thirty was wrong.

## Applying

    cd /path/to/mame
    git apply /path/to/patches/0001-upd7261-datasheet-step-rate.patch
    git apply /path/to/patches/0002-m6801-sci-start-bit-resync.patch

And, for an unattended run only, see the section below:

    git apply /path/to/patches/0003-pack-only-no-warning-screens.patch

`patch -p1 < ...` works as well.

A build cut down to this one machine is enough to exercise them:

    make SUBTARGET=mg1 SOURCES=src/mame/mg1/mg1.cpp

## The machine these came from

Two documents in [`docs/`](docs), written while doing it:

- **[Running 42nix 2.6 on the emulated MG-1](docs/running-42nix.md)** - where
  the ROMs and the disk image are published, how to assemble the ROM set MAME
  expects, the drive geometry `chdman` needs, the command line, logging in and
  shutting down cleanly, and how to skip the boot time `fsck`.
- **[What is on the disk](docs/whats-on-the-disk.md)** - the filesystems and
  accounts, Whitechapel's Oriel window system, the 39 programs in `/usr/wcw`
  including which graphics demos work and which do not, the C compiler, the
  manual pages and the games.

Two things from there are worth repeating here, because both look like
emulation faults and are not.

**The disk goes on the second hard disk slot.** The MG-1 addresses its drive as
unit 1 of the uPD7261, so `-hard1` gets `Hard disk error: Drive not ready` from
the system ROM and `-hard2` boots. 42nix says the same thing on the way up:
`hd0 at upd0 slave 1`.

**The driver's `TODO: mouse` is stale.** The IOP reads the mouse fifty times a
second, keeps the pointer position in the mailbox it shares with the host and
programs the cursor counters; the pointer walks the screen and stops at the
edge on its own. What is needed is starting MAME with `-mouse`, which defaults
to off. Note that the IOP scales the delta by shifting right, so injecting
steps of two or three counts per poll moves nothing and looks like a fault.

## A third patch, for unattended runs only

`patches/0003-pack-only-no-warning-screens.patch` is not a fix and is not meant
for upstream. The driver is marked `MACHINE_NOT_WORKING`, and with both fixes
above the machine boots 42nix, the keyboard is reliable and the mouse moves the
pointer; whether that flag should come off is the driver author's call, so the
two fixes leave it alone.

What the flag costs a harness is the warning screen MAME shows before it starts,
which waits for a keypress that a run starting on its own cannot answer. There
is no command line option for that screen: `-skip_gameinfo` covers the machine
information screen before it, and `skip_warnings` in `ui.ini` only suppresses
repeats, and only for a few days. This patch clears the flag so the screen never
appears.

Apply it only for that. On a tree meant for development, leave it out.

## Licence

BSD-3-Clause, the same licence as the MAME sources these patches touch. See
[LICENSE](LICENSE).
