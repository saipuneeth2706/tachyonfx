---
name: tachyonfx
description: Use when adding animated transitions/effects (fades, wipes, glitches, shimmer, color shifts) to a ratatui TUI in Rust
---

# tachyonfx Skill

tachyonfx adds animated effects to a ratatui TUI. This file teaches the mental model and the minimal wiring. For specifics, open the references only when you need them:

- **Which effect constructor / interpolation to use** → `reference/effects.md`
- **Spatial patterns, wave/signal API, effect DSL** → `reference/patterns-and-dsl.md`

## Dependencies

```toml
tachyonfx = "0.25"                                  # default: no_std, custom Duration (u32 ms)
tachyonfx = { version = "0.25", features = ["std"] }              # std::time::Duration, mpsc events
tachyonfx = { version = "0.25", features = ["dsl"] }             # effect DSL compiler (std required)
tachyonfx = { version = "0.25", features = ["std-duration"] }    # standard Duration instead of u32 ms
```

`dsl` requires `std`.

## Mental Model

tachyonfx is a post-render effect layer for ratatui. Effects are **stateful objects** created once and applied every frame. They transform already-rendered buffer cells (colors, characters, positions) via the `Shader` trait's `process(duration, buf, area) -> Option<Duration>` method. The returned `Option<Duration>` is the sequencing engine: `Some(overflow)` means the effect finished with leftover time, which containers forward to the next effect.

**Core types:**
- `Effect` — wraps `Box<dyn Shader>`. The only type you pass around. Is stateful — holds its internal timer/progress.
- `EffectTimer` — combines duration + an `Interpolation` easing curve. Construct via `EffectTimer::from_ms(ms, interp)`, or convert from `u32` (ms, linear), `(u32, Interpolation)`, `Duration`, or `(Duration, Interpolation)`.
- `Duration` — **custom `u32` milliseconds by default** (not `std::time::Duration`). Use the `std-duration` feature to switch.
- `EffectManager<K>` — optional lifecycle manager that ticks effects and garbage-collects completed ones.

**Rendering flow:** render widgets first, then apply effects. Each frame:
1. Create effects once (at startup or state change)
2. Render widgets into the buffer
3. Apply effects via `render_effect()` or `process_effects()`, passing the elapsed time since the last frame
4. The effect advances its internal timer and modifies buffer cells based on current alpha/progress

## Minimal End-to-End Example

```rust
use std::time::Instant;
use ratatui::{crossterm::event::{self, Event}, prelude::*, widgets::Block};
use tachyonfx::{
    CenteredShrink, Duration, Effect, EffectRenderer, EffectTimer, Interpolation,
    IntoEffect, Motion, fx,
};

fn main() -> io::Result<()> {
    let mut terminal = ratatui::init();
    let mut last_tick = Instant::now();

    // Create effects ONCE — they are stateful
    let mut effect: Effect = fx::ping_pong(fx::sweep_in(
        Motion::LeftToRight,
        10,    // gradient length in cells
        0,     // randomness (0 = uniform)
        Color::DarkGray,
        EffectTimer::from_ms(2000, Interpolation::QuadIn),
    ));

    loop {
        terminal.draw(|f| {
            // 1. Render widgets first
            Block::default()
                .style(Style::default().bg(Color::DarkGray))
                .render(f.area(), f.buffer_mut());
            let area = f.area().inner_centered(25, 2);
            Text::from(vec![
                Line::from("Hello, tachyonfx!"),
                Line::from("Press any key to exit."),
            ])
            .white()
            .centered()
            .render(area, f.buffer_mut());

            // 2. Apply effect AFTER widgets
            if effect.running() {
                f.render_effect(&mut effect, area, Duration::from_millis(33));
            }
        })?;

        if event::poll(std::time::Duration::from_millis(100))? {
            if let Event::Key(_) = event::read()? { break; }
        }
    }
    ratatui::restore();
    Ok(())
}
```

