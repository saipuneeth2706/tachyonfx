# tachyonfx Reference: Patterns, Wave/Signal API, and DSL

Assumes familiarity with `Effect`, `EffectTimer`, and the `fx::` constructor API — see `../SKILL.md` and `effects.md`.

## Spatial Pattern System

Patterns transform the global animation alpha (0.0–1.0) into per-cell alpha values, creating spatial progressions. Apply via `.with_pattern(pattern)` on any effect.

### Core Traits

```rust
// The trait all patterns implement
pub trait Pattern {
    type Context;
    fn for_frame(self, alpha: f32, area: Rect) -> PreparedPattern<Self::Context, Self>;
}

// Per-cell alpha computation
pub trait InstancedPattern {
    fn map_alpha(&mut self, pos: Position) -> f32; // -> 0.0..1.0
}
```

`PreparedPattern<S, P>` is the per-frame state. You never construct it directly — it's produced by `Pattern::for_frame()` internally.

`TransitionProgress` handles gradient smoothing. Its `map_threshold(global_alpha, cell_threshold) -> f32` maps discrete thresholds into smooth transitions with a configurable width.

### Pattern Catalog

All patterns live in `tachyonfx::pattern`. All implement `Into<AnyPattern>` so they can be passed to `.with_pattern()`.

| Pattern | Constructor | Description |
|---|---|---|
| `RadialPattern` | `::center()`, `::new(cx, cy)`, `::with_transition((cx,cy), w)` | Euclidean-distance circle from center |
| `DiamondPattern` | `::center()`, `::new(cx, cy)`, `::with_transition((cx,cy), w)` | Manhattan-distance diamond (no sqrt) |
| `DiagonalPattern` | `::top_left_to_bottom_right()`, `::top_right_to_bottom_left()`, `::bottom_left_to_top_right()`, `::bottom_right_to_top_left()`, `::new(dir, w)` | Corner-to-corner sweep |
| `SweepPattern` | `::left_to_right(w)`, `::right_to_left(w)`, `::up_to_down(w)`, `::down_to_up(w)`, `::new(dir, w)` | Directional linear sweep with gradient |
| `CheckerboardPattern` | `::new(cell_size, transition_w)`, `::with_cell_size(n)` | Alternating cell grid |
| `DissolvePattern` | `::new()` | Randomized organic dissolve (per-cell random threshold) |
| `CoalescePattern` | `::new()` | Reverse of dissolve — randomized reveal |
| `InvertedPattern` | `::new(pattern)` | Inverts: `alpha -> 1.0 - alpha` |
| `BlendPattern` | `::new(pattern_a, pattern_b)` | Crossfade between two patterns over time |
| `CombinedPattern` | `::multiply(a, b)`, `::max(a, b)`, `::min(a, b)`, `::average(a, b)`, `::new(op, a, b)` | Binary op on two patterns |
| `WavePattern` | `::new(WaveLayer)`, `.with_layer(WaveLayer)` | Wave-interference spatial field |
| `SpiralPattern` | `::center()`, `::new(cx, cy)`, `::with_transition((cx,cy), w)` | Spiral-arm reveal (no trig) |

Builder methods (on all patterns except `DissolvePattern`/`CoalescePattern`):
- `.with_transition_width(f32)` — gradient softness in terminal cells (min 0.1)
- `.with_center((f32, f32))` — normalized 0.0–1.0 coordinates (Radial, Diamond, Spiral only)

`SpiralPattern` also has `.with_arms(u16)`.

### Usage Examples

