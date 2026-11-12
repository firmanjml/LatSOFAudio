# LatSOFAudio source port for HP EliteBook 840 G6

This experimental source port targets the HP EliteBook 840 G6 board 8549 and PCI `8086:9DC8` / subsystem `103C:8549`. It is based on upstream commit `41fda77bd5321464fef5ba71a44437f993215d55`. The hp840g6-cnl branch contains the port. This is an HP-specific variant with the same bundle identifier as upstream. [Pinned upstream source](https://github.com/shubhambanekar/LatSOFAudio/tree/41fda77bd5321464fef5ba71a44437f993215d55).

## Verified reference inputs

The Ubuntu `firmware-sof-signed` package version `2025.12.2-1` contains both files below. Their hashes exactly match the files measured by the collector on the HP. The archive's default IPC3 link resolves to the Intel-signed image. The source package does not include either binary. [Ubuntu package information](https://packages.ubuntu.com/resolute/firmware-sof-signed).

| Input | SHA-256 |
| --- | --- |
| Intel-signed `sof-cnl.ri`, 557,824 bytes | `c5119cfca9c4bdf5ab87a87af08fb8fb2bf92767e0f070ea4eb52a877f10359e` |
| `sof-hda-generic-4ch.tplg`, 44,383 bytes | `7b1b185b35f1034ca1957f41e75c184f79ea1cae7b38bcef995acf7481145d20` |

The topology's `dmic01` link, ID 6, specifies a 48 kHz, 32-bit FIFO, a 2.4–4.8 MHz PDM clock range, 40–60% duty limits, a 200 ms unmute ramp, and controllers 0 and 1 with both inputs enabled. Both controllers have zero polarity, edge, and skew fields. The ACPI NHLT independently confirms PDM0 A/B and PDM1 A/B. The topology clock range is a constraint for SOF. The 2.4 MHz inferred from the NHLT divider is one programmed setting. [Linux IPC3 topology token mapping](https://github.com/torvalds/linux/blob/master/sound/soc/sof/ipc3-topology.c).

For a later authorized build, obtain the matching Intel-signed firmware and place it at `kext/LatSOFAudio/Firmware/sof-cnl.ri`. The Makefile embeds that file. The topology binary was used as a reference for the typed IPC3 DMIC request. This candidate retains LatSOFAudio's minimal capture graph in source and does not load a Linux topology binary at runtime.

## Source changes

- Limits device discovery to HP `8086:9DC8` / `103C:8549` and switches the firmware dependency to `sof-cnl.ri`.
- Uses the measured topology parameters for a typed, 216-byte IPC3 DMIC0 request. Carries four interleaved signed 32-bit channels at 48 kHz through HDA format `0x0043`, PCM parameters, the 16-byte DMA frame stride, and the Core Audio engine and filter state. The first candidate publishes only the 48 kHz path.
- Checks HDA BAR sizes and walks the linked capability list. Defers the code loader while the first output descriptor is programmed or an output is running. Takes the highest GCAP-reported input descriptor only if its visible configuration and tag checks pass. Keeps AppleHDA's PCI command and power settings intact and omits the Dell codec workaround.
- Requires successful topology replies and handles rejected or timed-out capture start IPCs. Removes publication of raw sample words through IORegistry and removes the legacy UserClient from the personality.

## Validation still required

The Windows-side checks cover source consistency and application of the patch to the pinned base. The kext has never been compiled, linked, installed, or run. The Windows and Ubuntu investigations did not record audio.

Test on a separately authorized macOS Sequoia setup after reviewing the load and rollback procedure. First confirm AppleHDA playback, then check the HP's HDA capability layout, firmware-ready IPC, stream ownership and device-visible DMA addressing under the active IOMMU. The driver still inherits the upstream code-loader, mailbox, DMA and power-management implementation. Visible descriptor checks do not confer exclusive ownership against AppleHDA. The conservative policy can leave the mic unavailable if AppleHDA retains the loader descriptor, including after sleep. Verify concurrent playback, channel order and signal, repeated start/stop, cleanup and sleep/wake before calling the port usable. The upstream README describes the Dell reference and should not be used as HP installation instructions.
