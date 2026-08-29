# CH375
"New" CH375 DOS Driver - All in one - Optimized

Using the original V1.9 DOS Driver as the base.

Idea is to have 8086/8088/V20/V30/286 support in one file with optimizations applied.

Currently only the V30 has been tested.

<img width="2048" height="1153" alt="image" src="https://github.com/user-attachments/assets/666eb8a8-1de3-4d1f-b66f-a9915038afb3" />

<img width="2048" height="1153" alt="image" src="https://github.com/user-attachments/assets/bdf1f6f5-cd49-4b0d-90e1-5e3390f0b087" />

# CH375 USB Disk Driver V2.1 — User Guide

Optimized by StevenC. Based on WCH's CH375DOS.SYS V1.9.

This driver lets MS-DOS read and write ordinary USB flash drives through a
CH375-based ISA card, so the stick appears as a normal drive letter. You can
`COPY`, `DIR`, `XCOPY` and run programs from it exactly as you would a floppy
or hard disk.

---

## What you need

**Hardware** — a CH375-based ISA card (the industrial boards sold on eBay and
AliExpress), and a free I/O address range. Any PC from an 8088 upward. No IRQ
is needed; the driver polls the card.

**DOS version** — 3.3 or later. The driver declares 32-bit sector addressing,
which DOS 4.0 and later use to handle volumes over 32 MB, so DOS 5 or 6 is the
comfortable choice.

**The USB stick must be formatted FAT16.** DOS cannot read FAT32 or exFAT, and
most modern sticks arrive as one or the other. Format it on a PC as FAT (not
FAT32) with a single partition of 2 GB or less. A 1 GB or 2 GB stick is ideal;
larger ones work if you make a small first partition.

---

## Installing

Copy `CH375EXT.SYS` to your boot drive — the root directory is easiest — then
add one line to `CONFIG.SYS`:

```
device=c:\CH375EXT.sys @260
```

Change `@260` to whatever I/O address your card is jumpered or configured for.
This is the one setting you must get right. Reboot.

---

## What you should see

```
Driver for CH375 USB-Disk V1.9
Copyright (C) W.ch 1998-2007
Optimized by StevenC  V2.1
CPU NEC V20/V30 or 80186, block I/O (REP INSB/OUTSB)
I/O address = 0260H, interrupt = 00  , add disk E:
```

Line by line:

**Lines 1–2** are the original vendor notice, unchanged.

**Line 3** identifies this build.

**Line 4** tells you which CPU was detected and which transfer method is in
use. This is worth reading — it is the quickest way to confirm the driver is
doing what you expect:

| you see | meaning |
|---|---|
| `CPU 8086/8088, byte loop` | Original 8086 or 8088. Correct — those CPUs lack the block-transfer instructions. |
| `CPU NEC V20/V30 or 80186, block I/O` | Best case. Block transfers active. |
| `CPU 80286 or later, block I/O` | Also block transfers. See the note on fast machines below. |
| `...byte loop` on a V20/V30 or better | Something forced the slow path — check for a `%` or `&` option on your `DEVICE=` line. |

**Line 5** confirms the I/O address the driver used and the drive letter it
claimed. `interrupt =` is blank because no IRQ is used.

If the stick is not plugged in at boot, the driver still loads and claims a
letter; plug in and it will pick the disk up.

---

## Command line options

Only `@` is normally needed.

| option | purpose |
|---|---|
| `@n` | **I/O base address in hex.** Required — must match your card. |
| `%n` | I/O delay. Default `0`. Adds spacing between port accesses for cards or machines that need it. Any nonzero value turns off the fast transfer paths, so use it only if you must. |
| `#n` | Interface mode. `91` or `92` select parallel-port operation. Leave it off for an ISA card. |
| `&n` | Force transfer method: `&1` block I/O, `&0` byte-by-byte. Not normally needed — the CPU is detected automatically. Use only for testing. |

Example with everything default except the address:

```
device=c:\CH375EXT.sys @260
```

---

## What performance to expect

Measured on an 8 MHz NEC V30:

