# What is on the 42nix 2.6 disk

A walk through the published Maxtor XT1140 image. Directory listings were read
out of the filesystem itself; anything described as tried was run on the
emulated machine.

## The system

42nix is Whitechapel's port of 4.2BSD to the NS32016. It is UNIX by descent,
not a reimplementation: `telnetd` still announces itself as `4.2 BSD UNIX`,
`/bin/login` carries the SCCS string
`@(#) Copyright (c) 1980 Regents of the University of California`, and there
are 2,599 `@(#)` markers across the image.

The kernel is `/vmunix`, 390,144 bytes, dated 12 May 1987, and it announces
itself as `WCW 42nix 2.6 (MG-1 #7): Tue May 12 13:23:59 BST 1987`. The
installation around it is later: `flar`, `tar` and `fdfmt` are from August
1987, `ben` and `saddle` from December 1990, `/bin/csh` from March 1991,
`/etc/termcap` from June 1991.

The machine calls itself `Amnesiac`. The login shell is `csh`, and its prompt
carries the history number: `#:1>`.

## Filesystems

Three, from `/etc/fstab`:

| device | mount | size | used |
| --- | --- | --- | --- |
| `/dev/hd0a` | `/` | 6,727 KB | 77% |
| `/dev/hd0h` | `/usr` | 24,303 KB | 87% |
| `/dev/hd0g` | `/users` | 71,555 KB | 0% |

Root holds the usual 4.2BSD layout: `bin dev etc lib tmp usr users flp
lost+found vmunix`, plus `.cshrc`, `.login` and `.profile` for root.

## Accounts

`/etc/passwd` has `root` (no password), `daemon`, `bin`, `man`, `ui`, `news`,
`gnu`, `uucp`, `single`, `guest`, `games`, `mono` and `df`. The last one is a
1980s joke that still works: its shell is `/bin/df`, so logging in as `df`
prints the disk usage and hangs up.

## Oriel

Oriel is Whitechapel's window system, and it is what you are looking at from
the moment the machine boots: the screen with the woven background, the
`console` window with its scroll bars and corner widgets, and an icon strip
along the top. The window manager is `/usr/wcw/wmgr`, the terminal is
`/usr/wcw/console`, and both are visible in `ps ax` on a running system.

All of Whitechapel's own programs live in `/usr/wcw`, already compiled. The
1987 `csh` has no `PATH` entry for it, so the easy way in is `cd /usr/wcw`.

The directory holds 39 programs:

```
ben        calc       clkicon    clock      console
cp42       desksave   enlarge    fdfmt      flar
fx80it     iced       kbset      logo       mkbflp
mkcolour   mkflp      mkhd       mkhdisk    mkmflp
mkrflp     msdir      msdos      mslist     msread
mswrite    newuser    newwin     pan        pattern
petal      rotate     saddle     screeninfo setflop
show       sstf       tidy       wmgr
```

### The graphics demos

These were run one at a time and watched:

| command | what happens |
| --- | --- |
| `clock &` | an analogue clock with moving hands, in its own titled window. This is the clock. |
| `clkicon &` | the same clock as a desktop icon |
| `petal &` | a full screen spirograph, with the Whitechapel logo in the corner. The best looking of the lot. |
| `saddle &` | a ruled surface drawn line by line, in a window |
| `calc &` | a calculator; it starts iconified, so open the icon |

Three results worth knowing before you go hunting:

`logo` answers `logo: must run on colour system` and stops. That is correct
behaviour, not a fault of the emulation: the MG-1 is monochrome.

`ben` is on the disk and exits without drawing anything. The `ben` that
[mg-1.uk documents as an analogue clock](https://mg-1.uk/wcw/wcw.html) is
example source code, not this binary. If you want that one, compile it.

`rotate` wants the foreground. Started with `&` it stops immediately with
`Stopped (tty input)`. Run it without `&` and leave with Ctrl-C.

Anything that opens a window prints `Done` as soon as you start it. They are
not failing: they fork and the parent exits, and the window stays.

### The rest of /usr/wcw

Not run, so described only by what they evidently are: `console` and `newwin`
open windows, `tidy` rearranges them, `desksave`, `enlarge`, `pan`, `pattern`
and `screeninfo` work on the screen, `iced` is Whitechapel's screen editor,
`show` a file viewer, `kbset` keyboard settings, `newuser` account creation,
`fx80it` something for an Epson FX-80 printer, and `sstf`, `cp42` and
`mkcolour` are unexplored.

Two groups stand out. `mkbflp`, `mkflp`, `mkmflp`, `mkrflp`, `setflop` and
`fdfmt` make and format floppies, and `mkhd` and `mkhdisk` do the same for hard
disks; `fdfmt` is the one with a
[manual page](http://mg-1.uk/man/manjs.html#/man/man1/fdfmt.1) online. And
`msdos`, `msdir`, `mslist`, `msread`, `mswrite` read and write MS-DOS floppies,
which is how you would have moved files to a PC in 1987.

`flar` is the archiver the installation media is built from; the 43
installation floppies are `flar` archives.

## The C compiler

`cc` is installed and produces working NS32016 binaries. Tried end to end:

```
echo 'main(){ printf("hello from the MG-1\n"); }' > /tmp/h.c
cc -o /tmp/h /tmp/h.c
/tmp/h
```

It takes about twenty seconds to compile, which is what it took at the time.
`ex` and `vi` are both there, and `TERM` is already set to `vt100em`.

## Manual pages

`/usr/man` has `man0` through `man9` plus a 45 KB `whatis`, so `man` and
`apropos` both work on the machine. Whitechapel's own manual pages are also
[online](http://mg-1.uk/man/manjs.html).

## Games

`/usr/games` has the 4.2BSD set, 30 of them:

```
adventure  arithmetic  backgammon  banner   bcd
boggle     btlgammon   canfield    cfscores cribbage
factor     fish        fortune     hack     hangman
mille      number      primes      quiz     rain
sail       snake       teachgammon trek     worm
worms      wump
```

`banner` and `factor` are the quickest to show someone. `rain` and `worms`
keep running until you press Ctrl-C. `hack` and `adventure` need a terminal
that behaves, which the Oriel console does.
