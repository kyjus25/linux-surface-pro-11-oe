---
id: how-to-bring-up-audio
title: "How To: Bring Up Audio on Surface Pro 11"
# prettier-ignore
description: How-to guide for installing Surface Pro 11 audio as a paired kernel + sp11-audio release (ADR0064), migrating off the retired CRD workaround stack, validating speakers and microphone, and rolling back.
---

# How To: Bring Up Audio on Surface Pro 11

Last updated: 2026-08-27

Surface Pro 11 audio is deployed as **two paired, immutable releases**:

- the kernel bundle, for example
  [`sp11-qcom-x1e-7.2.0-jg-0sp11v12`](https://github.com/ooaklee/linux-surface-pro-11-oe/releases/tag/sp11-qcom-x1e-7.2.0-jg-0sp11v12);
- the matching audio release, for example
  [`sp11-audio-v19c`](https://github.com/ooaklee/linux-surface-pro-11-oe/releases/tag/sp11-audio-v19c)
  (FullIO v19c topology + UCM).

The pairing is a contract: the kernel alone is not a working audio install.
`audioreach_tplg_init` requests `qcom/<card>-tplg.bin` when the sound card
probes at boot, and the UCM files provide the speaker, microphone, and
volume routes. The kernel release notes always name the compatible audio
release. See
[ADR0064](../adr/adr-0064-sp11-audio-release-strategy.md).

## Prerequisites

- A Surface Pro 11 running the installed Ubuntu system with the paired
  kernel installed (see
  [Reinstall Patched Kernel](how-to-reinstall-patched-kernel-from-usb.md)).
- aDSP/cDSP firmware in place (one-time install; see
  [Install Surface Pro 11 Firmware](how-to-install-sp11-firmware.md)).
- Root access for `/lib/firmware` and `/usr/share/alsa/ucm2`.

## Procedure

### 1. Download and verify the paired audio release

```bash
base=https://github.com/ooaklee/linux-surface-pro-11-oe/releases/download/sp11-audio-v19c
for f in SHA256SUMS X1E80100-Microsoft-Surface-Pro-11-tplg.bin \
         MICROSOFT-Surface-Pro-11in.conf SP11-HiFi.conf x1e80100.conf; do
  curl -fsLO "$base/$f"
done
sha256sum -c SHA256SUMS
```

`sha256sum -c` must pass for all four installable files. If it fails,
stop; do not install an unverified topology under the canonical firmware
name.

### 2. Back up any previous audio files

Preserve the current state before overwriting anything:

```bash
backup=/var/backups/sp11-audio-$(date '+%Y%m%d-%H%M%S')
sudo mkdir -p "$backup"
for f in \
  /lib/firmware/qcom/x1e80100/X1E80100-Microsoft-Surface-Pro-11-tplg.bin \
  /usr/share/alsa/ucm2/Qualcomm/x1e80100/MICROSOFT-Surface-Pro-11in.conf \
  /usr/share/alsa/ucm2/Qualcomm/x1e80100/SP11-HiFi.conf \
  /usr/share/alsa/ucm2/Qualcomm/x1e80100/Surface11-HiFi.conf \
  /usr/share/alsa/ucm2/Qualcomm/x1e80100/MICROSOFT-Surface-Pro-11.conf \
  /usr/share/alsa/ucm2/conf.d/x1e80100/x1e80100.conf; do
  if [ -e "$f" ]; then
    sudo mkdir -p "$backup$(dirname "$f")"
    sudo cp -a "$f" "$backup$f"
  fi
done
```

### 3. Install the four files

```bash
sudo install -Dm0644 X1E80100-Microsoft-Surface-Pro-11-tplg.bin \
  /lib/firmware/qcom/x1e80100/X1E80100-Microsoft-Surface-Pro-11-tplg.bin
sudo install -Dm0644 MICROSOFT-Surface-Pro-11in.conf \
  /usr/share/alsa/ucm2/Qualcomm/x1e80100/MICROSOFT-Surface-Pro-11in.conf
sudo install -Dm0644 SP11-HiFi.conf \
  /usr/share/alsa/ucm2/Qualcomm/x1e80100/SP11-HiFi.conf
sudo install -Dm0644 x1e80100.conf \
  /usr/share/alsa/ucm2/conf.d/x1e80100/x1e80100.conf
```

If a legacy `Surface11-HiFi.conf` exists, overwrite it with the new
`SP11-HiFi.conf` content so a stale copy can never load:

```bash
sudo install -m0644 SP11-HiFi.conf \
  /usr/share/alsa/ucm2/Qualcomm/x1e80100/Surface11-HiFi.conf
```

### 4. Retire the CRD-era workaround stack (older installs only)

Installs that predate the native pairing may still carry the retired CRD
workaround stack. It must not run alongside the paired topology:

```bash
rm -f ~/.config/pipewire/pipewire.conf.d/50-sp11-*.conf
rm -f ~/.config/wireplumber/wireplumber.conf.d/51-sp11-*.conf
sudo systemctl disable --now sp11-wsa-routing.service 2>/dev/null || true
sudo rm -f /etc/systemd/system/sp11-wsa-routing.service \
  /etc/systemd/system/multi-user.target.wants/sp11-wsa-routing.service
sudo systemctl daemon-reload
```

The legacy `50-sp11-speakers.conf` sink is fatal on the native pairing:
its target PCM does not exist under the paired topology, so PipeWire exits
with status 234 and crash-loops. The legacy routing service applies
CRD-era mixer routes and opens PCM1 at every boot, fighting the protected
speaker graph.

### 5. Reboot

```bash
sudo reboot
```

The topology is loaded by the AudioReach DSP when the sound card probes at
boot. A reboot is required after every topology change, even if PipeWire
restarts cleanly.

## Validation

```bash
uname -r                 # the paired kernel release
wpctl status             # a real sink, not Dummy Output
speaker-test -D default -c 2 -t sine -f 440 -s 1 -l 1   # left speaker only
speaker-test -D default -c 2 -t sine -f 440 -s 2 -l 1   # right speaker only
sudo dmesg | grep -Ei 'SP11 stage|SPVI|no backend' | tail -15
```

The dmesg check must include `SP11 stage SP/SPVI enabled with VI+CPS
feedback accepted` and no `no backend DAIs` messages.

Confirm the feedback-port Offset2 boot parameter reached the kernel (it
prevents volume-change pops):

```bash
grep -o 'soundwire_qcom[^ ]*' /proc/cmdline
cat /sys/module/soundwire_qcom/parameters/sp11_feedback_active_offset2_zero   # expect Y
```

For the internal microphone, check that the `HiFi` verb exposes `Speaker`
and `Mic`:

```bash
alsaucm -c hw:0 set _verb HiFi list _devices
wpctl status | grep -A6 Sources
```

## Troubleshooting

### Dummy Output after reboot

- Confirm the installed topology matches the release hash:
  `sha256sum /lib/firmware/qcom/x1e80100/X1E80100-Microsoft-Surface-Pro-11-tplg.bin`.
- `sudo dmesg | grep -iE 'tplg|qcom-apm'` for topology load errors; the
  boot-time opcode `0x1001021` is only the SPF readiness query, not a
  playback-graph failure.
- Confirm the UCM matcher selected the card:
  `alsaucm -c hw:0 set _verb HiFi list _devices`.

### PipeWire exits with status 234 in a crash loop

A legacy `50-sp11-*.conf` workaround file survived the migration. Remove
it (step 4) and restart the user audio services:

```bash
systemctl --user restart pipewire pipewire-pulse wireplumber
```

### Volume-change pops after reboot

The `soundwire_qcom.sp11_feedback_active_offset2_zero=1` boot parameter is
missing from the kernel command line. The support flow adds it via
`/etc/default/grub.d/99-surface-pro-11.cfg`; re-add it, then run
`sudo update-grub && sudo /usr/local/sbin/sp11-grub-inject-dtb`.

## Rollback

Restore the files saved in step 2 from the backup directory, then reboot so
the restored topology reloads at card probe. The previous kernel remains
installed by design; select it from the GRUB advanced menu for a
known-good fallback boot.

## Relationship to older audio lines

- **CRD workaround stack (retired).** The CRD topology, OE-authored UCM,
  manual PipeWire speaker sink, and WSA routing service were the original
  bring-up path
  ([ADR-0033](../adr/adr-0033-audio-topology-gap.md),
  [ADR-0035](../adr/adr-0035-audio-boot-race-alsactl.md),
  [ADR-0036](../adr/adr-0036-right-speaker-audio-position-reorder.md),
  [ADR-0044](../adr/adr-0044-sp11-ucm-single-wsa-macro-microphone.md)).
  ADR0064 retired this stack in favor of the paired releases.
- **Golden v32 v9/v10 pairing (superseded for v12+).** Documented in
  [`how-to-migrate-to-native-audio`](how-to-migrate-to-native-audio.md) and
  [ADR-0062](../adr/adr-0062-sp11-7-2-0-jg-0sp11v9-golden-v32-audio-line.md).
  Its topology defined no VA/DMIC capture graph, so the internal microphone
  was unavailable on that line
  ([issue #48](https://github.com/ooaklee/linux-surface-pro-11-oe/issues/48));
  the v19c pairing restores it.
- **DMIC clock.** The 2.4 MHz Denali DMIC clock remains the validated
  default; 4.8 MHz causes continuous capture static
  ([ADR-0045](../adr/adr-0045-sp11-2p4mhz-dmic-clock-test-kernel.md),
  [ADR-0046](../adr/adr-0046-sp11-default-2p4mhz-dmic-clock.md)). Capture
  remains slightly tinny or thin.

## References

- [ADR0064: Dedicated SP11 Audio Release Strategy](../adr/adr-0064-sp11-audio-release-strategy.md)
- Kernel release: [sp11-qcom-x1e-7.2.0-jg-0sp11v12](https://github.com/ooaklee/linux-surface-pro-11-oe/releases/tag/sp11-qcom-x1e-7.2.0-jg-0sp11v12)
- Audio release: [sp11-audio-v19c](https://github.com/ooaklee/linux-surface-pro-11-oe/releases/tag/sp11-audio-v19c)
- [Install Surface Pro 11 Firmware](how-to-install-sp11-firmware.md)
- [Publish the SP11 Audio Release](../../scripts/publish-sp11-audio-release.sh)
