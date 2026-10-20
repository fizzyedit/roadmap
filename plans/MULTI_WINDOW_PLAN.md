# Multi-window: OS windows as a first-class part of fizzy and of dvui

Status: in progress. Steps 1–3 and 3a (tear-off) are in fizzy `main` (#492, #501, #504, #510; #548, #577, #579, #591, #594, #605), and so is the window material for a plain dvui app, in-app windows included (#546, #549, #565, #586, #589, #590). Next: fizzy's own windows in the material (step 5 and roadmap#6), then Windows and Linux (step 6).

Fizzy's live resize is smooth on every window it draws, and the user wants that to carry an app
of many windows (2026-10-17). This plan is the order that work goes in, for fizzy, for its backend
as a dvui backend, and for a plain dvui app on it. It builds on, and does not repeat:
- the dvui upstreaming plan (kept private): the backend in dvui (`-Dbackend=fizzy`), the hooks, the proposal.
- `POPOUT_WINDOWS_PLAN.md`: why fizzy has one `dvui.Window` across OS windows (bands, viewports).
- `NATIVE_WINDOWS_PLAN.md`: fizzy's backend owning its windows; glass.
- `WORKSPACES_PLAN.md`: many workspaces as views, ⌘N for a new window.
- `UNIFIED_DRAG_PLAN.md`: every drag a view drag.

## Decisions (with the user, 2026-10-17)

**An OS window is an ordinary document window on macOS.** It goes full screen alone, moves alone,
and the person orders it. macOS has no window that both stays above another and has a full screen
of its own:

| | Stays above the main window | Own full screen |
|---|---|---|
| Child window (`addChildWindow`) | yes, but moves with its parent and goes into full screen with it | no |
| Above the normal level | yes, but taken into the app's full screen Space as a palette, on the way and on a swipe to it | no |
| Normal window | no: comes forward when clicked | yes |

fizzyedit/fizzy#483 used child windows; #492's first versions tried a level, the level dropped
around full screen, and reordering as the main window came forward. Each held for one moment and
the next test found another. #492 now makes them ordinary windows and deletes all of it
(+2/−265). This is how macOS apps' torn-out windows behave (VS Code, Xcode).

**A palette is a kind an app asks for**, not the default: an `NSPanel` above the app's windows
while it is active, following its full screen as an auxiliary window, with no full screen of its
own (the green button zooms). For tool windows.

**Windows and Linux keep owned and transient windows**: above their owner, which is the standard
there, and needs nothing like the above.

**dvui's demo stays in-app** (David, via the user, 2026-10-17): its floating windows are floating
windows; only its "I'm an OS window" demo makes OS windows. So a floating window becomes an OS
window only when it asks, and "every floating window an OS window" is a backend setting an app
chooses (`run-dvui`, fizzy).

**Drags between the app's windows stay the app's own**, routed by where the pointer is
(`heldAcrossWindows`), never OS drag and drop: the OS's drag image is a still bitmap, its loops
(Windows' `DoDragDrop`) block the frame loop, and nothing of it exists on the web. OS drag and drop
comes in only at the app's edges.

**Backend first** (the user, 2026-10-18): dvui has to keep working on the backends that aren't
fizzy's, so a feature goes in fizzy's backend wherever it can, and a dvui hook only where nothing
else will do. The tear-off reads its drag from what dvui already shows, with no hook. The window
material is fizzy's (`core.material`, roadmap#6), not dvui's. The hooks dvui keeps are
the ones a backend can't do without (`floatingWindowOs`, `floatingWindowTitle`, the `os_window`
ask; `drawRetained`, `screensSet`; and `floatingWindowBackground`, below).

**dvui changes are fine where they're needed** (the user, 2026-10-18), as long as each is explained,
contained and documented with its reason. The one this work added: `Backend.floatingWindowBackground`
(dvui-dev `3d90fdd`, tagged `fizzy-sdk-0.2.30-8`, fizzy's `docs/DEPENDENCIES.md` item 14), asked where
a floating window would fill its background, so the backend can draw it in the window's material.

## Steps

Each is its own PR. Fizzy's are into `zig/0.17` until it merges to `main`.

1. **Document windows.** Done, fizzyedit/fizzy#492: an OS-framed float is an ordinary window, with
   its own full screen.
2. **A float's full screen as smooth as the main window's.** Done, fizzyedit/fizzy#504. Measured
   at 56 fps against 120: the frame loop slept to 60 Hz whenever the main window was on another
   Space (`unseen`). Also fixed: a shadow redone every frame, a background that filled the screen,
   square corners inside rounded ones (now eased), and one-second stalls waiting on the main
   window's drawable on a Space nobody saw (`mainHeld`). Open: a ring, the backdrop's shadow,
   sometimes outside the picture on the way out of full screen. Traced once, but it didn't
   reproduce: 1,878 frames, the backdrop and the picture agreed on every one.
3. **OS windows only when asked.** Done, fizzyedit/fizzy#501: `FloatingWindowWidget.os_window` (set
   by `dvui.osWindow`), and the backend's `floating_os_windows` (`.asked`, `.all`, `.none`).
   dvui's demo stays in-app, as David wants. Also fizzyedit/fizzy#510: the pointer reaches the
   window it is over, for hover (AppKit sends mouse-moved to the key window) and for a drag in a
   full screen Space (windows on another Space skipped).
   Still to do from the earlier list: drops from outside (a generic dvui event, from SDL's drop
   events); an OS-window demo of dragging between windows. Tearing off (3a) became the
   demonstration of one `dvui.Window` across OS windows.
3a. **Tearing off a floating window.** Done, fizzyedit/fizzy#548, reworked in #577, with #579, #591,
   #594 and #605 from the user's testing. Backend only (`os_windows`), no dvui hook: the drag is read
   from the floating window holding the mouse capture, its rect moving at a constant size. As built:
   - **Past the app's windows it's an OS window at once** (#577): the OS's title bar and buttons
     under the pointer, the content a live miniature (laid out at its size, shown scaled, at most
     360 pt wide). The first design was a borderless card that grew on release and then gave way to
     a window opened for it; the swap jumped. While the drag holds it the window is **held**
     (`viewports.held`): the pointer passes through it, and it floats above the app's windows
     without becoming key, which would take the drag.
   - **Over a window's title bar it stays out** (#605): only a window's content counts
     (`windowContentAtPointer`), or dvui held its top below the title bar, off the pointer.
   - **Dropped on the desktop**, the same window grows down from its title bar to the floating
     window's size, onto the usable part of the screen under the pointer (`usableAtPointer`: it
     was moved on screen by the OS after growing past an edge), and is the floating window's window
     from then (`torn`). Its content is the floating window's less its header (#591: given the
     whole rect, it grew a header per trip out and back).
   - **Dropped on another of the app's windows**, it goes into that window's band (`heldPoint`),
     the window in front by the OS's own stacking where windows overlap (#579). The window under
     the drag comes forward (not key).
   - **Back in** (#605): let go by the OS's move with the pointer over the main window's content,
     it's the floating window there at once, dvui's header where the title bar was, and the OS
     window fades out over it in 150 ms, the pointer through it. Frames keep coming while the OS
     moves a window, so the let-go is seen at once. It used to wait for all of the window to be
     inside, and the next event, and then shrink back through a card that flashed a title.
   - **Only a floating window that can close tears off** (`open_flag`, noted for every floating
     window as dvui begins it: `closable`, #591). dvui's Debug window and anything else it draws
     late tear off too: the retained layer draws before the tear-off looks (#594).
   - Mechanisms worth knowing: the floating window is **parked** far left in the frame on a screen
     of its own while only its picture shows (dvui clipped it to the main window wherever its
     middle was); an OS window takes every subwindow overlapping its band, not only by the middle
     (a large float in a small window vanished).
   - macOS only (`viewports.tears`). The titled window removes one blocker for **Windows and Linux**
     (no carry window); they still need the question of the window under the pointer
     (`windowAtPointer`, `windowContentAtPointer`).
4. **The palette kind**: the backend's hook takes the kind (document or palette); macOS makes the
   palette an `NSPanel`-style window as above. Verify that a full screen auxiliary panel does not
   break the Space as the child windows did before relying on it.
5. **Fizzy on the same model.** Its floats and torn-out views as ordinary windows where they are
   not glass, and workspaces (`WORKSPACES_PLAN.md`, `FIZZY_WORKSPACES`) building on them: ⌘N, a
   workspace per window, dialogs in the asking window.
6. **Windows and Linux, checked**: the same cases in the UTM VMs, owned windows on Windows and
   transient on X11/Wayland.
7. **Dragging out** (later): when a drag leaves every window of the app, hand it to an OS drag
   session (`NSDraggingSession`, OLE, XDND/Wayland data device), starting with files from the
   tree on macOS and Windows.

## The window material

For a plain dvui app on fizzy's backend, done:
- `core.material` (`none`, `transparent`, `blur`, `frost`, `liquid`, and an opacity), the kinds each
  backend can show, and the effective one published each frame (fizzyedit/fizzy#546); every glass
  in fizzy drawn in it (#549).
- The backend applies it to a plain app's main window and its OS windows (#565). Liquid Glass there
  is fizzy's own window glass (the lens, the frost body, the clearing bevel), not a bare
  `NSGlassEffectView`. That needed `glass_look` in the backend (#565), then all of fizzy's in-app
  glass there (#586: `backend/src/glass/`, `core` re-exporting every name, the profiler through a
  hook).
- In-app floating windows in the material (#589): the frame is drawn through `FrameTarget` (#590,
  about 0.3 ms a frame, within noise), and dvui asks the backend for the background
  (`floatingWindowBackground`, above). A light theme's colour covers 0.3/0.7 as much, the ratio of
  fizzy's own defaults; glass never goes below `glass_look.min_opacity`, only a clear window does.
- A tape's captions in it too (#625).
- Measured 2026-10-19 (Debug, continuous redraw): every material held 120 fps, frost and glass
  about 0.3–0.6 ms of main-thread CPU a frame more than opaque. Earlier drops to 80 or 57 fps came
  with Spotlight indexing and other GPU load: the glass's blur passes bring a frame's GPU work near
  a refresh, and contention tips it.

**Next:** fizzy's own windows (it dresses them itself, `app_dresses_windows`) in the same material,
one path for both (roadmap#6); Windows (Acrylic) and Linux (compositor blur) for
`platform.window.plainMaterial`, which is macOS only.

## Open questions

- Fizzy's own floats: glass windows the app frames, or the OS's frame like `run-dvui`'s? Step 5
  needs the user's call.
- The ring on the way out of full screen (step 2): it needs a reproduction under the trace.
- A torn-off window dropped back sometimes landed to the right (the user, 2026-10-18). Traced, the
  rect it was given was right every time; the path it went through has since been replaced (#605).
  Watch for it.
