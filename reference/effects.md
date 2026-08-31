# tachyonfx Reference: Effects and Interpolation

Assumes familiarity with the mental model and `Effect`/`EffectTimer` — see `../SKILL.md`. This file is the complete constructor catalog and interpolation reference. For the pattern/wave/DSL systems, see `patterns-and-dsl.md`.

Every `timer` param accepts anything `Into<EffectTimer>`:

| Type | Meaning |
|---|---|
| `u32` | milliseconds, `Linear` interpolation |
| `(u32, Interpolation)` | milliseconds with easing |
| `Duration` | custom duration (u32 ms default), `Linear` |
| `(Duration, Interpolation)` | custom duration with easing |

## Use-Case Cheat Sheet

| I want to... | Use |
|---|---|
| Fade foreground to a color | `fade_to_fg` |
| Desaturate / mute a region | `saturate_fg(-x)` / `saturate(Some(-x), ...)` |
| Brighten / dim | `lighten_fg` / `darken_fg` |
| Cycle hue (shimmer/rainbow) | `hsl_shift_fg([hue_delta, ...], t)` |
| Wipe/reveal content with an edge | `sweep_in` / `sweep_out` |
| Slide with solid block gradient | `slide_in` / `slide_out` |
| Expand from center | `expand` |
| Stretch/fill with fractional block | `stretch` |
| Random organic fade-out | `dissolve` |
| Organic reveal from blanks | `coalesce` |
| Transform text through symbols | `evolve` / `evolve_into` / `evolve_from` |
| Explosion scatter | `explode` |
| Spatial progression shaping | pair with a pattern — see `patterns-and-dsl.md` |
| Add a beat / pause | `sleep` / `delay` |
| Loop indefinitely | `repeating` |
| Oscillate forward/back | `ping_pong` |

## Interpolation Table

All implement `Interpolation`. Use `(duration_ms, Interpolation::Variant)` tuple syntax.

| Category | Variants |
|---|---|
| Linear | `Linear` (default), `SmoothStep`, `Reverse` |
| Sine | `SineIn`, `SineOut`, `SineInOut` |
| Quad | `QuadIn`, `QuadOut`, `QuadInOut` |
| Cubic | `CubicIn`, `CubicOut`, `CubicInOut` |
| Quart | `QuartIn`, `QuartOut`, `QuartInOut` |
| Quint | `QuintIn`, `QuintOut`, `QuintInOut` |
| Expo | `ExpoIn`, `ExpoOut`, `ExpoInOut` |
| Circ | `CircIn`, `CircOut`, `CircInOut` |
| Back | `BackIn`, `BackOut`, `BackInOut` |
| Bounce | `BounceIn`, `BounceOut`, `BounceInOut` |
| Elastic | `ElasticIn`, `ElasticOut`, `ElasticInOut` |
| Special | `Spring` |

## Color Effects

