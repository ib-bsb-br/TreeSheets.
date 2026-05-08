# RK3588 + Debian 11 + X11/Ratpoison Performance Feasibility for a TreeSheets Fork

This document evaluates what is realistically implementable in a TreeSheets fork for your platform constraints:

- Rockchip RK3588 (Mali-G610 MC4)
- Debian 11 (Bullseye), Linux 5.10 BSP
- X11 + Ratpoison (no Wayland, no pure EGLFS session)

It is grounded in current TreeSheets source code architecture.

## 1) Ground truth from TreeSheets codebase

### 1.1 Rendering path is wxWidgets immediate-mode 2D

TreeSheets rendering is built around `wxDC` calls (`DrawText`, `DrawLine`, `DrawRectangle`, `DrawBitmap`, etc.) from core document/cell/grid/text code.

- Canvas paint invokes `doc->Draw(dc)` through `wxPaintDC`/`wxBufferedPaintDC`. See `src/tscanvas.h`.
- Document render path calls `Render(dc)` and selection overlays in `Draw(wxDC&)`. See `src/document.h`.
- Most primitives are CPU/2D `wxDC` operations in `src/grid.h`, `src/cell.h`, `src/text.h`, and helpers in `src/wxtools.h`.

Implication: by default, TreeSheets is not GL-scenegraph based and does not directly use OpenGL ES/Vulkan/OpenCL.

### 1.2 Build system does not currently enable a GPU renderer backend

`CMakeLists.txt` links against wxWidgets GUI libraries (`wx::aui wx::adv wx::core wx::xml wx::net`) and does not define a dedicated GL rendering module.

Implication: enabling Mali acceleration is not a flag flip; it requires architectural additions.

## 2) Feasibility matrix (for your exact stack)

### 2.1 High-feasibility, high-ROI changes (do first)

1. **Compiler and link optimizations tuned for RK3588**
   - Add ARMv8 tuning flags for A76/A55 mixed cores (`-mcpu=cortex-a76.cortex-a55` or conservative `-mcpu=native` when local-build only).
   - Keep/extend existing IPO/LTO path (`ENABLE_IPO` already exists).
   - Add PGO option in fork for representative workloads.

2. **Reduce over-invalidation and repaint area**
   - Replace broad `Refresh()` usage with tighter `RefreshRect()` where safe.
   - Cache layout/text metrics to avoid redundant recomputation during mouse move/selection drag.

3. **Event-loop and input-path micro-optimizations**
   - `OnMotion` currently triggers frequent updates and potential refreshes; throttle/coalesce hover updates.
   - Limit status bar updates when selection hash unchanged.

4. **I/O and startup improvements**
   - Profile large-sheet load/save hotspots.
   - Prefer faster compression settings or async save option if user enables it.

These are strongly feasible without replacing GUI stack and will give immediate wins under X11/Ratpoison.

### 2.2 Medium feasibility (targeted acceleration)

5. **Introduce optional wxGraphicsContext path for selected operations**
   - For anti-aliased lines/rounded shapes and batched drawing where backend can map better to accelerated paths.
   - Must keep wxDC fallback for compatibility.

6. **Add optional wxGLCanvas sub-view for specific heavy visuals only**
   - Feasible for isolated accelerated widgets (e.g., future minimap/heatmap/preview).
   - Not a drop-in accelerator for entire existing TreeSheets UI.

### 2.3 Low feasibility / high risk under your constraints

7. **Full-app OpenGL ES migration of current wxWidgets drawing model**
   - Requires substantial renderer rewrite (scene graph, text atlas, hit-testing parity).
   - High maintenance burden.

8. **OpenCL acceleration for current UI drawing path**
   - Not practical for dominant current bottlenecks (UI text/line draw and interaction).
   - OpenCL is more suitable for heavy compute kernels; TreeSheets workload is mostly UI-bound.

9. **Global Xorg Glamor tuning as primary strategy**
   - Usually unstable/low-benefit for tiling WM + mostly 2D text-heavy apps.
   - Better to optimize app-level invalidation and CPU drawing efficiency first.

## 3) What from the “possible stack” is actually worth integrating into TreeSheets

### 3.1 Worth integrating now

- RK3588-specific optimized build presets in CMake.
- Profiling instrumentation toggles (Tracy/perf markers or lightweight internal timing macros).
- Dirty-region repaint strategy.
- Font/text measurement caching for repeated draws.
- Optional runtime flags for high-refresh interaction behavior.

### 3.2 Worth integrating later (if benchmarks justify)

- Hybrid rendering experiments:
  - keep main UI with wxDC,
  - optional GL surface for narrowly scoped features.

### 3.3 Not recommended for near-term TreeSheets fork goals

- Porting TreeSheets from wxWidgets to Qt/QML just for RK3588 performance.
- End-to-end OpenCL/GLES interop pipeline for the existing document renderer.
- System-wide X stack mutation that can destabilize Ratpoison session.

## 4) Practical fork roadmap for maximum performance on RK3588

### Phase A (1–2 weeks): measurable wins with low risk

1. Add `TREE_SHEETS_RK3588_OPT` CMake option:
   - ARM CPU tuning flags.
   - Linker tuning (`-Wl,--as-needed`, optional mold/lld if available).
2. Add fine-grained repaint policy:
   - replace common full refreshes in motion/selection edits with bounded regions.
3. Add counters/timers around:
   - `Document::Draw`, `Document::Render`, `Grid::Render`, `Text::Render`.
4. Benchmark on your dual-monitor setup with representative large sheets.
5. Add reproducible `CMakePresets.json` profile:
   - `rk3588-bullseye-release` for local + CI parity.
   - Use this preset from CI workflow to avoid duplicated flag definitions.

### Phase B (2–4 weeks): deeper UI performance

5. Text rendering cache strategy (glyph extents/line layout reuse).
6. Optional frame pacing / coalesced update timer for drag/scroll bursts.
7. Optional “performance mode” runtime switch to reduce expensive visual effects.

### Phase C (experimental)

8. Prototype wxGLCanvas-backed auxiliary viewport.
9. Keep strict feature flag and fallback path; merge only if objective FPS/latency gains are sustained.

## 5) Final verdict (grounded to TreeSheets as it is)

Given TreeSheets’ current architecture, **the highest-feasibility path to maximize performance on RK3588 + Debian 11 + X11/Ratpoison is CPU-path optimization and smarter repainting, not wholesale GPU pipeline replacement**.

You can integrate “as much as possible” from your explored options, but in practical priority order:

1. Build/toolchain tuning for ARM big.LITTLE.
2. Render invalidation + caching improvements.
3. Event-loop throttling/coalescing.
4. Optional targeted GL experiments only after profiling proves need.

This strategy is consistent with source reality and gives the best risk-adjusted outcome for your exact software/firmware constraints.
