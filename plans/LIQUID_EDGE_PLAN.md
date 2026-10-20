# The liquid edge: controls that come out of their place's edge

Status: in progress, fizzyedit/fizzy#583 (step 2).

A pane's controls should not sit on top of its content while nobody is using them. Three of them
already try not to, each in its own way, and none gets it right at a card's rounded edge:

- **The arrange button** in a place's top-right corner. It became a glass piece filling the
  corner (fizzyedit/fizzy#555), then a small bubble on the corner's curve (#578). Against the
  window's own corner it is clamped inward and stops following any curve, and in the explorer,
  which has no card, it floats over the chrome.
- **The history dome** at a document pane's bottom edge (fizzyedit/fizzy#551, #562), which grows
  into the undo bar. At the card's edge it is cut by a rectangle clip.
- **The tape dome** (fizzyedit/fizzy#572): the tape player hiding into the same dome, its liquid
  the tape's progress.

The root of all three: **nothing can clip to a place's real shape.** dvui clips to rectangles, so
a control at a rounded edge is cut square, pokes past the curve, or is pulled inside it.

## The direction (decided, 2026-10-18)

Each place has an implicit **liquid edge**: its outline, which glass inside it can run into.
Controls bound to the place merge with that edge, metaball-style, and the place's outline holds
them in. Nothing at the edge is drawn on its own: only the necks where controls come near it.

- **The outline.** `Layout.cardShape`: a place's card as its rect and four radii, rounded or
  square, the corners its fill is drawn with. Where a card meets the window's edge it takes the
  window's own corner radius there, so a control at a window corner follows that curve.
- **The edge.** The place's outside, its surface 1px past the outline (`EDGE_GAP`): the outline's
  rounded-box distance inverted, one more shape in the `LiquidField`, in the program's last shape
  slot. Starting past the outline, it never shows as a sliver inside on its own.
- **Controls merge with it.** A control bound to the place is a shape in the same field and joins
  the band through the field's smooth union. Near the edge it grows a neck, so it reads as pulled
  out of the place's rim rather than a bevelled button sitting on it. "Sliding under the edge"
  is a control pulling back into it.
- **The boundary holds.** The field is cut by the place's outline (a max with its rounded-box
  distance), so nothing crosses it. The rim, bend and shadow are drawn only on the glass's own
  inner outline, never along the cut, so where the glass meets the edge it reads as carrying on
  past it.
- **Invisible unless a control is near.** The edge is never drawn on its own. It shows only where
  a control overlaps the place's edge, and is computed only there: the field's bounds are the
  controls' rects plus the merge distance, clipped to the place. Nothing per place, per frame,
  when no control is near an edge.
- **No glass** (frost, transparent, off): clipped rounded fills, no neck. Whatever material
  fizzyedit/plans#27 settles a shape in, the outline cuts it the same way.
- **A place with no card** (the explorer): its outline is the window's.

## Steps (one PR each)

- [ ] **layout: a region's card shape** (`Layout.cardShape`), taking the window's corner radius
      where a card meets the window's edge. Unit tests on the shapes of the shipped layouts.
- [ ] **core: glass that merges with its place's edge, and stays inside it**
      (fizzyedit/fizzy#583). `LiquidField.edge` (the place's outline, in the program's last shape
      slot), the cut, the rim from the inner outline only, `glassBubbles(..., .{ .edge = ... })`,
      and `LiquidField.sample` mirroring it. Compiled forms through `shaders.yml`.
- [ ] **layout: the arrange corner on the edge.** A drop at the card's top-right, mostly outside
      the place and cut by it: at rest a faint swell, clear glass (a slight bend, no tint, frost or
      dark line; the view's content corner lies just inside the curve, so anything coloured there
      covers it). Brought up only from outside the view's content (the padding and the curve), by
      the pointer or a tap, so reaching for a control in the view's top right never does. It then
      grows where it is into the button, under the pointer, still joined to the edge, its lens,
      frost and tint coming in as it grows, and sinks back as the pointer leaves it. Without the
      glass program, a small dot in the curve. Open: whether the rest state also carries the grid
      glyph as its hint. Replaces #578's peeking bubble and its window-edge clamp; the explorer's
      corner is the window's outline.
- [ ] **app: the history dome on the edge.** It swells out of the bottom edge and grows into the
      undo bar as now (`app/dome.zig`, `app/dome_rules.zig`), its liquid drawn in it.
- [ ] **app: the tape dome on the edge**, filling as the tape plays.

## Coordination

- #578 (the corner bubble) can land as the stopgap until the corner step replaces it.
- The domes' shared drawing (`app/dome.zig`, `app/dome_rules.zig`, from #572) draws at whatever
  shape it is given, with no placement or input of its own, so moving them onto the edge is a
  change of who places them, not of how they draw.
- New controls that come out of a place's edge are shapes merged with it, never their own clip or
  clamp.
