---
id: F-os-input-private-display
title: "Opt-in private X display per editor (Xvfb/Xephyr) so os_input never shares a pointer or stacking order with peers, with a hard refusal when the RHI falls back to a software Vulkan device"
status: IN-REVIEW
severity: Medium
category: feature
tags: [editor_start, os-input, xtest, x11, linux, xvfb, xephyr, shared-display, vulkan, lavapipe]
encounters: 1
costly: 0
lastSeen: 2026-09-30T12:00:00Z
---

# One X display per editor, opt-in and self-checking

UE 5.8 Linux, plugin `2580e7f4`. Several agents on one desktop share `:0`: one pointer, one
stacking order, one keyboard focus. The ownership gate (`B-drive-click-os-input-foreign-x-window`)
refuses when a peer's window covers the target, and `B-os-input-shared-pointer-race` asks for
serialization. Both leave the caller waiting while a peer is on top, and raising your own
window only moves the problem onto the peer. A private X server per editor removes the sharing:
its own root, pointer, stacking order and XTEST stream. The human's real mouse is untouched too.

**Why opt-in only:** rendering on a non-vendor X server is driver-dependent and unverified:
- NVIDIA proprietary: Vulkan presenting to a window on Xvfb/Xephyr (no NVIDIA GLX, no DRI3) is
  unverified for any driver version.
- Mesa (Intel/AMD): Xvfb/Xephyr lack DRI3; whether Mesa's X11 WSI has a working non-DRI3 path
  for the hardware drivers is unverified. The failure mode can be a silent drop to lavapipe.
- Windows has no equivalent (one input desktop per session; IddCx virtual monitors join the
  same desktop). Out of scope.

**Ask:**
1. `editor_start` (mode `visible`, Linux) gets an opt-in, e.g. `display:"private"`. The proxy
   (`Content/Python/mcp_proxy.py`, `_visible_launch_env()`) starts `Xvfb :N` (or `Xephyr :N`
   when the user wants to watch), picking a free N with `-displayfd`, sets `DISPLAY=:N` for the
   editor, and owns the X server's lifetime (killed when that editor exits).
2. After startup, read the Vulkan device the RHI picked from the editor log. If it is a CPU
   device (lavapipe / llvmpipe / `VK_PHYSICAL_DEVICE_TYPE_CPU`), stop the editor and refuse
   with an error naming the device and the display, instead of handing back a software-rendered
   editor.
3. Report the display in the `editor_start` / `editor_list` result, so raw `xdotool` fallbacks
   can target `DISPLAY=:N`.
4. Wiki: a compatibility table of confirmed GPU/driver/X-server combinations, starting empty,
   filled only from live runs.
5. Check that `Xvfb`/`Xephyr` exist up front; a missing binary is a clear refusal with the
   package name, not a spawn failure.

Also unblocks os_input for unattended runs: today offscreen editors have no X window at all
(`E-os-input-headless-editor-undocumented`).

## History
- `#1-filed-design` `OPEN` reporter - Filed from a design discussion on the wrong-window clicks. Host has `/usr/bin/Xephyr`, no Xvfb, RTX 3060 Ti on driver 580.126.09; the NVIDIA-on-Xephyr probe (`Xephyr :42 &` then `DISPLAY=:42 vulkaninfo --summary`) was not run. Must not become the default: an open-source user's GPU/driver may only render in software there.
- `#2-proxy-private-display` `IN-REVIEW` developer — `editor_start` takes `display: "desktop"|"xvfb"|"xephyr"` (default desktop; Linux + mode visible only, else `INVALID_DISPLAY`). Proxy side only, no plugin C++ needed (`Content/Python/mcp_proxy.py`): `start_private_display` checks the binary up front (`PRIVATE_DISPLAY_UNAVAILABLE` naming the `xvfb` / `xserver-xephyr` package), starts it with `-displayfd` (free `:N`) and `-terminate 10` (exits 10 s after its last client, so it dies with the detached editor even after the proxy exits, and a startup connect/close does not end it), `PRIVATE_DISPLAY_FAILED` when no display number arrives in 10 s; the editor gets `DISPLAY=:N`, `WAYLAND_DISPLAY` removed, `XDG_SESSION_TYPE=x11`, `SDL_VIDEODRIVER=x11`, and its own `-Abslog` when none was passed. After readiness `_check_private_display` reads the Vulkan device from that log (`rhiDevice`/`rhiDeviceType`); a CPU device (`VK_PHYSICAL_DEVICE_TYPE_CPU`, llvmpipe/lavapipe/softpipe/SwiftShader) stops the editor and the server and refuses `PRIVATE_DISPLAY_SOFTWARE_RHI` naming device and display. Results carry `display`/`displayServer`; `editor_list` rows carry `display` read from `/proc/<pid>/environ`. Note UE 5.8's `SelectPhysicalDevice` already skips CPU devices without `-AllowCPUDevices`, so the likelier failure on a non-DRI3 server is RHI init failing (exit or an invisible modal -> `EDITOR_START_TIMEOUT`, documented with the display for screenshots). Wiki `docs/wiki-src/mcp-transport.md`: contract plus the empty compatibility table. Live on this box: no Xvfb (`which Xvfb` empty), `/usr/bin/Xephyr` present; probed Xephyr only (64x64 screen): `-displayfd` gave `:1`, `xdotool` XTEST worked on it, server exited 10.2 s after the last client and survived a reconnect inside the window. No editor was started on a private display (shared box, editor slot owned by the manager), so the NVIDIA 580 Vulkan-on-Xephyr question stays open and the table stays empty. Not done: `editor_restart` / `editor_run_tests` do not take `display`. Tests: `Content/Python/tests/test_mcp_proxy_private_display.py` (12).
