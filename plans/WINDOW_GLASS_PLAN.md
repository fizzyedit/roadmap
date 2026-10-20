# Window glass for any dvui app: one opacity, the OS's frost per window, the backend's per float

Status: proposed. Part of fizzyedit/fizzy#281 (fizzy's backend serving any dvui app).

Fizzy's windows are glass: the desktop shows through them, frosted, and its floating windows, menus
and dialogs frost what lies behind them. Today that comes from fizzy's own code: two settings, a
dozen calls at startup, and widget forks that draw the frost. A plain dvui app on fizzy's backend
gets none of it. This plan makes it a standard the backend applies, so a dvui app asks for glass
with dvui's own `transparent` flag and gets the same look, and fizzy becomes one user of it.

## The standard, in three layers

1. **Each OS window wears the OS's material.** The main window and every window popped out of it
   (`dvui.osWindow`, fizzy's floats) alike: Liquid Glass or vibrancy on macOS, Acrylic on Windows,
   the compositor's blur on Linux. The OS blurs the desktop behind; the app cannot, and need not.
2. **A floating thing inside a window is frosted by the backend.** A dvui floating window, menu,
   dialog or tooltip drawn over a window's content gets that content blurred behind it, in that
   window. Once it is out in an OS window of its own, layer 1 takes over: it is an OS window.
3. **Behaviour is automatic.** Smooth live resize, smooth full screen and zoom, the window shown
   only with its first frame, geometry kept: no look to choose, so no app code.

Layers 1 and 2 are driven by one value, layer 3 by none.

## One value: opacity, and what 0 means

Two settings today, opacity and roughness. Outside macOS 26 the window follows only opacity: the
material is fixed (Acrylic, vibrancy, compositor blur) and opacity is the window colour's alpha over
it. Roughness reaches only the glass fizzy draws itself; on macOS 26 it also runs Liquid Glass from
a clear lens to frost.

There is no need for the glass to go clear. From here on:

| | behind the app's drawing |
|---|---|
| `transparent = false` | the window behaves the same; its backing is black, as any dvui app's |
| `transparent = true`, opacity 0 | fully transparent, no material: an overlay, a drag's glass layer, a clear full-screen window |
| `transparent = true`, opacity above 0, up to 1 | always frosted, at one tuned blur, the window colour ramping smoothly over it from mostly transparent to opaque |

- **The frost is fixed per platform, native where the OS has one:** one tuned frost on macOS (Liquid
  Glass's frost on 26, vibrancy before), Acrylic on Windows, the compositor's blur on Linux.
- **Where nothing can blur** (Linux without a compositor blur, Windows 10), above 0 is opaque:
  transparent always comes with blur, or not at all.
- **The ramp is the same on every OS:** only the material under it differs. It runs from a little
  colour over the frost (still reading as glass) to opaque, as a smooth curve tuned once.
- **0 is a mode, not the ramp's end:** the jump from 0 to just above it, from clear to frosted, is
  deliberate. An app's own slider (fizzy's) stops just above 0; a frost-free main window does not
  read. The API takes 0 exactly.
- **Roughness goes**, with the clear-lens end of the macOS ramp: `glass_look.window` becomes a
  function of opacity alone, plus each platform's frost constant. The settings migration keeps each
  person's opacity, per theme (light and dark).

`windowLook(opacity, caps) -> Look` is a pure function, `caps` saying what the OS offers (Liquid
Glass, vibrancy, Acrylic, compositor blur, none), tested without a screen: 0 is clear, above 0 has
the frost, the colour's alpha rises with opacity, 1 is opaque. Floating things inside a window
(layer 2) take the same frost and the same ramp (`floatLook(opacity)`), so a float and its window
agree.

## Layer 2: frost behind a floating thing, from the backend

dvui keeps each floating thing's drawing as its own subwindow's commands until the end of the frame
(`Window.drawRetained`, then the replay). The backend already takes subwindows out of that replay
for popped-out windows (`os_windows`, Popout). For the main window it takes over the replay too:

- after `drawRetained`, it replays the subwindows in dvui's stacking order into the window's frame
  target, itself;
- before each floating subwindow it frosts the region beneath it — what is drawn so far — and
  draws that, then the subwindow on top (the blur fizzy's `BlurBackdrop` already does, dual-Kawase
  down a persistent pyramid);
- it empties what it drew, so dvui's own replay draws nothing twice.

Which subwindows: every floating one in a window with glass on (`transparent`), and only where it
can show — a theme whose window colour is opaque shows no frost, so none is drawn. A frost is
recaptured only when what lies beneath it changed (the subwindows below and their rects hashed, as
`BlurBackdrop` witnesses a capture), and drawn from cache otherwise: one textured quad. A budget
caps how many are recaptured in one frame (menus and dialogs first), the rest drawing last frame's.

Everything this needs from dvui is public: subwindows' commands, `Window.renderCommands`, render
targets, and `drawRetained` (proposed upstream). Nothing for dvui to learn.

The blur itself moves out of fizzy's `core` into the backend package (or a small `glass` package the
backend and fizzy both use): `BlurBackdrop` reaches into `core` for `motion`, `liquid_glass`,
`LiquidField` and `FrameTarget`, which stay fizzy's; the capture, pyramid and blur are dvui-only.

