# Debian + Niri + Noctalia (next to GNOME)

A second desktop session on the same Debian install: **niri**, a scrollable-tiling Wayland compositor, with the **Noctalia** shell for the bar, launcher, notifications, and wallpaper picker. GNOME and its macOS look stay installed. At the login screen you pick which one to start.

The niri and Noctalia configs come from [pankajsagvekar/dotfiles](https://github.com/pankajsagvekar/dotfiles). That repo only contains config files, so this guide adds the install steps and the changes needed for this laptop (Debian testing, NVIDIA, 2560x1600 screen).

**Target:** Debian 14 "forky" (testing), GNOME 50 already installed, niri 26.04, Noctalia v5.

---

## Contents
1. [How the two sessions share one system](#1-how-the-two-sessions-share-one-system)
2. [Install niri](#2-install-niri)
3. [Install Noctalia](#3-install-noctalia)
4. [Companion apps](#4-companion-apps)
5. [NVIDIA fix](#5-nvidia-fix)
6. [Apply the dotfiles](#6-apply-the-dotfiles)
7. [Changes to make before first login](#7-changes-to-make-before-first-login)
8. [First login](#8-first-login)
9. [Key bindings](#9-key-bindings)
10. [Troubleshooting](#10-troubleshooting)
11. [Removing niri](#11-removing-niri)
12. [How long this stays accurate](#12-how-long-this-stays-accurate)

---

## 1. How the two sessions share one system

Both GNOME and niri are Wayland sessions. GDM lists every file in `/usr/share/wayland-sessions/`, so once niri is installed it shows up next to GNOME.

| Shared between both sessions | Separate per session |
|---|---|
| Home folder, apps, Flatpaks | Window management and shortcuts |
| `~/.config/gtk-3.0`, `~/.config/gtk-4.0` (app themes) | Top bar: GNOME Shell vs Noctalia |
| dconf / `gsettings` values | GNOME extensions (do not run in niri) |
| fish, Starship, fastfetch, fonts | Wallpaper (GNOME setting vs Noctalia) |
| GDM login screen and keyring | Quick Settings vs Noctalia control center |

The shared rows are the risk. Anything Noctalia writes into `~/.config/gtk-4.0` also changes how apps look in GNOME. Section 7 turns that off.

**Keep GDM.** Some niri guides install `greetd` and the Noctalia greeter. Two display managers fight over the login screen, and GDM already starts niri fine.

---

## 2. Install niri

First check whether Debian has packaged it by now. If it has, use the package and skip to section 3:
```bash
apt policy niri xwayland-satellite
```
If both show `(none)`, build them.

### Build dependencies
```bash
sudo apt install -y build-essential clang pkg-config git curl \
  libudev-dev libgbm-dev libxkbcommon-dev libegl1-mesa-dev libwayland-dev \
  libinput-dev libdbus-1-dev libsystemd-dev libseat-dev libpipewire-0.3-dev \
  libpango1.0-dev libdisplay-info-dev libxcb-cursor-dev xwayland
```

### Rust
niri 26.04 needs Rust 1.87 or newer. Check what Debian ships:
```bash
rustc --version 2>/dev/null || echo "not installed"
```
If it is missing or older, install the current toolchain with rustup. It lives in `~/.cargo` and does not touch Debian's packages:
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
fish_add_path ~/.cargo/bin
```

### niri as a .deb
niri's `Cargo.toml` already describes a Debian package (binary, `niri-session`, the GDM session file, portal config, and systemd units). `cargo-deb` builds it, so niri installs and uninstalls through apt like any other package.
```bash
cargo install cargo-deb
mkdir -p ~/src && cd ~/src
git clone --depth=1 --branch v26.04 https://github.com/niri-wm/niri.git
cd niri
cargo deb
sudo apt install ./target/debian/niri_*.deb
```
- `--branch v26.04` builds the release, not the development branch. The dotfiles use blur (`background-effect`), which needs 26.04 or newer.
- The package pulls in `alacritty` and `fuzzel` as dependencies. They are small, and handy as a fallback terminal and launcher if Noctalia fails to start.
- The build takes 5 to 15 minutes.

### xwayland-satellite (X11 apps)
niri runs X11 apps (Steam, some games, older tools) through xwayland-satellite. niri starts it on demand when it finds it in `PATH`.
```bash
cd ~/src
git clone --depth=1 --branch v0.8.3 https://github.com/Supreeeme/xwayland-satellite.git
cd xwayland-satellite
cargo build --release
sudo install -Dm755 target/release/xwayland-satellite /usr/local/bin/xwayland-satellite
```

---

## 3. Install Noctalia

Noctalia v5 is distributed through its own APT repository.
```bash
cd /tmp
wget https://pkg.noctalia.dev/deb/nickh-archive-keyring.deb
sudo apt install ./nickh-archive-keyring.deb
sudo wget -O /etc/apt/sources.list.d/noctalia.sources https://pkg.noctalia.dev/deb/noctalia-trixie.sources
sudo apt update
sudo apt install noctalia
```
This repository is published for Debian 13 (trixie). It usually works on testing because testing has newer libraries than trixie. Check [docs.noctalia.dev](https://docs.noctalia.dev) for a `forky` or `sid` sources file and use that instead if one exists.

---

## 4. Companion apps

The dotfiles call these programs by name. Install them, or change the key bindings in section 7 to apps you already use.
```bash
sudo apt install -y foot thunar firefox-esr \
  xdg-desktop-portal-gnome xdg-desktop-portal-gtk \
  wl-clipboard brightnessctl qt6ct
```
| Package | Why |
|---|---|
| `foot` | Terminal on Mod+Return; light and Wayland-native |
| `thunar` | File manager on Mod+M |
| `firefox-esr` | Browser on Mod+B |
| `xdg-desktop-portal-gnome`, `-gtk` | Screen sharing, file pickers, and screenshots for apps; niri's portal config asks for these |
| `wl-clipboard` | Clipboard on the command line (`wl-copy`, `wl-paste`) |
| `brightnessctl` | Backlight control fallback |
| `qt6ct` | The config sets `QT_QPA_PLATFORMTHEME=qt6ct` for Qt app styling |

Noctalia has its own polkit agent (password prompts) and notification daemon, so neither needs a separate package.

Set the Nerd Font in foot, so Starship's icons render:
```ini
# ~/.config/foot/foot.ini
font=JetBrainsMono Nerd Font Mono:size=11
```

---

## 5. NVIDIA fix

The NVIDIA driver keeps freed video memory instead of returning it, which makes niri use close to 1 GiB of VRAM instead of about 100 MiB. niri's documentation gives a per-process driver profile that fixes it:
```bash
sudo mkdir -p /etc/nvidia/nvidia-application-profiles-rc.d
sudo tee /etc/nvidia/nvidia-application-profiles-rc.d/50-limit-free-buffer-pool-in-wayland-compositors.json >/dev/null <<'EOF'
{
    "rules": [
        { "pattern": { "feature": "procname", "matches": "niri" },
          "profile": "Limit Free Buffer Pool On Wayland Compositors" }
    ],
    "profiles": [
        { "name": "Limit Free Buffer Pool On Wayland Compositors",
          "settings": [ { "key": "GLVidHeapReuseRatio", "value": 0 } ] }
    ]
}
EOF
```
The kernel line from the main README already has `nvidia-drm.modeset=1`, which niri also requires.

---

## 6. Apply the dotfiles

```bash
cd ~/src
git clone --depth=1 https://github.com/pankajsagvekar/dotfiles.git niri-dotfiles
cd niri-dotfiles

mkdir -p ~/.config/niri ~/.config/noctalia ~/Pictures/Wallpapers
cp niri/config.kdl niri/noctalia.kdl ~/.config/niri/
cp noctalia/config.toml noctalia/settings.toml ~/.config/noctalia/
cp -n wallpapers/* ~/Pictures/Wallpapers/
```
- `cp -n` skips files that already exist, so your own wallpapers stay. `~/Pictures/Wallpapers` is also the folder HyprQuickPaper uses in GNOME, so both sessions share one wallpaper collection.
- `niri/noctalia.kdl` holds focus-ring colours. Noctalia regenerates it from the wallpaper, so do not edit it by hand.

Do not log in yet. The configs were written for another laptop; make the changes in the next section first.

---

## 7. Changes to make before first login

### `~/.config/niri/config.kdl`

**Screen.** The `output "eDP-1"` block sets 1920x1200, the author's panel. This laptop is 2560x1600. Change it to:
```kdl
output "eDP-1" {
    mode "2560x1600"
    scale 1.5
}
```
`scale 1.5` gives the same apparent size as 1707x1067; use `1.25` for more space. After first login, `niri msg outputs` lists the exact modes and refresh rates.

**Polkit lines.** Delete both `spawn-at-startup` lines for `polkit-gnome` and `polkit-mate`. Noctalia's settings have `polkit_agent = true`, so a third agent would only compete for the same password prompts.

**Environment block.** Change these lines:
```kdl
environment {
    // ...keep the other lines...
    GDK_BACKEND "wayland,x11"      // was "wayland": falls back to X11 for apps without Wayland support
    SDL_VIDEODRIVER "wayland,x11"  // was "wayland": Minecraft needed SDL_VIDEODRIVER=x11 on this machine
    QT_QPA_PLATFORM "wayland;xcb"  // was "xcb": native Wayland stays sharp at scale 1.5
}
```
Remove `QT_STYLE_OVERRIDE "Fusion"` if you want qt6ct to control the Qt style; the two settings override each other. `XDG_SESSION_TYPE` and `XDG_CURRENT_DESKTOP` are already set by `niri-session` and can stay or go.

Also update the startup line that repeats `QT_QPA_PLATFORM=xcb`:
```kdl
spawn-at-startup "bash" "-c" "dbus-update-activation-environment --systemd WAYLAND_DISPLAY XDG_CURRENT_DESKTOP=niri QT_ENABLE_HIGHDPI_SCALING=1"
```

**Cursor.** `xcursor-theme "Fluent-dark-cursors"` is not installed here. Use the cursor from the macOS setup (`ls ~/.local/share/icons` shows its name), so both sessions match:
```kdl
cursor {
    xcursor-theme "<your cursor theme>"
    xcursor-size 24
    hide-when-typing
}
```

**Browser.** `Mod+B` starts `firefox-esr`. Change it to `"google-chrome"` if that is your main browser.

### `~/.config/noctalia/settings.toml`

**Stop Noctalia from rewriting GNOME's app theme.** This line makes Noctalia generate GTK and Qt colours from the wallpaper:
```toml
builtin_ids = [ "gtk3", "gtk4", "niri", "qt" ]
```
The GTK files it writes are the same `~/.config/gtk-3.0` and `~/.config/gtk-4.0` files that hold the Tahoe/WhiteSur `@import` lines from the GNOME setup. Change it to:
```toml
builtin_ids = [ "niri" ]
```
Apps then keep the macOS look in both sessions, and niri's focus ring still follows the wallpaper. If you would rather have Material You colours everywhere, keep `gtk4` and back up `~/.config/gtk-4.0` first.

**Telemetry.** `telemetry_enabled = true` is the author's choice. Set it to `false` if you do not want usage data sent.

**Plugins.** `auto_update = "all"` updates plugins from the official and community git repos on its own. Community plugins are third-party code. Turn automatic updates off in Noctalia's settings window (Mod+P), or remove `dotnetrob/cat` and `thepunkoff/pomodoro` from `enabled` if you do not want them.

**Screen-specific values.** `[wallpaper.monitors.eDP-1]` and the `ui_scale` / `scale` values were tuned for a 1920x1200 screen. Adjust them in Noctalia's settings window (Mod+P) after logging in.

### Check the config
```bash
niri validate
```
It reports the line and reason for any mistake. Fix those before logging in; a broken config falls back to niri's defaults with an error banner.

---

## 8. First login

1. Log out of GNOME.
2. On the GDM screen, click your name, then the gear icon in the bottom-right corner.
3. Pick **Niri** and log in.

GDM remembers the last session. To go back to GNOME, pick **GNOME** from the same gear menu.

To use the same default apps as GNOME (which program opens PDFs, links, and so on), link GNOME's list:
```bash
ln -s /usr/share/applications/gnome-mimeapps.list ~/.local/share/applications/niri-mimeapps.list
```

---

## 9. Key bindings

`Mod` is the Super (Windows) key. `Mod+Shift+/` shows the full list on screen.

| Keys | Action |
|---|---|
| Mod+Return | Terminal (foot) |
| Mod+D | App launcher |
| Mod+B / Mod+M | Browser / file manager |
| Mod+Q | Close window |
| Mod+H / Mod+L (or arrows) | Focus column left / right |
| Mod+J / Mod+K | Focus window down / up within a column |
| Mod+Shift + H/J/K/L | Move window |
| Mod+R | Cycle column width (25 / 50 / 75 / 100 %) |
| Mod+F | Fullscreen |
| Mod+T | Toggle floating |
| Mod+1 to Mod+0 | Switch workspace |
| Mod+Shift+1 to Mod+Shift+0 | Move window to workspace |
| **Mod+W** | **Wallpaper picker** (Noctalia; same key as HyprQuickPaper in GNOME) |
| Mod+Shift+W | Next wallpaper |
| Mod+V | Clipboard history |
| Mod+N | Notifications |
| Mod+E | Control center (Wi-Fi, Bluetooth, volume) |
| Mod+P | Noctalia settings |
| Mod+S / Print | Screenshot region / full screen |
| Mod+Escape | Power menu |
| Mod+Shift+T | Lock screen |
| Mod+Shift+C | Reload niri config |
| Mod+Ctrl+Q | Quit niri (back to GDM) |

Changing the wallpaper with Mod+W also recolours Noctalia and niri's focus ring, because the theme source is `wallpaper`.

---

## 10. Troubleshooting

| Problem | Check |
|---|---|
| Black screen or instant return to GDM | Log in to GNOME, run `journalctl --user -b -u niri` and read the last errors. Usually a config error; run `niri validate`. |
| No bar or launcher | Noctalia did not start. Open a terminal (Mod+Return) and run `noctalia` to see its error output. |
| X11 app does not open | `which xwayland-satellite` must print a path. `journalctl --user -b -u niri | grep X11` should show `listening on X11 socket`. |
| Screen sharing in Chrome or Discord fails | Both portals from section 4 must be installed. Log out and in after installing them. |
| High VRAM, stutter | Section 5 profile missing. `nvtop` should show niri near 100 MiB. |
| Apps look different from GNOME | `builtin_ids` still includes `gtk3` / `gtk4`. Fix it as in section 7 and restore `~/.config/gtk-4.0`. |
| Speakers silent | Same kernel and PipeWire as GNOME, so the fixes from that session apply. `pavucontrol` works in niri too. |

---

## 11. Removing niri

```bash
sudo apt purge niri noctalia
sudo rm /usr/local/bin/xwayland-satellite
sudo rm /etc/apt/sources.list.d/noctalia.sources
rm -r ~/.config/niri ~/.config/noctalia
rm ~/.local/share/applications/niri-mimeapps.list
```
GNOME is untouched. If Noctalia had already written GTK colours, restore the `@import` lines in `~/.config/gtk-4.0/gtk.css` and `gtk-dark.css` from the main README.

---

## 12. How long this stays accurate

Written in October 2026, against niri 26.04 (released April 2026) and Noctalia v5.

| Part | Stays usable | What ends it |
|---|---|---|
| GDM session switching, portals, NVIDIA profile | Several years | These are stable system interfaces |
| Building niri with `cargo deb` | Until Debian ships its own `niri` package | Then switch to `apt install niri` |
| niri `config.kdl` | 1 to 2 years | niri has so far kept old config options working across releases; new releases mostly add options |
| Noctalia install and `settings.toml` | Months | v5 is young; the `config_version` field already reads 14, so settings get migrated often. Recheck after each Noctalia update |
| Community plugins | Weeks to months | Plugin authors updating independently of Noctalia |

niri has shipped a release roughly every three to five months. Before upgrading, read its release notes for config changes, then run `niri validate`.
