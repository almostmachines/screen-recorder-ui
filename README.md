# screen-recorder-ui

A small Bash wrapper that puts a couple of [zenity](https://gitlab.gnome.org/GNOME/zenity)
dialogs in front of [gpu-screen-recorder](https://git.dec05eba.com/gpu-screen-recorder/),
so starting and stopping a recording is a keybinding away instead of a
remembered command line.

Ask it to record, pick a source, tick the audio you want, and it hands off to
`gpu-screen-recorder` in the background. Recordings land in `~/Videos` with a
timestamped filename. Desktop notifications report start, stop and the saved path.

## Usage

```
screen-recorder-ui [start|stop|toggle]
```

- `start` — prompt for a capture source and audio, then begin recording
  (if a recording is already running, it offers to stop it instead)
- `stop` — send `SIGINT` to the running recorder so it finalises the file
- `toggle` — stop if recording, otherwise start

With no argument, `start` is assumed.

Binding `toggle` to a media key (or `start`/`stop` to a pair of keys) is the
intended way to use it. For example, in sway:

```
bindsym XF86Tools exec screen-recorder-ui toggle
```

## Capture sources

The source list adapts to the session:

**Wayland**

- **Screen / window / region** — via the XDG desktop portal picker
- **Window rectangle (slurp)** — offered only under sway, when `swaymsg`, `jq`
  and `slurp` are all present. Window geometry is read out of the sway tree and
  fed to `slurp`, so windows snap as selection targets rather than being traced
  freehand.

**X11**

- **Focused window**
- **Whole screen**

**Either**

- **Webcam** — devices are enumerated with `gpu-screen-recorder --list-v4l2-devices`;
  if there's more than one you get a picker.

Video is captured at 60 fps, except webcam capture, which uses 30.

## Audio

A checklist offers *System audio (what you hear)* — recorded from
`default_output` — and *Microphone*, from `default_input`. Both, either or
neither; leaving both unchecked gives a silent video.

## Requirements

- `bash`
- `gpu-screen-recorder`
- `zenity`
- `notify-send` (optional, for notifications)
- `swaymsg`, `jq`, `slurp` (optional; enable the sway window-rectangle source)

## How it works

`start` collects the answers, builds the `gpu-screen-recorder` command line,
then re-executes itself with the hidden `__run` subcommand under `nohup`. That
supervisor process launches the recorder, records its PID and output path in
`$XDG_RUNTIME_DIR/screen-recorder-ui/`, waits for it to exit, and then notifies
you about the result — cleaning up a zero-byte file if the recording failed or
was cancelled.

`stop` reads that PID file and verifies the process still exists *and* is
actually `gpu-screen-recorder` before signalling it, so a recycled PID can't be
killed by mistake. Stale state files are removed when the check fails.

## Configuration

Two things are settable from the environment or by editing the top of the script:

| Name | Default | Meaning |
| --- | --- | --- |
| `GSR_BIN` | `gpu-screen-recorder` | Recorder binary to invoke |
| `VIDEO_DIR` | `$HOME/Videos` | Where recordings are written |

State and logs live in `${XDG_RUNTIME_DIR:-/tmp}/screen-recorder-ui/`; the
recorder's own output is captured in `gpu-screen-recorder.log` there, which is
the first place to look if a recording doesn't appear.