```rust
use tachyonfx::{fx, pattern::*};

// Radial dissolve from center
fx::dissolve(1000).with_pattern(RadialPattern::center());

// Sweep reveal from left with wide gradient
fx::dissolve(1000).with_pattern(SweepPattern::left_to_right(25));

// Diamond-shaped evolve
fx::evolve(EvolveSymbolSet::CircleFill, 500)
    .with_pattern(DiamondPattern::center().with_transition_width(5.0));

// Combine patterns: only active where both radial AND diagonal overlap
fx::dissolve(1000).with_pattern(
    CombinedPattern::multiply(
        RadialPattern::center(),
        DiagonalPattern::top_left_to_bottom_right(),
    )
);

// Crossfade from dissolve pattern to radial over the effect's lifetime
fx::dissolve(1000).with_pattern(
    BlendPattern::new(DissolvePattern::new(), RadialPattern::center())
);

// Invert a sweep so it reveals from the trailing edge
fx::dissolve(1000).with_pattern(
    InvertedPattern::new(SweepPattern::left_to_right(20))
);
```

### AnyPattern

`AnyPattern` is an enum wrapping all concrete pattern types. It exists so shaders can store patterns without knowing the concrete type. All pattern types implement `Into<AnyPattern>` via `From` impls. You don't need to construct `AnyPattern` directly — just pass any pattern to `.with_pattern()`.

```rust
pub enum AnyPattern {
    Radial(RadialPattern),
    Diamond(DiamondPattern),
    Diagonal(DiagonalPattern),
    Sweep(SweepPattern),
    Checkerboard(CheckerboardPattern),
    Dissolve(DissolvePattern),
    Coalesce(CoalescePattern),
    Inverted(InvertedPattern),
    Blend(BlendPattern),
    Combined(CombinedPattern),
    Wave(WavePattern),
    Spiral(SpiralPattern),
}
```

### Checkerboard

Alternating black/white cell grid. Useful for reveal overlays.

```rust
// 4x4 cells with soft transitions
fx::dissolve(1000).with_pattern(CheckerboardPattern::new(4, 2));

// 8x8 cells, default transition width
fx::dissolve(1000).with_pattern(CheckerboardPattern::with_cell_size(8));
```

### Spiral

Generates a spiral-arm reveal from center. No trig functions — uses integer approximations for performance. Arms wrap counterclockwise; center is offset by (0.2, 0.2) to start slightly off-center.

```rust
// Default spiral from center, 4 arms
fx::dissolve(2000).with_pattern(SpiralPattern::center());

// Custom origin and arm count
fx::dissolve(2000).with_pattern(
    SpiralPattern::new(0.3, 0.4)
        .with_arms(6)
        .with_transition_width(0.2)
);
```

### Combined Pattern Operations

`CombinedPattern` applies a binary operation to two patterns. Available operations: `Multiply`, `Max`, `Min`, `Average`. Multiply is the most common — it acts as an AND gate (both patterns must be active at a cell for the cell to be active).

```rust
use tachyonfx::pattern::CombinedOp;

// AND: cells must be within radial AND diagonal
fx::dissolve(1000).with_pattern(
    CombinedPattern::multiply(RadialPattern::center(), DiagonalPattern::top_left_to_bottom_right())
);

// OR: whichever pattern activates the cell first
fx::dissolve(1000).with_pattern(
    CombinedPattern::max(RadialPattern::center(), DissolvePattern::new())
);

// MIN: stricter AND (minimum of two thresholds)
fx::dissolve(1000).with_pattern(
    CombinedPattern::min(SweepPattern::left_to_right(20), DiamondPattern::center())
);

// AVERAGE: blends two patterns
fx::dissolve(1000).with_pattern(
    CombinedPattern::average(RadialPattern::center(), DissolvePattern::new())
);
```

## Wave / Signal API

The wave system generates oscillating spatial signals for `WavePattern`. All types live in `tachyonfx::wave`.

### SignalSampler Trait

```rust
pub trait SignalSampler {
    fn sample(&self, x: f32, y: f32, t: f32) -> f32; // -> -1.0..1.0
}
```

Implemented for `Oscillator`, `Modulator`, `WaveLayer`, and `[WaveLayer]`.

- `x`, `y` are **cell coordinates relative to the effect area**, normalized to roughly -1.0..1.0 range
- `t` is animation progress (0.0 at start, 1.0 at end)

