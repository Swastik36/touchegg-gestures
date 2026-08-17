# 3-finger touchpad gestures — Touchégg (portable setup)

Windows-style 3-finger gestures via **Touchégg** + Cinnamon:

| Gesture | Action |
|---|---|
| 3-finger swipe **UP** | Mint menu |
| 3-finger swipe **DOWN** | Expo (workspace view, all workspaces) |
| 3-finger swipe **LEFT** | Next workspace |
| 3-finger swipe **RIGHT** | Previous workspace |

## Setup

```bash
sudo apt install touchegg
sudo systemctl disable --now touchegg      # stop the root daemon — it ignores user config
sudo usermod -a -G input "$USER"           # grant access to /dev/input/event* (needs re-login)
mkdir -p ~/.config/touchegg
cp touchegg.conf ~/.config/touchegg/touchegg.conf
```

> **Why the `input` group is required (found the hard way):**
> Input devices are owned by `root:input` with mode `crw-rw----`, so a
> user-level daemon cannot open them without group membership. The **root
> systemd daemon** can read devices but runs with a different config path
> (`/root/.config/...`), so it silently ignores your user config — gestures
> either do nothing or run stale settings. The correct setup is: **user-level
> daemon + `input` group**. The group change takes effect on next login.
> To test without re-logging in, start the daemon with the group applied:
> `sg input -c 'nohup /usr/bin/touchegg --daemon >/dev/null 2>&1 &'`

**Touchégg is a two-process system** — both must run (daemon gathers gestures,
client executes actions from the config):

```bash
nohup /usr/bin/touchegg --daemon >/dev/null 2>&1 &
nohup /usr/bin/touchegg >/dev/null 2>&1 &
```

Autostart both at login (`~/.config/autostart/touchegg.desktop`):

```
[Desktop Entry]
Type=Application
Name=Touchégg
Comment=Touchpad gesture daemon (3-finger gestures)
Exec=/usr/bin/touchegg
X-GNOME-Autostart-enabled=true
NoDisplay=false
```

## Notes

- Swipe-down opens Expo via **`RUN_COMMAND`** (Touchégg v2 renamed `EXECUTE`) —
  a D-Bus call to Cinnamon's JS eval:
  `gdbus call --session --dest org.Cinnamon --object-path /org/Cinnamon --method org.Cinnamon.Eval "imports.ui.main.expo.toggle()"`
  Expo has **no keyboard binding** in Cinnamon — the D-Bus call is the only
  programmatic trigger besides the hot corner.
- Workspace switching relies on Cinnamon's `Ctrl+Alt+Left/Right` bindings
  (check: `gsettings get org.cinnamon.desktop.keybindings.wm switch-to-workspace-left`).
- Disable Cinnamon's own gesture engine so it doesn't fight Touchégg:
  `gsettings set org.cinnamon.gestures enabled false`
  (it's off by default on some versions, on by default on others).

## Troubleshooting

- **Nothing happens** → verify both processes: `pgrep -a -x touchegg` (two lines:
  daemon + client). If the root daemon is running, `sudo systemctl disable --now touchegg`.
- **Daemon logs `Error opening device /dev/input/eventX`** (run with `--debug`) →
  the daemon cannot read your touchpad. You are missing the `input` group:
  `sudo usermod -a -G input "$USER"`, then re-login (or start via
  `sg input -c '...'` for an immediate test). Seen on a clean IdeaPad Slim 3
  install: root daemon opened devices but ignored the user config, and the
  user daemon couldn't open devices at all — both symptoms point here.
- **"Ignoring this gesture… action not supported"** in the client log
  (run client with `--debug`) → wrong action type in the config. Touchégg v2
  uses `RUN_COMMAND` (not `EXECUTE`) with a `<command>` parameter.
- **Gesture goes the wrong way** → swap the LEFT/RIGHT mappings above.
- **Cinnamon native gestures interfere** → `gsettings set org.cinnamon.gestures enabled false`.