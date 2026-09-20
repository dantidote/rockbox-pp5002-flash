# Rockbox on PP5002 iPods with flash storage

Rockbox's PP5002 (iPod 1G, 2G, 3G) disk driver assumes the original Toshiba
hard drive. With a CF card, an SD adapter, or the
[Sphinxmoth](https://github.com/dantidote/sphinxmoth) bridge it fails in three
ways described below. This repo holds the driver patch that fixes them,
ready-to-copy builds, and the workflow that produces them. Found and tested
on Sphinxmoth hardware; nothing in the patch is specific to it.

## Which file do I need?

**Use the latest pipeline build for either bridge image and either iPod
generation.** It is a current Rockbox development build with the driver
patch applied (timing keyed to the clock, paced writes, and the inter-block
settle that fixes the intermittent `error -4` on database builds):

| iPod | File |
|---|---|
| 1G / 2G | [`rockbox-ipod1g2g-pp5002-flash.ipod`](https://github.com/dantidote/rockbox-pp5002-flash/releases/download/rockbox-d23a19dc2d-pp5002-flash/rockbox-ipod1g2g-pp5002-flash.ipod) |
| 3G | [`rockbox-ipod3g-pp5002-flash.ipod`](https://github.com/dantidote/rockbox-pp5002-flash/releases/download/rockbox-d23a19dc2d-pp5002-flash/rockbox-ipod3g-pp5002-flash.ipod) (built the same way, not yet hardware-tested) |

Install a Rockbox **development build** (not 4.0) with Rockbox Utility, then
in disk mode copy the file over `.rockbox/rockbox.ipod`, renaming it to
`rockbox.ipod`. It must sit on a development install: codecs and plugins are
version-locked to the binary, and a 4.0 install under this file makes every
track skip. The full `.zip` on the
[release page](https://github.com/dantidote/rockbox-pp5002-flash/releases/tag/rockbox-d23a19dc2d-pp5002-flash)
is a complete install if you would rather unzip than swap a file.

Verified 2026-09-19 on an iPod 2G with the raw-pad PIO bridge image: three
consecutive database builds on a large library, where the unpatched dev
build failed one of two. Test on a fix3b board pending.

The older hand-patched 4.0 files in `releases/` are kept for reference; they
boot and play but can still fail long writes (problem 3 below).

## Why Rockbox needs help on this board

Two separate things:

1. **Rockbox 4.0 crashes at boot on every iPod 1G/2G**, flash mod or not
   (`Data abort at 0006e8ec`, an unaligned halfword store in the PP5002
   core-wake assembly). Rockbox fixed it upstream on 2025-08-04, after the
   4.0 branch point, so dev builds are fine and the next release will be.
   The 3G release happens not to be affected. Our 4.0 files carry the
   seven-byte fix.

2. **Rockbox's PP5002 ATA driver uses one fixed, fast IDE timing** (0x10)
   tuned for the original Toshiba drive. The fix3b bridge image passes PIO
   data through a clock synchroniser, which adds about 45 ns to every
   register read; at 80 MHz that is more than the PP5002 tolerates, and
   init fails with `ATA error: -11` or `-32`. Slowing the timing exposes a
   second problem: Rockbox's PIO write loop never waits for the controller,
   so with a slower bus the controller drops words and every write fails
   with `-4`.

3. **Long writes fail intermittently with `-4` on every bridge image**,
   typically during a database build or a large theme install. Between the
   data blocks of a multi-sector write, Rockbox checks BSY-clear then DRQ
   once. A hard drive raises BSY within nanoseconds of the last word; a
   flash card can leave the previous block's DRQ standing for microseconds,
   so the check passes on stale state, the next block is pushed into a card
   that is not listening, words are lost and the transfer ends with DRQ
   stuck.

`pp5002-flash.patch` fixes the second and third problems at the
source, the way the retail firmware and Rockbox's own PP502x port already
do for the timing: select the IDE timing from the CPU clock, wait for the
controller's idle bit after each data word, and wait (bounded,
`ATA_DRQ_SETTLE_POLLS`) for the card to go BSY or drop DRQ before trusting
the next-block check. `upstream-commit-message.txt` is the message for
sending it to Rockbox; once merged, this repo becomes unnecessary for new
releases.

The hand-patched 4.0 binaries in `releases/` predate the source patch and
pin the CPU at 80 MHz instead of keying the timing to the clock; they are
what has actually been tested on hardware.

## Automated builds

`.github/workflows/build.yml` builds Rockbox for `ipod1g2g` and `ipod3g`
with the patch applied. It also carries Rockbox's own core-wake fix
(`upstream-core-wake-c33602375.patch`, the upstream commit verbatim) and
applies it when the ref being built predates that commit, so 4.0-based
builds do not inherit the boot crash; on newer refs it is detected as
already present and skipped. The workflow runs:

- **Weekly** (Monday 06:17 UTC): finds the newest upstream release tag
  (`vX.Y` or `vX.Y-final`) and, if there is no `rockbox-<tag>-pp5002-flash`
  release here yet, builds and publishes one.
- **On demand** (Actions, "Run workflow"): any Rockbox ref; `publish`
  controls whether a release is created. Branch builds are tagged by the
  upstream commit (`rockbox-<commit>-pp5002-flash`), so every tested build
  keeps its own URL.

The arm-elf-eabi toolchain (`tools/rockboxdev.sh --target=a`, gcc 9.5) is
built once and cached; the first run takes 20 to 30 minutes.

Each release carries `rockbox-ipod1g2g-pp5002-flash.ipod`,
`rockbox-ipod3g-pp5002-flash.ipod`, a full `.zip` install for each, and
`SHA256SUMS.txt`.

**A build that compiles is not a build that works.** The workflow tests
nothing on hardware. Before a workflow build replaces a tested file here or
on the website, boot it on a fix3b board, do a write (plug in FireWire and
let Rockbox flush to disk mode), and play a track. If the patch stops
applying because upstream changed the driver, the workflow fails at the
"Apply the flash-storage patch" step; check whether upstream now carries the
fix before rebasing.

## License

The patch is offered under Rockbox's license (GPL v2 or later). The
binaries in `releases/` are Rockbox 4.0 release builds with the changes
described in their READMEs; Rockbox source is at
https://git.rockbox.org/.
