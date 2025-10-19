# Arducam Mega Driver Port to nRF5340 (Zephyr RTOS v2.7.0)

This repository documents the port of the **Arducam Mega** camera driver from **Raspberry Pi** to the **nRF5340** using **Zephyr RTOS v2.7.0**.  
The project runs in **non-secure mode** (`nrf5340dk_nrf5340_cpuapp_ns`) to enable future **OpenThread** integration for wireless image streaming.

---

## 🎯 Project Overview

This work demonstrates:
- SPI communication tuning between nRF5340 and Arducam Mega  
- DMA-safe buffer alignment for large image bursts  
- Manual Chip-Select control (P0.25) for consistent frame captures  
- Off-tree Zephyr module integration  
- Real-time debugging via UART0 (921600 baud)

The Raspberry Pi driver used automatic CS and loose SPI timing, which failed on nRF5340’s stricter SPI1 controller.  
By introducing manual CS handling and DMA-aligned buffers, stable preview and partial high-resolution capture were achieved.

---

## 🛠 Hardware Setup

**MCU:** nRF5340 DK (non-secure build)  
**Camera:** Arducam Mega (5 MP)  
**Interface:** SPI1 + UART0  
**Power:** Stable 3.3 V

| nRF5340 Pin | Arducam Mega Signal |
|--------------|---------------------|
| P0.06 | SCK |
| P0.07 | MOSI |
| P0.26 | MISO |
| P0.25 | CS (Chip Select) |
| UART0 TX | RX |
| UART0 RX | TX |
| 3.3 V | VCC |
| GND | GND |

Optional: Attach a logic analyzer to SPI1 and UART0 for debugging.

---

## 🧰 Software Requirements

- **Zephyr RTOS v2.7.0** (via nRF Connect SDK)  
- **Toolchain:** GNU Arm Embedded (from nRF Connect SDK)  
- **Dependencies:** `west`, `cmake ≥ 3.20`, `python ≥ 3.8`  
- **Repository:** <https://github.com/NealRollins2022/my-arducam>

---

## ⚙️ Installation & Build

### 1. Setup Zephyr Workspace
```bash
mkdir -p ~/ncs/v2.7.0
cd ~/ncs/v2.7.0
west init -m https://github.com/nrfconnect/sdk-nrf --mr v2.7.0
west update
west zephyr-export
```

If `west update` hangs, delete `.git/index.lock` in the `zephyr/` folder.

---

### 2. Add Off-Tree Module
Edit `west.yml` to include the driver:
```yaml
remotes:
  - name: neal
    url-base: https://github.com/NealRollins2022
projects:
  - name: my-arducam
    remote: neal
    path: modules/my-arducam
```

---

### 3. CMake Configuration
Add the off-tree path in `CMakeLists.txt`:
```cmake
find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(camera_demo)
target_sources(app PRIVATE src/main.c)
target_include_directories(app PRIVATE $ENV{ZEPHYR_BASE}/../modules/my-arducam/drivers/video)
```

---

### 4. Device Tree Overlay (`boards/nrf5340dk_nrf5340_cpuapp_ns.overlay`)
```dts
&i2c0 { status = "disabled"; };
&spi0 { status = "disabled"; };
&i2c1 { status = "disabled"; };

&uart0 {
    status = "okay";
    current-speed = <921600>;
};

&spi1 {
    compatible = "nordic,nrf-spim";
    status = "okay";
    pinctrl-0 = <&spi1_default>;
    pinctrl-1 = <&spi1_sleep>;
    pinctrl-names = "default", "sleep";
    cs-gpios = <&gpio0 25 GPIO_ACTIVE_LOW>;

    arducam0: arducam@0 {
        compatible = "arducam,mega";
        reg = <0>;
        spi-max-frequency = <4000000>;
        status = "okay";
    };
};

&pinctrl {
    spi1_default: spi1_default {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK, 0, 6)>,
                    <NRF_PSEL(SPIM_MOSI, 0, 7)>,
                    <NRF_PSEL(SPIM_MISO, 0, 26)>;
        };
    };
    spi1_sleep: spi1_sleep {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK, 0, 6)>,
                    <NRF_PSEL(SPIM_MOSI, 0, 7)>,
                    <NRF_PSEL(SPIM_MISO, 0, 26)>;
            low-power-enable;
        };
    };
};
```

---

### 5. Build and Flash
```bash
west build -b nrf5340dk_nrf5340_cpuapp_ns path/to/sample
west flash
```

---

## 🧩 Driver Modifications

