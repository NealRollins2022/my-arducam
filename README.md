# ⬛ **ARDCUAM MEGA DRIVER PORT TO nRF5340 (ZEPHYR RTOS v3.6.99)**  

![Light Badge](https://img.shields.io/badge/Theme-Light-grey?style=flat-square)
![Dark Badge](https://img.shields.io/badge/Theme-Dark-black?style=flat-square)
![Status](https://img.shields.io/badge/Status-Development-blue?style=flat-square)
![Platform](https://img.shields.io/badge/Platform-nRF5340-brightgreen?style=flat-square)
![RTOS](https://img.shields.io/badge/Zephyr-v2.7.0-blueviolet?style=flat-square)

---

## 🧭 **Table of Contents**

1. [⚡ Overview](#-overview)  
2. [🎯 Purpose](#-purpose)  
3. [⬛ Project Overview](#-project-overview)  
4. [⬛ Demo Results](#-demo-results)  
5. [⬛ Hardware Requirements](#-hardware-requirements)  
6. [⬛ Software Requirements](#-software-requirements)  
7. [⬛ Installation Steps](#-installation-steps)  
8. [⬛ Driver Modifications](#-driver-modifications)  
9. [⬛ Journey and Challenges](#-journey-and-challenges)  
10. [⬛ Lessons Learned](#-lessons-learned)  
11. [⬛ Screenshots and Logs](#-screenshots-and-logs)  
12. [⬛ References](#-references)

---

## ⚡ **Overview**

This repository contains a **ported and modified driver** for the **Arducam Mega camera module**, adapted from its original **Raspberry Pi implementation** to run on the **nRF5340** using **Zephyr RTOS  v3.6.99**.

It showcases **low-level embedded system development** involving:
- 🧩 **SPI communication**
- ⚙️ **DMA-safe buffer management**
- 🔄 **Manual and automatic chip-select (CS) handling**
- 🧠 **Off-tree Zephyr module integration**

---

## 🎯 **Purpose**

This work demonstrates my ability to:
- Tackle complex **driver porting challenges**
- Debug **hardware-specific timing and transfer issues**
- Optimize for **real-time embedded performance**

It aligns with the technical demands of **advanced wireless and SDR development**, such as roles at **Shared Spectrum Company** and similar organizations.

---

## ⬛ **PROJECT OVERVIEW**

The goal was to enable the **Arducam Mega camera** on the **nRF5340**, supporting both preview and high-resolution capture modes.

The original Raspberry Pi driver relied on different SPI timing and automatic CS handling, which caused issues on the nRF5340 due to its stricter SPI controller and lack of persistent CS across transactions.

### ⚙️ **Challenges Faced**

- **SPI Timing and CS Handling:** required manual CS for full-frame capture  
- **DMA Buffer Alignment:** ensured DMA-safe buffers to prevent corruption  
- **Zephyr Off-Tree Module Setup:** handled module integration  
- **Capture Failures:** debugged CAP_DONE timeouts and FIFO misreads

---

## ⬛ **DEMO RESULTS**

- 🎞️ **Preview Mode:** low-resolution streaming confirms SPI/DMA setup  
- 🖼️ **Capture Mode:** partial high-resolution capture achieved  
- 🧾 **Logs:** detailed SPI and CAP_DONE sequence verification  

```plaintext
[14:04:53]: Port Connect Success
[14:04:54]: Command:55 FF AA, Send Success!
[14:04:55]: Command:55 0F AA, Send Success!
[14:05:59]: Video Open Success
```

---

## ⬛ **HARDWARE REQUIREMENTS**

- 🧠 MCU: **nRF5340 DK**  
- 📸 Camera: **Arducam Mega 5 MP**  
- 🔌 Wiring:
  - SPI SCLK/MOSI/MISO → nRF5340 SPI1  
  - CS → GPIO P0.25  
  - Power 3.3 V regulated  
- 🧰 Optional: logic analyzer for SPI/CS timing  

---

## ⬛ **SOFTWARE REQUIREMENTS**

- Zephyr RTOS  v3.6.99  
- nRF Connect SDK toolchain ver2.7 
- `west`, CMake 3.20+, Python 3.8+  

**Repositories**
- 📂 [my-arducam](https://github.com/NealRollins2022/my-arducam)  
- 📂 [arducam_AI_module](https://github.com/NealRollins2022/arducam_AI_module)

---

## ⬛ **INSTALLATION STEPS**

### 1️⃣ Setup Zephyr Workspace
```bash
mkdir -p ~/ncs/v2.7.0
cd ~/ncs/v2.7.0
west init -m https://github.com/nrfconnect/sdk-nrf --mr v2.7.0
west update
west zephyr-export
```

### 2️⃣ Add Arducam Module  
Edit `west.yml`:
```yaml
projects:
  - name: my-arducam
    remote: neal
    path: modules/my-arducam
  - name: arducam_AI_module
    remote: neal
    path: modules/arducam
```
Then run:
```bash
west update
```

### 3️⃣ Configure Build  
`CMakeLists.txt`
```cmake
find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(camera_demo)
target_sources(app PRIVATE src/main.c)
```

**Device-tree overlay**
```dts
&spi1 {
    status = "okay";
    cs-gpios = <&gpio0 25 GPIO_ACTIVE_LOW>;
    arducam_mega0: arducam-mega@0 {
        compatible = "arducam,mega";
        spi-max-frequency = <4000000>;
    };
};
```

### 4️⃣ Build and Flash  
```bash
west build -b nrf5340dk_nrf5340_cpuapp_ns path/to/sample
west flash
```

---

## ⬛ **DRIVER MODIFICATIONS**

### 1️⃣ Manual Chip-Select Handling
```c
cs_select();
arducam_mega_write_reg(&cfg->bus, ARDUCHIP_FIFO, FIFO_CLEAR_ID_MASK);
arducam_mega_write_reg(&cfg->bus, ARDUCHIP_FIFO, FIFO_START_MASK);
...
cs_deselect();
```

### 2️⃣ DMA-Safe Buffer
```c
#define MAX_SPI_BURST 4096
#define DMA_BUF_SIZE (MAX_SPI_BURST + 8)
static uint8_t dma_buf[DMA_BUF_SIZE] __aligned(4);
```

### 3️⃣ CAP_DONE Polling
```c
uint16_t tries = 1000;
do {
    if (tries-- == 0) return -ETIMEDOUT;
    k_msleep(5);
} while (!(arducam_mega_read_reg(&cfg->bus, ARDUCHIP_TRIG) & CAP_DONE_MASK));
```

---

## ⬛ **JOURNEY AND CHALLENGES**

> “A year of debugging, iteration, and persistence.”

- 🧩 Fixed off-tree module linking  
- ⏱️ Tuned capture timing  
- 🔄 Hybrid CS (auto + manual)  
- 💾 Resolved DMA alignment  
- 🧠 Improved structured logging  

---

## ⬛ **LESSONS LEARNED**

- ⚙️ Hardware-aware porting is critical  
- 🔄 Manual CS ensures stable SPI bursts  
- 💾 Proper alignment prevents corruption  
- 🧩 Accurate `west.yml` avoids conflicts  
- 🧮 Systematic logging accelerates debug  

---

## ⬛ **SCREENSHOTS AND LOGS**

```plaintext
[00:02:18.909,484] <inf> mega_camera: Starting capture...
[00:02:18.909,545] <inf> mega_camera: spi_write_dt returned: 0
[00:02:19.759,216] <err> mega_camera: Capture timeout! CAP_DONE never set
```
<!-- 
![Preview Mode](images/preview.png)
![Partial Capture](images/capture.png)
-->

---

## ⬛ **REFERENCES**

- 📘 [Zephyr RTOS v2.7.0 Docs](https://docs.zephyrproject.org)  
- 🔗 [nRF Connect SDK Docs](https://developer.nordicsemi.com/nRF_Connect_SDK/doc/latest)  
- 📚 [Arducam Mega Protocol](https://www.arducam.com/downloads/shield)

---

**Author:** Neal O. Rollins  
📧 nealrollins2022 @ github  
🕓 Updated October 2025
