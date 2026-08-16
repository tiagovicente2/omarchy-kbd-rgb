# kbd-rgb

An [Omarchy](https://omarchy.org/) Quattro shell plugin that controls the RGB
keyboard backlight on ASUS Vivobook laptops over the **HID LampArray** interface
(ITE5570 controller), using [VRGB](https://github.com/vrgb-dev/vrgb).

The plugin runs as a shell **service** that:

- registers a **vector system tray icon** (freedesktop StatusNotifierItem) that renders
  a sharp, anti-aliased keyboard illuminated in your active color (or rainbow gradient in rainbow mode);
- provides **two-way hardware brightness key synchronization** (via UPower DBus and sysfs);
- provides a **Theme Accent mode** that automatically tracks your active Omarchy theme's accent color;
- opens a modern control panel on click — 12-color curated palette with contrast indicators, custom hex input,
  brightness slider with quick step buttons, and Theme / Static / Rainbow / Off modes;
- reapplies your color after a reboot or full power cycle, surviving firmware power resets.

The keyboard on these laptops does **not** speak the ASUS WMI dialect that
`asusctl`/OpenRGB use for ROG and TUF laptops — it exposes the Microsoft HID
LampArray protocol instead. That is why this plugin shells out to `vrgb`
rather than the usual ASUS tools.

## How it works

```
omarchy-shell (service)
├── Service.qml   state + vrgb apply queue + control window (PanelWindow)
└── sni.py        StatusNotifierItem (Cairo vector tray icon) + UPower DBus hotkey sync
                        │ left/right click
                        ▼
        omarchy-shell kbd-rgb toggle
```

The tray icon is a real StatusNotifierItem, so it appears inside the existing
`omarchy.tray` widget and is pinnable/hidable like any other tray app.

## Prerequisites

Install VRGB (Arch-based systems first):

```bash
paru -S vrgb 2>/dev/null || {
  git clone https://github.com/vrgb-dev/vrgb /tmp/vrgb
  cd /tmp/vrgb && sudo ./install.sh
}
sudo udevadm control --reload-rules && sudo udevadm trigger
vrgb set ff0000 100   # sanity check: keyboard turns red
```

The helper also needs `python3` with `dbus-python` and `pycairo` (pre-installed on
Omarchy). Some ITE5570 systems ignore HID commands until the ASUS WMI module
has been loaded. If color changes do nothing:

```bash
sudo modprobe asus-nb-wmi
echo asus-nb-wmi | sudo tee /etc/modules-load.d/asus-nb-wmi.conf
```

## Install

From a checkout of this repo:

```bash
mkdir -p ~/.config/omarchy/plugins/kbd-rgb
cp manifest.json Service.qml Model.js sni.py ~/.config/omarchy/plugins/kbd-rgb/
omarchy-shell shell rescanPlugins
```

Then enable it by adding its id to `plugins[]` in
`~/.config/omarchy/shell.json`:

```json
{
  "plugins": [
    { "id": "kbd-rgb" }
  ]
}
```

The shell hot-reloads `shell.json` on save. Verify with:

```bash
omarchy-shell shell listPlugins | grep kbd-rgb
```

A keyboard icon should appear in the system tray. Restart the shell if it does
not: `omarchy restart shell`.

## Usage & IPC Commands

- **Left- or right-click the tray icon** to open the control panel.
- Choose between **Theme**, **Static**, **Rainbow**, or **Off**.
- Pick from the 12 curated color presets, or type a custom 6-digit hex color.
- Drag the brightness slider or tap the quick-step buttons (`Off`, `33%`, `67%`, `100%`).
- Clicking outside the panel (or pressing Escape) dismisses it.

### Command-line & Hyprland Shortcuts

You can control `kbd-rgb` directly via IPC from scripts or Hyprland keybindings:

```bash
omarchy-shell kbd-rgb toggle              # Open / close control panel
omarchy-shell kbd-rgb togglePower         # Toggle between off and previous on mode
omarchy-shell kbd-rgb stepBrightness 10   # Increase brightness by 10%
omarchy-shell kbd-rgb stepBrightness -10  # Decrease brightness by 10%
omarchy-shell kbd-rgb nextPreset          # Cycle to the next preset color
omarchy-shell kbd-rgb setMode theme       # Match current Omarchy theme accent
omarchy-shell kbd-rgb setHex 00E5FF       # Set custom hex color
omarchy-shell kbd-rgb setBrightness 80    # Set brightness percentage
omarchy-shell kbd-rgb status              # Get current JSON status
```

## License

MIT