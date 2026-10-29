# Linux/Unix Graphics Output and Windowing Architecture

## Preserve the working rendering paths

Preserve the current native Wayland/GPU presentation work, ca2 event-based camera
input, tap/hold tuning, bitmap publishing, icon history, and lifecycle fixes.
Restoring or improving off-screen rendering must coexist with that work. Do not
globally replace a working backend or force all applications into swap-chain mode.

The following separates the intended selection policy from what is currently
implemented. This document does not itself change runtime defaults or overrides.

## Settings and default behavior

- `graphics3d_output.txt` selects output mode. `off_screen` is the default when
  unset or unrecognized. `on_screen` explicitly enables window swap-chain output.
- The runtime also accepts the legacy `swap_chain` value as on-screen output.
  Prefer `on_screen` for new settings; the Bred options code does not currently
  now normalizes the legacy alias consistently with the Aura runtime.
- `graphics3d.txt` selects the GPU API, such as `opengl` or `vulkan`; it does not
  select on-screen versus off-screen output.
- `draw2d.txt` selects the 2D implementation. GPU API and draw2d settings can be
  operating-system-specific through `application_file_setting_by_operating_system`.
- Legacy `windowing.txt` is deactivated by renaming it to `aaa_windowing.txt`
  (numbered backups preserve any existing file).
- Linux raw overrides use `forced_raw_windowing.txt`, with
  `forced_raw_windowing_for_on_screen.txt` and
  `forced_raw_windowing_for_off_screen.txt` taking precedence for their modes.
  An absent or empty specific setting falls back to the general setting. Valid
  values are `wayland`, `x11`, and `xcb` (also accepting windowing-name aliases).
- Preserve the normal settings lookup: application cloud settings take precedence
  over the application roaming settings. On this Linux setup roaming settings are
  under `~/.config/<application-id>/`. Windowing also has an appconfig fallback.

## Intended off-screen policy

For `off_screen`, or no output setting, Linux should normally use `draw2d_cairo`
and the desktop's supported toolkit windowing/ambient stack. Off-screen 3D can
still be GPU-rendered: its result is supplied as an image/texture for composition
and presentation by the ordinary application window. Off-screen does not mean
that the 3D scene must be rendered on the CPU.

The available version-matched toolkit families are:

| Desktop/toolkit family | Windowing | Acme windowing | Operating ambient |
| --- | --- | --- | --- |
| KDE6 | `windowing_kde6` | `acme_windowing_kde6` | `operating_ambient_kde6` |
| KDE5 | `windowing_kde5` | `acme_windowing_kde5` | `operating_ambient_kde5` |
| GNOME/GTK3 | `windowing_gtk3` | `acme_windowing_gtk3` | `operating_ambient_gtk3` |
| GNOME/GTK4 | `windowing_gtk4` | `acme_windowing_gtk4` | `operating_ambient_gtk4` |

Apply the same principle to other supported desktop/toolkit families. Current
desktop detection uses the desktop family together with build capabilities such
as `HAS_KDE6` and `HAS_GTK4`; the desktop name alone does not select every toolkit
version.

The user's KDE5 example explicitly named
`windowing_kde5/acme_windowing_kde6/operating_ambient_kde6`. The current selector
uses one toolkit ID for these factories and therefore normally selects the KDE5
chain shown above. The mixed-version example is a compatibility detail to resolve,
not verified current behavior. Preserve that distinction in future changes.

## Intended on-screen policy

On Linux/Unix, explicit `on_screen` output should normally use:

| Native session/protocol | Windowing | Acme windowing | Operating ambient |
| --- | --- | --- | --- |
| X11 | `windowing_x11` | `acme_windowing_x11` | `operating_ambient_x11` |
| Wayland | `windowing_wayland` | `acme_windowing_wayland` | `operating_ambient_wayland` |

Use compatible GPU draw2d and 3D implementations for swap-chain composition:
OpenGL uses `draw2d_opengl` or the supported `draw2d_nanovg` option; Vulkan uses
`draw2d_vulkan` or `draw2d_vkvg`. Cairo is not the compositor for this GPU-owned
window swap-chain path. This policy does not rule out future toolkit integrations
that explicitly support the necessary native surface and presentation ownership.

Keep native windowing GPU-agnostic. It supplies native surfaces, configuration,
events, and generic presentation-ownership provisions. GPU backends own EGL,
OpenGL/Vulkan resources and swap-chain presentation. In particular, `gpu_opengl`
owns `wl_egl_window` and EGL objects; native Wayland does not depend on Qt or EGL.

## Current implementation and selection gaps

Verified in this workspace on 2026-10-28:

- Aura graphics initialization defaults swap-chain output to false and enables it
  only for `on_screen` or `swap_chain`. Bred lists `off_screen` first as its fallback.
- Linux draw2d falls back to Cairo for ordinary/off-screen output. Explicit draw2d
  overrides can still select another implementation. On-screen selection chooses
  a GPU implementation compatible with the GPU API.
- Linux selects the desktop toolkit for off-screen output and automatically
  selects raw Wayland or X11 for on-screen output. A valid forced raw setting
  can select raw windowing in either mode. The most specific setting for the
  current mode takes precedence over the general setting, even if invalid; an
  invalid override is ignored and the automatic choice is used.
- XCB uses its own windowing and Acme windowing factories with the X11 ambient.
- Legacy `windowing.txt` settings are archived in cloud, roaming and appconfig
  locations and are never promoted into the new forced setting.
- These raw overrides apply only to Linux desktop builds. Other systems retain
  desktop toolkit selection.

When changing selection, account for initialization order and explicit overrides
centrally. Do not reinterpret `graphics3d.txt` as an output setting.

## Change and verification rules

- Test off-screen/Cairo/toolkit presentation independently from on-screen/GPU/raw
  presentation. Preserve both modes and the unset-setting off-screen default.
- Skip redundant CPU image presentation when a GPU swap chain owns presentation.
  Keep explicit bitmap-source publishing, screenshots, and image mapping working
  through their proper leases and synchronization.
- Apply hide/close requests even when no further visible frame will be rendered.
  A presentation wait must not prevent native hiding and application teardown.
- Keep camera input driven by ca2 messages and monotonic elapsed time. Preserve
  configurable tap distance, hold delay, and continuous speed across backends.
- Report which operating system, windowing backend, GPU API, and output mode were
  actually tested. Local Wayland/OpenGL results do not verify every platform/API.

## Source entry points

- `source/app/aura/platform/application.cpp`: early graphics configuration and
  draw2d selection.
- `source/app/bred/platform/application.cpp`: output/API options and fallbacks.
- `source/app/acme/platform/system.cpp`: ambient/toolkit selection and Acme factories.
- `source/app/aura/platform/system.cpp`: draw2d fallback and ambient/windowing factories.
- `source/app/acme/operating_system/__eoperating_ambient.cpp`: desktop-to-toolkit mapping.
- `source/app/acme/platform/application.cpp`: cloud/roaming settings lookup.
- `operating_system/operating_system-posix/acme_windowing_wayland/native/` and
  `operating_system/operating_system-posix/windowing_wayland/native/`: native
  transport and Aura adapter.
- `source/app/gpu_opengl/`: OpenGL/EGL presentation integration.