### Oscillator

Signal: `func(kx*x + ky*y + kt*t + phase)`, where `x`/`y` are cell coordinates relative to the effect area, `t` is animation progress (0.0–1.0).

```rust
Oscillator::sin(kx, ky, kt)      // sine wave
Oscillator::cos(kx, ky, kt)      // cosine wave
Oscillator::triangle(kx, ky, kt) // triangle wave
Oscillator::sawtooth(kx, ky, kt) // sawtooth wave
    .phase(radians)              // phase offset (float)
    .modulated_by(modulator)     // attach FM/AM modulator
```

**Wave shape behavior:**
- `sin` — smooth oscillation, peaks at ±1.0, crosses zero regularly
- `cos` — same shape as sin but offset by π/2 (starts at peak)
- `triangle` — linear ramp up/down, peaks at ±1.0, sharp corners
- `sawtooth` — linear ramp with hard reset, ramps from -1.0 to +1.0

**Frequency parameters:**
- `kx` — spatial frequency along x (columns). Higher = more oscillations per column. `kx=0.3` gives ~3 visible wave cycles across a 40-column area.
- `ky` — spatial frequency along y (rows). Higher = more oscillations per row. `ky=0.0` = horizontal-only waves.
- `kt` — temporal frequency. Higher = faster animation over the effect's lifetime. `kt=1.0` = one full cycle over the effect's duration. `kt=2.0` = two cycles.

### Modulator

Same signal formula as Oscillator but with an `intensity` scaling factor and a `ModTarget`.

```rust
Modulator::sin(kx, ky, kt)
    .phase(radians)
    .intensity(f32)       // amplitude scaling (default 1.0)
    .on_phase()           // FM synthesis (default)
    .on_amplitude()       // AM synthesis
```

**Modulation targets:**
- `ModTarget::Phase` — offsets the parent oscillator's phase input (FM synthesis). `intensity` controls how much phase modulation occurs. High intensity = chaotic, frequency-divided output. Low intensity = subtle warble.
- `ModTarget::Amplitude` — scales the parent's output around 1.0 (AM synthesis). `intensity` controls modulation depth. A modulator signal of 0.0 means no scaling; ±1.0 means full inversion/boost.

**FM synthesis intuition:** When a modulator with `kt=0` (static) modulates a carrier's phase, the carrier's frequency shifts based on spatial position — producing frequency-domain variation across the terminal. When `kt>0`, the modulation evolves over time, creating shimmering or warbling effects.

**AM synthesis intuition:** A modulator multiplying the carrier's amplitude creates volume pulsing. When the modulator has spatial variation (`kx` or `ky > 0`), different cells pulse at different rates — creating ripple or wave interference patterns.

```rust
// FM: modulator warps the carrier's phase
let carrier = Oscillator::sin(0.3, 0.0, 1.0);
let modulator = Modulator::sin(0.5, 0.5, 0.0)
    .intensity(3.0)    // strong phase warping
    .on_phase();

// AM: modulator controls carrier amplitude
let carrier = Oscillator::sin(0.3, 0.0, 1.0);
let modulator = Modulator::sin(0.0, 0.8, 0.5)
    .intensity(0.5)
    .on_amplitude();
```

### WaveLayer

One layer in a wave interference pattern. Holds a primary oscillator, optional second oscillator with combinator, amplitude, and post-transform.

```rust
WaveLayer::new(oscillator)           // single oscillator, amplitude 1.0
    .multiply(second_oscillator)     // element-wise product
    .average(second_oscillator)      // arithmetic mean
    .max(second_oscillator)          // element-wise max
    .amplitude(f32)                  // output scaling
    .power(n)                        // raise signal to power n (sharpens peaks)
    .abs()                           // absolute value (doubles frequency visually)
```

**Builder methods (chained on `WaveLayer::new(osc)`):**

