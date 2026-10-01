# MyFCS Runtime Reverse Engineering Report

> Repository: `Chookri/MyFCS`  
> Branch/ref inspected: `main`  
> Source tree SHA: `7915283557aeb37b1a1c9876bae3f59b1866ddd6`  
> Scope: JavaScript runtime files in the repository  
> JS files inspected: **173**

## 1. Purpose

This document records the runtime reverse-engineering map of MyFCS for use as a persistent engineering reference for Tradigy.

The objective is **not** to rewrite MyFCS. It is to identify:
- which file executes what role;
- how webpack chunks enter the runtime;
- what the runtime lifecycle is;
- what each indicator consumes and produces;
- which modules are useful as implementation references;
- which files are bundled/minified/obfuscated and therefore require runtime tracing rather than direct source reading.

## 2. Important limitation

The repository source was inspected statically through its actual JavaScript files. The environment used for this report could not launch a full browser copy of the repository, so this is a **static runtime-path reconstruction**, not a Chrome DevTools recording of live execution.

Where a runtime path is inferred from webpack wrappers, method names, engine references, and renderer calls, it is marked as a reconstruction rather than a claim that every branch was observed at runtime.

## 3. Runtime architecture discovered

### Main runtime

`src/fcsapi-chart.js` is the main public bundle. It is approximately 731 KB and is webpack-packed. It contains a large number of runtime methods including:

- data processing/alignment;
- chart rendering;
- price-axis rendering;
- indicator price labels;
- line/dot/area/candle rendering;
- drawing rendering;
- markers;
- Heikin Ashi;
- crosshair;
- replay-related chart behavior;
- lazy extra-chart loading;
- websocket/connection-style methods.

It also contains strong signs of minification/obfuscation, including string-table/decoder style code. Direct decompilation is therefore less useful than mapping its runtime boundaries.

### Webpack chunks

The `src/chunks/` directory contains the actual runtime modules loaded into the webpack chunk array:

`self.webpackChunkfcsapi_chart = self.webpackChunkfcsapi_chart || []`

Each chunk generally follows:

`chunk load → webpack module registration → exported class/function → chart engine lifecycle`

### Indicator lifecycle

The indicator modules consistently expose the following runtime model:

`getSettingsConfig() → calculate() → render()`

Typical data flow:

`engine.data → indicator calculation → this.data → renderer → chart panel/main chart`

Many indicators also expose drawing helpers such as:

- `drawLine`
- `drawHistogram`
- `drawHLine`
- `drawFill`
- `drawLabel`
- signal arrows/markers
- pattern-specific drawings

This is particularly useful for Tradigy because the indicator calculation and the chart renderer are separable runtime layers.

## 4. Core runtime modules

### `src/fcsapi-chart.js`

Classification: **main webpack bundle / public chart runtime**

Observed runtime method families include:
- connection/heartbeat;
- data processing and timestamp alignment;
- cache management;
- UI creation/event listeners;
- price-axis and bid/ask rendering;
- indicator price labels;
- line/dot/fill/crosshair rendering;
- drawing rendering;
- markers;
- Heikin Ashi;
- chart/lazy-load checks.

Reverse-engineering priority: **VERY HIGH**.

Recommended method:
1. pretty-print;
2. identify module boundaries;
3. resolve webpack module IDs;
4. trace public chart creation;
5. trace data loading;
6. trace timeframe/replay;
7. trace indicator registration;
8. trace rendering calls.

### `src/chunks/src_core_UIManager_js.js`

Classification: core toolbar/UI runtime.

Important runtime areas include:
- timeframe dropdown;
- favorite timeframes;
- chart type;
- indicators;
- templates;
- theme;
- screenshot;
- shortcuts;
- replay controls;
- multi-chart layout;
- settings.

Reverse-engineering priority: **VERY HIGH**.

### `src/chunks/src_core_UIManager-helper_js.js`

Classification: UI settings and indicator UI helper.

Important runtime areas:
- indicator updates;
- active engine/UI manager lookup;
- indicator categories;
- result rendering;
- appearance/scales/visibility settings;
- chart color settings;
- settings population;
- apply/cancel/reset;
- color/slider/checkbox handling.

