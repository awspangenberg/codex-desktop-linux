# Codex Micro

This opt-in Linux feature enables the Work Louder Codex Micro integration that
already ships in the upstream Codex desktop app. It does two narrowly scoped
things:

1. enables the upstream Codex Micro feature gate locally; and
2. adds the verified `node-hid@3.3.0` Linux prebuild for the current app's
   nested Work Louder dependency.

The feature is disabled by default.

## Enable

Copy `linux-features/features.example.json` to the gitignored
`linux-features/features.json`, then add `codex-micro` to `enabled` and rebuild:

```json
{
  "enabled": [
    "codex-micro"
  ]
}
```

```bash
./install.sh
```

The feature rejects the build if the current DMG no longer contains the
expected Codex Micro gate/route or the exact nested `node-hid` loader contract.
Only the pinned x64 and arm64 prebuilds are supported; there is no source-build
fallback.

## Runtime libraries

Native packages declare the required `libudev.so.1` and `libusb-1.0.so.0`
dependencies. For AppImage, source, or user-local installs, install them
yourself:

```bash
# Debian / Ubuntu
sudo apt install libudev1 libusb-1.0-0

# Fedora
sudo dnf install systemd-libs libusb1

# Arch Linux
sudo pacman -S systemd-libs libusb
```

## Device access

Feature-enabled Debian, RPM, and pacman packages install
`/usr/lib/udev/rules.d/70-codex-micro.rules`. Native package scripts do not
reload the system udev daemon. Reload after the first install and after a
package rebuild that enables or disables the feature, then reconnect USB or
Bluetooth:

```bash
sudo udevadm control --reload-rules
```

AppImage, source, user-local, Home Manager, and direct flake installs cannot
change host udev policy. Install the copy staged for your install mode once.

For a source build:

```bash
codex_rule="codex-app/.codex-linux/features/codex-micro/70-codex-micro.rules"
test -r "$codex_rule" || {
  echo "The source build was not built with codex-micro enabled" >&2
  exit 1
}
sudo install -Dm0644 \
  "$codex_rule" \
  /etc/udev/rules.d/70-codex-micro.rules
```

For a user-local install, use the installed app copy:

```bash
codex_rule="$HOME/.local/opt/codex-desktop-linux/codex-app"
codex_rule="$codex_rule/.codex-linux/features/codex-micro/70-codex-micro.rules"
test -r "$codex_rule" || {
  echo "The user-local app was not built with codex-micro enabled" >&2
  exit 1
}
sudo install -Dm0644 \
  "$codex_rule" \
  /etc/udev/rules.d/70-codex-micro.rules
```

For an AppImage, replace the placeholder with the image you use, then extract
only its staged rule:

```bash
codex_appimage="$(readlink -f /path/to/codex-desktop.AppImage)" || exit 1
codex_extract_dir="$(mktemp -d)" || exit 1
(
  set -e
  trap 'rm -rf -- "$codex_extract_dir"' EXIT
  cd "$codex_extract_dir"
  "$codex_appimage" --appimage-extract \
    'opt/codex-desktop/.codex-linux/features/codex-micro/70-codex-micro.rules' \
    >/dev/null
  codex_rule="$codex_extract_dir/squashfs-root/opt/codex-desktop"
  codex_rule="$codex_rule/.codex-linux/features/codex-micro/70-codex-micro.rules"
  test -r "$codex_rule" || {
    echo "The AppImage was not built with codex-micro enabled" >&2
    exit 1
  }
  sudo install -Dm0644 \
    "$codex_rule" \
    /etc/udev/rules.d/70-codex-micro.rules
)
```

For Home Manager or another installed Nix profile, resolve the selected Nix
store output from the installed launcher:

```bash
codex_executable="$(command -v codex-desktop)" || {
  echo "codex-desktop is not on PATH" >&2
  exit 1
}
codex_executable="$(readlink -f "$codex_executable")" || exit 1
codex_package_root="$(dirname "$(dirname "$codex_executable")")"
codex_rule="$codex_package_root/lib/udev/rules.d/70-codex-micro.rules"
test -r "$codex_rule" || {
  echo "The installed Nix package was not built with codex-micro enabled" >&2
  exit 1
}
sudo install -Dm0644 \
  "$codex_rule" \
  /etc/udev/rules.d/70-codex-micro.rules
```

For a direct flake build that is not installed on `PATH`, use the
feature-enabled package output. This example assumes the usual `result`
out-link; substitute a custom `--out-link` or the path returned by
`--no-link --print-out-paths` when applicable:

```bash
codex_package_root="$(readlink -e ./result)" || {
  echo "The direct flake output link does not exist" >&2
  exit 1
}
codex_rule="$codex_package_root/lib/udev/rules.d/70-codex-micro.rules"
test -r "$codex_rule" || {
  echo "The flake output was not built with codex-micro enabled" >&2
  exit 1
}
sudo install -Dm0644 \
  "$codex_rule" \
  /etc/udev/rules.d/70-codex-micro.rules
```

After any manual install, reload the rules and reconnect USB or Bluetooth:

```bash
sudo udevadm control --reload-rules
```

The USB rule imports `usb_id` before matching the observed Work Louder
VID/PID/interface (`303a:8360`, interface `00`). The Bluetooth rule matches the
same vendor HID channel on the Bluetooth HID bus. Both rules use `uaccess` and
`0660`; they do not make hidraw devices world-writable.

NixOS installs the rule automatically when the feature is selected through the
module. Other install modes require the matching manual procedure above.
When disabling a manually installed copy, remove
`/etc/udev/rules.d/70-codex-micro.rules`, reload the rules, and reconnect the
device. Removing or disabling a native package build removes its packaged copy,
but still requires the same reload and reconnect.

## Bluetooth

Pair the Micro through the desktop Bluetooth settings before opening Codex.
Channel selection and pairing mode are device operations; see the Work Louder
Micro setup guide.

## Verify

With the Micro connected, identify its hidraw node and exercise the actual rule:

```bash
udevadm info --attribute-walk --name=/dev/hidrawN
sudo udevadm test "$(udevadm info --query=path --name=/dev/hidrawN)"
getfacl /dev/hidrawN
```

The test output must show the Codex Micro rule, imported USB properties for a
USB connection, the `uaccess` tag, and mode `0660`. Then open
`Settings -> Codex Micro` and verify connection state, buttons, dial, joystick,
battery/status reporting, and lighting controls.

Run focused checks with:

```bash
node --test linux-features/codex-micro/test.js
node --test scripts/lib/linux-features.test.js
```
