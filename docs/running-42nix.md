# Running 42nix 2.6 on the emulated MG-1

Everything below was done with MAME 0.289 plus the two patches in this
repository. Without them the machine still boots, but the boot takes about
fifteen minutes and the keyboard mangles roughly one character in thirty.

## The media

All of it comes from Tony Duell's collection, published at
[mg-1.uk](https://mg-1.uk/). Nothing in this repository redistributes it.

### ROMs

From the ROMs section of <https://mg-1.uk/>:

| file | size | what it is |
| --- | --- | --- |
| [`U291_and_U292-SYS_2.51_198x0113.bin`](https://mg-1.uk/U291_and_U292-SYS_2.51_198x0113.bin) | 64 KiB | system ROM, the code the CPU runs at power on |
| [`U285_IOP_3.0_19870724.bin`](https://mg-1.uk/U285_IOP_3.0_19870724.bin) | 32 KiB | ROM for the I/O processor |

Both are also mirrored on the Internet Archive, as
[WCW_MG-1_SYS_2.51_19860912](https://archive.org/details/WCW_MG-1_SYS_2.51_19860912)
and
[WCW_MG-1_IOP_3.0_19870724](https://archive.org/details/WCW_MG-1_IOP_3.0_19870724).

### Disks

From <https://mg-1.uk/42nix/42nix.html>:

| file | size | what it is |
| --- | --- | --- |
| [`42nix_2.6_Maxtor_XT1140.img.bz2`](http://archive.mg-1.uk/42nix/42nix_2.6_Maxtor_XT1140.img.bz2) | 8.7 MiB | the one you want: a 121 MB hard disk with 42nix 2.6 already installed |
| [`42nix_2.6_Maxtor_XT1140.emu.bz2`](http://archive.mg-1.uk/42nix/42nix_2.6_Maxtor_XT1140.emu.bz2) | 23 MiB | the same drive in Gesswein MFM emulator format, for real hardware |
| [`42nix_2.6_floppies.tar.bz2`](http://archive.mg-1.uk/42nix/42nix_2.6_floppies.tar.bz2) | 7.9 MiB | the 43 installation floppies |
| [`42nix_00_2.5_Boot_Floppy_(2.6_Bootstrap)_hacked.img`](http://archive.mg-1.uk/42nix/42nix_00_2.5_Boot_Floppy_%282.6_Bootstrap%29_hacked.img) | 800 KiB | boot floppy |

The hard disk image uncompresses to exactly 126,904,320 bytes.

## Building the ROM set

MAME wants the system ROM split into its two original chips and the IOP ROM cut
down to the 8 KiB that is actually mapped. The system ROM file holds the two
chips interleaved: even bytes of the first 16 KiB are U291, odd bytes are U292.

```python
import zipfile

sysrom = open('U291_and_U292-SYS_2.51_198x0113.bin', 'rb').read()[:16384]
iop    = open('U285_IOP_3.0_19870724.bin',           'rb').read()[:8192]

with zipfile.ZipFile('mg1.zip', 'w', zipfile.ZIP_DEFLATED) as z:
    z.writestr('even_2.51__sys__13.1_u291.u291', sysrom[0::2])
    z.writestr('odd_2.51__sys__13.1_u292.u292',  sysrom[1::2])
    z.writestr('3.0__iop__24.7.87.u285',         iop)
```

Check them against what MAME expects:

| member | CRC32 | SHA1 |
| --- | --- | --- |
| `even_2.51__sys__13.1_u291.u291` | `677cab3c` | `d0197b45ddb1ddd8cd125727312b06dcae0f984a` |
| `odd_2.51__sys__13.1_u292.u292` | `b0134a98` | `a81bd4987030b09799bad0c3bc758ea8aed8cd2f` |
| `3.0__iop__24.7.87.u285` | `733cd089` | `31ffdd85b4ae2ac35dcde292a0d42860baaba88d` |

One more ROM is required and is not published at mg-1.uk: `2716.z4`, 2 KiB, the
firmware of the keyboard's 8039, which MAME looks for in `mg1_kbd_device.zip`
(or inside `mg1.zip`). It is part of MAME's `mg1` ROM set. Its hashes are
`ae325c37` / `c6e90b10241fa61e12124ef9a5cea15212d5866c`.

`mame -rompath roms -verifyroms mg1` should say `romset mg1 is good` before you
go any further. The v2.60 system ROMs listed by `-listroms` are a second BIOS
option and are not needed if you select the 2.51 one.

## Building the disk

The drive is a Maxtor XT1140: 918 cylinders, 15 heads, 18 sectors of 512 bytes.
That geometry has to be given to `chdman`, it cannot be guessed from the file.

```
bunzip2 42nix_2.6_Maxtor_XT1140.img.bz2
chdman createhd -i 42nix_2.6_Maxtor_XT1140.img -o xt1140.chd \
       --chs 918,15,18 --sectorsize 512
```

## Running it

```
mame mg1 -rompath roms -bios 260 -hard2 xt1140.chd -mouse -window -nomaximize
```

Three things in that command line are not obvious.

**The disk goes on `-hard2`, not `-hard1`.** The MG-1 addresses its drive as
unit 1 of the uPD7261 controller. On `-hard1` the system ROM answers
`Hard disk error: Drive not ready - drive 0 cyl 0 head 0 sector 0`. 42nix says
the same thing on the way up: `hd0 at upd0 slave 1`.

**`-mouse` is needed**, because MAME leaves mouse input off by default and the
pointer will not move without it. To hand the pointer back to the host: right
Control and right Alt together.

**`-bios 260` or `-bios 251`** both boot from the disk. The driver notes that
floppy support behaves better with 2.51.

The system ROM shows a menu; press `0` for "Boot from hard disk 0". To have
MAME do it for you, add `-autoboot_command "0" -autoboot_delay 25`.

## Logging in and shutting down

Log in as `root`. It has no password in this image.

To stop the machine cleanly, from the shell:

```
/etc/halt
```

Wait for it to report that it has halted, then close the emulator. 42nix keeps
the disk mounted with writes outstanding, so quitting the emulator on a running
system risks the filesystem. MAME writes changes to a diff file rather than to
the CHD, so deleting the diff directory always brings the disk back to the
state it shipped in.

## The fifteen minute boot, and how to skip it

With the uPD7261 patch a cold boot reaches the login prompt in about 100
seconds of emulated time. Most of that is `fsck`, which `/etc/rc` runs on all
three filesystems on every boot.

42nix already carries a way out. `/etc/rc` starts with

```sh
if [ -r /fastboot ]; then
	rm -f /fastboot
	echo 'Fast boot ... skipping disk checks'	>/dev/console
elif [ $1x = autobootx ]; then
	echo 'Automatic reboot in progress...'		>/dev/console
	/etc/fsck -p					>/dev/console
```

Editing that test so it is always true cuts the boot to a few seconds of disk
work. The three filesystems in the published image are clean, checked with
42nix's own `fsck`. Keep in mind that skipping the checks means a disk that was
not shut down properly will not be repaired.

## What it costs

Measured with a window open and 42nix sitting at the shell prompt: about 30 per
cent of one core of a modern x86, and roughly 260 MB resident. It runs at the
real speed of an 8 MHz NS32016, no faster.