Reverse-engineering priority: **HIGH**.

### `src/chunks/src_core_ShortcutManager_js.js`

Classification: keyboard command runtime.

Important commands include:
- chart movement/zoom;
- reset;
- start/end;
- date navigation;
- log/percent/invert scale;
- indicators;
- drawings;
- screenshot;
- templates/layout;
- replay play/pause;
- replay stepping.

Reverse-engineering priority: **HIGH** for Tradigy keyboard UX.

### `src/chunks/src_core_TimeRangeSelector_js.js`

Classification: timeframe/date/range runtime.

Important runtime areas:
- timezone clock;
- range selection;
- date picker;
- period parsing;
- candle-count calculation;
- jump/live behavior;
- display range;
- active range.

Reverse-engineering priority: **VERY HIGH** for Replay and timeframe behavior.

### `src/chunks/src_core_Renderer-charts_js.js`

Classification: chart renderer.

Observed rendering entry point includes candle rendering and related chart primitives.

Reverse-engineering priority: **VERY HIGH**.

### `src/chunks/src_core_InfoOverlay_js.js`

Classification: candle/chart information overlay.

Runtime:
`hovered candle/profile data → overlay state → DOM update`

Methods include:
- create overlay;
- set information;
- profile data;
- hovered candle;
- last candle;
- search modal;
- destroy.

### `src/chunks/src_core_ProfilePanel_js.js`

Classification: market/profile modal.

Runtime:
`profile data → modal → tabs → formatted sections/rows`

Tabs include overview/performance/trading/statistics.

### `src/chunks/src_indicators_technicals_BaseIndicator_js.js`

Classification: indicator base class.

Core lifecycle methods:
- `calculate`
- `render`
- `setVisible`
- `updateConfig`

This is one of the most important files for reproducing the indicator architecture in Tradigy.

### Indicator index chunks

- `src_indicators_financial_pine-index_js.js`
- `src_indicators_signals_pine-index_js.js`
- `src_indicators_technicals_pine-index_js.js`

These are very small registration/index chunks rather than large calculation engines.

## 5. Drawing runtime

### `src/chunks/drawings.js`

Size: approximately 298 KB.

This is a major subsystem rather than a simple drawing definition.

Observed runtime method families:
- point creation/update/completion;
- render;
- JSON serialization/deserialization;
- volume/time calculations;
- accuracy/quality calculations;
- hover/tooltip;
- table/grid operations;
- row/column insertion/removal;
- resizing.

This module should be treated as a **drawing framework**.

For Tradigy, it is more useful to reverse-map the drawing lifecycle than to copy individual drawing implementations blindly.

## 6. Auxiliary runtime

### `src/chunks/other.js`

Size: approximately 111 KB.

Contains auxiliary runtime services including modal/instance management and generic calculation/render/update functionality.

Priority: **MEDIUM-HIGH**.

## 7. Indicator runtime model

All indicator families use the same broad lifecycle.

### Technical indicators

`engine.data → calculate() → this.data → render()`

Typical outputs:
- main-chart lines;
- oscillator lines;
- histograms;
- horizontal levels;
- fills;
- special overlays.

### Pattern indicators

`engine.data → candle-pattern test → render() → marker/shape/label`

### Signal indicators

`engine.data → signal calculation → render() → arrow/zone/label/line`

This distinction matters for Tradigy:

**Technical indicator = numeric series**

**Pattern = event/pattern annotation**

**Signal = event + visual marker/zone**

## 8. Complete per-file runtime inventory

The table below intentionally includes every JavaScript file in the inspected repository.

