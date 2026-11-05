# Raspberry Pi 3B + Canon LASER SHOT LBP-1210 as a Wi-Fi print server

## Why this is not a plain "install CUPS" job

The LBP-1210 is a CAPT ("winprinter") device. It has no built-in page
language; the host computer rasterises every page and streams compressed
bitmaps (SCoA format) over USB with a bidirectional handshake. So:

- A generic/raw CUPS queue cannot drive it. A real CAPT driver has to run on the Pi.
- Canon's official Linux CAPT driver (`cndrvcups-capt` 2.71) ships x86/x86_64
  binaries only. It does not run on the ARM CPU of the Pi 3B.
- The open-source `captdriver` project (newer LBP2900/3000 family) does **not**
  support the LBP-1210 because it lacks SCoA.
- **`captppd`** (https://github.com/darkvision77/captppd, with the `libcapt`
  protocol library) is an open-source, reverse-engineered CAPT v1 driver whose
  target list is LBP800 / LBP810 / LBP1120 / **LBP1210** / LBP3200. Release
  1.0.0 (January 2026) ships prebuilt `arm64` and `armhf` Debian packages,
  a `captusb://` CUPS backend built on libusb, and a `LBP1210CAPTPPD.ppd`.
  600 dpi, multi-page jobs, toner saving/density, media type and cartridge
  cleaning are supported; 300 dpi and custom page sizes are not.

That is the driver this setup uses. Clients on the LAN never need a Canon
driver: they send PDF/IPP to the Pi, and the Pi does all rendering.

## Hardware / OS

| Item | Choice |
|------|--------|
| Board | Raspberry Pi 3B (1 GB RAM, 2.4 GHz Wi-Fi only) |
| OS | **Raspberry Pi OS Lite (64-bit)**, Trixie (Debian 13) or Bookworm (Debian 12) |
| Storage | 8 GB+ microSD |
| Power | 5 V / 2.5 A supply (USB printer port draws little, but under-powered Pis drop Wi-Fi) |
| Printer | USB cable to the Pi. The parallel port is not supported by captppd. |

Use **64-bit**. The prebuilt `armhf` (32-bit) package depends on `libcups2`,
which 32-bit Trixie renames to `libcups2t64` without a compatibility alias, so
it will not install there. On 64-bit Trixie the alias exists and the arm64
package installs cleanly. (32-bit Bookworm also works.)

## Step 1 – Flash and bring the Pi onto Wi-Fi

1. Raspberry Pi Imager → "Raspberry Pi OS Lite (64-bit)".
2. In the Imager's OS customisation set: hostname (e.g. `printpi`), username +
   password, Wi-Fi SSID/password (2.4 GHz network), country, and **enable SSH**.
3. Boot, then `ssh <user>@printpi.local`.
4. On your router, give the Pi a DHCP reservation so its IP never changes.

## Step 2 – Run the setup script on the Pi

Copy `pi-setup.sh` to the Pi (`scp pi-setup.sh <user>@printpi.local:`), plug in
and power on the printer, then:

```sh
chmod +x pi-setup.sh
./pi-setup.sh
```

What it does:

1. Installs `cups`, `cups-filters`, `ghostscript`, `avahi-daemon`, `libusb-1.0-0`.
2. Downloads and installs `captppd_1.0.0_<arch>.deb` from the GitHub release
   (backend `/usr/lib/cups/backend/captusb`, PPDs in `/usr/share/cups/model/`,
   and a USB quirks file so the stock CUPS `usb` backend ignores the printer).
3. Blacklists the `usblp` kernel module (the backend uses libusb directly).
4. Adds your user to `lpadmin`; runs `cupsctl --remote-any --remote-admin --share-printers`.
5. Finds the printer via `lpinfo -v` (`captusb://Canon/LBP1210?drv=capt&serial=...`)
   and creates the shared queue `LBP1210` with `LBP1210CAPTPPD.ppd`, A4, 600 dpi.
6. Prints the URLs to use from clients.

Set `QUEUE_NAME=Laser ./pi-setup.sh` if you want a different queue name.

Test from the Pi:

```sh
echo "hello" | lp -d LBP1210
lpstat -o
```