```rust
// Fade foreground to a target color. Supports spatial patterns.
pub fn fade_to_fg<T, C>(fg: C, timer: T) -> Effect
//     T: Into<EffectTimer>, C: Into<Color>
fx::fade_to_fg(Color::Red, (1000, Interpolation::CircOut));
fx::fade_to_fg(Color::from_u32(0x504945), 500);

// Fade foreground FROM a color toward the original.
pub fn fade_from_fg<T, C>(fg: C, timer: T) -> Effect
fx::fade_from_fg(Color::DarkGray, (800, Interpolation::QuadInOut));

// Fade both fg and bg to targets.
pub fn fade_to<T, C>(fg: C, bg: C, timer: T) -> Effect
let c = Color::from_u32(0x1d2021);
fx::fade_to(c, c, (1000, Interpolation::CircOut));

// Fade both FROM toward originals.
pub fn fade_from<T, C>(fg: C, bg: C, timer: T) -> Effect
let c = Color::from_u32(0x1d2021);
fx::fade_from(c, c, (1000, Interpolation::CircOut));

// Instantly set colors (no interpolation). Timer = persist duration.
pub fn paint<T, C>(fg: C, bg: C, timer: T) -> Effect
pub fn paint_fg<T, C>(fg: C, timer: T) -> Effect
pub fn paint_bg<T, C>(bg: C, timer: T) -> Effect
fx::paint(Color::Cyan, Color::DarkGray, 1000);
fx::paint_fg(Color::Red, 100);
fx::paint_bg(Color::Blue, 100);

// Shift hue/saturation/lightness. Array = [hue_delta, sat_delta, light_delta].
// Panics if BOTH fg and bg are None. Supports spatial patterns.
pub fn hsl_shift<T>(hsl_fg_change: Option<[f32; 3]>, hsl_bg_change: Option<[f32; 3]>, timer: T) -> Effect
pub fn hsl_shift_fg<T>(hsl_fg_change: [f32; 3], timer: T) -> Effect
fx::hsl_shift_fg([120.0, 25.0, 25.0], 1000);
fx::hsl_shift(Some([120.0, 25.0, 25.0]), Some([-20.0, -50.0, 15.0]), 1000);

// Adjust saturation. Negative = desaturate, positive = boost. Panics if both None. Supports patterns.
pub fn saturate<T>(fg: Option<f32>, bg: Option<f32>, timer: T) -> Effect
pub fn saturate_fg<T>(fg: f32, timer: T) -> Effect
fx::saturate_fg(-0.5, 1000);
fx::saturate(Some(-0.5), None, 1000).with_pattern(SweepPattern::left_to_right(25));

// Adjust lightness 0.0-1.0. 1.0 lighten = white, 1.0 darken = black. Panics if both None. Supports patterns.
pub fn lighten<T>(fg: Option<f32>, bg: Option<f32>, timer: T) -> Effect
pub fn lighten_fg<T>(fg: f32, timer: T) -> Effect
pub fn darken<T>(fg: Option<f32>, bg: Option<f32>, timer: T) -> Effect
pub fn darken_fg<T>(fg: f32, timer: T) -> Effect
fx::lighten(Some(0.5), Some(0.3), 1000);
fx::darken_fg(0.7, (800, Interpolation::SineInOut));
fx::lighten_fg(0.5, 1000);
```

## Text / Character Effects

```rust
// Randomly replace chars with spaces (organic). dissolve_to also transitions bg.
// Supports .with_rng(seed) for deterministic output and spatial patterns.
pub fn dissolve<T>(timer: T) -> Effect
pub fn dissolve_to<T>(style: Style, timer: T) -> Effect
fx::dissolve(1000);
fx::dissolve((1000, Interpolation::QuadOut));
fx::dissolve(1000).with_rng(SimpleRng::new(42));
fx::dissolve_to(Style::default(), 1000);

// Reverse of dissolve — restores text from spaces.
pub fn coalesce<T>(timer: T) -> Effect
pub fn coalesce_from<T>(style: Style, timer: T) -> Effect
fx::coalesce((1500, Interpolation::QuintIn));
fx::coalesce_from(Style::default().bg(Color::DarkGray), 1000);

// Transform chars through symbol progressions.
// EvolveSymbolSet: Circles, BlocksHorizontal, BlocksVertical, CircleFill, Quadrants, Shaded, Squares.
// Can pass (EvolveSymbolSet, Style) for styled evolution.
pub fn evolve<T>(symbols: impl Into<EvolveSymbolConfig>, timer: T) -> Effect
pub fn evolve_into<T>(symbols: impl Into<EvolveSymbolConfig>, timer: T) -> Effect   // stops at alpha=1.0, reveals original
pub fn evolve_from<T>(symbols: impl Into<EvolveSymbolConfig>, timer: T) -> Effect  // starts original, evolves into symbols
fx::evolve(EvolveSymbolSet::CircleFill, 500);
fx::evolve((EvolveSymbolSet::Quadrants, style), 500).with_pattern(RadialPattern::center());
fx::evolve_into(EvolveSymbolSet::Circles, (1000, Interpolation::SineOut));
fx::evolve_from(EvolveSymbolSet::Quadrants, 1500).with_pattern(DissolvePattern::new());

// Explosion — cells fly outward from center, replaced with black.
// force = distance, force_rng_factor = chaos (0 = uniform). Supports spatial patterns.
pub fn explode(force: f32, force_rng_factor: f32, timer: impl Into<EffectTimer>) -> Effect
fx::parallel(&[
    fx::fade_to_fg(Color::from_u32(0x404040), timer),
    fx::explode(15.0, 2.0, timer),
]);
```

## Motion / Spatial Effects

