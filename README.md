`omi-firmware` publishes signed builds of a fork of the [Omi](https://github.com/BasedHardware/omi) CV1 pendant firmware (nRF5340), for Omi pendant owners who use the Omi Capture Android app. The fork makes button gestures work without a phone connected and refuses firmware updates until the owner taps the pendant, and this repository's releases are where Omi Capture installs it from.

# What It Does

- **Release packages**: Each release, tagged `firmware-v<revision>` (for example `firmware-v3.0.22.7`), carries `dfu_application.zip`, an MCUboot DFU package holding only the application core image (image 0), `dfu_application.zip.sha256`, and `THIRD_PARTY_NOTICES.md`, the notices of the third-party code in that image.
- **Gestures without a phone**: A double tap stores a bookmark and a triple tap a track request next to the audio on the pendant; both reach the phone with its next sync.
- **Sync request**: A triple tap with no phone connected advertises a "sync requested" flag for up to 120 s, so a nearby phone can connect and sync at once.
- **Update lock**: From `3.0.22.7`, Bluetooth firmware updates are refused until five quick taps open a five-minute update window.
- **Haptic confirmations**: Each gesture and the update window opening and closing have their own buzz pattern.
- **Explicit release**: When the app asks for it, the pendant frees stored audio only after the app confirms it holds it.

# How It Works

1. Omi Capture's **Pendant** page, with the **Custom** source selected, lists this repository's releases and offers the newest one.
2. You tap the pendant five times; it buzzes twice (long) and accepts an update for five minutes.
3. Omi Capture uploads the image over Bluetooth and the pendant restarts into it.

# How it Really Works

- The firmware revision is `<upstream version>.<fork build>`, for example `3.0.22.7`, and the image is signed `<upstream version>+<fork build>`, for example `3.0.22+7`. Stock firmware revisions have three parts, so a fourth marks this fork.
- The pendant's bootloader refuses an image whose `major.minor.revision` is lower than the installed one and ignores the build number, so a fork build and the stock release of the same base replace each other.
- Images are signed with MCUboot's public sample key, the key the pendant's bootloader trusts. Anyone can sign images for it; the update lock is what keeps others in Bluetooth range from installing firmware outside the five-minute window.
- The update window lives in RAM: every boot starts locked, including the restart that installs an update. Safe mode, entered after three failed boots or by holding the button for 3 s at power-on, accepts updates without the taps so a broken image can always be replaced.
- Only image 0 ships. The network core image (image 1) is never included, because a bad one cannot be repaired over Bluetooth.
- Omi Capture reads each release's signed version from the package with HTTP Range requests before offering it, and checks the downloaded image against the bootloader rule before it flashes.

# Prerequisites

- An Omi CV1 pendant. The firmware runs on no other Omi device.
- Omi Capture built from a revision that reads this repository's releases and version 3 event records. Older Omi Capture builds skip the event records this firmware writes and still free them from the pendant, so every bookmark and track request made before the app is updated is lost.
- The pendant charged above 15%.

# Installation

> **Warning:** Accept no stock Omi update whose version is above this fork's base (for example stock `3.0.23` over a fork built on `3.0.22`). Its bootloader then refuses every custom image until a custom build on that stock version or later exists.

With Omi Capture:

1. Update Omi Capture first (see Prerequisites).
2. Turn off Omi Capture's live streaming and close every other app connected to the pendant.
3. On the **Pendant** page, select the **Custom** source and tap **Check releases**.
4. Start the install. When the pendant runs `3.0.22.7` or later, Omi Capture shows **Tap the pendant 5 times to allow the update**: tap the button five times quickly and wait for two long buzzes. Stock firmware and fork builds before `3.0.22.7` have no update lock and skip this step.
5. Wait for the upload and the restart. The **Pendant** page then shows the new revision.

With another MCUmgr client, such as [nRF Connect Device Manager](https://www.nordicsemi.com/Products/Development-tools/nRF-Connect-Device-Manager):

1. Download `dfu_application.zip` and `dfu_application.zip.sha256` from the release and check the package:

   ```sh
   sha256sum -c dfu_application.zip.sha256
   ```

2. Close every app connected to the pendant, Omi Capture included.
3. On a pendant running `3.0.22.7` or later, tap the button five times quickly and wait for two long buzzes.
4. In Device Manager, select the pendant, choose **Image** → **Firmware Upgrade**, pick `dfu_application.zip` and select **Test and Confirm** (or **Confirm only**).
5. Wait for the upload and the restart. An upload that fails with `ACCESS_DENIED` met a closed update window: tap five times again and restart it.

# Rollback

"Allowed" means the pendant's bootloader installs the candidate over the installed image.

| Installed | Candidate | Result |
| :--- | :--- | :--- |
| Stock signed `0.0.0+0` (3.0.21 and every older published CV1 release) | Custom `3.0.22+n` | Allowed |
| Custom `3.0.22+n` | Stock signed `0.0.0+0` | **Refused** |
| Custom `3.0.22+n` | Stock 3.0.22 signed `3.0.22+0` | Allowed |
| Stock 3.0.22 signed `3.0.22+0` | Custom `3.0.22+n` | Allowed |
| Custom `3.0.22+n` | Custom `3.0.22+m`, any `m` | Allowed |
| Custom `3.0.22+n` | Stock 3.0.23 or later | Allowed |
| Stock 3.0.23 or later | Custom `3.0.22+n` | **Refused** until a custom build on 3.0.23 or later exists |

- **Another fork build**: install any release here, older or newer, the same way.
- **Back to stock**: every CV1 release published by Based Hardware up to 3.0.21 is signed `0.0.0+0` and is refused over this fork. Return to stock with the unmodified upstream 3.0.22 built and signed `3.0.22+0`, which this repository does not publish; Omi Capture installs it only through **Restore stock firmware (factory reset)** on its **Pendant** page, which syncs the pendant first and asks for a typed confirmation and the five taps. The stock pendant then has no update lock.
- **Safe mode**: when the firmware does not start, the pendant enters safe mode after three failed boots, or when the button is held for 3 s at power-on. Safe mode accepts an update without the five taps; install a working release as above.
- **SWD**: when the pendant neither boots nor advertises, the last resort is opening the case and flashing it with a J-Link.

MCUboot overwrites the old image during an update and keeps no copy, so a power loss during that copy can leave no bootable image. Install with the pendant charged.

# License

[MIT](LICENSE), the license of the upstream Omi firmware. The images also contain third-party components under their own licenses: [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md) for the newest release, and each release's own `THIRD_PARTY_NOTICES.md` asset for its image.
