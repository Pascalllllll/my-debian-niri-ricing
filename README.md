# Debian + niri + DankMaterialShell (next to GNOME)

A second desktop session on the same Debian install, built for ricing: **niri**, a scrollable-tiling Wayland compositor, with **DankMaterialShell (DMS)** for the bar, launcher, notifications, lock screen, wallpaper picker, and wallpaper-based colour theming. GNOME stays installed as a fallback. At the login screen you pick which one to start.

Both come as `.deb` packages from the DankLinux repositories, so nothing has to be compiled and `apt upgrade` keeps them current.

**Target:** Debian 14 "forky" (testing), GNOME 50 already installed, Intel Arrow Lake iGPU + NVIDIA RTX 5060 Mobile (hybrid).

---

## Contents
1. [How the two sessions share one system](#1-how-the-two-sessions-share-one-system)
2. [Install niri and DMS](#2-install-niri-and-dms)
3. [Start DMS only in niri](#3-start-dms-only-in-niri)
4. [NVIDIA and hybrid graphics](#4-nvidia-and-hybrid-graphics)
5. [Generate the config](#5-generate-the-config)
6. [Edit before first login](#6-edit-before-first-login)
7. [First login](#7-first-login)
8. [Ricing with DMS](#8-ricing-with-dms)
9. [Key bindings](#9-key-bindings)
10. [Troubleshooting](#10-troubleshooting)
11. [Removing niri and DMS](#11-removing-niri-and-dms)
12. [How long this stays accurate](#12-how-long-this-stays-accurate)

All commands in this guide work in both fish and bash. They avoid bash-only syntax such as `<<'EOF'` heredocs, which fish does not support.

---

## 1. How the two sessions share one system

GNOME and niri are both Wayland sessions. GDM lists every session file in `/usr/share/wayland-sessions/`, so once niri is installed it appears next to GNOME.

| Shared between both sessions | Separate per session |
|---|---|
| Home folder, apps, Flatpaks | Window management and shortcuts |
| `~/.config/gtk-3.0`, `~/.config/gtk-4.0` (app themes) | Bar and panels: GNOME Shell vs DMS |
| dconf / `gsettings` values | GNOME extensions (do not run in niri) |
| fish, Starship, fastfetch, fonts | Wallpaper (GNOME setting vs DMS) |
| GDM login screen and keyring | Notifications, lock screen, idle |

Two rules keep GNOME working:
- **DMS must not start inside GNOME.** Section 3 ties it to the niri session.
- **Decide whether DMS may theme GTK apps.** Its GTK colours land in the shared folders above and also change apps in GNOME. Section 8 covers the choice.

**Keep GDM.** DMS offers its own login screen (`dms-greeter`, based on greetd). Two display managers fight over the login screen, and GDM already starts niri.

---

## 2. Install niri and DMS

### Repositories
DMS lives in two repositories on openSUSE's build service: `danklinux` (niri, matugen, dgop, and other companions) and `dms` (the shell). Both have a `Debian_Testing` build.

```bash
sudo install -d -m 0755 /etc/apt/keyrings

# niri and companions
curl -fsSL https://download.opensuse.org/repositories/home:AvengeMedia:danklinux/Debian_Testing/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/danklinux.gpg
echo "deb [signed-by=/etc/apt/keyrings/danklinux.gpg] https://download.opensuse.org/repositories/home:/AvengeMedia:/danklinux/Debian_Testing/ /" | \
  sudo tee /etc/apt/sources.list.d/danklinux.list

# DMS
curl -fsSL https://download.opensuse.org/repositories/home:/AvengeMedia:/dms/Debian_Testing/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/avengemedia-dms.gpg
echo "deb [signed-by=/etc/apt/keyrings/avengemedia-dms.gpg] https://download.opensuse.org/repositories/home:/AvengeMedia:/dms/Debian_Testing/ /" | \
  sudo tee /etc/apt/sources.list.d/avengemedia-dms.list

sudo apt update
```
`signed-by` limits each key to its own repository, so these keys cannot sign packages that claim to come from Debian.

### Take quickshell from Debian
The DankLinux repository still carries its own, deprecated `quickshell` build. Its version number is higher than Debian's, so apt picks it over Debian's package unless told otherwise. Block it with a pin:
```bash
printf '%s\n' 'Package: quickshell quickshell-git' 'Pin: origin download.opensuse.org' 'Pin-Priority: -1' | \
  sudo tee /etc/apt/preferences.d/quickshell-from-debian
sudo apt update
apt policy quickshell
```
`Pin-Priority: -1` means "never install from this source". The pin matches the host written in the sources file, so it also covers the mirrors openSUSE redirects to. `apt policy` should now show a Candidate from `deb.debian.org`.

### Packages
```bash
sudo apt install quickshell
sudo apt install niri dms kitty \
  xdg-desktop-portal-gnome xdg-desktop-portal-gtk
```
| Package | Why |
|---|---|
| `quickshell` | The framework DMS is built on. Debian testing ships it; DankLinux's own build for Debian is deprecated. |
| `niri` | The compositor. Pulls in `xwayland-satellite` for X11 apps (Steam, some games). |
| `dms` | The shell and the `dms` command. Pulls in `matugen` (colour generation), `dgop` (system stats), `danksearch` (file search in the launcher), `cava` (audio visualiser), and `qt6ct`. |
| `kitty` | Terminal. DMS ships a kitty config that follows the wallpaper colours. |
| `xdg-desktop-portal-gnome`, `-gtk` | Screen sharing, file pickers, and screenshots for apps; niri's portal config asks for these. |

Check the niri version:
```bash
niri --version
```
It should be 26.04 or newer. Window and layer blur (`background-effect`) does not exist in older versions.

---

## 3. Start DMS only in niri

The `dms` package installs a systemd user service whose `[Install]` section says `WantedBy=graphical-session.target`. GNOME also reaches that target. If you run the usual `systemctl --user enable dms`, DMS starts inside GNOME too: a second bar appears, and DMS takes the notification service name GNOME Shell needs.

On Debian the package goes further: its install script enables the service for every user at once, through a link in `/etc/systemd/user/graphical-session.target.wants/`. Remove that global link, then tie DMS to niri:
```bash
sudo systemctl --global disable dms
systemctl --user add-wants niri.service dms
find ~/.config/systemd/user /etc/systemd/user -name dms.service
```
The last command should list exactly one link, the one in `niri.service.wants`. This starts DMS whenever niri starts, and never in GNOME. A `dms` package upgrade may recreate the global link; repeat the check after upgrading. Do not also add `spawn-at-startup "dms" "run"` to the niri config; DMS would run twice.

For the same reason, put environment variables in the niri config (section 6), not in `~/.config/environment.d/`. Files there apply to every session, GNOME included.

---

## 4. NVIDIA and hybrid graphics

This laptop has two GPUs: the Intel iGPU drives the built-in screen, and the RTX 5060 renders games on request (PRIME offload). niri draws the desktop on the Intel GPU by default. Leave it that way; it is the stable and battery-friendly setup.

### VRAM fix
The NVIDIA driver keeps freed video memory instead of returning it, which makes niri use close to 1 GiB of VRAM instead of about 100 MiB. This matters whenever the NVIDIA GPU is involved, for example with an external monitor on a port wired to it. niri's documentation gives this per-process driver profile:
```bash
sudo mkdir -p /etc/nvidia/nvidia-application-profiles-rc.d
echo '{"rules":[{"pattern":{"feature":"procname","matches":"niri"},"profile":"Limit Free Buffer Pool On Wayland Compositors"}],"profiles":[{"name":"Limit Free Buffer Pool On Wayland Compositors","settings":[{"key":"GLVidHeapReuseRatio","value":0}]}]}' | \
  sudo tee /etc/nvidia/nvidia-application-profiles-rc.d/50-limit-free-buffer-pool-in-wayland-compositors.json
python3 -m json.tool /etc/nvidia/nvidia-application-profiles-rc.d/50-limit-free-buffer-pool-in-wayland-compositors.json
```
The last command prints the JSON back if it is valid. The kernel line from the main README already has `nvidia-drm.modeset=1`, which niri also needs.

### Running games on the NVIDIA GPU
Same as in GNOME. For one command:
```fish
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia <command>
```
In Steam, put this in a game's *Launch Options*:
```
__NV_PRIME_RENDER_OFFLOAD=1 __GLX_VENDOR_LIBRARY_NAME=nvidia %command%
```

---

## 5. Generate the config

Do this before logging in to niri for the first time, from a terminal in GNOME:
```bash
dms setup headless --compositor niri --terminal kitty --skip-existing
```
- `headless` skips the interactive questions.
- `--terminal kitty` deploys a kitty config and makes Mod+T open kitty.
- `--skip-existing` leaves any config you already have untouched.

It writes:

| File | Who edits it |
|---|---|
| `~/.config/niri/config.kdl` | You. Input, environment, animations, window rules. Ends with `include` lines for the files below. |
| `~/.config/niri/dms/binds.kdl` | You, or DMS Settings > Input > Keyboard shortcuts. Key bindings. |
| `~/.config/niri/dms/layout.kdl`, `colors.kdl` | DMS only. Marked "DO NOT EDIT"; DMS rewrites them from the **Niri** page in Settings and from the wallpaper colours. |
| `~/.config/niri/dms/outputs.kdl`, `cursor.kdl`, `input.kdl`, `alttab.kdl` | DMS Settings (Displays, Cursor, and so on). |

Changes you make by hand in the "DO NOT EDIT" files are lost on the next theme change. Put your own layout tweaks in `config.kdl` instead.

---

## 6. Edit before first login

### Environment
Open `~/.config/niri/config.kdl` and extend the `environment` block:
```kdl
environment {
    XDG_CURRENT_DESKTOP "niri"
    QT_QPA_PLATFORM "wayland;xcb"         // native Wayland, X11 as fallback
    QT_QPA_PLATFORMTHEME "gtk3"           // Qt apps follow the GTK colours DMS generates
    QT_QPA_PLATFORMTHEME_QT6 "gtk3"
    ELECTRON_OZONE_PLATFORM_HINT "auto"   // Chrome, VS Code, Discord run natively on Wayland
    SDL_VIDEODRIVER "wayland,x11"         // Minecraft needed SDL_VIDEODRIVER=x11 on this machine; this falls back to it
}
```

### Super+W for the wallpaper picker
DMS binds the wallpaper browser to Mod+Y and uses Mod+W for tabbed columns. To match the GNOME setup, where Super+W opens the wallpaper carousel, swap them in `~/.config/niri/dms/binds.kdl`:
```kdl
// was: Mod+W { toggle-column-tabbed-display; }
Mod+Alt+W { toggle-column-tabbed-display; }

// was: Mod+Y hotkey-overlay-title="Browse Wallpapers" { ... }
Mod+W hotkey-overlay-title="Browse Wallpapers" {
    spawn "dms" "ipc" "call" "dash" "toggle" "wallpaper";
}
```
Each key may appear only once across all included files; a duplicate makes the config invalid.

### Wallpapers
DMS reads wallpapers from a folder you choose in Settings. Use the same folder as HyprQuickPaper in GNOME, so both sessions share one collection:
```bash
mkdir -p ~/Pictures/Wallpapers
```

### Optional: Alt as the Mod key
On a 65% keyboard the Super key can sit somewhere awkward, for example right of the space bar. niri can use Alt as `Mod` instead. Every binding written as `Mod+...` then moves to Alt.

The cost: niri takes those Alt combinations before apps see them. Alt+Left / Alt+Right (back and forward in browsers), Alt+1 to Alt+9 (browser tabs), Alt+F (menus), and Alt+Up / Alt+Down (move line in VS Code) stop working inside apps. If your keyboard supports VIA or QMK, remapping a key in the keyboard firmware avoids this; the steps below are the software route.

**1. Switch the Mod key.** Add it to the existing `input` block in `config.kdl`:
```bash
sed -i '0,/^input {/s//input {\n    mod-key "Alt"\n    mod-key-nested "Super"/' ~/.config/niri/config.kdl
grep -A3 '^input {' ~/.config/niri/config.kdl
```

**2. Fix the bindings that now collide.** With Mod = Alt, some DMS defaults turn into the same key twice, which makes the config invalid:

| Default | Becomes | Change to | Reason |
|---|---|---|---|
| `Mod+Tab` (overview) | Alt+Tab | remove | Alt+Tab stays the window switcher from `recent-windows`; Mod+O still opens the overview |
| `Alt+Space` (spotlight bar) | same as Mod+Space | `Mod+Shift+Space` | Mod+Space keeps the launcher |
| `Mod+Alt+L` (lock) | Alt+L, same as focus right | `Super+L` | Same key as Windows; locking is rare, so the far Super key is fine |
| `Mod+Alt+W` (tabbed column) | Alt+W, same as wallpapers | `Mod+Ctrl+W` | Free combination |
| `Super+X` (power menu) | unchanged | `Mod+X` | Keeps it on the near key |

```bash
sed -i '/^\s*Mod+Tab repeat=false { toggle-overview; }/d' ~/.config/niri/dms/binds.kdl
sed -i 's/^\(\s*\)Alt+Space hotkey-overlay-title="Spotlight Bar"/\1Mod+Shift+Space hotkey-overlay-title="Spotlight Bar"/' ~/.config/niri/dms/binds.kdl
sed -i 's/^\(\s*\)Mod+Alt+L hotkey-overlay-title="Lock Screen"/\1Super+L hotkey-overlay-title="Lock Screen"/' ~/.config/niri/dms/binds.kdl
sed -i 's/^\(\s*\)Mod+Alt+W {/\1Mod+Ctrl+W {/' ~/.config/niri/dms/binds.kdl
sed -i 's/^\(\s*\)Super+X hotkey-overlay-title="Power Menu/\1Mod+X hotkey-overlay-title="Power Menu/' ~/.config/niri/dms/binds.kdl
grep -nE '^\s*(Mod\+Alt|Alt\+Space|Super\+X|Mod\+Tab )' ~/.config/niri/dms/binds.kdl
niri validate
```
The `grep` line should print nothing. If `niri validate` reports a duplicate key, the message names it; change one of the two the same way.

### Check the config
```bash
niri validate
```
It reports the file, line, and reason for any error. A config that fails to load leaves niri on its built-in defaults with an error banner, so fix errors before logging in.

---

## 7. First login

1. Log out of GNOME.
2. On the GDM screen, click your name, then the gear icon in the bottom-right corner.
3. Pick **Niri** and log in.

The DMS bar should appear within a few seconds. GDM remembers the last session; pick **GNOME** from the same menu to go back.

Two checks after the first login:
```bash
systemctl --user status dms     # should be "active (running)"
dms doctor                      # lists missing optional features and config problems
```

To use the same default apps as GNOME (which program opens PDFs, links, and so on), link GNOME's list:
```bash
ln -s /usr/share/applications/gnome-mimeapps.list ~/.local/share/applications/niri-mimeapps.list
```

---

## 8. Ricing with DMS

Open Settings with **Mod+Comma**. Everything below is set there unless a command is shown.

### Wallpaper and slideshow
- **Mod+W** (after section 6) opens the wallpaper browser. Picking one applies it and recolours the whole desktop.
- **Wallpaper & colors > Automatic cycling** turns on a slideshow: pick the folder (`~/Pictures/Wallpapers`), the interval, and whether the order is random.
- From a terminal or a key binding: `dms ipc call wallpaper next` and `dms ipc call wallpaper prev`.

### Colours from the wallpaper
DMS runs matugen on every wallpaper change and builds a Material You palette from it. That palette goes to the bar, panels, niri's focus ring, kitty, and (if you allow it) GTK and Qt apps.
- **Wallpaper & colors > Theme & colors > Source color** decides which part of the wallpaper seeds the palette. *Colorful* favours a vivid accent over a large dull area; *Dominant* (the default) takes the most common colour.
- Light and dark mode switch the palette between its light and dark variants.

### Blur and transparency
niri 26.04 can blur what sits behind DMS surfaces.
1. **Interface style > Background blur**: on. If it says your compositor does not support it, niri is older than 26.04.
2. **Surface opacity** on the same page: lower it until the blur shows. Blur only shows through transparent pixels; at full opacity nothing changes.
3. **Blur border** colour and opacity: a thin outline that smooths the edge where blur meets rounded corners.

For blur behind ordinary windows, add a rule to `config.kdl`. The window needs some transparency for it to show:
```kdl
window-rule {
    match app-id="^kitty$"
    opacity 0.9
    background-effect {
        blur true
    }
}
```

### Gaps, corners, borders
The **Niri** page in Settings sets gaps, window corner radius, and border width. DMS writes them to `dms/layout.kdl`.

### Animations
niri supports custom GLSL shaders for opening and closing windows. The `window-open` / `window-close` blocks with an expanding circle in [pankajsagvekar/dotfiles](https://github.com/pankajsagvekar/dotfiles/blob/main/niri/config.kdl) are a good starting point. Paste them into the `animations` block of `config.kdl`, replacing the existing `window-open` and `window-close` entries, then run `niri validate`.

### GTK and Qt apps: decide once
DMS always generates `dank-colors.css` in `~/.config/gtk-3.0` and `~/.config/gtk-4.0`. The **Apply GTK colors** switch under **Wallpaper & colors > App theming** decides whether apps actually use it.

| | Apply GTK colors **off** | Apply GTK colors **on** |
|---|---|---|
| Apps in niri | Keep the current GTK theme (Tahoe/WhiteSur) | Follow the wallpaper colours |
| Apps in GNOME | Unchanged | Also follow the wallpaper colours, because the folders are shared |
| Desktop consistency in niri | Bar and apps use different palettes | Everything matches |

For a full rice, turn it on. Back up the current files first, so the GNOME look can be restored:
```bash
cp -r ~/.config/gtk-3.0 ~/.config/gtk-3.0.before-dms
cp -r ~/.config/gtk-4.0 ~/.config/gtk-4.0.before-dms
apt policy adw-gtk3
```
If `adw-gtk3` is available, install it first. With it, DMS patches a copy of the theme and GTK3 apps switch between light and dark live; without it, DMS falls back to a plain CSS import.

### Terminal
The kitty config from section 5 already uses the wallpaper palette. Set the font so Starship's icons render, in `~/.config/kitty/kitty.conf`:
```
font_family JetBrainsMono Nerd Font Mono
font_size   11
```
fish, Starship, and fastfetch work unchanged; they do not depend on the desktop.

### Bar and widgets
The **Bar** pages (General, Appearance, Bar widgets) control position, look, widgets, and their order. Media controls, weather, system stats (from `dgop`), and an audio visualiser (from `cava`) are already installed.

### Lock screen and idle
**Power & battery > Power & sleep** sets auto-lock and suspend timers, with separate values on battery and on AC. **Security > Lock screen** sets what the lock screen shows.

---

## 9. Key bindings

`Mod` is the Super (Windows) key, or Alt if you followed *Alt as the Mod key* in section 6; the table shows that variant where keys differ. **Mod+Shift+/** shows the full list on screen, read from your actual config.

| Keys | Action |
|---|---|
| Mod+T | Terminal (kitty) |
| Mod+Space | App launcher |
| Mod+Shift+Space (Alt+Space with Super as Mod) | Spotlight bar (search apps, files, calculator) |
| Mod+Q | Close window |
| Mod+H / Mod+L (or Left / Right) | Focus column left / right |
| Mod+J / Mod+K (or Down / Up) | Focus window down / up in a column |
| Mod+Shift + H/J/K/L | Move window |
| Mod+F | Maximize column |
| Mod+Shift+F | Fullscreen |
| Mod+Shift+T | Toggle floating |
| Mod+Ctrl+W (Mod+Alt+W with Super as Mod) | Tabbed column (moved from Mod+W in section 6) |
| Mod+O (also Mod+Tab with Super as Mod) | Overview of all workspaces |
| Mod+1 to Mod+9 | Switch workspace |
| Mod+U / Mod+I | Workspace down / up |
| **Mod+W** | **Wallpaper browser** (moved from Mod+Y in section 6) |
| Mod+V | Clipboard history |
| Mod+N | Notification center |
| Mod+Shift+N | Notepad |
| Mod+M or Ctrl+Alt+Delete | Task manager |
| Mod+Comma | DMS Settings |
| Mod+Shift+W | Create a window rule for the focused window |
| Mod+X (Super+X with Super as Mod) | Power menu |
| Super+L (Mod+Alt+L with Super as Mod) | Lock screen |
| Mod+Shift+E | Quit niri (back to GDM) |
| Alt+Tab | Switch between recent windows |

---

## 10. Troubleshooting

| Problem | Check |
|---|---|
| `File has unexpected size ... Mirror sync in progress?` | An openSUSE mirror is behind the main server. Wait 15 to 30 minutes, then `sudo apt update` and retry. If it is `quickshell`, the pin in section 2 is missing. |
| DMS bar shows up in GNOME too | It was enabled globally, usually by the package. `sudo systemctl --global disable dms` (and `systemctl --user disable dms` if you enabled it yourself), then `systemctl --user add-wants niri.service dms` again. |
| No bar in niri | `systemctl --user status dms` and `journalctl --user -u dms -b`. Run `dms doctor`. |
| Black screen or instant return to GDM | Log in to GNOME and run `journalctl --user -b -u niri`. Usually a config error; run `niri validate`. |
| "Duplicate bind" error | The same key is defined in `config.kdl` and `dms/binds.kdl`. Keep one. |
| X11 app does not open | `which xwayland-satellite` must print a path. `journalctl --user -b -u niri | grep X11` should show `listening on X11 socket`. |
| Screen sharing in Chrome or Discord fails | Both portals from section 2 must be installed. Log out and in after installing them. |
| High VRAM, stutter with an external monitor | Section 4 profile missing. `nvtop` should show niri near 100 MiB. |
| Blur toggle does nothing | `niri --version` below 26.04, or Surface Opacity still at 100%. |
| GNOME apps changed colour | Apply GTK colors is on (section 8). Turn it off and copy the `.before-dms` folders back. |
| Speakers silent | Same kernel and PipeWire as GNOME, so the fixes from that session apply. `pavucontrol` works in niri too. |

---

## 11. Removing niri and DMS

```bash
systemctl --user remove-wants niri.service dms 2>/dev/null
sudo apt purge dms niri quickshell matugen dgop danksearch
sudo rm /etc/apt/sources.list.d/danklinux.list /etc/apt/sources.list.d/avengemedia-dms.list
sudo rm /etc/apt/keyrings/danklinux.gpg /etc/apt/keyrings/avengemedia-dms.gpg
sudo rm /etc/apt/preferences.d/quickshell-from-debian
sudo apt update
rm -r ~/.config/niri ~/.config/DankMaterialShell
rm ~/.local/share/applications/niri-mimeapps.list
```
GNOME stays as it was, except for the GTK folders if Apply GTK colors was on. Restore them with:
```bash
rm -r ~/.config/gtk-3.0 ~/.config/gtk-4.0
mv ~/.config/gtk-3.0.before-dms ~/.config/gtk-3.0
mv ~/.config/gtk-4.0.before-dms ~/.config/gtk-4.0
```

---

## 12. How long this stays accurate

Written in October 2026, against niri 26.04 and the DMS docs for version 1.6.

| Part | Stays usable | What ends it |
|---|---|---|
| GDM session switching, portals, NVIDIA profile, `add-wants` | Several years | These are stable system interfaces |
| DankLinux repositories for Debian Testing | As long as DankLinux publishes them | If Debian packages niri and DMS itself, switch to those |
| `config.kdl` (environment, rules, animations) | 1 to 2 years | niri has so far kept old config options working; new releases mostly add options |
| DMS Settings layout and menu names | Months | DMS releases often; menu names in section 8 may move. `dms doctor` and the docs at danklinux.com/docs follow the current version |
| `dms/binds.kdl` defaults | Months | New DMS versions may change default keys; `dms setup` never overwrites an existing file, so your edits stay |

Before a big upgrade, back up `~/.config/niri` and `~/.config/DankMaterialShell`. After it, run `niri validate` and `dms doctor`.
