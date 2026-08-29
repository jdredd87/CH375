# CH375

"New" CH375 DOS Driver - All in one - Optimized

Using the original V1.9 DOS Driver as the base.

One binary supports 8086, 8088, V20, V30 and 286. The driver probes the CPU at
load time and selects the fastest transfer path it can safely use, then patches
itself so nothing is re-tested at run time. No separate builds, no configuration.

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
claimed. The `interrupt =` field reflects the IRQ setting, which is unused
here because the driver polls the card.

If the stick is not plugged in at boot, the driver still loads and claims a
letter; plug in and it will pick the disk up.

---

## One driver, every CPU

The original V1.9 driver moved data one byte at a time through two nested
subroutine calls, which re-read three configuration variables *per byte*. That
is why it managed 27 KB/s on hardware capable of ten times more.

This build keeps a single binary for every supported CPU. At load time it
probes the processor and picks a transfer path:

| CPU | transfer path | why |
|---|---|---|
| 8086 / 8088 | tight inline byte loop | these parts have no block-I/O instructions |
| NEC V20 / V30 / 80186 | `REP INSB` / `REP OUTSB` | a whole 64-byte USB packet in one instruction |
| 80286 and later | `REP INSB` / `REP OUTSB` | same path; see the warning below |

**Detecting a V20/V30 is harder than it looks**, and the obvious methods fail.
`PUSH SP` behaves identically on the 8086 and the 80186, so it cannot separate
them. Shift-count masking exists on Intel parts because they have a barrel
shifter — which the V20/V30 lack, so that fails too. On both of the usual
tests, a V30 looks exactly like an 8086.

What works is testing the property the driver actually depends on. The 8086
decodes opcodes `60h`–`6Fh` as aliases of the conditional jumps `70h`–`7Fh`, so
`60h` (`PUSHA`) executes there as `JO rel8`. With the overflow flag clear it is
not taken, and simply swallows the byte that follows:

```asm
        xor     ax, ax        ; CF=0 and, critically, OF=0
        db      060h          ; PUSHA / on an 8086: JO $+2
        stc                   ; swallowed by that JO on an 8086
        jnc     .no_v30
        db      061h          ; POPA -- only reached on a V20/V30 or better
```

That asks directly whether `6Ch`/`6Eh` will be `INSB`/`OUTSB` or jumps, using no
instruction that can fault on the older part. A second probe (`PUSH SP`, valid
once we know we are on a 186-class part) separates the 286 and later.

**The driver then rewrites itself.** Rather than testing conditions on every
packet forever, INIT patches eleven sites in the hot paths — each assembled as
a pair of `NOP`s that becomes a `JMP` only if the fast path cannot be used. By
the time the first sector is read, the driver contains just the code your
machine needs.

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

For comparison, that read rate is around twenty times a 360 K floppy and in the
same territory as a period IDE hard disk on an XT-class machine. Copying a
10 MB file takes about 35 seconds.

On an original 8086 or 8088 the driver falls back to byte-at-a-time transfers,
because those CPUs lack the block instructions. It is still considerably faster
than stock, but slower than the figures above. Those configurations have not
been measured.

**The practical rule:** The slower the machine, the more this driver matters. 
On an 8 MHz V30 it is the difference between unusable and pleasant. 
On a 486 the stock driver is already tolerable.

---

## Why a faster machine barely helps

This is the part that surprises people, so it is worth showing the numbers.

Data arrives from the CH375 in 64-byte USB packets. On the 8 MHz V30 one packet
costs about 1,656 CPU cycles, and they divide like this:

| | cycles | scales with CPU speed? |
|---|---:|---|
| CH375 USB transaction | ~950 | **no** — the chip's own USB engine |
| `REP INSB`, 64 bytes | ~570 | **no** — paced by the ISA bus clock |
| everything the driver does | ~136 | yes |
| **total** | **1,656** | |

**Only 8% of that time is code.** The rest is hardware, and neither part gets
faster when you drop in a quicker processor.

The **CH375's USB transaction** is the chip talking to the flash drive. That
takes as long as it takes, whatever is driving it.

The **ISA bus** runs at roughly 8 MHz on essentially every machine, by design —
that is what keeps cards from the early 1980s working in later systems. An `IN`
instruction is paced by the bus, not the CPU, so a 486 does not issue port reads
any faster than a V30 does. It just waits longer between them.

Put a faster CPU under this driver and only the 136-cycle slice shrinks:

