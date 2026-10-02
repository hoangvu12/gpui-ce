# Pane GPUI CE maintenance branch

This branch retains GPUI CE, its workspace layout and all upstream licenses.
Maintainer: [hoangvu12](https://github.com/hoangvu12).
Consumer: [Pane #63](https://github.com/hoangvu12/pane/issues/63), under
[the launcher UI specification](https://github.com/hoangvu12/pane/issues/61).

## Source provenance and patch scope

- CE base: [gpui-ce/gpui-ce at 17d9c8e8fdb30a329d817ca06bff424e8e848f1a](https://github.com/gpui-ce/gpui-ce/commit/17d9c8e8fdb30a329d817ca06bff424e8e848f1a).
- Reference: [hoangvu12/zui at ec16c62b83caa58b61cd8039db4b5721bacf948f](https://github.com/hoangvu12/zui/blob/ec16c62b83caa58b61cd8039db4b5721bacf948f/crates/gpui_windows/src/directx_renderer.rs).
  Zui is a separate Zed GPUI lineage. Only its source-over alpha correction is
  adapted here; no Zui history, framework replacement or blur engine is imported.
- Both Windows renderer packages are Apache-2.0. Their existing LICENSE-APACHE
  files and CE attribution remain intact. The new regression is Apache-2.0,
  matching the surrounding renderer package.

In `create_blend_state` and `create_blend_state_for_path_sprite`, destination
alpha uses `INV_SRC_ALPHA` instead of `ONE`. RGB blending is unchanged.
The resulting alpha is `As + Ad * (1 - As)`, so two 50% layers retain 25% of
an external backdrop instead of saturating the surface alpha to 100%.
Path rasterization already uses source-over; subpixel text deliberately does
not write alpha. Those states are unchanged.

The CE baseline already has backdrop/content filter infrastructure and a shared
DirectX render/readback path. No additional in-window blur implementation is
needed for this correction. CE macOS still selects `NSVisualEffectMaterial::Selection`;
its native-material evaluation belongs to Pane's later platform ticket and is
not changed here. Neither source inspection nor these tests certifies desktop
blur. Pane's prototype also needed removal of obscuring panel shadows.

## Renderer-output regression

On Windows with a working Direct3D 11 adapter, from this fork checkout:

```powershell
$env:GPUI_ALPHA_OUTPUT_DIR = "$PWD/target/alpha-output"
cargo +1.98.1 test --locked -p gpui_ce_windows --features test-support --lib translucent_ -- --nocapture
```

The existing hidden-window renderer fixture renders to the actual swapchain and
copies the GPU texture into a staging resource for RGBA readback. It does not
show a window, acquire focus, simulate input or capture the desktop. Failures to
create a device are test failures, not skipped passes.

The tests independently exercise quad-on-quad and path-on-quad output, including
path rasterization and path-sprite composition. They check clear, single-layer
and overlapping pixels; the overlap must be approximately premultiplied RGBA
`[64, 0, 128, 191]`. Tests also composite the measured pixel on green to check
that the remaining backdrop contribution is 64. An optional output directory
receives actual RGBA readbacks and black/green comparison previews even when an
assertion fails. These previews illustrate alpha, not native compositor blur.

For a negative control, revert only the two production destination-alpha edits
in an isolated fork checkout, leaving the tests intact. Both tests must fail
with overlap alpha 254/255 (8-bit rounding). Restoring only the ordinary blend correction must leave
the path test failing; restoring both must pass. Run the full Windows renderer
library tests after restoring the patch.

## Updating or retiring

Keep changes on `pane/source-over-alpha` and publish immutable commits. Do not
force-update a commit used by Pane. Rebase/cherry-pick onto a reviewed CE base
in a new branch, audit the small diff and run the positive and negative output
regressions. Submit the narrow fix upstream when ready. Once CE contains the
same correction and the regression passes there, Pane can return to an upstream
commit after its launcher suites and native Windows checks pass.

Move every Pane GPUI dependency together (renderer, platform and editable
controls, including dev dependencies), update its Cargo.lock and source allow
list, and confirm only one gpui-ce package identity. Do not copy a platform crate
into Pane or edit a Cargo cache to make a build pass.

## Recorded native GPU result (2026-10-02)

Windows 11 Pro 10.0.26200 x86_64, NVIDIA GeForce RTX 5050 driver
32.0.16.1074, Rust 1.98.1: both uncorrected cases failed with overlap RGBA
`[64, 0, 127, 254]`; ordinary-only correction passed the quad case and failed
the path case; both corrections passed with `[64, 0, 127, 191]`. All 15 Windows
renderer library tests passed. The initial command without `--features test-support`
could not compile the baseline test harness; the commands above include it.
No native macOS/Linux or desktop material claim follows from this result.
