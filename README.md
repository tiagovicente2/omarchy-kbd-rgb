# kbd-rgb

An [Omarchy](https://omarchy.org/) Quattro shell plugin that controls the RGB
keyboard backlight on ASUS Vivobook laptops over the **HID LampArray** interface
(ITE5570 controller), using [VRGB](https://github.com/vrgb-dev/vrgb).

The plugin runs as a shell **service** that:

- registers a **system tray icon** (freedesktop StatusNotifierItem) that shows
  the current keyboard color as a small swatch;
- opens a themed control panel on click — preset colors, hex input, brightness
  slider, and Static / Rainbow / Off modes;
- reapplies your color after a reboot or full power cycle, because the keyboard
  resets to its firmware default when fully powered off.

The keyboard on these laptops does **not** speak the ASUS WMI dialect that
`asusctl`/OpenRGB use for ROG and TUF laptops — it exposes the Microsoft HID
LampArray protocol instead. That is why this plugin shells out to `vrgb`
rather than the usual ASUS tools.

## How it works

```
omarchy-shell (service)
├── Service.qml   state + vrgb apply queue + control window (PanelWindow)
└── sni.py        StatusNotifierItem → shows in omarchy.tray
                        │ left/right click
                        ▼
        omarchy-shell kbd-rgb toggle
```

The tray icon is a real StatusNotifierItem, so it appears inside the existing
`omarchy.tray` widget and is pinnable/hidable like any other tray app. It
updates to a solid color swatch of the current keyboard color.

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

The helper also needs `python3` with `dbus-python` (already installed on
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
`~/.config/omarchy/shell.json` (third-party plugins are enabled iff their id
appears there):

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

## Usage

- **Left- or right-click the tray icon** to open the control panel.
- Pick a preset swatch, type a hex color (Enter), or drag the brightness
  slider. Choose Static, Rainbow, or Off.
- Clicking anywhere outside the panel (or pressing Escape) closes it.
- Every change is saved to the `kbd` profile and re-applied on shell startup by
  the service, so the color survives reboots and full power cycles.

The tray swatch recolors live as you change the color.

## Development

```bash
omarchy plugin validate .
node --check Model.js
python3 -m py_compile sni.py
omarchy-shell kbd-rgb ping
omarchy-shell kbd-rgb status
omarchy-shell kbd-rgb setHex ff8800
```

`Service.qml` reuses Omarchy's MIT-licensed shell kit (`PanelWindow`,
`BorderSurface`, `Button`, `PanelSlider`, `qs.Commons`).

## License

MIT