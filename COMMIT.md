# fix(screen-corners): keep corner overlays above the bar after a bar config change

## Summary

Rounded screen corners silently disappeared behind the bar as soon as any bar
setting changed. For a bottom bar this meant the two bottom corners went square
while the top two stayed rounded, and they stayed that way until screen corners
were toggled off and on again.

The corner overlays and the bar both live on the wlr-layer-shell **`Top`** layer.
Same-layer surfaces stack by **creation order**, and the protocol gives no way to
restack after the fact — so whichever surface is created last ends up on top.

`ScreenCorners::onConfigReload()` only rebuilt its surfaces when `enabled`, `size`,
or the eligible output count changed. A bar-only change left the corner surfaces
untouched while the bar recreated its own, putting the bar above the corners and
clipping the overlay away wherever the bar sits.

The bar recreates its surfaces for more than just radii — see
`barConfigSurfaceFieldsEqual` in `src/shell/bar/bar.cpp:688-713`, which returns true
only when two configs would produce an identical layer-shell surface. Any of
`thickness`, `layer`, all four corner radii, `concave_edge_corners`, `margin_ends`,
`margin_edge`, `shadow`, or the monitor overrides differing makes it false, and the
bar is recreated. Corner radii are in that list because they feed the concave-corner
bulge, which changes the surface size:

```cpp
// Corner radii feed the concave-corner bulge, which changes the surface size.
&& a.radiusTopLeft == b.radiusTopLeft
&& a.radiusTopRight == b.radiusTopRight
```

## Reproducing

Requires a bar that actually overlaps a screen corner — i.e. `margin_edge = 0` and
`margin_ends = 0`, so it runs flush into the corners. With `margin_ends > 0` the bar
stops short of the corners and the bug is invisible even though the stacking is
still wrong.

**1. Enable screen corners** in `~/.local/state/noctalia/settings.toml`:

```toml
[shell.screen_corners]
enabled = true
size = 32
```

**2. Make the bar reach the corners** (Settings → Bar → Layout):

```toml
[bar.default]
position = "bottom"
margin_edge = 0        # flush against the bottom edge
margin_ends = 0        # full width, so it covers both bottom corners
```

**3. Reload and confirm the corners are rounded** — all four should be masked:

```
noctalia msg config-reload
grim -g "0,1404 36x36" bl.png     # bottom-left; corner pixel should be black
```

**4. Change any bar setting that forces a surface recreate.** The cheapest is a
corner radius:

```toml
radius_top_left = 20
radius_top_right = 20
```

**5. Reload again** — the two bottom corners are now square. The top two are
unaffected, because no surface covers them. Every subsequent bar change keeps them
hidden; only toggling `screen_corners.enabled` off and on brings them back, since
that is the one path that recreated the corner surfaces.

Probing the bottom-left corner pixel across those steps, before the fix:

```
screen_corners just toggled   px=(0,0,0)     -> MASK VISIBLE
after bar radius change       px=(52,30,17)  -> mask hidden (bar on top)
after another bar change      px=(52,30,17)  -> mask hidden
```

## The fix

`src/shell/screen_corners/screen_corners.cpp` — also rebuild the corner surfaces
when the bar, widgets, or dock config changed, so the corners are created last and
land back on top:

```cpp
const auto& changed = m_config->lastChange();
const bool stackingMayHaveChanged = changed.bars || changed.widgets || changed.dock;

if (cfg.enabled != m_lastEnabled || cfg.size != m_lastSize
    || m_instances.size() != eligibleOutputCount(*m_wayland) || stackingMayHaveChanged) {
  destroySurfaces();
  ...
  ensureSurfaces();
}
```

This is ordering-dependent and only correct because the corner rebuild runs after
the bar's: `Bar::initialize()` registers its reload callback at
`src/app/application_ui.cpp:775`, screen corners register theirs at `:841`, and
`ConfigService` dispatches callbacks in registration order
(`src/config/config_service.cpp:634-636`). Moving either registration earlier or
later than the other reintroduces the bug.

`widgets` and `dock` are included alongside `bars` because a widget change also
routes through `Bar::reload()` (`src/shell/bar/bar.cpp:1236-1248`) and the dock is
another configurable `Top`-layer surface with the same recreate behaviour.

## Verification

Built and hot-swapped, then re-ran the reproduction: the mask survives repeated bar
changes, and all four corners read black at the extreme pixel.

```
baseline              px=(0, 0, 0)  -> MASK VISIBLE
bar radius -> 21      px=(0, 0, 0)  -> MASK VISIBLE
bar radius -> 20      px=(0, 0, 0)  -> MASK VISIBLE
bar radius -> 24      px=(0, 0, 0)  -> MASK VISIBLE
```

Tested on niri, single output, scale 1.

## Known gaps

- **Not fixed: on-demand `Overlay` panels.** The launcher, control center, session
  and wallpaper panels are created on `LayerShellLayer::Overlay` when opened, so
  they are created after the corners and cover them while open. Same root cause,
  different trigger.
- **Rejected alternative: moving the corners to `Overlay`.** It would put them above
  the bar unconditionally, but it is not a guarantee either — the panels above are
  already on `Overlay` and would still be created later. It also trades a narrow bug
  for a broad layering change. The fix here is narrower and addresses the reported
  case.
- A full guarantee would need the corner surfaces recreated after *any* other
  surface, which layer-shell cannot express; a compositor-side or protocol-level
  restack would be the real answer.