| Modification | Reason | Change | Impact |
|---------------|--------|--------|--------|
| **Manual CS** | No persistent CS on nRF5340 | Added `cs_select()` / `cs_deselect()` using P0.25 | Enabled full-frame capture |
| **DMA Buffers** | Corruption with non-DMA memory | Added aligned DMA buffers (`MAX_SPI_BURST + 8`) | Stable transfers |
| **CAP_DONE Polling** | Timeout too short | Increased tries (1000) and delay (5 ms) | Reliable completion |
| **FIFO Reads** | Batched reads failed | Switched to single register reads | Accurate FIFO size |
| **Preview Mode** | Auto CS works fine | Kept automatic CS for streaming | Simpler preview |

---

## 💻 Code Examples

**Manual CS**
```c
cs_select();
arducam_mega_write_reg(&cfg->bus, ARDUCHIP_FIFO, FIFO_CLEAR_ID_MASK);
arducam_mega_write_reg(&cfg->bus, ARDUCHIP_FIFO, FIFO_START_MASK);
cs_deselect();
```

**DMA Buffer**
```c
#define MAX_SPI_BURST 4096
#define DMA_BUF_SIZE (MAX_SPI_BURST + 8)
static uint8_t dma_buf[DMA_BUF_SIZE] __aligned(4);
```

**CAP_DONE Polling**
```c
uint16_t tries = 1000;
do {
    if (tries-- == 0) {
        LOG_ERR("Capture timeout!");
        return -ETIMEDOUT;
    }
    k_msleep(5);
} while (!(arducam_mega_read_reg(&cfg->bus, ARDUCHIP_TRIG) & CAP_DONE_MASK));
```

**FIFO Length Read**
```c
drv_data->fifo_length  = arducam_mega_read_reg(&cfg->bus, FIFO_SIZE1);
drv_data->fifo_length |= (arducam_mega_read_reg(&cfg->bus, FIFO_SIZE2) << 8);
drv_data->fifo_length |= (arducam_mega_read_reg(&cfg->bus, FIFO_SIZE3) << 16);
```

---

## 📋 Sample UART Log
```plaintext
[14:04:53]: Port Connect Success
[14:04:54]: Command:55 FF AA, Send Success!
[14:04:56]: The camera is successfully initialized.
[14:04:59]: Video Open Success, The camera cannot be set in this mode
[14:04:59]: Command:55 02 03 AA, Send Success!
[14:05:46]: Command:55 10 AA, Send Success!
[14:05:58]: Correct data is not obtained
```

---

## 🚧 Journey & Challenges

- **West manifest errors:** duplicate remotes caused `west update` hangs — fixed by using a single `neal` remote.  
- **CMake module paths:** `zephyr-export` did not include off-tree modules; explicit include was required.  
- **SPI timing:** Needed delay and polling adjustments for CAP_DONE bit.  
- **CS handling:** Preview worked with auto CS, but capture required manual P0.25 control.  
- **DMA:** Non-aligned buffers corrupted frames; DMA-safe buffers fixed this.  
- **Debugging:** UART0 logs (921600 baud) proved critical for timing and register validation.

---

## 💡 Lessons Learned

- Porting drivers requires deep understanding of SPI timing and CS persistence.  
- Manual CS is essential for large bursts on nRF5340.  
- Always use DMA-aligned buffers to avoid corruption.  
- Off-tree Zephyr modules need explicit include paths.  
- Logging over UART0 is invaluable for low-level debugging.

---

## 🧭 Troubleshooting

| Issue | Fix |
|--------|------|
| CAP_DONE Timeout | Increase `k_msleep(5)` → `10 ms` or `tries = 2000` |
| Partial Capture | Ensure video buffer matches `fifo_length` |
| No Response | Check P0.25 CS pin and ground reference |
| West hangs | Delete `.git/index.lock` in `zephyr/` |
| README renders as code block | Ensure closing ``` at end of each block and UTF-8 encoding |

---

## 📚 References

- [Arducam Mega Host Protocol](https://www.arducam.com/docs/arducam-mega/arducam-mega-getting-started/packs/HostCommunicationProtocol.html)  
- [Zephyr RTOS v2.7.0 Docs](https://docs.zephyrproject.org/2.7.0/)  
- [nRF Connect SDK](https://developer.nordicsemi.com/nRF_Connect_SDK/doc/)  

---

## 🖼 Screenshots (Placeholders)

```markdown
![Preview Mode](images/preview.png)
![Partial Capture](images/capture.png)
![Log Output](images/log.png)
```
*(Add actual images under `my-arducam/images/` before committing.)*

---

## 🔗 How to Add README

```bash
cd my-arducam
git add README.md
git commit -m "Add clean README with SPI1, UART0, CS=P0.25"
git push origin main
```

Then verify rendering at <https://github.com/NealRollins2022/my-arducam>.