```rust
// Sweep content in/out with colored gradient edge.
// gradient_length = transition cells, randomness = max per-row/col offset (0 = uniform).
pub fn sweep_in<T, C>(direction: Motion, gradient_length: u16, randomness: u16, faded_color: C, timer: T) -> Effect
pub fn sweep_out<T, C>(direction: Motion, gradient_length: u16, randomness: u16, faded_color: C, timer: T) -> Effect
//     T: Into<EffectTimer>, C: Into<Color>
fx::sweep_in(Motion::LeftToRight, 10, 0, Color::Black, (1200, Interpolation::QuadOut));
fx::sweep_out(Motion::UpToDown, 15, 5, Color::Cyan, 2000);

// Slide content with block-character gradient (██▇▆▅▄▃▂▁). Color revealed behind cells.
pub fn slide_in<T, C>(direction: Motion, gradient_length: u16, randomness: u16, color_behind_cells: C, timer: T) -> Effect
pub fn slide_out<T, C>(direction: Motion, gradient_length: u16, randomness: u16, color_behind_cells: C, timer: T) -> Effect
fx::slide_in(Motion::UpToDown, 5, 0, Color::Cyan, 1000);
fx::slide_out(Motion::LeftToRight, 10, 0, Color::DarkGray, 1500);

// Unidirectional stretch using partial blocks. Fills area with style, leading edge = fractional block.
pub fn stretch<T>(direction: Motion, style: Style, timer: T) -> Effect
fx::stretch(Motion::UpToDown, Style::default().bg(Color::Black), (1000, Interpolation::BounceOut));

// Bidirectional from center. ExpandDirection::Horizontal | Vertical.
pub fn expand<T>(direction: ExpandDirection, style: Style, timer: T) -> Effect
fx::expand(ExpandDirection::Horizontal, Style::default().bg(Color::Blue), 1000);
fx::expand(ExpandDirection::Vertical, Style::default().bg(Color::Red), (800, Interpolation::CubicOut));

// Move an effect's render area by offset over time.
// MUST be applied before rendering widgets — changes the draw area. Check effect.area() for current rect.
pub fn translate<T>(fx: Effect, translate_by: Offset, timer: T) -> Effect
let inner = fx::evolve_from((EvolveSymbolSet::Quadrants, style), timer).with_pattern(DissolvePattern::new());
fx::translate(inner, Offset { x: 0, y: -8 }, timer).with_area(content_area);

// Translate a pre-rendered auxiliary buffer onto main buffer. For complex, rarely-changing content.
pub fn translate_buf<T>(translate_by: Offset, aux_buffer: RefCount<Buffer>, timer: T) -> Effect
```

## Timing / Control Effects

```rust
// No-op pause. Spacer in sequences.
pub fn sleep<T>(duration: T) -> Effect
fx::sleep(1000);
fx::sleep((500, Interpolation::Linear));

// Sugar for sequence(&[sleep(duration), effect]).
pub fn delay<T>(duration: T, effect: Effect) -> Effect
fx::delay(800, fx::dissolve(200));

// Freeze inner effect at a specific alpha. set_raw_alpha bypasses interpolation.
pub fn freeze_at(alpha: f32, set_raw_alpha: bool, effect: Effect) -> Effect
fx::freeze_at(0.5, false, fx::dissolve(1000));

// Remap alpha to a sub-range. Useful for partial effects or seamless sequential transitions.
pub fn remap_alpha(alpha_start: f32, alpha_end: f32, effect: Effect) -> Effect
let fade = fx::fade_to_fg(Color::Cyan, 3000).with_color_space(ColorSpace::Rgb);
fx::remap_alpha(0.1, 0.5, fade); // only the 10%-50% portion

// RepeatMode: Forever, Times(u32), Duration(Duration). repeating = repeat(e, Forever).
pub fn repeat(effect: Effect, mode: RepeatMode) -> Effect
pub fn repeating(effect: Effect) -> Effect
fx::repeat(fx::fade_to_fg(Color::Red, 1000), RepeatMode::Times(3));
fx::repeating(fx::fade_to_fg(Color::Red, 1000));

// Play forward then reverse. Total = 2x original.
pub fn ping_pong(effect: Effect) -> Effect
fx::ping_pong(fx::coalesce((1500, Interpolation::QuintIn)));

// Add dead time before/after. prolong_start: inner gets zero duration (stays initial).
// prolong_end: inner holds final state.
pub fn prolong_start<T>(duration: T, effect: Effect) -> Effect
pub fn prolong_end<T>(duration: T, effect: Effect) -> Effect
fx::prolong_start(1000, fx::fade_from_fg(Color::Gray, 500));
fx::prolong_end(500, fx::fade_to_fg(Color::DarkGray, 500));

// never_complete: runs indefinitely, never reports done (inner holds end state).
// timed_never_complete: never_complete with a time cap.
// with_duration: completes when EITHER inner or duration finishes.
pub fn never_complete(effect: Effect) -> Effect
pub fn timed_never_complete(duration: Duration, effect: Effect) -> Effect
pub fn with_duration(duration: Duration, effect: Effect) -> Effect
fx::never_complete(fx::fade_to_fg(Color::Red, 1000));
fx::timed_never_complete(Duration::from_millis(1000), fx::dissolve(2000));
fx::with_duration(Duration::from_millis(1000), fx::dissolve(2000));

// Ensure effect runs exactly once before completing. Critical for zero-duration effects in sequences.
pub fn run_once(effect: Effect) -> Effect
fx::sequence(&[
    fx::fade_to_fg(Color::Red, 1000),
    fx::run_once(fx::dissolve(500)),
    fx::fade_to_fg(Color::Blue, 1000),
]);

// Completes after one processing tick. Synchronization in sequences.
pub fn consume_tick() -> Effect
```

