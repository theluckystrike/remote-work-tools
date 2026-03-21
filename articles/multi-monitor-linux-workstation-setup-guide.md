---
layout: default
title: "Multi-Monitor Linux Workstation Setup Guide"
description: "Configure a multi-monitor Linux workstation for remote development. Covers xrandr, Wayland, i3, per-monitor DPI, and workspace assignment across displays."
date: 2026-03-21
author: theluckystrike
permalink: /multi-monitor-linux-workstation-setup-guide/
categories: [guides]
reviewed: true
score: 8
intent-checked: true
voice-checked: true
tags: [remote-work-tools]
---

{% raw %}

A multi-monitor Linux workstation requires configuration that Windows and macOS handle automatically. The payoff is total control: workspaces pinned to specific monitors, per-display scaling, hotkey-driven layout switching, and no proprietary drivers needed for most hardware.

This guide covers the complete setup: detecting displays, configuring layouts, handling mixed DPI, setting up workspace assignment in i3/Sway, and persisting everything across reboots.

## Check What Linux Sees

```bash
# X11 systems
xrandr --query

# Sample output:
# Screen 0: minimum 320x200, current 5120x1440, maximum 16384x16384
# HDMI-1 connected primary 2560x1440+0+0 (normal left inverted right) 600mm x 340mm
# DP-1 connected 2560x1440+2560+0 (normal left inverted right) 600mm x 340mm
# HDMI-2 disconnected

# Wayland systems
wlr-randr          # if using wlroots compositor (Sway, Hyprland)
# or
kscreen-doctor -o  # KDE Plasma
gnome-randr        # GNOME
```

The `+0+0` and `+2560+0` are the offsets — they tell you which monitor is to the left or right. A display at `+2560+0` starts at pixel 2560 horizontally, placing it directly to the right of a 2560-wide primary display.

## Set Display Layout with xrandr

```bash
# Two monitors side by side — HDMI-1 primary, DP-1 to the right
xrandr --output HDMI-1 --primary --mode 2560x1440 --rate 144 --pos 0x0 \
       --output DP-1 --mode 2560x1440 --rate 60 --right-of HDMI-1

# Three monitors — center primary, left and right flanking
xrandr --output DP-2 --mode 1920x1080 --rate 60 --pos 0x0 \
       --output HDMI-1 --primary --mode 2560x1440 --rate 144 --right-of DP-2 \
       --output DP-1 --mode 1920x1080 --rate 60 --right-of HDMI-1

# Stacked vertical layout
xrandr --output HDMI-1 --primary --mode 2560x1440 --rate 144 --pos 0x0 \
       --output DP-1 --mode 1920x1080 --pos 320x1440

# Turn off a display
xrandr --output HDMI-2 --off
```

## Persist Layout with autorandr

`xrandr` commands reset on reboot. `autorandr` saves named profiles and auto-applies them when the same monitors are connected.

```bash
# Install
sudo apt install autorandr    # Debian/Ubuntu
sudo pacman -S autorandr      # Arch

# Configure your layout with xrandr first, then save it
autorandr --save office-dual

# List saved profiles
autorandr --list

# Manually apply a profile
autorandr office-dual

# Auto-detect and apply the matching profile (run in .profile or .xinitrc)
autorandr --change
```

Add `autorandr --change` to your session startup so it fires whenever monitors change — docking/undocking works automatically.

## Handle Mixed DPI (HiDPI + 1080p)

The most common setup: a 4K laptop screen at 200% scaling plus a 1080p external at 100%. This is the hardest multi-monitor problem on Linux.

**X11 (limited, partial fix):**

```bash
# Set global DPI (affects all displays equally — not ideal for mixed)
xrandr --dpi 144

# Workaround: use xrandr --scale to fake per-monitor DPI
# Scale the 1080p monitor up to match the HiDPI environment
xrandr --output DP-1 --scale 2x2 --mode 1920x1080 --fb 7680x2160

# This makes the 1080p monitor "virtually" 4K while keeping crisp text on the HiDPI display
# The --fb flag sets the virtual framebuffer size to accommodate both
```

**Wayland (clean per-monitor scaling):**

Wayland handles per-monitor scaling natively. Use Sway or Hyprland for the cleanest result.

