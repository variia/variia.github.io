---
title: "Restoring HDMI-CEC on Raspberry Pi OS Bookworm with FKMS"
date: 2026-02-22 11:00:00 +0100
author: Iván Ádám Vári
description: Custom libcec packages that restore HDMI-CEC (TV remote control) functionality on Raspberry Pi OS Bookworm when using the FKMS display driver instead of KMS.
keywords: raspberry-pi,hdmi,cec,fkms,kms,kodi,libcec,bookworm
categories:
  - Raspberry Pi
tags:
  - raspberry-pi
  - linux
  - kodi
  - hdmi
---

Raspberry Pi OS moved from the FKMS display driver (`vc4-fkms-v3d`) to KMS (`vc4-kms-v3d`). For most users this is fine, but
if you run Kodi as a media center, KMS brings some unwelcome side effects — elevated CPU load from D-state wait inflation,
colour accuracy issues with certain TVs, and a generally less mature video playback path compared to the older driver.

Reverting to FKMS fixes all of that, but breaks one thing: **HDMI-CEC stops working**. Your TV remote can no longer control Kodi,
because the standard `libcec` package only knows how to talk to the kernel's `/dev/cec0` device — which FKMS does not create.

I rebuilt `libcec` with the deprecated `HAVE_RPI_API` flag enabled, which activates the firmware-level VCHI CEC adapter that
bypasses the kernel path entirely. The result is a set of drop-in `.deb` packages that restore full CEC functionality on
Bookworm with FKMS.

The packages are available as an APT repository hosted on this domain, so installation is a few commands:

```bash
echo "deb [trusted=yes] https://ivanvari.com/rpi-libcec-fkms/repo bookworm main" \
  | sudo tee /etc/apt/sources.list.d/rpi-libcec-fkms.list
sudo apt-get update
sudo apt-get install libcec6 cec-utils
```

Full documentation, build instructions, and source are on GitHub:
**[github.com/variia/rpi-libcec-fkms](https://github.com/variia/rpi-libcec-fkms)**
