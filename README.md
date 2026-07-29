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
- Installed browser binary selected in `BIN`
- (Optional) `xauth` for X11 sessions

## Quick start

```bash
chmod +x bc
./bc
```

By default, the browser defined here is launched:

```bash
BIN=brave-origin
```

You can also start a shell inside the same isolation profile:

```bash
./bc --bash
```

## Configuration

All configuration is done directly in `bc`.

### 1) Choose the application

```bash
# BIN options: brave-origin, brave, chromium, helium-browser, firefox, librewolf, zen-browser
BIN=brave-origin
```

`BIN -> config path` mapping is handled by the `APP` associative array.

### 2) Config/cache directories

If the app does not use `~/.config/<APP>` and `~/.cache/<APP>`, set these manually:

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

### 4) Environment variables

Pass additional variables through:

```bash
ENV=()
```

Entry format: `NAME=value`.

In most setups, keep this list empty and let the script handle display-related variables automatically.

Example with custom variables:

```bash
ENV=(MOZ_ENABLE_WAYLAND=1 GTK_THEME=Adwaita:dark)
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

## Example: Firefox profile

Set in `bc`:

```bash
BIN=firefox
```

Then adjust `HOME_RW` / `HOME_MASKED` as needed for your data layout.

## Troubleshooting

- No GUI: verify `DISPLAY` / `WAYLAND_DISPLAY` and socket availability.
- No audio: verify `pipewire-0` and `pulse` under `XDG_RUNTIME_DIR`.
- Missing file access: add paths to `HOME_RW`, `HOME_RO`, `RW`, or `RO`.
- App does not start: ensure `BIN` points to a valid executable in your system.

## License

This project is licensed under GNU GPLv3 — see `LICENSE`.
