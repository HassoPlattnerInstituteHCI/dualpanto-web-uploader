# DualPanto Web Firmware Uploader

A GitHub-hosted, browser-based firmware uploader for the [DualPanto](https://github.com/HassoPlattnerInstituteHCI/dualpantoframework) —
the web version of the [dualpanto-testing](https://github.com/HassoPlattnerInstituteHCI/dualpanto-testing) upload workflow
(`python -m unittest test_firmware.UploadDualPantoFramework`), with **no local setup at all**:
no Python, no PlatformIO, no drivers configuration in an IDE.

**How it works for the user:**

1. Open the GitHub Pages site in Chrome or Edge.
2. Pick the **branch** (= version) of `dualpantoframework` you want.
3. Click **Connect** and pick the DualPanto's **serial port** in the browser dialog.
4. Hold the flash button on the back of the panto, click **Upload firmware** — done.

## Architecture

GitHub Pages can only serve static files — it cannot run a C++ compiler. So compilation
and uploading are split:

```
┌────────────────────────────────────────────────────────────────────┐
│ GitHub Actions (.github/workflows/build-firmware.yml)              │
│                                                                    │
│  list all branches of dualpantoframework                           │
│    └─> matrix: for each branch                                     │
│          checkout → pio run -e esp32dev  (same as the PyCharm flow)│
│          collect bootloader.bin / partitions.bin /                 │
│                  boot_app0.bin / firmware.bin                      │
│    └─> generate manifest.json (branch, commit, build date, files)  │
│    └─> deploy site + binaries to GitHub Pages                      │
└────────────────────────────────────────────────────────────────────┘
                                │ static hosting
                                ▼
┌────────────────────────────────────────────────────────────────────┐
│ Browser (site/index.html)                                          │
│                                                                    │
│  loads manifest.json → branch dropdown                             │
│  Web Serial API → user picks the COM port                          │
│  esptool-js → flashes the ESP32 at 0x1000/0x8000/0xE000/0x10000    │
│  (with MD5 verification) → hard reset → optional serial monitor    │
└────────────────────────────────────────────────────────────────────┘
```

Because the firmware is **pre-compiled per branch** by CI, uploading takes seconds and
nobody needs a GitHub account or token. The trade-off: the site serves the state of each
branch **as of the last workflow run** (each branch's commit hash and build time are shown
in the UI, and the nightly schedule keeps it fresh).

## Setup (one-time)

1. **Create a new GitHub repository** (e.g. `dualpanto-web-uploader`, public) and push this
   folder's contents to it:

   ```bash
   cd dualpanto-web-uploader
   # if the workflow arrived in "github/" instead of ".github/", rename it first:
   [ -d github ] && mv github .github
   git init && git add -A && git commit -m "DualPanto web firmware uploader"
   git branch -M main
   git remote add origin https://github.com/<you>/dualpanto-web-uploader.git
   git push -u origin main
   ```

2. **Enable GitHub Pages via Actions:** repo → *Settings* → *Pages* → under **Build and
   deployment**, set *Source* to **GitHub Actions**.

3. **Run the workflow once:** repo → *Actions* → *Build firmware & deploy site* →
   *Run workflow*. It compiles every branch of `dualpantoframework` (~10–20 min the first
   time; branches that don't compile are listed as "failed" in the manifest and simply
   don't appear in the dropdown).

4. Open `https://<you>.github.io/dualpanto-web-uploader/` — done.

### Keeping binaries fresh

- The workflow re-builds **every night** (03:23 UTC) and on every push to this repo.
- You can always trigger a rebuild manually via *Actions → Run workflow* — optionally with
  a **branch filter** regex (e.g. `^(master|develop|BIS)$`) for a quick partial rebuild.
- *(Optional, needs write access to the firmware repo)* For instant rebuilds when the
  firmware changes, add a small workflow to `dualpantoframework` that sends a
  `repository_dispatch` event of type `firmware-updated` to this repo:

  ```yaml
  # dualpantoframework/.github/workflows/notify-uploader.yml
  on: push
  jobs:
    notify:
      runs-on: ubuntu-latest
      steps:
        - run: |
            curl -X POST -H "Authorization: Bearer ${{ secrets.UPLOADER_PAT }}" \
              -H "Accept: application/vnd.github+json" \
              https://api.github.com/repos/<you>/dualpanto-web-uploader/dispatches \
              -d '{"event_type":"firmware-updated"}'
  ```

## Configuration

- **Firmware repo:** set in one place, the `FIRMWARE_REPO` env var at the top of
  [`.github/workflows/build-firmware.yml`](.github/workflows/build-firmware.yml)
  (and mirrored in the checkout step). Point it at a fork if needed.
- **Flash offsets:** the standard arduino-esp32 1.x layout (bootloader `0x1000`,
  partition table `0x8000`, `boot_app0` `0xE000`, app `0x10000`) is written into each
  branch's `info.json` by CI, so the web page stays layout-agnostic.
- **Hardware config:** some branches (e.g. `BIS`) have `firmware/include/config/config.hpp`
  checked in; others (e.g. `develop`, `BIS22*`) generate it at build time from a hardware
  description JSON. For those, CI runs the branch's own generator
  (`node utils/scripts/generateHardwareConfig.js <target>`) with target **`fiona`** —
  the same config the BIS class devices use (the `BIS` branch's checked-in `config.hpp`
  hash matches `hardware/fiona.json` exactly). Override the target per run via the
  workflow's `hardware_target` input. The UI shows which config each build used
  (`hw: fiona` = generated, `hw: branch default` = checked in).
- **Legacy branches:** very old branches (`master`, `jotaro-develop`, `ember-2020`, …) use
  the ancient Arduino-IDE project layout without `firmware/platformio.ini` — they can't be
  built by this pipeline (nor by the classic PyCharm flow) and are listed under "failed"
  in the manifest. The project's actual mainline is the `BIS` / `develop` line.

## Browser support & limitations

- Requires the **Web Serial API**: Chrome, Edge, Opera (desktop). Safari and Firefox do
  not support it. GitHub Pages serves over HTTPS, which Web Serial requires.
- The CP210x driver is still needed on the computer (as in the classic workflow) so the
  serial port exists at OS level:
  [Silicon Labs CP210x VCP drivers](https://www.silabs.com/developers/usb-to-uart-bridge-vcp-drivers?tab=downloads).
- As with the PyCharm flow, you may need to **hold the flash button** on the back of the
  DualPanto while the flasher connects.
- If the upload fails at the default 460800 baud, lower the baud rate to 115200 in the UI.

## Repository layout

```
.github/workflows/build-firmware.yml   CI: compile all branches + deploy Pages
site/index.html                        the flasher web app (single file)
site/vendor/esptool-js/                vendored esptool-js 0.6.0 bundle (Apache-2.0)
```

## Local development

```bash
cd site
python3 -m http.server 8000     # Web Serial needs http://localhost or https://
```

`manifest.json` won't exist locally; either copy one from a Pages deployment or create a
small fake one to test the UI.

## Credits

- Flashing engine: [esptool-js](https://github.com/espressif/esptool-js) (Espressif, Apache-2.0), vendored in `site/vendor/esptool-js/`.
- Firmware: [dualpantoframework](https://github.com/HassoPlattnerInstituteHCI/dualpantoframework) (HPI HCI).
- Classic workflow reference: `dualpanto-testing` (see `BIS.md` / `Readme.md` there).
