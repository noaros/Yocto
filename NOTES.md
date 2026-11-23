# Embedded Linux Portfolio — Project Notes

Goal: switch from traditional software to embedded Linux by building resume projects
on a **BeaglePlay** with **Yocto**.

_Last updated: 2026-11-23_

---

## Current status

- [x] Picked a board: **BeaglePlay** (TI AM625)
- [x] Ordered parts from SparkFun (see [Parts](#parts))
- [ ] **Qwiic BME280 sensor: out of stock. Order later** (SparkFun SEN-15440)
- [ ] Set up the Yocto build host (Ubuntu 22.04/24.04, 32GB RAM ideally, 100GB+ free disk)
- [ ] Set up this repo: Scarthgap + `meta-ti` + a custom layer for `MACHINE = "beagleplay"`
- [ ] Do the first build and boot it from microSD

**Next step:** set up the repo and start the first Yocto build on the PC while the parts ship.

---

## Hardware decisions

| Option | Verdict | Why |
|---|---|---|
| **BeaglePlay** | **Chosen** | 64-bit Cortex-A53, open K3 boot chain (TF-A + OP-TEE + U-Boot), 16GB eMMC, Gigabit Ethernet + Wi-Fi, Qwiic/Grove/mikroBUS, M4F + R5 cores |
| BeagleBone Black | Not chosen | Simpler boot and a reference machine in Poky, but 32-bit Cortex-A8 from 2013, so it looks dated on a resume |
| PocketBeagle 2 | Not chosen | Cheap and tiny, but microSD only, no networking, newer and less mature Yocto support |
| Raspberry Pi | Not chosen | Boots from a closed GPU firmware blob, limited documentation, reads as a hobby board |
| STM32 Nucleo | Not for Linux | Cortex-M has no MMU, so no real Linux. Useful later for the FreeRTOS project |
| STM32MP157F-DK2 / MP257F-DK | Alternative | Real Linux boards from ST with Yocto (OpenSTLinux) and A + M cores |

### BeaglePlay facts to remember
- SoC: TI AM625. 4x Cortex-A53 at 1.4GHz, plus Cortex-M4F and Cortex-R5. 2GB DDR4, 16GB eMMC
- Boot chain (TI K3): R5 SPL → A53 SPL → TF-A → OP-TEE → U-Boot → Linux
- Yocto needs **`meta-ti`** and a **multiconfig** build, because the R5 core uses its own toolchain (`beagleplay-k3r5`)
- Ships with Debian on the eMMC
- **Boot from microSD:** hold the **USR button** while powering on
- Debug UART: 3-pin header, 3.3V, **115200 baud**

---

## Parts

All from SparkFun except where noted.

| Item | SKU | Price | Status |
|---|---|---|---|
| BeaglePlay | DEV-22659 | $99.00 | Ordered |
| USB-to-TTL serial cable, **3.3V** | CAB-12977 | $18.95 | Ordered |
| USB-C wall adapter, 5.1V/3A | TOL-16272 | $4.25 | Ordered |
| Flexible Qwiic cable, 200mm | PRT-17258 | $1.95 | Ordered |
| microSD card, 32GB | (any name brand) | ~$10 | Ordered |
| **Qwiic BME280 sensor** | **SEN-15440** | $16.95 | **Out of stock, still needed** |

Links:
- BeaglePlay: https://www.sparkfun.com/beagleplay.html
- Serial cable: https://www.sparkfun.com/usb-to-ttl-serial-cable.html
- Power supply: https://www.sparkfun.com/usb-c-wall-adapter-5-1v-3a-black.html
- Qwiic cable: https://www.sparkfun.com/flexible-qwiic-cable-200mm.html
- BME280 (Qwiic): https://www.sparkfun.com/sparkfun-atmospheric-sensor-breakout-bme280-qwiic.html

### Notes on parts
- **Serial cable:** do **not** use SparkFun DEV-09717 (it has 5V on VCC). Connect only **GND, TX and RX**,
  with TX and RX crossed. Leave the VCC wire unconnected. Open the console with `picocom -b 115200 /dev/ttyUSB0`.
- **Why the serial cable matters:** it's the only way to see bootloader and early kernel output, and to
  use the U-Boot prompt. SSH only works once Linux and the network are up.
- **Why the microSD card matters:** flash `.wic` images to it, boot with USR held down, recover a broken
  eMMC, and use it to write images to the eMMC. It also keeps the factory Debian on the eMMC untouched.
- **Other places to buy the BME280:** DigiKey part SEN-15440, or Adafruit's STEMMA QT BME280 (also Qwiic-compatible).
  The ENS160/BME280 combo board (SparkFun) also works. I²C address: **0x77** (alternate 0x76).
- **Other places to buy the BeaglePlay:** DigiKey part **102110762** (don't order TI part BEAGL-PLAY-SBC), Newark, or RS.
- **Optional later:** an STM32 Nucleo board (e.g. F446RE or H723ZG) for the FreeRTOS project, and a
  cheap logic analyzer (~$15) for driver debugging.

---

## Projects (in order)

Each project builds on the previous one. It's better to finish 2–3 projects in depth than to start all 6.
Put each on GitHub with a README that includes an architecture diagram, the hardware used, build steps,
lessons learned, and photos or a demo video.

1. **Custom Yocto distro.** Custom layer, machine/image recipes, smaller image, faster boot, sstate cache.
2. **Kernel driver for the BME280.** I²C character device or IIO driver, Device Tree overlay, sysfs,
   debugging with ftrace/dmesg. (Mainline already has a `bmp280` IIO driver. Write your own as a learning
   exercise and compare the two.)
3. **Secure OTA updates.** A/B partitions on the eMMC with RAUC, SWUpdate or Mender. U-Boot boot counting
   with rollback. Signed bundles, verified boot (FIT signing / dm-verity).
4. **Edge IoT device.** C++ daemon (epoll, libgpiod, I²C), MQTT telemetry to a dashboard, systemd units
   with a watchdog, power management.
5. **Embedded CI/CD.** GitHub Actions/Jenkins builds Yocto images, QEMU boot tests, hardware-in-the-loop
   tests, SBOM (SPDX) and `cve-check`. This is where a traditional software background helps most.
6. **Real-time core / RTOS (optional).** FreeRTOS on the BeaglePlay's M4F, talking to Linux over
   **remoteproc/RPMsg**, or on a separate STM32 Nucleo board.

---

## Resume bullet points (draft)

Only use these once the work is done. Replace `[X]` with your own measured numbers.

### Custom Embedded Linux distro (Yocto)
- Built a custom Linux distribution for the TI AM625-based BeaglePlay using the **Yocto Project (Scarthgap)**,
  writing a custom BSP layer, machine configuration and image recipes.
- Cut boot time from [X]s to [Y]s and root filesystem size by [Z]% by removing unused packages and tuning the kernel config.
- Wrote BitBake recipes to cross-compile and package C/C++ and Python applications, with `.bbappend` patches for upstream components.
- Set up reproducible builds with sstate cache and a shared download mirror, cutting rebuild times by [X]%.
- Configured the TI K3 multi-stage boot chain (**TF-A, OP-TEE, U-Boot**) using a Yocto multiconfig build.

### Linux kernel driver
- Wrote a **Linux kernel driver** for an I²C environmental sensor (BME280), exposing data through sysfs/IIO and `/dev`.
- Wrote **Device Tree overlays** to describe the hardware and bind the driver without changing board files.
- Implemented interrupt-driven GPIO handling with threaded IRQs and workqueues, and verified timing with ftrace and a logic analyzer.
- Debugged kernel panics and race conditions with dmesg, kgdb and kernel debug config options.

### Secure OTA updates
- Built an **A/B partition OTA update system** with RAUC/SWUpdate/Mender, with automatic rollback if boot fails.
- Configured **U-Boot** boot counting and environment-based slot selection.
- Signed update bundles with X.509 certificates and enabled **verified boot** (FIT image signing / dm-verity).

### Edge IoT device
- Built an end-to-end IoT sensor node: Yocto-based Linux on ARM64, a C++ data-acquisition daemon, and
  **MQTT** telemetry to a cloud dashboard.
- Wrote a low-latency userspace daemon using epoll, libgpiod and the I²C/SPI interfaces, sampling at [X] Hz with under [Y]% CPU.
- Packaged services as systemd units with watchdog integration for automatic recovery.
- Lowered idle power draw by [X]% using CPU frequency scaling and runtime power management.

### Embedded CI/CD
- Built a **CI pipeline** that builds Yocto images, runs **QEMU** boot and integration tests, and publishes versioned artifacts.
- Set up **hardware-in-the-loop testing** to flash and check real devices automatically (pytest/LAVA).
- Generated an **SBOM (SPDX)** and ran CVE scans with Yocto `cve-check` to support security compliance.

### Real-time core / RTOS (optional)
- Wrote **FreeRTOS** firmware for the Cortex-M4F with tasks, queues and interrupt-driven peripherals,
  communicating with Linux over **remoteproc/RPMsg**.
- Implemented a CRC-checked framing protocol between the real-time core and the Linux host.

### Resume keywords
Yocto, BitBake, Device Tree, U-Boot, TF-A, OP-TEE, kernel modules, cross-compilation, ARM Cortex-A/M,
I²C/SPI/UART/CAN, FreeRTOS, remoteproc/RPMsg, OTA, systemd, GDB/JTAG, QEMU, SBOM.

---

## Build host requirements
- Linux PC running Ubuntu 22.04 or 24.04 LTS
- 16GB RAM minimum, 32GB recommended
- 100GB+ free disk space
- The first full build takes about 1–3 hours