| File | Size | Runtime role | Runtime path |
|---|---:|---|---|
| `src/chunks/drawings.js` | 298,413 | Drawing subsystem | drawing tool load → point/control state → geometry/render → interaction → JSON persistence |
| `src/chunks/indicator.editorchoice.aroon.js` | 5,028 | Editor-choice indicator | load → getSettingsConfig → calculate → render → plot output |
| `src/chunks/indicator.editorchoice.atr.js` | 4,934 | Editor-choice indicator | load → getSettingsConfig → calculate → render → plot output |
| `src/chunks/indicator.editorchoice.macd.js` | 6,992 | Editor-choice indicator | load → getSettingsConfig → calculate → render → plot output |
| `src/chunks/indicator.editorchoice.sma.js` | 6,385 | Editor-choice indicator | load → getSettingsConfig → calculate → render → plot output |
| `src/chunks/indicator.editorchoice.stochastic.js` | 7,240 | Editor-choice indicator | load → getSettingsConfig → calculate → render → plot output |
| `src/chunks/indicator.financial.matest-simple.js` | 3,937 | Financial/Pine test indicator | load → getSettingsConfig → calculate → render → plot output |
| `src/chunks/indicator.financial.matest.js` | 4,245 | Financial/Pine test indicator | load → getSettingsConfig → calculate → render → plot output |
| `src/chunks/indicator.financial.osctest.js` | 8,462 | Financial/Pine test indicator | load → getSettingsConfig → calculate → render → plot output |
| `src/chunks/indicator.financial.test-alma-only.js` | 3,400 | Financial/Pine test indicator | load → getSettingsConfig → calculate → render → plot output |
| `src/chunks/indicator.financial.test-hma-only.js` | 3,083 | Financial/Pine test indicator | load → getSettingsConfig → calculate → render → plot output |
| `src/chunks/indicator.financial.test-moving-averages.js` | 4,259 | Financial/Pine test indicator | load → getSettingsConfig → calculate → render → plot output |
| `src/chunks/indicator.financial.test-vwap-only.js` | 3,003 | Financial/Pine test indicator | load → getSettingsConfig → calculate → render → plot output |
| `src/chunks/indicator.patterns.doji.js` | 5,199 | Pattern indicator | load → getSettingsConfig → calculate/pattern test → render → pattern drawing/label |
| `src/chunks/indicator.patterns.dragonfly-gravestone.js` | 5,635 | Pattern indicator | load → getSettingsConfig → calculate/pattern test → render → pattern drawing/label |
| `src/chunks/indicator.patterns.engulfing.js` | 4,234 | Pattern indicator | load → getSettingsConfig → calculate/pattern test → render → pattern drawing/label |
| `src/chunks/indicator.patterns.hanging-inverted.js` | 5,514 | Pattern indicator | load → getSettingsConfig → calculate/pattern test → render → pattern drawing/label |
| `src/chunks/indicator.patterns.harami-cross.js` | 5,804 | Pattern indicator | load → getSettingsConfig → calculate/pattern test → render → pattern drawing/label |
| `src/chunks/indicator.patterns.ladder-top-bottom.js` | 5,732 | Pattern indicator | load → getSettingsConfig → calculate/pattern test → render → pattern drawing/label |
| `src/chunks/indicator.patterns.long-legged-doji.js` | 4,465 | Pattern indicator | load → getSettingsConfig → calculate/pattern test → render → pattern drawing/label |
| `src/chunks/indicator.patterns.marubozu.js` | 3,885 | Pattern indicator | load → getSettingsConfig → calculate/pattern test → render → pattern drawing/label |
| `src/chunks/indicator.patterns.mat-hold.js` | 4,923 | Pattern indicator | load → getSettingsConfig → calculate/pattern test → render → pattern drawing/label |
| `src/chunks/indicator.patterns.meeting-lines.js` | 6,166 | Pattern indicator | load → getSettingsConfig → calculate/pattern test → render → pattern drawing/label |
| `src/chunks/indicator.patterns.morning-evening-star.js` | 4,703 | Pattern indicator | load → getSettingsConfig → calculate/pattern test → render → pattern drawing/label |
| `src/chunks/indicator.patterns.piercing-darkcloud.js` | 4,672 | Pattern indicator | load → getSettingsConfig → calculate/pattern test → render → pattern drawing/label |
| `src/chunks/indicator.patterns.rising-falling-window.js` | 4,794 | Pattern indicator | load → getSettingsConfig → calculate/pattern test → render → pattern drawing/label |
| `src/chunks/indicator.patterns.separating-lines.js` | 6,132 | Pattern indicator | load → getSettingsConfig → calculate/pattern test → render → pattern drawing/label |
| `src/chunks/indicator.patterns.shooting-star-hammer.js` | 5,448 | Pattern indicator | load → getSettingsConfig → calculate/pattern test → render → pattern drawing/label |
| `src/chunks/indicator.patterns.spinning-top.js` | 4,873 | Pattern indicator | load → getSettingsConfig → calculate/pattern test → render → pattern drawing/label |
| `src/chunks/indicator.patterns.tasuki-gap.js` | 5,800 | Pattern indicator | load → getSettingsConfig → calculate/pattern test → render → pattern drawing/label |
| `src/chunks/indicator.patterns.three-inside.js` | 5,657 | Pattern indicator | load → getSettingsConfig → calculate/pattern test → render → pattern drawing/label |
| `src/chunks/indicator.patterns.three-methods.js` | 5,978 | Pattern indicator | load → getSettingsConfig → calculate/pattern test → render → pattern drawing/label |
| `src/chunks/indicator.patterns.three-outside.js` | 5,303 | Pattern indicator | load → getSettingsConfig → calculate/pattern test → render → pattern drawing/label |
| `src/chunks/indicator.patterns.three-soldiers-crows.js` | 4,503 | Pattern indicator | load → getSettingsConfig → calculate/pattern test → render → pattern drawing/label |
| `src/chunks/indicator.patterns.tweezer.js` | 4,101 | Pattern indicator | load → getSettingsConfig → calculate/pattern test → render → pattern drawing/label |
| `src/chunks/indicator.signals.bb-breakout-signal.js` | 5,015 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.belt-hold-signal.js` | 5,229 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.bollinger-squeeze-signal.js` | 6,478 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.breakaway-gap-signal.js` | 5,936 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.cup-handle-signal.js` | 5,051 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.dark-cloud-piercing-signal.js` | 5,421 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.doji-signal.js` | 5,073 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.double-top-bottom-signal.js` | 6,008 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.engulfing-signal.js` | 4,748 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.exhaustion-gap-signal.js` | 5,599 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.fibonacci-retracement-signal.js` | 7,703 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.gap-signal.js` | 7,204 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.harami-signal.js` | 4,989 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.harmonic-pattern-signal.js` | 7,869 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.head-shoulders-signal.js` | 5,689 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.higher-high-lower-low-signal.js` | 5,509 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.inside-outside-bar-signal.js` | 5,341 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.island-reversal-signal.js` | 5,855 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.ma-cross-signal.js` | 4,944 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.macd-signal.js` | 4,953 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.marubozu-signal.js` | 4,947 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.money-flow-divergence-signal.js` | 7,116 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.morning-evening-star-signal.js` | 5,353 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.order-block-signal.js` | 7,367 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.pinbar-signal.js` | 4,887 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.range-breakout-signal.js` | 7,669 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.rsi-divergence-signal.js` | 8,373 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.rsi-signal.js` | 4,941 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.stochastic-divergence-signal.js` | 7,317 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.strong-trend-signal.js` | 5,462 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.support-resistance-signal.js` | 5,482 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.swing-point-signal.js` | 5,175 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.three-line-strike-signal.js` | 4,833 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.three-soldiers-crows-signal.js` | 6,388 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.trend-signal.js` | 5,454 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.triangle-breakout-signal.js` | 6,998 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.triple-ma-cross-signal.js` | 5,652 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.tweezer-signal.js` | 5,480 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.volume-profile-signal.js` | 7,528 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.volume-spike-signal.js` | 4,493 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.signals.vwap-touch-signal.js` | 6,529 | Signal indicator | load → getSettingsConfig → calculate → render → marker/label/zone drawing |
| `src/chunks/indicator.technicals.adl.js` | 3,778 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.adx-strength.js` | 5,712 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.adx.js` | 5,612 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.alligator.js` | 5,574 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.aroon.js` | 5,255 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.atr.js` | 4,489 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.awesome-oscillator.js` | 5,419 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.balance-of-power.js` | 4,561 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.bb-bandwidth.js` | 6,744 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.bb.js` | 4,539 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.bbtrend.js` | 6,267 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.bop.js` | 4,456 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.bull-bear-power.js` | 4,652 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.bw-mfi.js` | 4,150 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.cci.js` | 5,673 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.chaikin-osc.js` | 4,989 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.chaikin-oscillator.js` | 4,873 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.chandelier-exit.js` | 5,198 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.choppiness-index.js` | 4,838 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.choppiness.js` | 5,323 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.cmf.js` | 5,757 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.cmo.js` | 6,858 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.connors-rsi.js` | 8,346 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.coppock-curve.js` | 5,006 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.dema.js` | 3,816 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.donchian.js` | 4,547 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.double-ema.js` | 3,289 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.dpo.js` | 4,727 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.ease-of-movement.js` | 4,797 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.elder-force-index.js` | 4,546 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.elder-ray.js` | 5,001 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.ema.js` | 4,185 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.envelope.js` | 5,156 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.eom.js` | 5,150 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.fisher-transform.js` | 6,061 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.force-index.js` | 4,687 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.hma.js` | 5,655 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.ichimoku.js` | 6,133 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.imi.js` | 4,707 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.keltner-channels.js` | 5,367 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.keltner.js` | 4,748 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.klinger-oscillator.js` | 5,128 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.klinger.js` | 6,593 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.kst.js` | 5,906 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.linear-regression.js` | 5,952 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.macd.js` | 6,130 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.mass-index.js` | 5,014 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.mcginley-dynamic.js` | 5,157 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.mfi.js` | 6,003 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.momentum.js` | 4,725 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.obv.js` | 3,758 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.parabolic-sar.js` | 4,286 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.pivot-points.js` | 7,323 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.ppo.js` | 5,918 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.price-channel.js` | 4,863 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.pvt.js` | 3,672 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.qqe.js` | 6,342 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.rainbow-ma.js` | 10,010 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.roc.js` | 4,751 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.rsi.js` | 7,690 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.rvi.js` | 6,199 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.schaff-trend.js` | 6,211 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.sma.js` | 6,383 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.smi.js` | 5,789 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.squeeze-momentum.js` | 7,125 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.stddev-channel.js` | 5,589 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.stoch-rsi.js` | 5,879 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.stochastic.js` | 6,935 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.super-trend.js` | 5,039 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.supertrend.js` | 6,723 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.tdi.js` | 7,333 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.tema.js` | 3,935 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.trix.js` | 5,229 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.tsi.js` | 6,309 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.ultimate-oscillator.js` | 5,676 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.volume-oscillator.js` | 4,554 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.volume.js` | 9,772 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.vortex.js` | 4,585 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.vwap.js` | 3,234 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.waddah-attar-explosion.js` | 5,930 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.wave-trend.js` | 5,514 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.wavetrend.js` | 6,952 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.williams-r.js` | 5,975 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/indicator.technicals.zigzag.js` | 4,475 | Technical indicator | load → getSettingsConfig → calculate → render → line/histogram/fill output |
| `src/chunks/other.js` | 111,428 | Shared auxiliary runtime | lazy chunk load → modal/utility/runtime services → chart callbacks |
| `src/chunks/src_core_InfoOverlay_js.js` | 10,105 | Core UI/runtime | chunk load → UI/core service init → DOM/events → chart state mutation/render |
| `src/chunks/src_core_ProfilePanel_js.js` | 21,738 | Core UI/runtime | chunk load → UI/core service init → DOM/events → chart state mutation/render |
| `src/chunks/src_core_Renderer-charts_js.js` | 11,849 | Core UI/runtime | chunk load → UI/core service init → DOM/events → chart state mutation/render |
| `src/chunks/src_core_ShortcutManager_js.js` | 34,284 | Core UI/runtime | chunk load → UI/core service init → DOM/events → chart state mutation/render |
| `src/chunks/src_core_TimeRangeSelector_js.js` | 27,799 | Core UI/runtime | chunk load → UI/core service init → DOM/events → chart state mutation/render |
| `src/chunks/src_core_UIManager-helper_js.js` | 179,827 | Core UI/runtime | chunk load → UI/core service init → DOM/events → chart state mutation/render |
| `src/chunks/src_core_UIManager_js.js` | 185,938 | Core UI/runtime | chunk load → UI/core service init → DOM/events → chart state mutation/render |
| `src/chunks/src_indicators_financial_pine-index_js.js` | 431 | Indicator framework/index | chunk load → indicator registration/framework → calculate/render lifecycle |
| `src/chunks/src_indicators_signals_pine-index_js.js` | 187 | Indicator framework/index | chunk load → indicator registration/framework → calculate/render lifecycle |
| `src/chunks/src_indicators_technicals_BaseIndicator_js.js` | 2,317 | Indicator framework/index | chunk load → indicator registration/framework → calculate/render lifecycle |
| `src/chunks/src_indicators_technicals_pine-index_js.js` | 218 | Indicator framework/index | chunk load → indicator registration/framework → calculate/render lifecycle |
| `src/fcsapi-chart.js` | 731,067 | Main public chart runtime / bundle | library load → module bootstrap → chart engine/UI/data/replay/drawing runtime |

## 9. High-value reverse-engineering targets for Tradigy

### Tier A — Chart engine/runtime

1. `src/fcsapi-chart.js`
2. `src/chunks/src_core_Renderer-charts_js.js`
3. `src/chunks/src_core_TimeRangeSelector_js.js`
4. `src/chunks/src_core_UIManager_js.js`
5. `src/chunks/src_core_UIManager-helper_js.js`
6. `src/chunks/drawings.js`

### Tier B — Indicator framework

1. `src/chunks/src_indicators_technicals_BaseIndicator_js.js`
2. technical indicator chunks
3. signal indicator chunks
4. pattern indicator chunks

### Tier C — UX/support

1. `InfoOverlay`
2. `ProfilePanel`
3. `ShortcutManager`
4. `other.js`

## 10. Decompilation assessment

### `fcsapi-chart.js`

**Possible:** yes.

**Expected quality:** partial-to-high after multiple passes.

Recommended pipeline:

`raw JS → beautify → identify webpack modules → resolve string decoder → replace decoded strings → simplify wrappers → rename variables → map module dependencies → runtime trace`

Do not expect the original author source names/comments to be recoverable perfectly.

### `src/chunks/*.js`

Most indicator chunks are already much closer to ordinary source structure than the main bundle.

Recommended pipeline:

`chunk → beautify → inspect class → inspect getSettingsConfig → inspect calculate → inspect render → map renderer calls`

For many indicators, a clean functional reconstruction is practical.

## 11. Tradigy integration rule

The reverse-engineering work should **not modify MyFCS core behavior** unless explicitly required.

The preferred architecture is:

`MyFCS/FCS runtime`
↓
`Tradigy adapter/interface`
↓
`Tradigy data / replay / journal / alerts`

This keeps FCS as the chart engine while Tradigy owns:
- TwelveData integration;
- journal;
- blind replay;
- alerts;
- MFE/MAE;
- trade capture;
- user concepts;
- Tradigy-specific UI.

## 12. Next runtime tracing targets

The next pass should trace these concrete runtime chains:

### Chain A — Chart startup

`examples/index.html → fcsapi-chart.js → chart constructor/init → engine → renderer`

### Chain B — Data loading

`symbol/timeframe → data source → processData → engine.data → render`

### Chain C — Timeframe switching

`timeframe selection → range calculation → data reload → engine state → render`

### Chain D — Replay

`Replay start → replay state → visible candle boundary → data/render → step → play/pause`

### Chain E — Indicator

`indicator selection → lazy chunk load → BaseIndicator → config → calculate → render`

### Chain F — Drawing

`tool selection → drawing instance → points → interaction → render → serialization`

### Chain G — Tradigy alert

`candle update → indicator calculation → condition → alert event`

## 13. Status

**Repository inventory:** COMPLETE  
**JavaScript file inventory:** COMPLETE — 173 files  
**Static runtime lifecycle mapping:** COMPLETE at architecture/file level  
**Detailed live browser call-stack tracing:** NEXT PASS  
**Deobfuscation of main bundle:** NEXT PASS  
**Tradigy adapter design:** NEXT PASS

This report is intended to remain in the MyFCS source tree and be updated as runtime chains are confirmed.


# 9. Runtime Trace Pass 2 — Concrete Execution Chains

> This section is a deeper static reconstruction from the actual bundled runtime code. It maps concrete method calls and state mutations that were found in `src/fcsapi-chart.js` and the readable core chunks.

## 9.1 Chart bootstrap / runtime object graph

The main chart engine creates the Replay Manager directly during chart initialization:

```
chart engine constructor
  └─ replayManager = new ReplayManager(engine)
```

The same initialization sequence creates/loads the major runtime services around it:

```
engine
 ├─ indicatorManager
 ├─ drawingManager
 ├─ interactionManager
 ├─ replayManager
 ├─ UIManager (lazy-loaded)
 ├─ InfoOverlay (lazy-loaded)
 ├─ ProfilePanel
 ├─ TimeRangeSelector
 ├─ MultiChartManager
 └─ SocketManager
```

This confirms that Replay, Indicators, Drawings and UI are not independent applications; they mutate shared chart-engine state.

## 9.2 Timeframe selection → data load

The readable `TimeRangeSelector` shows the concrete path:

```
selectRange(range)
  → updateTimeframeDropdown(period)
  → loadRangeData(period, length, ...)
  → timeframeAggregator.calculateAggregation(period)
  → choose source timeframe
  → build API URL
  → fetch(...)
  → response.json()
  → optional timeframeAggregator.processData(...)
  → chart.setData(data, info)
  → zoomToTimeRange() / zoomToShowAll()
```

The API request is assembled from:

- `symbol`
- `period`
- authentication parameters
- optional `from` / `to`
- optional `length`
- `is_chart=1`

This is a strong integration boundary for Tradigy: the chart engine expects normalized candle data after the data-source/aggregation layer.

## 9.3 Timeframe dropdown → engine.setPeriod()

The UI path is:

```
UIManager.createTimeframeDropdown()
  → customSelectChange
  → engine.chartInstance.setPeriod(period)
```

Keyboard timeframe changes use:

```
ShortcutManager.setTimeframe(period)
  → uiManager.updateTimeframeDropdown(period)
  → chartInstance.setPeriod(period)
```

Therefore Tradigy should treat `setPeriod()` as the central timeframe transition point rather than making the UI directly reload candles.

## 9.4 Replay runtime — confirmed state model

The main bundle contains a dedicated Replay Manager class. Its reconstructed public methods are:

- `start(startIndex)`
- `pause()`
- `stop()`
- `setSpeed()`
- `togglePlayPause()`
- `getCurrentTime()`

Obfuscated methods can be mapped from their behavior to:

- set replay data/index
- skip forward
- skip backward
- jump/set index
- get progress
- update replay UI
- toggle auto-fit

### Replay start

The concrete state transition is approximately:

```
ReplayManager.start(startIndex)
  → originalData = engine.data
  → originalScrollOffset = engine.scrollOffset
  → save original price range / autoscale state
  → replayStartIndex = startIndex
  → replayEndIndex = replayData.length - 1
  → currentIndex = replayStartIndex
  → isActive = true
  → engine.data = originalData.slice(0, currentIndex + 1)
  → recalculate indicators
  → calculateVisibleRange()
  → render()
  → updateReplayUI()
```

### Replay play loop

The runtime uses `requestAnimationFrame` and a speed-controlled frame interval.

On each replay step:

```
currentIndex++
engine.data = originalData.slice(0, currentIndex + 1)
engine.scrollOffset += 1
indicatorManager.recalculateAll()
optional auto-fit price range
calculateVisibleRange()
render()
updateReplayUI()
```

This is the critical Blind Backtest property:

**future candles remain stored in `originalData`, but are not exposed through `engine.data` while replay is active.**

That means indicator calculation can be made candle-causal if the calculation itself only reads `engine.data`.

### Replay manual stepping

Forward:

```
skipForward(n)
  → current replay index advances
  → engine.data = replayData.slice(0, index + 1)
  → indicatorManager.recalculateAll()
  → calculateVisibleRange()
  → render()
```

Backward follows the same pattern and recalculates indicators from the shortened visible dataset.

### Replay stop

```
stop()
  → cancel animation
  → isActive = false
  → engine.data = originalData
  → restore scrollOffset
  → restore price range / autoscale
  → recalculate indicators
  → render()
  → reconnect live socket when applicable
```

## 9.5 Replay selection interaction

The interaction manager has an explicit `replaySelectionMode`.

The mouse interaction path is approximately:

```
Replay selection mode
  → mouse down records selection point/index
  → mouse up resolves selected candle
  → replay manager receives selected index
  → ReplayManager starts from that index
```

This is separate from ordinary drawing/cursor interaction.

## 9.6 Indicator execution boundary

The Indicator Manager contains the concrete lifecycle:

```
add(indicatorConfig)
  → loadIndicatorClass(type)
  → new Indicator(id, config, engine)
  → indicator.init()
  → optional panel creation
  → render
```

Its recalculation method is explicitly:

```
recalculateAll()
  → indicators.forEach(indicator => indicator.calculate())
```

Its rendering method then calls visible main-chart indicators and panel rendering.

This gives Tradigy a clean execution boundary:

```
engine.data
   ↓
indicator.calculate()
   ↓
indicator.data / plots / signal state
   ↓
indicator.render()
   ↓
chart renderer / panels / annotations
```

## 9.7 Data update boundary

The engine also recalculates indicators after normal data updates:

```
appendData(...)
  → parseData(...)
  → merge/update candles
  → calculateTimeLabelIndices()
  → calculateVisibleRange()
  → calculatePriceRange()
  → indicatorManager.recalculateAll()
  → render()
```

A separate update path for historical/more data similarly ends in indicator recalculation and render.

Therefore a Tradigy live-alert architecture can hook at the candle-update boundary before/around indicator recalculation, provided alert evaluation is kept causal.

## 9.8 Important architecture conclusion for Tradigy

The most useful reusable abstraction discovered is not the visual renderer. It is the **state pipeline**:

```
DATA SOURCE
   ↓
NORMALIZED CANDLES
   ↓
ENGINE.DATA
   ↓
INDICATOR CALCULATION
   ↓
SIGNAL / DRAWING STATE
   ↓
RENDER
   ↓
ALERT / JOURNAL HOOK
```

For Blind Backtest:

```
ORIGINAL DATA
   ↓
REPLAY INDEX
   ↓
ENGINE.DATA = ORIGINAL.slice(0, index + 1)
   ↓
RECALCULATE INDICATORS
   ↓
EVALUATE ALERTS
   ↓
RENDER / JOURNAL SNAPSHOT
```

This is the runtime boundary Tradigy should reproduce.

## 9.9 Current reverse-engineering confidence

### High confidence — directly observed in source

- TimeRangeSelector API/data-load path.
- UI timeframe → `chartInstance.setPeriod()`.
- Replay Manager existence and constructor placement.
- Replay `originalData` preservation.
- Replay `engine.data = originalData.slice(...)`.
- Replay indicator recalculation.
- Replay render/visible-range update.
- Indicator Manager `recalculateAll()`.
- Indicator loading and `init()` path.
- Normal append-data → indicator recalculation → render path.

### Medium confidence — reconstructed from obfuscated property mappings

- Exact original names of several Replay Manager private methods.
- Exact method name used by the interaction manager to submit the selected replay index.
- Some private engine property names inside the main bundle.

### Not yet runtime-observed

- A live Chrome DevTools call-stack capture.
- Exact network timing between API response and chart render.
- Exact lazy chunk-load timing for every indicator.
- Full alert/event execution path because MyFCS does not appear to expose the complete Tradigy-style alert engine as a separate readable subsystem.

## 9.10 Next target

The next reverse-engineering pass should focus on:

1. exact replay-selection method mapping;
2. `setData()` internals;
3. `setPeriod()` data-fetch transition;
4. indicator lazy-loading/chunk registration;
5. drawing manager → `drawings.js` execution boundary;
6. alert/event hooks;
7. a Tradigy adapter interface that can sit around these boundaries without modifying the FCS chart renderer.
