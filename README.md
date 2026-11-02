# No More Manual Retime

**English** · [한국어](README.ko.md) · [Introduction page](https://vfx-compositor.github.io/no-more-manual-retime/) · [Nukepedia](https://www.nukepedia.com/tools/python/misc/no-more-manual-retime-auto-retime-matching/)

No More Manual Retime matches an editorial retime to its plate inside Nuke. Give it the offline edit movie and the original camera plate. It finds which plate frame every edit frame uses, then builds a TimeWarp, the matching reframe, and a QC difference tree for you.

It does not need an EDL, XML, or matching timecode. It is built for the offline you actually receive: graded, compressed, reframed, and covered in burn-ins.

The result is one TimeWarp node. A change that speed cannot explain, such as a cut inside the shot, is flagged with a warning.

| | |
|---|---|
| Input | Offline edit movie and camera plate |
| Output | TimeWarp, reframe (Transform), QC difference tree, report notes |
| Runs in | Nuke 15.1 or later with a GUI, plus numpy |
| License | MIT |

**Processing is fully local.** Analysis and burn-in timecode recognition run on your computer, in numpy: no neural network, no model download, no upload, no API key. Your source media is only read, never renamed, moved, or overwritten.

## Download

All you need is the single file `no_more_manual_retime.py`.

- **Releases:** the latest `no_more_manual_retime_v1.0.2.zip` from the [Releases page](../../releases) (the tool file, README, license).
- **Single file:** `no_more_manual_retime.py` in this repository. Use the **Raw** view to copy it.

## Install

You can run the tool without installing anything, or register it as a menu command.

**Try it once (Script Editor).** In Nuke, open a Script Editor panel, paste the entire contents of `no_more_manual_retime.py` into the lower Python input area, and press Cmd+Enter (macOS) or Ctrl+Enter (Windows, Linux).

**Menu command**

1. Copy `no_more_manual_retime.py` into a `python` folder inside your personal `.nuke` folder (create it if needed).
2. Append this to `~/.nuke/menu.py` (create the file if it does not exist; do not overwrite an existing one):

```python
import nuke

nuke.pluginAddPath("./python")
import no_more_manual_retime

nuke.menu('Nuke').addCommand('NMMR/No More Manual Retime', 'no_more_manual_retime.run()', 'ctrl+alt+r')
nuke.menu('Nuke').addCommand('NMMR/No More Manual Retime (Batch Selected)', 'no_more_manual_retime.run_batch_selected()')
```

3. Restart Nuke. **NMMR > No More Manual Retime** appears in the menu bar.

`ctrl+alt+r` is Cmd+Option+R on macOS. Pick another shortcut if it is already taken.

### numpy

Nuke does not ship with numpy. If it is missing, the first run opens an install window with three options, and Nuke does not need to restart afterwards:

- **Offline:** put the numpy wheel for your platform in a `vendor/` folder next to the script (or browse to it in the window).
- **Online:** press **Install online (pip)**; it runs Nuke's own Python with `pip install --target` into `~/.nuke_pylibs/`, outside your Nuke configuration.
- **Shared folder:** point every workstation at one numpy folder with the `NMMR_NUMPY_PATH` environment variable.

Nuke before 16 uses Python 3.10 and Nuke 16 or later uses Python 3.11, so download the wheel that matches. For example:

```bash
pip download numpy --only-binary=:all: --no-deps -d vendor --python-version 3.11 --platform macosx_14_0_arm64
```

## Use

1. Bring the edit movie and the plate into your script as Read nodes.
2. Select both (optional) and run **NMMR > No More Manual Retime**.
3. Check that **EDIT** is the editorial reference and **PLATE** is the source footage. **First Frame** defaults to **Plate start**.
4. Press **Run**.

The tool creates, next to the plate, a TimeWarp, a FrameRange, a Reformat and Transform that bring the plate to the edit format, a QC difference tree (edit against the retimed plate), and report notes. Everything from one run sits in one backdrop and is undone with a single Ctrl+Z (Cmd+Z on macOS).

Select several plate Read nodes and use **NMMR > No More Manual Retime (Batch Selected)** to solve them one after another.

### Reading the result

Each plate gets a **match score** from 0 to 100%. About 85% and above is good: spot-check the QC tree. Around 70 to 85% is fair: scrub the QC difference around the frames listed in the report. Below that the report is marked *review required*. The score is separate from the **cut suspicion** flag: a shot can match well and still contain one suspected cut.

### Fixing a hard spot

Load the timeline to see the edit as a filmstrip. You can **exclude** a range (it is filled from timecode or constant speed) or **pin** a frame to force EDIT = PLATE at that point. Both can also be typed, for example exclusions `1005-1012, 1030` and pins `1020=1088, 1040=1102`.

If your offline carries a source-timecode burn-in, **Calibrate TC…** reads it without external OCR and uses it as a hint. Timecode is only a hint: the picture still decides.

## What it handles

- Constant speed, offsets, ramps, holds, and reverse playback, written as whole plate frames (up to 16 plate frames per edit frame by default, adjustable from 4 to 16; few cases are measured at 8× and above).
- Graded or LUT-baked offlines: matching uses rank-normalized luminance.
- Reframes: scale, reposition, and flop, including zooms that change over the shot.
- Burn-ins, logos, and watermarks, trimmed and masked automatically.
- Slow motion made by interpolation (frame blend, optical flow): each edit frame gets a fractional plate position; the TimeWarp carries that lookup.
- Near-still stretches, where the picture cannot say which frame is which: the tool warns and bridges them with a constant-speed line.
- Batches of shots, and split-screen composites (Beta, limited validation).
- An optional QC review render that joins every result into one movie or JPEG sequence.

## Limitations

- Rotation is not detected. Exclude rotating sections or pin them.
- Scale outside 0.8 to 1.6× is not searched.
- Retimes are whole plate frames, except in interpolated slow stretches, where the TimeWarp carries a fractional lookup. The default output renders the lower of the two plate frames.
- Slow motion made by interpolation (such as Kronos) is still weak, especially when combined with a zoom.
- The QC render ignores tone differences but not everything: burn-ins that exist only in the edit leave faint bright edges even on a correct retime. Judge by large, structured flashes.
- In OCIO or ACES projects, set the colorspace on both edit and plate Reads. Left at default, the tool can stop or image-only accuracy can drop. Set it on untagged ProRes edits too.
- The timecode Text overlay burns into the plate if left on at render time. Disable that node for clean renders.
- Frames cut out inside a shot are not followed; they get a warning. Jumps of two or three frames get no warning, and on real footage 4- and 8-frame jumps went unflagged without readable timecode.
- Verified on macOS with Nuke 16.1. Nuke 15.1, Windows and Linux are not yet verified in a real Nuke session.

## Accuracy

Measured on 144 edit conditions from 16 shots not used in development (12,504 frames per mode, matching code identical to version 1.0.1). The numbers come from real Nuke thumbnail caches replayed on CPU, not GUI renders. The burn-in timecode was generated for the test, not read from real footage. Failures are included. Low-motion and dark shots are the weakest.

| Case | Conditions | Image only | Image + burn-in TC | Conditions at 98%+ (image / TC) |
|---|---:|---:|---:|---:|
| General retime (0.5×, 1×, 2×, ramp, hold, reverse) | 102 | 97.35% | 99.87% | 90/102 / 100/102 |
| Reframe | 34 | 92.59% | 99.80% | 25/34 / 33/34 |
| 4×/8× (reported separately) | 8 | 90.10% | 100.00% | 7/8 / 8/8 |

Accuracy is the share of frames where the TimeWarp outputs the exact plate frame. Within one frame: 99.01% / 99.91% (general), 95.34% / 100.00% (reframe), 90.62% / 100.00% (4×/8×).

## Optional: path setup

By default the tool uses the Read nodes in your script. If your shots follow a folder layout, the gear button in the window header opens **Path setup**, where you describe the layout once and the tool finds the latest plate and edit for the shot you are working on. Settings are stored in `~/.nuke/no_more_manual_retime_paths.json`; a shared file can be given with `NMMR_PATHS`. The first-run language can be set with `NMMR_LANGUAGE=ko` or `en`.

## License

[MIT](LICENSE). Copyright (c) 2026 Jumin Park.
