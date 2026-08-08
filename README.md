# browser-confine

A lightweight launcher that runs a browser inside an isolated environment built with `systemd-run` and `bubblewrap` (`bwrap`).

The `bc` script limits filesystem/device access and applies syscall hardening, while keeping the minimum required pieces for GUI, audio, and networking.

## What this project does

- Starts the browser as a dedicated `systemd --user` unit.
- Starts from a clean/minimal `/` view and mounts back only explicitly required paths.
- Clears the environment (`--clearenv`) and sets only selected variables.
- Isolates `/home` and mounts only explicitly allowed paths.
- Filters `/dev` (everything not in `DEV_ALLOW` is masked).
- Provides fine-grained access lists: `RW`, `RO`, `MASKED`, `HOME_RW`, `HOME_RO`, `HOME_MASKED`.

## Requirements

- Linux with `systemd --user`
- `bubblewrap` (`bwrap`)
- (Optional) `xauth` for X11 sessions

## Supported browsers

The launcher works with browsers predefined in `bc` under `BIN`:

- `brave-origin`
- `brave`
- `chromium`
- `falkon`
- `floorp`
- `helium-browser`
- `firefox`
- `librewolf`
- `vivaldi`
- `zen-browser`

## Quick start

```bash
mkdir -p ~/.local/bin
cd ~/.local/bin
wget https://raw.githubusercontent.com/mtriam/browser-confine/main/bc
chmod +x bc
~/.local/bin/bc
```

By default, the script auto-detects an installed browser from the supported list. You can also specify a browser explicitly:

```bash
~/.local/bin/bc firefox
~/.local/bin/bc brave
```

### Runtime options

```bash
~/.local/bin/bc --bash           # Start an interactive shell inside the sandbox
~/.local/bin/bc --fast-start     # Warm up the browser headlessly
~/.local/bin/bc --app-menu       # Create a desktop application menu entry
~/.local/bin/bc --remove-app-menu  # Remove the desktop application menu entry
```

### Auto-detection

The script automatically detects the first available browser from the supported list when `BIN` is empty. You can also pass the browser name as the first argument to override the default or auto-detected choice.

Examples:

```bash
~/.local/bin/bc firefox
~/.local/bin/bc brave --fast-start
~/.local/bin/bc chromium --bash
```

The `--fast-start` flag runs the browser in headless mode with a temporary profile for the duration specified by `FAST_START_TIME`. It then stops the browser and removes the temporary profile. This is useful for autostart to speed up the first real launch.

## Configuration

All configuration is done directly in `bc`.

### 1) Choose the application

```bash
# BIN options: brave-origin, brave, chromium, falkon, floorp, helium-browser, firefox, librewolf, vivaldi, zen-browser
BIN=brave-origin
```

`BIN -> config path` mapping is handled by the `APP` associative array.

### 2) Config/cache directories

If the app does not use `~/.config/<APP>` and `~/.cache/<APP>` or `~/.mozilla` or `~/.floorp`, set these manually:

```bash
CONFIG_DIR=
CACHE_DIR=
```

### 3) Data access rules

- `HOME_RW` – writable paths under `~`
- `HOME_RO` – read-only paths under `~`
- `HOME_MASKED` – masked paths under `~` (tmpfs or `/dev/null`)
- `RW` / `RO` / `MASKED` – same model for absolute/global paths

Non-existing paths are automatically ignored.

### 4) KeePassXC-Browser integration

Enable KeePassXC-Browser integration for password management:

```bash
KEEPASSXC_BROWSER=true
```

When enabled, this automatically adds:
- `/run/user/$UID` to read-only paths (`RO`)
- `/run/user/$UID/systemd` to masked paths (`MASKED`)

### 5) Environment variables

Pass additional variables through:

```bash
ENV=()
```

Entry format: `NAME=value`.

The script automatically handles display-related variables.

Example with custom variables:

```bash
ENV=(MOZ_ENABLE_WAYLAND=1 GTK_THEME=Adwaita:dark)
ENV=(DISPLAY=)
```

### 5) Device filtering (`/dev`)

`DEV_ALLOW` defines what remains available. Everything else in `/dev` is masked using `tmpfs` (directories) or `/dev/null` (files).

Implementation detail: the script first binds /dev, then masks entries not matched by DEV_ALLOW. This keeps allowed glob patterns dynamic, and devices that appear later (for example a newly plugged YubiKey) are not blocked.

Alternative models are possible (fully open `/dev`, or mounting only a fixed device subset), but fixed mounts usually do not handle hot-plugged devices during runtime.

## GUI and audio handling

The script auto-supports:

- Wayland (`WAYLAND_DISPLAY` socket)
- X11 (`DISPLAY` + optional `XAUTHORITY`)
- PipeWire/PulseAudio (`$XDG_RUNTIME_DIR/pipewire-0`, `$XDG_RUNTIME_DIR/pulse`)
- User session D-Bus (`/run/user/$UID/bus`)

## Security notes

The script applies hardening such as:

- `NoNewPrivileges=yes`
- `ProtectSystem=full`
- `ProtectKernel*` and `ProtectProc=invisible`
- Empty capability set (`CapabilityBoundingSet=`)
- Multiple `SystemCallFilter` exclusions

This is practical hardening, not a formal guarantee against all attack classes.

### File access and the file picker

The browser runs inside a sandbox, but it launches `xdg-desktop-portal` for file operations such as opening or saving files.

`xdg-desktop-portal` runs outside the browser sandbox, so the file picker can access the user's files and the user can select any file.

However, **the browser itself does not gain unrestricted access to the selected file or the rest of the filesystem**. It can only access the paths that are explicitly exposed to it by the sandbox configuration.

So even though the user can browse and select files outside the sandbox in the file picker, the browser remains restricted to its allowed filesystem paths.

## Troubleshooting

- If the browser profile and cache do not exist yet, start the browser once without ~/.local/bin/bc. 
  This   will create them so they can be bind-mounted into the sandbox.
- Run ~/.local/bin/bc --bash to inspect the environment and mounted/masked directories. 
  Then start the browser to view its logs and error messages.
- No GUI: verify `DISPLAY` / `WAYLAND_DISPLAY` and socket availability.
- No audio: verify `pipewire-0` and `pulse` under `XDG_RUNTIME_DIR`.
- Missing file access: add paths to `HOME_RW`, `HOME_RO`, `RW`, or `RO`.
- App does not start: ensure `BIN` points to a valid executable in your system.

## License

This project is licensed under GNU GPLv3 — see `LICENSE`.
