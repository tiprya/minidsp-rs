# Flex HT support — fork summary

This fork (branch `flexht`) adds support for the **miniDSP Flex HT**
(`hw_id=31, dsp_version=115`) to minidsp-rs.

## Problem

The Flex HT probed as `Generic`:

```
0: Found Generic with serial 901213 at usb:1-1.4%3A1.3?vid=2752&pid=004b [hw_id: 31, dsp_version: 113]
MasterStatus { preset: 0, source: NotInstalled, volume: Gain(-37.5), mute: false, dirac: false }
```

As a result:
- Input source could not be changed / was reported as `NotInstalled`
- No input or output levels were reported
- PEQ, routing, crossover, and compressor controls were unavailable

This matches upstream issue
[mrene/minidsp-rs#698 - Device definition for Flex HT](https://github.com/mrene/minidsp-rs/issues/698).

## Device research (ground truth)

- Toolchain / testing was done against a real unit (serial 901213) over USB.
- The Flex HT was probed before and after a firmware upgrade (dsp 113 -> 115).
  Pre-upgrade it reported `hw_id: 31, dsp_version: 113`; after the console
  upgraded the firmware it reports `hw_id: 31, dsp_version: 115`.
- The Flex HT runs the same firmware architecture as the Flex HTx
  (`hw_id=32`). After the Flex HTx moved to dsp 115 in upstream PR
  [#767](https://github.com/mrene/minidsp-rs/pull/767), its memory layout
  (input PEQ on channels 1-8, outputs as channels 9-16) is **byte-for-byte
  identical** to the Flex HT at dsp 115.
- The symbol/address table was taken from the official MiniDSP Device Console
  config export of the actual unit (`<dspversion>115</dspversion>`, 576 items).
  It was programmatically diffed against the Flex HTx `config.xml`: identical
  names and identical addresses.
- Source IDs were confirmed with the live unit / the user's Home Assistant
  automations:
  - `Toslink 1`, `Spdif 2`, `Usb 3`, `Hdmi 4` (same as Flex HTx, minus `Analog`).

## Changes

- `devtools/src/codegen/flexht/config.xml`
  Console export of the unit's dsp-115 config (576 item symbol table).
- `devtools/src/codegen/flexht/mod.rs`
  Codegen rules: `product_name = "FlexHt"`, sources
  `[Toslink, Spdif, Usb, Hdmi]` (no Analog), 8 inputs (with input PEQ), 8
  outputs (crossover + compressor). Structurally identical to `flexhtx/mod.rs`.
- `devtools/src/codegen/mod.rs`, `devtools/src/main.rs`
  Register the `flexht` codegen target.
- `protocol/src/device/flexht.rs` (generated)
  `cargo run -p minidsp-devtools codegen` output. Verified byte-identical to
  `flexhtx.rs` except `product_name` and `sources`. DO NOT edit by hand.
- `protocol/src/device/mod.rs`
  Gate `pub mod flexht` behind the `device_flexht` feature.
- `protocol/src/device/probe.rs`
  - Add `FlexHt` variant to `DeviceKind`.
  - `(31, 115) => FlexHt` probe mapping.
  - `by_kind` arms for `FlexHt`.
- `protocol/src/source.rs`
  Source mapping for `hw_id 31`: `[(Toslink, 1), (Spdif, 2), (Usb, 3), (Hdmi, 4)]`.
- `protocol/Cargo.toml`
  Add `device_flexht` feature; include it in `all_devices`.

## Build / packaging

- Cross-compiled `minidsp` + `minidspd` for `aarch64-unknown-linux-gnu`
  (the piCorePlayer host target).
  - Toolchain used a per-build rustup install (temp dir, no root).
  - `cargo-zigbuild` + Zig as linker; libusb taken from Debian arm64
    `libusb-1.0-0-dev` extracted into a scratch sysroot (pkg-config cross
    flags). No system packages were installed on the build host.
- Packaged the aarch64 `.deb` (`cargo deb`) and converted to a piCorePlayer
  `.tcz` with the deb2tcz method (same layout the existing tcz used:
  `usr/bin/{minidsp,minidspd}`, `etc/minidsp/config.toml`,
  `lib/systemd/system/minidsp.service`, `lib/udev/rules.d/99-minidsp.rules`).

## How to build for your pi (aarch64 cross-compile, no root)

These are the exact steps used to produce the binaries in this repo's
releases path for the Raspberry Pi (piCorePlayer, aarch64). They require only a
user account on a build host (tested on an Arch Linux x86_64 box) — **no
root/sudo** is needed. Everything lives in one temp dir that can be deleted
afterwards.

### 1. Fresh rustup + aarch64 target (in a temp dir, user-level)

```sh
BUILD=/tmp/minidsp-build
mkdir -p "$BUILD"
export CARGO_HOME="$BUILD/.cargo" RUSTUP_HOME="$BUILD/.rustup"
curl -sSf https://sh.rustup.rs -o "$BUILD/rustup-init.sh"
cd "$BUILD" && sh rustup-init.sh -y --no-modify-path --profile minimal \
  --default-toolchain stable --default-host x86_64-unknown-linux-gnu
"$CARGO_HOME/bin/rustup" target add aarch64-unknown-linux-gnu
"$CARGO_HOME/bin/rustup" component add rustfmt
export PATH="$CARGO_HOME/bin:$PATH"
```

### 2. Zig + cargo-zigbuild (handles C deps + linking for the cross target)

```sh
ZIGVER=0.16.0
curl -sSL -o "$BUILD/zig.tar.xz" \
  "https://ziglang.org/download/$ZIGVER/zig-x86_64-linux-$ZIGVER.tar.xz"
tar -C "$BUILD" -xf "$BUILD/zig.tar.xz"
ln -sf "$BUILD/zig-x86_64-linux-$ZIGVER/zig" "$BUILD/zig"
export PATH="$BUILD:$PATH"          # so `zig` resolves
cargo install cargo-zigbuild --locked
```

### 3. libusb for the aarch64 target (headers + .a + .pc, no system install)

The `hidapi` C backend needs libusb for the target. Rather than installing a
cross sysroot system-wide, extract the Debian arm64 libusb dev package into a
scratch sysroot and point pkg-config at it:

```sh
mkdir -p "$BUILD/debs" "$BUILD/sysroot"
BASE="https://deb.debian.org/debian"
curl -sSL -o "$BUILD/debs/libusb0.deb" "$BASE/pool/main/libu/libusb-1.0/libusb-1.0-0_1.0.26-1_arm64.deb"
curl -sSL -o "$BUILD/debs/libusbdev.deb" "$BASE/pool/main/libu/libusb-1.0/libusb-1.0-0-dev_1.0.26-1_arm64.deb"
for f in "$BUILD"/debs/*.deb; do
  d="$BUILD/dig-$(basename "$f")"; mkdir -p "$d"; ( cd "$d" && ar x "$f" && tar xf data.tar.xz -C "$BUILD/sysroot" )
done

export PKG_CONFIG_ALLOW_CROSS=1
export PKG_CONFIG_SYSROOT_DIR="$BUILD/sysroot"
export PKG_CONFIG_PATH="$BUILD/sysroot/usr/lib/aarch64-linux-gnu/pkgconfig"
# sanity: pkg-config --cflags --libs libusb-1.0  should print the sysroot paths
```

### 4. Clone the fork + build

```sh
cd "$BUILD"
git clone -q --depth 1 -b flexht https://github.com/tiprya/minidsp-rs.git src
cd src
cargo zigbuild --release --target aarch64-unknown-linux-gnu \
  -p minidsp -p minidsp-daemon
# binaries at:
#   target/aarch64-unknown-linux-gnu/release/minidsp
#   target/aarch64-unknown-linux-gnu/release/minidspd
```

Optional: (re)generate the device file after editing codegen rules:

```sh
cargo run -p minidsp-devtools -- codegen protocol/src/device
rustfmt --edition 2021 protocol/src/device/*.rs
```

### 5. Package as a piCorePlayer tcz (do this ON the pi)

Build the `.deb` (mirrors the official release packaging: binaries + config +
systemd unit + udev rule), then convert deb -> tcz with the deb2tcz method
from [tinycore forum](https://forum.tinycorelinux.net/index.php?topic=2325.0):

```sh
# on the build host:
cargo install cargo-deb --version 2.7.0
export PKG_CONFIG_ALLOW_CROSS=1 PKG_CONFIG_SYSROOT_DIR="$BUILD/sysroot" \
       PKG_CONFIG_PATH="$BUILD/sysroot/usr/lib/aarch64-linux-gnu/pkgconfig"
cargo deb --target aarch64-unknown-linux-gnu -p minidsp --no-build --no-strip
# ~> target/aarch64-unknown-linux-gnu/debian/minidsp_0.1.12-1_arm64.deb

# on the pi (piCorePlayer) as the tc user:
tce-load -i squashfs-tools findutils   # if not already loaded
mkdir -p /tmp/deb2tcz/pkg && cd /tmp/deb2tcz
ar x /home/tc/minidsp_0.1.12-1_arm64.deb     # extract data.tar.xz
tar xf data.tar.xz -C pkg
rm -rf pkg/usr/share                        # strip doc/man
mksquashfs pkg /home/tc/minidsp-new.tcz -noappend
```

### 6. Install on the pi (back up the old one first!)

```sh
cd /mnt/mmcblk0p2/tce/optional
# back up the current extension + binaries
cp minidsp.tcz minidsp.tcz.bak.$(date +%Y%m%d_%H%M%S)
# replace
cp -v /home/tc/minidsp-new.tcz /mnt/mmcblk0p2/tce/optional/minidsp.tcz
md5sum /mnt/mmcblk0p2/tce/optional/minidsp.tcz | awk '{print $1"  minidsp.tcz"}' \
  > /mnt/mmcblk0p2/tce/optional/minidsp.tcz.md5.txt
# reboot so piCorePlayer loads the new extension
sudo reboot
# verify
minidsp probe        # should print: Found FlexHt ... [hw_id: 31, dsp_version: 115]
minidsp              # master status with real source + input/output levels
minidspd --version
```

### 7. Clean up the build host

```sh
rm -rf "$BUILD"      # removes rustup, zig, sysroot, debs, clone, target (~3 GB)
```

The pi is a piCorePlayer host (`Living_room_pi`, `tc@192.168.100.32`); the
Flex HT is connected over USB (`vid=2752 pid=004b`).

## Verification (on hardware, read-only)

- `minidsp probe` -> `0: Found FlexHt with serial 901213 ... [hw_id: 31, dsp_version: 115]`
- `minidsp` -> `MasterStatus { preset: 0, source: Hdmi, ... }` plus 8x8 input
  and output levels (previously `NotInstalled` and empty).
- `minidsp debug id` -> `Detected sources: [(Toslink, 1), (Spdif, 2), (Usb, 3), (Hdmi, 4)]`
- `minidspd --version` -> `minidsp-daemon 0.1.12` (daemon functional).
- Read-only dumps of the crossover / PEQ regions return live DSP data,
  confirming the device definition addresses real processing memory.

## Notes / follow-ups

- The pre-upgrade dsp-113 layout is not mapped; at dsp 113 the unit would have
  used the pre-#767 output-PEQ layout. Only `(31, 115)` is currently probed as
  `FlexHt`. If units remain on dsp 113 a second device definition (or mapping)
  would be needed.
- PEA writes have not yet been exercised end-to-end with a deliberate write
  test; the layout is byte-identical to the Flex HTx map (hardware-validated in
  upstream PR #767) so input PEQ writes are expected to work.
- Saved device settings from the firmware-upgrade path (preset 1 + 2 +
  InputPEQ backups) live under the MiniDSP Device Console directory on the
  host and are restorable there.
- Old binary backups: pi `tce/optional/minidsp.tcz.bak.*` and
  `/home/tc/minidsp-backup-*`.

## Running the daemon (minidspd) for Home Assistant

The `minidspd` daemon exposes the device over HTTP so Home Assistant (or any
other host) can read status (volume, source, mute, preset, input/output levels)
and send volume/source/preset commands.

### 1. Persistent daemon config

The `.tcz` ships a read-only `config.toml` on the squashfs, so put an
override config somewhere persistent. `/home/tc` is in piCorePlayer's
persist list (`/opt/.filetool.lst` -> `home`), so use it:

```sh
cat > /home/tc/minidspd.toml <<'EOF'
[http_server]
bind_address = "0.0.0.0:5380"
EOF
```

Binding `0.0.0.0` exposes the API on the LAN (pi is `192.168.100.32`,
Home Assistant is `192.168.100.160:8123`). Use `127.0.0.1:5380` if you only
want local access.

Note: the HTTP server must be enabled for the API to serve. If the config
omits `[http_server]`, the daemon only serves the plugin-compatible TCP server
(default `0.0.0.0:5333`).

### 2. init.d control script (the piCorePlayer way)

piCorePlayer has no systemd; services live as init.d scripts under
`/usr/local/etc/init.d/` and are driven with busybox `start-stop-daemon`
(this survives SSH session close, unlike a raw `&`). Save as
`/usr/local/etc/init.d/minidspd`:

```sh
#!/bin/sh
# minidspd daemon control script for piCorePlayer
DAEMON=/usr/bin/minidspd
CONFIG=/home/tc/minidspd.toml
PIDFILE=/var/run/minidspd.pid
DESC="miniDSP daemon"

start() {
    echo -n "Starting $DESC: "
    start-stop-daemon --start --quiet --background --make-pidfile \
        --pidfile "$PIDFILE" --exec "$DAEMON" -- -c "$CONFIG"
    echo "OK"
}
stop() {
    echo -n "Stopping $DESC: "
    start-stop-daemon --stop --quiet --pidfile "$PIDFILE" --oknodo
    echo "OK"
}
restart() { stop; sleep 1; start; }
case "$1" in
    start) start ;;
    stop) stop ;;
    restart) restart ;;
    status)
        if [ -f "$PIDFILE" ] && kill -0 "$(cat "$PIDFILE")" 2>/dev/null; then
            echo "$DESC is running (pid $(cat "$PIDFILE"))"
        else
            echo "$DESC is not running"
        fi ;;
    *) echo "Usage: $0 {start|stop|restart|status}"; exit 1 ;;
esac
```

```sh
sudo cp <script> /usr/local/etc/init.d/minidspd
sudo chmod +x /usr/local/etc/init.d/minidspd
sudo /usr/local/etc/init.d/minidspd start
sudo /usr/local/etc/init.d/minidspd status
```

### 3. Auto-start on boot

Add the start to `bootlocal.sh` and persist both the script and the
filetool list:

```sh
sudo sh -c 'echo "\n# miniDSP daemon (HTTP API for Home Assistant)\n/usr/local/etc/init.d/minidspd start" >> /opt/bootlocal.sh'

# make sure the init.d script survives reboot too
echo "usr/local/etc/init.d/minidspd" | sudo tee -a /opt/.filetool.lst

# persist everything (also backs up bootlocal.sh + /home/tc config)
sudo filetool.sh -b
```

`/opt` and `home` are already in the persist list, so `bootlocal.sh` and
`/home/tc/minidspd.toml` are covered.

### 4. Verify the API

```sh
curl -s http://127.0.0.1:5380/devices
#   [{"url":"usb:...vid=2752&pid=004b","product_name":"FlexHt",
#     "version":{"hw_id":31,"dsp_version":115,"serial":901213}}]

curl -s http://127.0.0.1:5380/devices/0
#   {"master":{"preset":0,"source":"Hdmi","volume":-36.0,"mute":false},
#    "input_levels":[...], "output_levels":[...], ...}
```

### 5. HTTP API summary

Base URL: `http://<pi>:5380`

| Method | Path | Purpose |
|--------|------|---------|
| GET  | `/devices` | List discovered devices |
| GET  | `/devices/:index` | Master status (volume, source, mute, preset, levels) |
| POST | `/devices/:index/volume/:direction` | `direction` = `up` / `down` (relative gain) |
| POST | `/devices/:index/source/:source` | `source` = `toslink` / `spdif` / `usb` / `hdmi` |
| POST | `/devices/:index/preset/:preset` | `preset` = 0-based config slot |
| POST | `/devices/:index/config` | Full master config write (see schema) |
| WS   | `/devices/:index/ws` | WebSocket bridge (live updates) |
| GET  | `/openapi.json` | Full OpenAPI spec |

Home Assistant can poll `GET /devices/0` with a RESTful sensor for volume /
input/output levels, and drive volume/source with `rest_command` POSTs.
(Configure those in HA's `configuration.yaml`; the daemon side is complete once
the API above responds.)