```bash
# Sway config: ~/.config/sway/config
output HDMI-A-1 resolution 3840x2160 scale 2
output DP-1 resolution 1920x1080 scale 1

# Position them correctly
output HDMI-A-1 pos 0 0
output DP-1 pos 1920 0
```

## i3 Config: Assign Workspaces to Monitors

i3's workspace-to-output binding keeps code on one monitor and browser on another permanently.

```bash
# ~/.config/i3/config

# Name your outputs (get names from xrandr --query)
set $mon1 HDMI-1
set $mon2 DP-1

# Assign workspaces to outputs
workspace 1 output $mon1
workspace 2 output $mon1
workspace 3 output $mon1
workspace 4 output $mon1
workspace 5 output $mon2
workspace 6 output $mon2
workspace 7 output $mon2
workspace 8 output $mon2

# Move focused container to specific monitor
bindsym $mod+Shift+Left move container to output left
bindsym $mod+Shift+Right move container to output right

# Focus monitor
bindsym $mod+Left focus output left
bindsym $mod+Right focus output right
```

Workspaces 1-4 open on the primary monitor, 5-8 on the secondary. Browser starts on workspace 5 (secondary). Terminal and editor on workspace 1 (primary). No manual dragging.

## Sway Config for Wayland

```bash
# ~/.config/sway/config

# Output configuration
output HDMI-A-1 {
    resolution 2560x1440
    position 0 0
    refresh_rate 144
    scale 1
}
output DP-1 {
    resolution 1920x1080
    position 2560 0
    refresh_rate 60
    scale 1
}

# Workspace-to-output binding
workspace 1 output HDMI-A-1
workspace 2 output HDMI-A-1
workspace 3 output HDMI-A-1
workspace 5 output DP-1
workspace 6 output DP-1
workspace 7 output DP-1
```

## Font Rendering at Mixed DPI

```bash
# ~/.config/fontconfig/fonts.conf
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "fonts.dtd">
<fontconfig>
  <match target="font">
    <edit name="antialias" mode="assign">
      <bool>true</bool>
    </edit>
    <edit name="hinting" mode="assign">
      <bool>true</bool>
    </edit>
    <edit name="hintstyle" mode="assign">
      <const>hintslight</const>
    </edit>
    <edit name="rgba" mode="assign">
      <const>rgb</const>
    </edit>
    <edit name="lcdfilter" mode="assign">
      <const>lcddefault</const>
    </edit>
  </match>
</fontconfig>
```

```bash
# Rebuild font cache
fc-cache -fv

# Set DPI for X11 apps via .Xresources
echo "Xft.dpi: 144" >> ~/.Xresources
xrdb -merge ~/.Xresources
```

## Wallpaper Per Monitor

```bash
# feh — sets different wallpaper per display (X11)
sudo apt install feh
feh --bg-fill /path/to/left.jpg --bg-fill /path/to/right.jpg

# Add to i3 config to restore on session start
exec_always --no-startup-id feh --bg-fill /path/to/left.jpg --bg-fill /path/to/right.jpg

# Sway — built-in per-output background
output HDMI-A-1 background /path/to/left.jpg fill
output DP-1 background /path/to/right.jpg fill
```

## Status Bar Per Monitor

i3bar and Waybar can show a bar on every monitor:

```bash
# i3 config — two bars, one per output
bar {
    output HDMI-1
    status_command i3status
    position top
}

bar {
    output DP-1
    status_command i3status
    position top
    # Simpler bar on secondary — no clock/battery
    mode hide
    hidden_state hide
    modifier Mod4
}
```

## Test Layout Without Committing

```bash
# X11: test a layout change, it reverts on next login
xrandr --output HDMI-1 --mode 1920x1080

# Confirm it works before saving with autorandr
autorandr --save test-layout

# Revert to previous profile if something breaks
autorandr previous
```

## Related Reading

- [How to Set Up a Linux Workstation for Remote Work](/remote-work-tools/how-to-set-up-linux-workstation-for-remote-work/)
- [Best Ultrawide Monitor for Programming Remote Work](/remote-work-tools/best-ultrawide-monitor-for-programming-remote-work/)
- [Ergonomic Desk Setup Guide for Developers 2026](/remote-work-tools/ergonomic-desk-setup-developers-2026/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
