# No More Manual Retime

**English** · [한국어](README.ko.md) · [Introduction page](https://vfx-compositor.github.io/no-more-manual-retime/) · [Nukepedia](https://www.nukepedia.com/tools/python/misc/no-more-manual-retime-auto-retime-matching/)

No More Manual Retime matches an editorial retime to its plate inside Nuke. Give it the offline edit movie and the original camera plate. It finds which plate frame every edit frame uses, then builds a TimeWarp, the matching reframe, and a QC node set for you.

It does not need an EDL, XML, or matching timecode. It is built for the offline you actually receive: graded, compressed, reframed, and covered in burn-ins.

The result is one TimeWarp node. A change that speed cannot explain, such as a cut inside the shot, is flagged with a warning.

| | |
|---|---|
| Input | Offline edit movie and camera plate |
| Output | TimeWarp, reframe (Transform), QC node set, report notes |
| Runs in | Nuke 15.1 or later with a GUI, plus numpy |
| License | MIT |

**Processing is fully local.** Analysis and burn-in timecode recognition run on your computer, in numpy: no neural network, no model download, no upload, no API key. Your source media is only read, never renamed, moved, or overwritten.

## Download

All you need is the single file `no_more_manual_retime.py`.

- **Releases:** the latest `no_more_manual_retime_v1.1.0.zip` from the [Releases page](../../releases) (the tool file, README, license).
- **Single file:** `no_more_manual_retime.py` in this repository. Use the **Raw** view to copy it.

## What's new in 1.1.0

Retime results are the same as 1.0.2: a 288-run replay (144 conditions, image only and image + burn-in TC) gave every edit frame the same plate frame and the same TimeWarp curve. False cut warnings where there is no cut went down, and warnings on real cuts stayed the same (42 runs with known cuts). What changed is the tools around the result and some defaults.

**Changed defaults and output nodes (read before upgrading)**

- **Reframe output (Beta)** is on by default. A saved choice from an earlier version still wins.
- **Follow existing Write settings** is on by default. The QC review Write takes codec, quality and channels from the Writes already in your script. It takes a colorspace only when it is an output or display one (sRGB, Rec.709 and the like), never log or linear. Turn it off for the fixed 1.0.2 settings.
- **The result block is one straight line.** Each plate's nodes are chained in this order: TimeWarp, FrameRange, fit, optical, Difference, MP label, end Dot. The edit's nodes sit to the left of it. The `NMMR_timecode` node is no longer created. The plate TC (MP) label sits in the QC branch only and is on by default.
- **QC review output** goes to `retime_QC_reviews/` beside the script (was `qc_review/`). Each shot is one Group, and all Groups feed one AppendClip and one Write.
- **Language** moved from the window header to the top of **Settings** (gear button, Cmd+, / Ctrl+,).

**New**

- **TC only** burn-in mode: timing from a trusted burn-in TC without image matching. On synthetic burn-ins it accepted 83 of 144 conditions and was 99.4% exact there, in under half the analysis time of the default mode (539.8 s vs 1,194.3 s in total). TC only skips image matching so it is faster. In stretches where the image is hard to match such as a still frame or a black screen the plate frame still comes from the burn-in TC as long as it can be read. It refuses rather than guesses on long gaps, misreads or a missing plate TC.
- **QC render mode:** render now in Nuke, or create only the Write node for a render farm. The format stays as you chose it (MOV or JPEG).
- **Split composite (Beta):** a drawn region gets its matching plate automatically. Two clips in one frame are reported as a composite instead of cuts.
- **Difference placement** option (B pipe or side branch) and batch result placement.
- **Progress:** the bar keeps moving during long Nuke calls and shows `last progress n s ago` when work stalls.
- **MP label as one Group:** the plate TC (MP) label is one Group with knobs for position, size, opacity, weight and color. A custom position shows a Transform handle in the Viewer.
- **One window when a run ends:** a finished run always ends with one window. A single run shows the completion headline and the QC review result, and warnings appear in the same window below the headline. A batch shows the counts of succeeded, needs review, skipped and failed shots once at the end. A cancelled run shows no completion window.

**Fixed**

- The MP TC (plate timecode) shown in QC now matches the plate frame actually used, even in ramps, reverse and holds. It used to assume a constant speed, so those sections showed the wrong timecode.
- QC review Writes no longer stop with `Invalid LUT selected` in OCIO projects.
- Large scripts: analysis creates fewer temporary nodes.
- Running the QC review twice in the same minute no longer points at the earlier output. The second folder gets `_2`, then `_3` and so on, so nothing is overwritten.
- On first launch the language follows the OS display language (Korean when the OS is Korean, English for everything else) even when Nuke is started without a `LANG` setting. A language you chose in Settings still wins.
- The window now opens in Nuke 15.1. It failed to open there before.
- Opening another script while the window is open no longer leaves the previous script's batch rows and Done message behind.
- Creating the QC Write node only twice in the same minute no longer gives both Writes the same path. The second one gets `_2`.
- EDIT and PLATE selection and refresh are locked during a run. The source cannot change in the middle of an analysis.
- **Find edit again** and **Remove row** could be pressed while a batch was running, which put the table out of step with the run. The batch tools are now locked until the run ends.
- A QC review render that failed because of a permission or disk problem was shown only as cancelled. It is now reported as a failure with the reason.

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

The tool creates, next to the plate, a TimeWarp, a FrameRange, a Reformat and Transform that bring the plate to the edit format, a QC node set (edit against the retimed plate), and report notes. Everything from one run sits in one backdrop and is undone with a single Ctrl+Z (Cmd+Z on macOS).

Select several plate Read nodes and use **NMMR > No More Manual Retime (Batch Selected)** to solve them one after another.

### Reading the result

Each plate gets a **matching similarity** from 0 to 100%. About 85% and above is good: spot-check the QC node set. Around 70 to 85% is fair: scrub the QC difference around the frames listed in the report. Below that the report is marked *review required*. The score is separate from the **Possible cut** flag: a shot can match well and still contain one possible cut.

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
- The plate TC (MP) label is drawn only in the QC branch. The TimeWarp output carries no burn-in.
- Frames cut out inside a shot are not followed; they get a warning. Jumps of two or three frames get no warning, and on real footage 4- and 8-frame jumps went unflagged without readable timecode.
- Verified on macOS with Nuke 16.1. On macOS Nuke 15.1v8 the window opens, a single run completes and the QC render works (2026-11-10). Windows and Linux are not yet verified in a real Nuke session.

## Accuracy

Measured on 144 edit conditions from 16 shots not used in development (12,504 frames per mode, matching results identical to versions 1.0.1, 1.0.2 and 1.1.0). The numbers come from real Nuke thumbnail caches replayed on CPU, not GUI renders. The burn-in timecode was generated for the test, not read from real footage. Failures are included. Low-motion and dark shots are the weakest.

| Case | Conditions | Image only | Image + burn-in TC | Conditions at 98%+ (image / TC) |
|---|---:|---:|---:|---:|
| General retime (0.5×, 1×, 2×, ramp, hold, reverse) | 102 | 97.35% | 99.87% | 90/102 / 100/102 |
| Reframe | 34 | 92.59% | 99.80% | 25/34 / 33/34 |
| 4×/8× (reported separately) | 8 | 90.10% | 100.00% | 7/8 / 8/8 |

Accuracy is the share of frames where the TimeWarp outputs the exact plate frame. Within one frame: 99.01% / 99.91% (general), 95.34% / 100.00% (reframe), 90.62% / 100.00% (4×/8×).

## Optional: path setup

By default the tool uses the Read nodes in your script. If your shots follow a folder layout, the gear button in the window header opens **Settings** (language at the top, then path setup), where you describe the layout once and the tool finds the latest plate and edit for the shot you are working on. Settings are stored in `~/.nuke/no_more_manual_retime_paths.json`; a shared file can be given with `NMMR_PATHS`. The first-run language can be set with `NMMR_LANGUAGE=ko` or `en`.

## License

[MIT](LICENSE). Copyright (c) 2026 Jumin Park.
