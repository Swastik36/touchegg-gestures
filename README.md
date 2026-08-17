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
mkdir -p ~/.config/touchegg
cp touchegg.conf ~/.config/touchegg/touchegg.conf
```

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
- **"Ignoring this gesture… action not supported"** in the client log
  (run client with `--debug`) → wrong action type in the config. Touchégg v2
  uses `RUN_COMMAND` (not `EXECUTE`) with a `<command>` parameter.
- **Gesture goes the wrong way** → swap the LEFT/RIGHT mappings above.
- **Cinnamon native gestures interfere** → `gsettings set org.cinnamon.gestures enabled false`.