The first page takes 20–40 s on a Pi 3B (Ghostscript at 600 dpi), then it is
printer-speed.

## Step 3 – Add the printer on each client

Avahi advertises the queue as an IPP Everywhere / AirPrint printer, so most
clients find it automatically.

**Windows 10/11**
Settings → Bluetooth & devices → Printers & scanners → Add device. If it does
not appear within a minute: "Add manually" → "Select a shared printer by name"
→ enter

```
http://<pi-ip>:631/printers/LBP1210
```

Driver prompt: Manufacturer "Microsoft" → "Microsoft IPP Class Driver".
(Any PDF-capable generic driver also works; do NOT install a Canon CAPT driver
on the PC.)

**macOS**
System Settings → Printers & Scanners → Add. The Pi queue shows up via Bonjour;
pick it, leave "Use: AirPrint" or "Generic PostScript/PDF".

**Linux**
`cups-browsed` or the GNOME/KDE printer dialog discovers it. Manual:

```sh
lpadmin -p LBP1210 -E -v ipp://<pi-ip>:631/printers/LBP1210 -m everywhere
```

**Android / iOS**
Android's default print service and iOS AirPrint both discover it on the same
Wi-Fi network.

## Troubleshooting

| Symptom | Check |
|---------|-------|
| `lpinfo -v` shows no `captusb://` line | `lsusb` must list `04a9:2617 Canon, Inc.`. Try another cable/port; run `sudo /usr/lib/cups/backend/captusb` directly. |
| Job stuck "Processing", nothing printed | `sudo cupsctl --debug-logging`, then `grep '\] \[Job' /var/log/cups/error_log`. Confirm the queue uses `captusb://` (`lpstat -v`) and the PPD is "Canon LBP1210, captppd 1.0.0", not a Canon "CAPT ver.x" PPD. |
| Printed once, later jobs hang | Power-cycle the printer, `sudo cupsenable LBP1210`. Report at https://github.com/darkvision77/captppd/issues with the log. |
| Clients cannot see the Pi | Same Wi-Fi network/VLAN; `systemctl status avahi-daemon`; Pi 3B is 2.4 GHz only. |
| Windows "cannot connect" | Use `http://`, not `https://`, and the `:631` port. |
| Web UI refuses admin login | Log out/in on the Pi so the `lpadmin` group applies. |
| Toner density / toner save | CUPS web UI → Printers → LBP1210 → Administration → "Set Default Options". |

CUPS web UI: `http://<pi-ip>:631/` (Administration tab asks for your Pi user).

## If captppd does not work for your unit

1. **Build captppd from source** on the Pi (needed only on 32-bit Trixie):

   ```sh
   sudo apt install git cmake g++ python3 libcups2-dev libusb-1.0-0-dev
   git clone https://github.com/darkvision77/captppd && cd captppd
   cmake -S. -B build && cmake --build build && sudo cmake --install build
   ```

2. **USB-over-IP instead of a CUPS driver.** Run VirtualHere USB Server (free
   for one device) or `usbip` on the Pi, attach the printer over the network
   from a Windows/macOS client, and use Canon's native LBP-1210 driver there.
   Only one client at a time can hold the printer, and the client must have
   the Canon driver installed. Keeps the bidirectional CAPT channel intact.

3. **Canon's x86 driver under emulation** (`qemu-user-static` + i386 chroot
   running `ccpd`/`captfilter`). Works for some people, is slow on a Pi 3B,
   and is fragile across CUPS/kernel updates. Last resort.

## Sources

- captppd: https://github.com/darkvision77/captppd (release 1.0.0 assets include arm64/armhf .deb)
- libcapt (protocol + SCoA): https://github.com/darkvision77/libcapt
- captdriver wiki stating LBP1210 needs SCoA and is unsupported there: https://github.com/mounaiban/captdriver/wiki
- Canon CAPT Linux driver 2.71 (x86 only): https://asia.canon/en/support/0100459601
- LBP-1120 on Raspberry Pi write-up (older Boichat driver, same printer family): https://sudonull.com/post/76280-Canon-Laser-Shot-LBP-1120-printer-and-Raspberry-Pi-based-print-server
