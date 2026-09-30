---
id: E-os-input-wayland-unverified
title: "os_input under a Wayland session (editor on XWayland) passes IsAvailable because DISPLAY is set, but whether XTEST events reach the editor there is unverified and nothing warns"
status: OPEN
severity: Medium
category: ergonomic
tags: [drive, os-input, xtest, wayland, xwayland, linux, gnome, platform-support]
encounters: 1
costly: 0
lastSeen: 2026-09-30T12:00:00Z
---

# Wayland sessions are neither detected nor supported explicitly

UE 5.8 Linux, plugin `2580e7f4`. `FDriveOsInput::IsAvailable`
(`Source/PinWright/Private/Handlers/Drive/DriveOsInput.cpp`) only checks that libX11/libXtst load
and that `XOpenDisplay(NULL)` succeeds. On a Wayland desktop (Ubuntu's GNOME default, Fedora,
KDE Plasma 6) the editor usually runs through XWayland, so `DISPLAY` is set and the check
passes. Whether XTEST motion/button events then reach the editor is unverified: XWayland
routes XTEST through libei and the portal on newer versions, and pointer confinement and
relative mode (the reasons os_input exists) are compositor-mediated there. The
`FindForeignWindowAt` ownership gate is also blind to native Wayland windows stacked above the
editor. So os_input can quietly do nothing, and the report is `no_change_within_budget`.

This host is an X11 session, so it was not reproduced. For an open-source plugin, Wayland is
likely the most common Linux desktop.

**Ask:**
1. Detect a Wayland session (`WAYLAND_DISPLAY` set or `XDG_SESSION_TYPE=wayland` in the
   editor's environment). Add a field such as `session:"wayland"` to os_input results, and a
   warning that real-input delivery is unverified there. Or refuse up front until verified.
2. Verify live on one GNOME and one KDE Wayland session; record the outcome in the drive wiki.
3. If it does not work, document the options: log in to an X11 session, or use a private
   X display (`F-os-input-private-display`).

## History
- `#1-filed-platform-gap` `OPEN` reporter - Filed from the multi-editor wrong-window discussion; code reading at `2580e7f4`, not reproduced (host session is X11 on `:0`).