| Method | Effect | Use case |
|---|---|---|
| `.multiply(osc2)` | `output = osc1 * osc2` | AND gate — both signals must be high |
| `.average(osc2)` | `output = (osc1 + osc2) / 2` | Smooth blend of two wave shapes |
| `.max(osc2)` | `output = max(osc1, osc2)` | OR gate — whichever signal is stronger |
| `.amplitude(f32)` | Scales output by factor | Control overall brightness |
| `.power(n)` | `output = signal^n` | Sharpens peaks (>1), softens (<1) |
| `.abs()` | `output = |signal|` | Doubles visual frequency (all peaks positive) |

Multiple layers are averaged when sampled as a `[WaveLayer]`. This is the primary way to combine layers — just create multiple `WaveLayer` instances and pass them as a slice to `WavePattern::new()`.

**Signal math:** The raw oscillator output is in [-1.0, 1.0]. The `.amplitude()` multiplier scales this range. `.power(n)` is applied after amplitude. `.abs()` is applied last before the layer is passed to `WavePattern`.

```rust
// Layer 1: horizontal wave, amplitude-boosted
let layer1 = WaveLayer::new(Oscillator::sin(0.4, 0.0, 1.0))
    .amplitude(1.5);

// Layer 2: vertical wave, power-sharpened
let layer2 = WaveLayer::new(Oscillator::cos(0.0, 0.6, 1.0))
    .power(3);

// Both layers averaged by WavePattern
WavePattern::new(vec![layer1, layer2]);
```

### WavePattern

Wraps one or more `WaveLayer`s into a spatial alpha field.

```rust
WavePattern::new(layer)              // single layer
    .with_layer(layer)               // add more layers (averaged)
    .with_contrast(i32)              // >1 pushes toward black/white extremes
    .with_transition_width(f32)      // soft edge width, normalized 0..1 (default 0.15)
```

**Signal processing pipeline:**
1. Each `WaveLayer` is sampled per-cell (averaged if multiple layers)
2. Combined signal rescaled from [-1.0, 1.0] to [0.0, 1.0]
3. Contrast-boosted (if `contrast > 1`): pushes values toward 0.0 or 1.0
4. Mixed with global animation progress (`alpha`)
5. Output alpha = (contrast_boosted_signal + alpha) / 2.0 — signal modulates when the effect activates each cell

**Contrast parameter:** Controls how "binary" the wave field is. `contrast=1` (default) = smooth sine-like alpha. `contrast=3` = sharp peaks/valleys. `contrast=5` = near-binary on/off. Higher values make the wave pattern more dramatic but may lose gradient detail.

**Transition width:** Controls softness at wave boundaries. `transition_width=0.15` (default) gives a gentle fade at wave edges. `transition_width=0.0` = hard cutoff. `transition_width=1.0` = full-area soft gradient.

### Full Wave Examples

```rust
use tachyonfx::{fx, pattern::WavePattern, wave::{Modulator, Oscillator, WaveLayer}};

// Example 1: FM-modulated wave pattern
let carrier = Oscillator::sin(0.3, 0.0, 1.0);
let modulator = Modulator::cos(0.5, 0.5, 0.0)
    .intensity(2.0)
    .on_phase();  // FM modulation

let layer = WaveLayer::new(carrier.modulated_by(modulator))
    .amplitude(0.8)
    .power(2);

let pattern = WavePattern::new(layer)
    .with_contrast(3)
    .with_transition_width(0.1);

fx::dissolve(2000).with_pattern(pattern);

// Example 2: Multi-layer interference — horizontal + vertical waves
let horizontal = WaveLayer::new(Oscillator::sin(0.4, 0.0, 1.0)).amplitude(0.6);
let vertical = WaveLayer::new(Oscillator::cos(0.0, 0.6, 1.0)).amplitude(0.6);

fx::dissolve(2000).with_pattern(
    WavePattern::new(vec![horizontal, vertical]).with_contrast(2)
);

// Example 3: Sawtooth wave for sharp sweeping reveal
let saw = WaveLayer::new(Oscillator::sawtooth(0.3, 0.0, 0.5))
    .power(4);

fx::dissolve(2500).with_pattern(
    WavePattern::new(saw).with_transition_width(0.05)
);

// Example 4: AM-modulated pulsing
let carrier = Oscillator::sin(0.2, 0.3, 1.0);
let am = Modulator::sin(0.0, 0.0, 0.8)
    .intensity(0.5)
    .on_amplitude();

let layer = WaveLayer::new(carrier.modulated_by(am));
fx::dissolve(2000).with_pattern(WavePattern::new(layer));

// Example 5: Triangle wave for gentle gradient
let tri = WaveLayer::new(Oscillator::triangle(0.15, 0.0, 1.0))
    .abs();  // positive-only peaks
fx::dissolve(2000).with_pattern(WavePattern::new(tri));
```

