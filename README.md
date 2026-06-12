# NAS540 — UART Recovery & Custom Kernel Debug

**Device:** Zyxel NAS540
**SoC:** Marvell Comcerto 2000 EVM (ARMv7, Dual-Core)
**RAM:** 1 GB
**Bootloader:** Barebox
**OS:** Custom Debian (bullseye) — USB/SD boot

---

## Table of Contents

1. [Initial State](#initial-state)
2. [Why 6.6 Kernel?](#why-66-kernel)
3. [Upgrade Attempts](#66-kernel-upgrade-attempts)
4. [Kernel Panic Analysis](#error-analysis--kernel-panic)
5. [Recovery via UART](#recovery-process)
6. [The Real Fix: Editing Barebox Boot Script](#the-real-fix-editing-barebox-boot-script)
7. [Final System State](#final-system-state)
8. [Why 6.6 Never Worked](#why-66-never-worked)
9. [Reference Commands](#reference-commands)
10. [Key Takeaways](#key-takeaways)
11. [Source Guide Corrections](#source-guide--corrections)

---

## Initial State

- Custom Debian installed via forum guide (`nas5xx-debian-howto`), booting from USB
- Running kernel: **3.2.54** (custom nas5xx build)
- Boot flow: Barebox → `usb_key_func.sh` (on USB FAT partition) → USB EXT partition (Debian rootfs)
- **6.6.84+nas5xx** kernel files present on the system but inactive

---

## Why 6.6 Kernel?

- 3.2 kernel lacks XFS module (`/lib/modules/3.2.54` has no XFS)
- A newly installed 1.8TB disk was formatted as XFS
- 6.6 kernel has XFS: `/lib/modules/6.6.84+nas5xx/kernel/fs/xfs/xfs.ko`
- Goal: boot 6.6, mount the XFS disk, copy data

---

## 6.6 Kernel Upgrade Attempts

### Attempt 1 — `info_setenv next_bootfrom 2` (FAILED)

```bash
sudo /firmware/sbin/info_setenv next_bootfrom 2
sudo reboot
```

→ System booted with 3.2 again. This command only writes a Zyxel-specific environment variable, but **Barebox doesn't read it**. Useless.

### Attempt 2 — `zy-kernel2-write` (BRICKED)

```bash
sudo cp -p /boot/vmlinuz-6.6.84+nas5xx /boot/uImage
sudo bash /usr/local/bin/zy-kernel2-write
```

- 6.6 kernel written to NAND slot 2, `next_bootfrom=2` saved
- Bad block warning: `Bad block at 0x380000` (normal for NAND)
- **Reboot → NAS unresponsive. No SSH. No USB LED. Dead.**

---

## Error Analysis — Kernel Panic

Captured via serial cable (FT232RL, 115200 baud):

```
[5.160000] ubi0 error: ubi_read_volume_table: the layout volume was not found
[5.160000] ubi0 error: ubi_attach_mtd_dev: failed to attach mtd2, error -22
[5.450000] VFS: Cannot open root device "ubi0:rootfs" or unknown-block(0,253): error -19
[5.510000] Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,253)
```

**Root cause:** The default Barebox bootargs passed to the 6.6 kernel include `root=ubi0:rootfs` — it expects rootfs on NAND UBIFS. But rootfs is on a USB EXT4 partition. Kernel panic hits before the USB boot script even runs.

Secondary issue in the serial log:

```
[1.790000] ls1024a-usb3-phy: cannot get "phy" reset control: -517
```

`-517` = `EPROBE_DEFER` — the USB3 PHY driver never fully initializes on this kernel build. Even when manually specifying `root=/dev/sda2`, USB disconnects after 1.5–2 minutes.

---

## Recovery Process

### Step 1 — Fixing uImage on USB (DIDN'T WORK)

USB plugged into a Fedora machine:

- FAT partition (sda1): uImage = 6.6 kernel (7.58MB)
- EXT partition (sda2): `/boot/uImage` = 6.6 kernel (7.58MB)

Replaced both with the working 3.2 kernel image:

```bash
sudo cp /mnt/ext/boot/vmlinuz-3.2.0-6-nas5xx /mnt/ext/boot/uImage
file /mnt/fat/uImage
# → u-boot legacy uImage, Linux-3.2.0-6-nas5xx ✓
```

**Result:** NAS still didn't boot.
**Why:** uImage on USB is only a reference — the real kernel is loaded from **NAND**, and `next_bootfrom=2` was still pointing to it.

### Step 2 — UART Serial Connection

**Hardware:**

- FT232RL USB-UART adapter (3.3V)
- Jumper wires: TX→RX, RX→TX, GND→GND
- Pinout found on the PCB next to the buzzer (verify with multimeter, don't trust wire colors)

```bash
cu -s 115200 -l /dev/ttyUSB0
```

**Barebox exploration:**

```bash
# List available devices
devinfo

# List NAND partitions
ls /dev/
# → nand0.kernel1.bb, nand0.kernel2.bb, nand0.config, nand0.rootfs1, etc.

# Boot the old kernel (temporary fix — survives only until reboot)
bootm /dev/nand0.kernel1.bb
```

System booted with 3.2. But `next_bootfrom=2` meant the next reboot would load 6.6 again and panic.

### Step 3 — Trying to Flash kernel1 over kernel2 (FAILED)

Goal: overwrite kernel2 with the working 3.2 kernel so no matter what Barebox picks, it works.

```bash
# Barebox nand command didn't support raw copy
nand read /dev/nand0.kernel2.bb /dev/nand0.kernel1.bb
# Error: "nand: unknown command"
```

### Step 4 — Trying `zy-bb-env-write` (FAILED)

The guide mentions `zy-nand-get` to fetch Barebox environment, then `zy-bb-env-write` to flash it back with a patch.

```bash
# On the NAS (in Debian)
sudo /usr/local/bin/zy-nand-get
# → /boot/barebox directory was EMPTY
```

The Barebox environment on NAND was either incompatible or corrupted. `patch` also failed because the files weren't there.

### Step 5 — Attempted Fix via `info_setenv` (FAILED)

```bash
sudo /firmware/sbin/info_setenv next_bootfrom 1
# Output: "Open /dev/mtd2 void"
```

This writes to NAND config partition, but **Barebox doesn't read this variable**. Useless.

---

## The Real Fix: Editing Barebox Boot Script

After all failed attempts, the final solution was to **directly rewrite Barebox's boot script** using `echo` in the Barebox shell (the `edit` command was buggy on this firmware).

```bash
# Barebox shell — recreate /env/bin/boot line by line
echo -o /env/bin/boot "#!/bin/sh"
echo -a /env/bin/boot 'bootargs="root=/dev/sda2 rootwait rw console=ttyS0,115200n8 init=/etc/preinit pcie_gen1_only=yes mac_addr=,,"'
echo -a /env/bin/boot "bootm /dev/nand0.kernel1.bb"
saveenv
reset
```

**What this does:**

- `-o` overwrites the file, `-a` appends
- Bootargs: rootfs is `/dev/sda2` (USB), not NAND UBIFS
- `console=ttyS0,115200n8` fixes garbled serial output (those `␀` characters)
- `bootm /dev/nand0.kernel1.bb` forces Barebox to always load the working 3.2 kernel
- `saveenv` writes everything to NAND

**Result:** After `reset`, the NAS boots directly into Debian 3.2 kernel from USB. No key press needed. Survives power cycles. **Fully recovered.**

---

## Final System State

| Component            | Status                                                                                               |
| -------------------- | ---------------------------------------------------------------------------------------------------- |
| **Kernel**           | 3.2.0-6-nas5xx (kernel1, working)                                                                    |
| **Root filesystem**  | USB EXT4 partition (`/dev/sda2`)                                                                     |
| **Boot method**      | Barebox `/env/bin/boot` → `bootm /dev/nand0.kernel1.bb`                                              |
| **Boot args**        | `root=/dev/sda2 rootwait rw console=ttyS0,115200n8 init=/etc/preinit pcie_gen1_only=yes mac_addr=,,` |
| **Debian**           | Bullseye                                                                                             |
| **OMV**              | Web panel active (SMB open, NFS disabled)                                                            |
| **Serial console**   | Working with proper baud rate                                                                        |
| **Boot without USB** | Not supported (rootfs is on USB)                                                                     |

---

## Why 6.6 Never Worked

1. **Wrong default bootargs:** The 6.6 kernel's default command line pointed to `ubi0:rootfs` (NAND UBIFS), while rootfs is on USB EXT4.
2. **No initrd support in Barebox:** The new kernel apparently needs an initrd to find USB rootfs, but this old Barebox version has no `fatls` or initrd loading capability.
3. **xHCI USB3 PHY driver broken:** the 6.6 kernel build is missing a proper USB3 PHY reset control (`-517 EPROBE_DEFER`). Even with correct bootargs, USB disconnects after ~2 minutes.
4. **Barebox too old:** The bootloader would need to be updated to fully support the 6.6 kernel, but that's extremely risky on this platform.

**Bottom line:** The 6.6 kernel build was incomplete for this hardware.

---

## Reference Commands

**Barebox kernel selection:**

```bash
bootm /dev/nand0.kernel1.bb       # 3.2 kernel
bootm /dev/nand0.kernel2.bb       # 6.6 kernel
```

**Directly editing Barebox boot script (when edit command is broken):**

```bash
echo -o /env/bin/boot "#!/bin/sh"
echo -a /env/bin/boot 'bootargs="root=/dev/sda2 rootwait rw console=ttyS0,115200n8 init=/etc/preinit pcie_gen1_only=yes mac_addr=,,"'
echo -a /env/bin/boot "bootm /dev/nand0.kernel1.bb"
saveenv
reset
```

**Serial connection:**

```bash
cu -s 115200 -l /dev/ttyUSB0
```

**List NAND partitions from Debian:**

```bash
cat /proc/mtd
```

**For future kernel attempts with initrd:**

- Barebox would need `nand -l` (didn't exist on this build) or a Barebox update
- Alternative: pack initrd into the kernel image itself

---

## Key Takeaways

- ✅ UART pinout identification and FT232 serial connection
- ✅ Barebox bootloader commands and scripting
- ✅ Complete recovery from a bricked NAS (kernel panic → serial → boot script rewrite)
- ✅ Kernel panic analysis: wrong rootfs target, USB PHY driver failure
- ✅ Barebox `/env/bin/boot` script editing via `echo`
- ✅ NAND bad block handling
- ✅ USB boot vs NAND boot architecture on embedded ARM
- ✅ The guide's instructions were incomplete/inaccurate for this hardware revision

---

## Source Guide & Corrections

This project followed the **nas5xx-debian-howto** ([German forum, Seafile distribution](https://seafile.servator.de/nas/zyxel/images/nas5xx-debian-howto.txt)).

### Inaccuracies found in the original guide

| #   | Topic                           | What the guide says                                          | What actually happened                                                                           |
| --- | ------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------ |
| 1   | **Teardown / clips**            | Teardown instructions for one revision don't match all units | Plastic clip layout differs — had to figure it out independently                                 |
| 2   | **UART pinout**                 | Provided pin diagram and wire colors                         | Always verify with a multimeter; colors vary                                                     |
| 3   | **`next_bootfrom` command**     | Claims `info_setenv next_bootfrom 2` is sufficient           | Barebox ignores this variable entirely. Only `zy-kernel2-write` sets it                          |
| 4   | **Kernel 6.x compatibility**    | Guide says newer kernels work                                | 6.6 kernel has broken xHCI USB3 PHY driver on this SoC — cannot keep USB stable                  |
| 5   | **Serial recovery (`b1`)**      | Says `b1` boots the old kernel                               | `b1` → `command not found` (patch not applied on this revision)                                  |
| 6   | **Barebox environment**         | `zy-nand-get` should fetch Barebox env                       | `/boot/barebox` was empty — env missing or incompatible                                          |
| 7   | **ZyXEL scripts unreliability** | Guide assumes scripts work universally                       | `zy-bb-env-write` couldn't patch because files didn't exist. Direct Barebox editing was required |

### The real fix (not in the guide)

Directly rewriting `/env/bin/boot` in Barebox using `echo -o` / `echo -a` — because the guide's `b1`, `zy-bb-env-write`, and `info_setenv` all failed on this revision.