Key points:
- `EffectRenderer` (`.render_effect()`) is implemented for `Frame` and `Buffer`
- Pass the elapsed time since the last frame as `last_tick`
- `effect.running()` checks if still active (for conditional rendering)
- `effect.process(duration, buf, area) -> Option<Duration>` is the raw `Shader` driver

## Composition API Basics

Every combinator returns `Effect`, so effects nest arbitrarily. All constructors live in `tachyonfx::fx`.

```rust
// Sequential — time spills forward: leftover time immediately starts the next effect.
fx::sequence(&[fx::sweep_in(/* ... */), fx::coalesce(800)]);

// Simultaneous — done when the LAST child finishes.
fx::parallel(&[fx::fade_to_fg(Color::Red, 500), fx::dissolve(500)]);

// Delay = sugar for sequence(&[sleep(duration), effect]).
fx::delay(800, fx::dissolve(200));

// Repeat — RepeatMode::Forever, Times(u32), or Duration(Duration).
fx::repeat(fx::fade_to_fg(Color::Red, 1000), RepeatMode::Times(3));
fx::repeating(effect); // = repeat(effect, Forever)

// Play forward then reverse (total = 2x duration).
fx::ping_pong(fx::dissolve(1000));
```

Effect builder methods (fluent, chain on any `Effect`):
```rust
fx::dissolve(1000)
    .with_area(Rect::new(0, 0, 40, 10))   // restrict to area
    .with_filter(CellFilter::Text)         // only specific cells
    .with_color_space(ColorSpace::Hsl)     // color interpolation (Rgb/Hsl/Hsv)
    .with_rng(SimpleRng::new(42))          // deterministic
    .with_pattern(RadialPattern::center()) // spatial progression
    .reversed();                           // flip direction
```

**Duration math:** `sequence` = sum of children; `parallel` = max child; `ping_pong` = 2x inner; `repeat(_, Times(n))` = inner * n; `repeat(_, Duration(d))` = d.

## Common Mistakes

1. **Recreating effects each frame.** Effects are stateful; a fresh effect is always at alpha=0.0. Create once, then `render_effect(&mut effect, ...)` every frame.

2. **Using `std::time::Duration` with default features.** tachyonfx `Duration` is `u32` milliseconds. Use `tachyonfx::Duration::from_millis()` or enable `std-duration`.

3. **Applying effects before rendering widgets.** Effects modify existing buffer content — render widgets first.

4. **Passing a plain `u32` timer and expecting easing.** `1000` defaults to `Linear`. Use `(1000, Interpolation::BounceOut)` for easing.

5. **Confusing `translate` ordering.** `translate` changes the draw area for widgets, so it must be applied before rendering, not after. Check `effect.area()` for the current rect.

6. **`with_filter` only sets if no filter exists.** Composition containers set filters internally; `with_filter` on a composed effect may be a no-op.

7. **Calling `process_effects` inside `terminal.draw()`.** `EffectManager::process_effects` needs `&mut Buffer`, conflicting with `Frame`'s borrow. Call it outside `draw()`.

8. **Checking `done()` on child effects of a composable.** For `sequence`/`parallel`, only the top-level container's `done()` matters; children are consumed internally.

9. **Zero-duration effects in sequences get skipped.** Wrap with `fx::run_once()` to force execution.

10. **`parallel` right-aligns on reversal.** `ping_pong(parallel(...))` delays shorter effects so all finish together — intentional, but non-obvious.

11. **Forgetting color params accept `C: Into<Color>`.** `Color::from_u32(0xff0000)`, `Color::Red`, `Color::Indexed(42)` all work.

12. **`hsl_shift` (and `saturate`/`lighten`/`darken`) panic if both fg and bg args are `None`.** At least one must be `Some`.

## For More Detail

- **Full effect constructor catalog and interpolation table** → `reference/effects.md` (all 40+ `fx::` constructors with verified signatures and usage, `ShaderFnContext` fields, `Glitch` builder).
- **Pattern/wave/DSL system** → `reference/patterns-and-dsl.md` (spatial patterns like Radial/Diamond/Sweep/Wave/Combined, the wave/signal API — Oscillator/Modulator/WaveLayer — and the effect DSL compiler).