## Effect DSL

The DSL (feature-gated behind `dsl`, requires `std`) provides a string-based language for defining effects at runtime. Useful for config files, scripting, and the interactive browser editor (TachyonFX FTL).

### Quick Start

```rust
use tachyonfx::dsl::EffectDsl;

let dsl = EffectDsl::new(); // pre-registered with all standard effects
let effect = dsl.compiler()
    .compile("fx::dissolve(500)")
    .unwrap();
```

### Binding Variables

```rust
use ratatui::prelude::Color;
use tachyonfx::{dsl::EffectDsl, Motion};

let effect = dsl.compiler()
    .bind("motion", Motion::LeftToRight)
    .bind("c", Color::from_u32(0x1d2021))
    .compile("fx::sweep_in(motion, 10, 0, c, (1000, QuadOut))")
    .unwrap();
```

### Registering Custom Effects

```rust
let dsl = EffectDsl::new()
    .register("my_effect", |args| {
        // args provides typed accessors: .duration()?, .color()?, .f32()?, etc.
        Ok(fx::sleep(args.duration()?).into())
    });
```

### Parsing Without Compiling

```rust
use tachyonfx::dsl::EffectExpression;

let expr = EffectExpression::parse(
    "fx::sequence(&[fx::dissolve(500), fx::fade_to(Color::Red, 1000)])"
).unwrap();

// Round-trip back to string
let dsl_string = expr.to_string();
```

### Round-Tripping Effects to DSL

Any effect can attempt conversion to a DSL expression:

```rust
let effect = fx::dissolve(500);
match effect.to_dsl() {
    Ok(expr) => println!("DSL: {expr}"),
    Err(e) => eprintln!("Not supported: {e}"),
}
```

### DSL Syntax Reference

```
// Function calls
fx::dissolve(500)
fx::fade_to_fg(Color::Red, (1000, Interpolation::QuadOut))
fx::sequence(&[fx::dissolve(500), fx::coalesce(500)])

// Let bindings (terminated with ;)
let timer = (1000, QuadOut);
let c = Color::from_u32(0xff0000);
fx::fade_to_fg(c, timer);

// Method chains
fx::dissolve(1000).with_pattern(SweepPattern::left_to_right(25))

// Enums (module::Variant)
Motion::LeftToRight
Interpolation::BounceOut
RepeatMode::Times(3)
EvolveSymbolSet::CircleFill
CellFilter::Text

// Struct init
Glitch::builder()
    .cell_glitch_ratio(0.1)
    .action_start_delay_ms(0..500)
    .action_ms(100..300)
    .build()
```

### Error Handling

`DslError` has 20+ variants covering tokenization, parsing, type-checking, and compilation errors. Each includes source location (`ExprSpan` with start/end byte offsets) for debugging.

`DslParseError` wraps `DslError` with additional context (line/column numbers, surrounding source text).

### DSL Limitations

- Requires `std` feature (not available in `no_std`)
- Custom closures (`effect_fn`, `effect_fn_buf`) cannot be expressed in DSL
- `Glitch` builder is supported but its internal `glitch_cells` field is skipped
- Some internal effects (`resize_area`) may not round-trip cleanly