Fizzy's own floats draw their frost through its `FloatingWindowWidget` fork today, with looks the
backend's frost does not have (Liquid Glass lens on macOS 26, the field merging drops). They can
stay so; whether fizzy later lets the backend frost its plain floats is its own choice.

## Layer 3: behaviour, automatic

What fizzy's `Entry` switches on today and a plain `dvui.App` never does (`restoreWindowState`,
`setWindowStyle`, `macosLaunchComplete`, `showWindow`) moves into the backend's own window setup:

- the macOS monitor: live resize drawn step by step, full screen, zoom and Space transitions;
- Windows' live-resize steps;
- the window shown with its first frame, not before;
- geometry kept, through dvui's own `persist_window_geometry` and `pref_path`; fizzy keeps its
  `layout.zon` store as an option of that.

Fizzy keeps what is its own: the native menu bar's model, file dialogs' folders, gestures, open-file
events (all in `platform`, for an app that wants them).

## What a dvui app writes

```zig
pub const dvui_app: dvui.App = .{ .config = .{ .options = .{
    .size = .{ .w = 960, .h = 640 },
    .title = "My app",
    .transparent = true, // glass: the OS's material, frost behind floating windows
} }, .frameFn = frame };
```

and a theme whose window colour is not opaque, so the glass shows; an app that draws an opaque
background over the whole window sees none. Opacity has a default per platform and theme, above 0;
`backend.setWindowOpacity(win, value)` changes it, live (fizzy's slider), 0 for a clear window.
Without `transparent`, nothing changes: a black backing, as dvui has it.

## Upstream

Nothing beyond what is proposed already: `transparent` is dvui's, and `drawRetained` lets a backend
take the replay. A float in dvui's options would mean nothing on dvui's own backends, which cannot
blur, so opacity stays a backend option until a second backend wants it. Content under a
transparent title bar (fizzy draws there, keeping floats and menus below the traffic lights) would
need dvui to know a safe area at the top of the window: a possible later proposal, opt-in until then.

## Steps

Each a PR, in this order.

1. `windowLook` and `floatLook`, per platform, with their tests; fizzy's main window and floats read
   their material and base colour from them. One "Window opacity" setting, light and dark; the
   roughness setting migrated away.
2. Tune, on screen: macOS's one frost (26 and before), and the colour ramp's curve.
3. Layer 3: the window setup moves into the backend; `run-dvui` resizes and goes full screen as
   fizzy does, with no code of its own.
4. Layer 1 from the backend: `transparent` honoured with the platform's material and the base colour
   cleared at the window's opacity; `setWindowOpacity`. Fizzy calls it instead of drawing its base.
5. The blur moved out of `core` into the backend (or a `glass` package).
6. Layer 2: the backend's replay with frost behind floating subwindows, cached and budgeted.
7. `run-dvui` with `transparent = true` and a see-through theme: the dvui demo frosted, its floating
   windows frosting what is behind them, popped-out windows wearing the OS's material.
8. Windows: a dragged view shown outside its window. The carry window is macOS's only today
   (`viewports.carries`); Windows gets one, a plain Acrylic window following the pointer (no
   merging glass bubbles there: DWM cannot merge them).

## Open

- Windows at full opacity: Acrylic off (cheaper, the colour exact) or kept under a solid fill?
- The frost budget per frame, and its cost on the web backend, measured before step 6 lands
  (`blur-cost-model`: a frost's cost is its render-target switches).
- dvui apps on the web: fizzy's web backend has the blur, not the OS material; layer 1 there is the
  page behind the canvas, if anything.