## Composition Effects

```rust
// Run one after another. Time spills forward: leftover time immediately starts next effect.
// Reports completion after the last finishes.
pub fn sequence(effects: &[Effect]) -> Effect
fx::sequence(&[
    fx::sweep_in(Motion::LeftToRight, 10, 0, Color::DarkGray, 2000),
    fx::coalesce((800, Interpolation::SineOut)),
]);

// Run all simultaneously. Reports done when ALL finish.
// When reversed (ping_pong), shorter effects are right-aligned (delayed) so all finish together.
pub fn parallel(effects: &[Effect]) -> Effect
fx::parallel(&[
    fx::coalesce(timer).with_pattern(RadialPattern::center()),
    fx::hsl_shift_fg([240.0, 30.0, 15.0], timer).reversed(),
]);
```

## Custom / Builder Effects

```rust
// Custom effect via per-cell iterator. state passed as &mut each frame.
pub fn effect_fn<F, S, T>(state: S, timer: T, f: F) -> Effect
where
    S: Clone + ThreadSafetyMarker + 'static,
    T: Into<EffectTimer>,
    F: FnMut(&mut S, ShaderFnContext, CellIterator) + ThreadSafetyMarker + 'static,
fx::effect_fn((), 1000, |_state, ctx, cell_iter| {
    let alpha = ctx.alpha();
    for (_pos, cell) in cell_iter {
        cell.set_fg(cell.fg().lerp(&Color::Red, alpha));
    }
});

// Custom effect with full buffer access.
pub fn effect_fn_buf<F, S, T>(state: S, timer: T, f: F) -> Effect
where
    S: Clone + ThreadSafetyMarker + 'static,
    T: Into<EffectTimer>,
    F: FnMut(&mut S, ShaderFnContext, &mut Buffer) + ThreadSafetyMarker + 'static,
fx::effect_fn_buf((), 1000, |_state, ctx, buf| {
    let offset = ctx.timer.remaining().as_millis() as usize;
    for (i, pos) in buf.area.positions().enumerate() {
        let cell = &mut buf[pos];
        cell.set_fg(Color::Indexed(((offset + i) % 256) as u8));
    }
});
```

`ShaderFnContext` fields:
- `ctx.alpha() -> f32` — current interpolated progress (0.0–1.0)
- `ctx.last_tick: Duration` — elapsed time since last frame
- `ctx.timer: &EffectTimer` — full timer access
- `ctx.area: Rect` — the area being processed
- `ctx.filter()` — current cell filter

```rust
// Glitch via builder (bon Builder derive).
// use tachyonfx::fx::Glitch;
Glitch::builder()
    .cell_glitch_ratio(0.1)        // 10% of cells glitched
    .action_start_delay_ms(0..500)  // delay before each glitch
    .action_ms(100..300)            // glitch duration range
    .rng(SimpleRng::default())      // optional deterministic RNG
    .selection(CellFilter::Text)    // optional cell filter
    .build()
    .into_effect()
```

## Other Effects

```rust
// Responsive layout wrapper (area follows a shared rect).
pub fn dynamic_area(area: RefRect, effect: Effect) -> Effect

// Render to offscreen buffer.
pub fn offscreen_buffer(fx: Effect, target: RefCount<Buffer>) -> Effect

// Send event on start (std feature only).
pub fn dispatch_event<T>(sender: std::sync::mpsc::Sender<T>, event: T) -> Effect
where T: Clone + core::fmt::Debug + ThreadSafetyMarker + 'static,

// Deprecated: term256_colors() (since 0.16.0), resize_area (since 0.19.0).
```