| | this driver | original V1.9 |
|---|---:|---:|
| write | 270 KB/s | 28 KB/s |
| read | 302 KB/s | 27 KB/s |

Roughly **9× faster writing and 11× faster reading** than the stock driver.

For comparison, that read rate is around twenty times a 360 K floppy and in
the same territory as a period IDE hard disk on an XT-class machine. Copying
a 10 MB file takes about 35 seconds.

On an original 8086 or 8088 the driver falls back to byte-at-a-time transfers,
because those CPUs lack the block instructions. It is still considerably
faster than stock, but slower than the figures above. Those configurations
have not been measured.

Speed is limited mainly by the CH375 chip's own USB timing, not by your CPU,
so a faster machine will not help much beyond this point.

---

## Using it day to day

The stick behaves like any other DOS drive. `DIR E:`, `COPY`, `XCOPY /S`,
running programs from it, using it as a target for `BACKUP` — all normal.

**Before unplugging**, make sure nothing has files open, and if you use a
disk cache run whatever flush it provides. DOS may hold unwritten data
briefly. Unplugging mid-write can corrupt the filesystem, exactly as it would
on a hard disk.

**Swapping sticks** while DOS is running is risky. DOS caches directory
information per drive, and although the driver supports media-change
detection, the safest habit is to swap only when nothing is open — or reboot
after a swap.

---

## If something goes wrong

**Nothing prints at boot, or the machine hangs.**
Almost always the wrong `@` address. Check your card's jumpers or
documentation and try the correct value. A wrong address means the driver
talks to whatever else lives there, which can wedge the machine.

**`CH375: disk not found`**
No USB stick detected. Check it is inserted, try reseating it, and try
another stick — some draw more power than an ISA card can supply.

**`CH375: error boot sector`**
The stick was found but DOS cannot make sense of its filesystem. Almost
always FAT32 or exFAT. Reformat as FAT16 with a partition of 2 GB or less.

**`CH375: I/O data error`**
A transfer failed. Try `%1` on the `DEVICE=` line to add bus spacing:

```
device=c:\CH375EXT.sys @260 %1
```

If that fixes it, try `%2` or `%3` only if `%1` is not enough. Bear in mind
each step turns off the fast paths, so you trade speed for reliability.

**`CH375: interrupt timeout` or `CH375: wait serial`**
The card is not responding as expected. Check the address first, then power
and seating.

**It loads but the drive letter does not work.**
Check `LASTDRIVE` in `CONFIG.SYS` — if you have several drives you may need
`LASTDRIVE=Z`.

**Files copy but come back corrupted.**
Stop using the stick for anything important and try `%1`. See the next section
for how to confirm a fix.

---

## Checking your setup is reliable

Worth doing once, before trusting the stick with real data:

1. Copy a large file — 10 MB or more — from your hard disk to the stick.
2. Copy it back under a different name.
3. Compare: `FC /B C:\ORIGINAL.ZIP C:\RETURNED.ZIP`

`no differences encountered` means reads and writes are both sound. If you
see byte offsets listed, add `%1` and repeat. Use a compressed file — a `.ZIP`
or a disk image — because any corrupted byte shows up rather than hiding in
slack space.

Running `CHKDSK E:` afterwards is a useful second check; new lost clusters
after a copy would point at a problem.

---

## A note on fast machines

This build was tuned and verified on an 8 MHz V30. The timing between port
accesses was tightened to gain speed, which is safe at that clock rate.

On a 386, 486 or faster machine, the same code drives the card harder, and if
the card cannot keep up the failure is **silently wrong data rather than an
error message**. If you are running this on anything faster than roughly a
286, do the verification above before trusting it, and use `%1` or `%2` if it
fails.

---

## Limitations

- FAT16 only, one partition, 2 GB maximum. No FAT32, no exFAT, no NTFS.
- Not bootable — DOS must already be running before the driver loads.
- Uses about 4.3 KB of conventional memory.
- One card and one USB stick at a time.

---

Hobbyist software, provided as-is with no warranty. The original driver
remains © W.ch 1998–2007; this is an optimized derivative.