| CPU | projected read | gain |
|---|---:|---:|
| 8 MHz V30 (measured) | 302 KB/s | — |
| 386DX-33 | ~322 KB/s | +6.6% |
| 486DX-33 | ~325 KB/s | +7.7% |
| 486DX2-66 | ~327 KB/s | +8.3% |
| *a driver with literally zero overhead* | *329 KB/s* | *+8.9%* |

A 486DX2-66 lands within 2 KB/s of a theoretically perfect driver. **There is
about 8% of headroom left in software in total, on any CPU.** That is the wall,
and this build is already against it.

It also means the 386/486 instructions you would normally reach for buy nothing
here. `REP INSD` looks like the obvious win — 32 bits per transfer — but on an
8-bit ISA card the chipset splits each 32-bit read into four sequential byte
reads at the same port. Identical bus cycles, no gain. The same goes for 32-bit
registers and addressing modes: they would shave part of a slice that is already
only 8% of the total.

**The one lever that genuinely moves the number is a 16-bit-capable card.**
Two bytes per bus cycle would halve the 570 to ~285, taking a packet to roughly
1,400 cycles and read speed to around 355 KB/s — more than every software
optimisation in this project combined. Generic CH375 ISA cards are 8-bit, so
that needs different hardware, not different code.

---

## Using it day to day

The stick behaves like any other DOS drive. `DIR E:`, `COPY`, `XCOPY /S`,
running programs from it, using it as a target for `BACKUP` — all normal.

**Before unplugging**, make sure nothing has files open, and if you use a disk
cache run whatever flush it provides. DOS may hold unwritten data briefly.
Unplugging mid-write can corrupt the filesystem, exactly as it would on a hard
disk.

**Swapping sticks** while DOS is running is risky. DOS caches directory
information per drive, and although the driver supports media-change detection,
the safest habit is to swap only when nothing is open — or reboot after a swap.

---

## If something goes wrong

**Nothing prints at boot, or the machine hangs.**
Almost always the wrong `@` address. Check your card's jumpers or documentation
and try the correct value. A wrong address means the driver talks to whatever
else lives there, which can wedge the machine.

**`CH375: disk not found`**
No USB stick detected. Check it is inserted, try reseating it, and try another
stick — some draw more power than an ISA card can supply.

**`CH375: error boot sector`**
The stick was found but DOS cannot make sense of its filesystem. Almost always
FAT32 or exFAT. Reformat as FAT16 with a partition of 2 GB or less.

**`CH375: I/O data error`**
A transfer failed. Try `%1` on the `DEVICE=` line to add bus spacing:

```
device=c:\CH375EXT.sys @260 %1
```

If that fixes it, try `%2` or `%3` only if `%1` is not enough. Bear in mind each
step turns off the fast paths, so you trade speed for reliability.

**`CH375: interrupt timeout` or `CH375: wait serial`**
The card is not responding as expected. Check the address first, then power and
seating.

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

`no differences encountered` means reads and writes are both sound. If you see
byte offsets listed, add `%1` and repeat. Use a compressed file — a `.ZIP` or a
disk image — because any corrupted byte shows up rather than hiding in slack
space.

Running `CHKDSK E:` afterwards is a useful second check; new lost clusters after
a copy would point at a problem.

---

## A note on fast machines

This build was tuned and verified on an 8 MHz V30. The timing between port
accesses was tightened by measurement on that machine, which is safe at that
clock rate.

On a 386, 486 or faster machine the picture changes, and **not in your favour**.
As the section above shows there is almost nothing to gain — but there is
something to lose. The settling delay after each port write is a short jump,
worth about 1.5 µs on an 8 MHz V30 and a small fraction of that on a 486. The
gap the card was given shrinks as the CPU gets faster, and if the CH375 cannot
keep up, the failure is **silently wrong data rather than an error message**.

So on anything faster than roughly a 286: run the verification above before
trusting it, and use `%1` or `%2` if it fails. You would be trading a few
percent of speed for correctness — and on those machines the few percent was
never really there.

---

## Limitations

- FAT16 only, one partition, 2 GB maximum. No FAT32, no exFAT, no NTFS.
- Not bootable — DOS must already be running before the driver loads.
- Uses about 4.3 KB of conventional memory.
- One card and one USB stick at a time.
- USB flash drives and card readers only. USB CD drives will not work: they use
  2048-byte sectors and the MMC command set, and DOS treats optical media as a
  character device behind MSCDEX rather than as a block device. That would be a
  different driver, not an option on this one.

---

Hobbyist software, provided as-is with no warranty. The original driver remains
© W.ch 1998–2007; this is an optimized derivative.
