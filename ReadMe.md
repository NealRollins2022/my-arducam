

# Arducam Mega Driver Port to nRF5340 (Zephyr RTOS v2.7.0)

This repository hosts a ported driver for the [Arducam Mega](https://www.arducam.com/docs/arducam-mega/arducam-mega-getting-started/packs/HostCommunicationProtocol.html) camera module, adapted from Raspberry Pi to the [nRF5340](https://www.nordicsemi.com/Products/nRF5340) using [Zephyr RTOS v2.7.0](https://docs.zephyrproject.org/2.7.0/). Built in non-secure mode to enable future OpenThread stack integration, it demonstrates expertise in embedded systems, SPI communication, DMA buffer management, chip select (CS) handling, and off-tree Zephyr module integration. These skills align with requirements for SDR and wireless communications roles, such as those at Shared Spectrum Company.

## 🎯 Project Overview

I spent over a year porting the Arducam Mega driver to support preview and high-resolution capture on the nRF5340 in a non-secure build. The Raspberry Pi driver relied on automatic CS and loose SPI timing, which failed on the nRF5340’s stricter SPI1 controller due to lack of persistent CS. Key challenges included SPI timing, DMA alignment, Zephyr module setup, CMake configuration, and capture timeouts. Through iterative debugging, I achieved stable preview and partial high-resolution captures, highlighting real-time software and hardware debugging skills.

## 📸 Demo

- **Preview Mode**: Streams low-resolution video (e.g., QVGA) using automatic CS on SPI1, confirming SPI and DMA setup.
- **Capture Mode**: Achieved partial high-resolution captures (top half displayed), indicating correct data transfer but buffer sizing issues.
- **Logs**: Display initialization, SPI commands, and CAP_DONE polling via UART0 (921600 baud).

**Screenshots** (to be added):
- Preview video on nRF5340: `![Preview Mode](images/preview.png)`
- Partial high-resolution capture: `![Partial Capture](images/capture.png)`
- UART log output: `![Log Output](images/log.png)`

<!-- [SCREENSHOT PLACEHOLDER: Preview Mode]
*Save screenshot as `images/preview.png` and add: `![Preview Mode](images/preview.png)`.* -->
<!-- [SCREENSHOT PLACEHOLDER: Partial Capture]
*Save screenshot as `images/capture.png` and add: `![Partial Capture](images/capture.png)`.* -->
<!-- [SCREENSHOT PLACEHOLDER: Log Output]
*Save screenshot as `images/log.png` and add: `![Log Output](images/log.png)`.* -->

**Sample Log** (via UART0):
```plaintext
[14:04:53]: Port Connect Success
[14:04:54]: Command:55 FF AA, Send Success!
[14:04:56]: The camera is successfully initialized.
[14:04:59]: Video Open Success, The camera cannot be set in this mode
[14:04:59]: Command:55 02 03 AA, Send Success!
[14:05:46]: Command:55 10 AA, Send Success!
[14:05:58]: Correct data is not obtained

 Hardware SetupMCU: nRF5340 (e.g., nRF5340 DK, non-secure build)
Camera: Arducam Mega (5MP)
Wiring:
```plaintext
nRF5340    Arducam MegaSPI1 SCLK   SCK  (P0.06)
SPI1 MOSI   MOSI (P0.07)
SPI1 MISO   MISO (P0.26)
P0.25       CS
UART0 TX    RX
UART0 RX    TX
3.3V        VCC
GND         GND

Power: Stable 3.3V supply.
Optional: Logic analyzer for SPI1/UART0 debugging.

 Software RequirementsZephyr RTOS: v2.7.0 (via nRF Connect SDK)
Toolchain: nRF Connect SDK toolchain (GNU Arm Embedded)
Dependencies: West (pip install west), CMake 3.20+, Python 3.8+
Repositories:Primary: https://github.com/NealRollins2022/my-arducam.git
Secondary: https://github.com/NealRollins2022/arducam_AI_module.git

 InstallationSet Up Zephyr Workspace:bash

mkdir -p ~/ncs/v2.7.0
cd ~/ncs/v2.7.0
west init -m https://github.com/nrfconnect/sdk-nrf --mr v2.7.0
west update
west zephyr-export

Verify: west --version (~v1.2.0). If west update hangs, delete .git/index.lock in zephyr/.
Add Arducam Module:
Edit ncs/v2.7.0/west.yml:yaml

manifest:
  version: 0.12
  remotes:
    - name: ncs
      url-base: https://github.com/nrfconnect
    - name: zephyrproject-rtos
      url-base: https://github.com/zephyrproject-rtos
    - name: mcu-tools
      url-base: https://github.com/mcu-tools
    - name: neal
      url-base: https://github.com/NealRollins2022
  projects:
    - name: zephyr
      remote: ncs
      revision: v3.3.0
      import: true
    - name: my-arducam
      remote: neal
      revision: main
      path: modules/my-arducam
    - name: arducam_AI_module
      remote: neal
      revision: main
      path: modules/arducam
  self:
    path: nrf

Update: west update
Configure Build:
Add to CMakeLists.txt (explicit include for off-tree module):cmake

find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(camera_demo)
target_sources(app PRIVATE src/main.c)
target_include_directories(app PRIVATE $ENV{ZEPHYR_BASE}/../modules/my-arducam/drivers/video)

Add Device Tree overlay (boards/nrf5340dk_nrf5340_cpuapp_ns.overlay):dts

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
        status = "okay";
        reg = <0>;
        spi-max-frequency = <4000000>;
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

Build and Flash (non-secure):bash

west build -b nrf5340dk_nrf5340_cpuapp_ns path/to/sample
west flash

 Driver ModificationsThe Raspberry Pi driver used automatic CS and loose SPI timing, incompatible with nRF5340’s SPI1 controller. Key changes:Modification
Why
Change
Impact
Manual CS
nRF5340 lacks persistent CS for bursts.
Added cs_select()/cs_deselect() using P0.25.
Enabled high-resolution capture.
DMA Buffers
Non-DMA-safe buffers caused corruption.
Used aligned buffers (MAX_SPI_BURST + 8).
Prevented partial captures.
CAP_DONE Polling
Fast polling caused timeouts.
Increased tries (1000), delay (5ms).
Reduced CAP_DONE timeouts.
FIFO Length Reads
Batched reads failed on nRF5340.
Used single-register reads.
Ensured accurate FIFO sizes.
Auto CS for Preview
Preview worked with auto CS.
Removed manual CS for non-capture ops.
Simplified preview mode.

Code Examples:Manual CS:c

cs_select();
arducam_mega_write_reg(&cfg->bus, ARDUCHIP_FIFO, FIFO_CLEAR_ID_MASK);
arducam_mega_write_reg(&cfg->bus, ARDUCHIP_FIFO, FIFO_START_MASK);
cs_deselect();

DMA Buffer:c

#define MAX_SPI_BURST 4096
#define DMA_BUF_SIZE (MAX_SPI_BURST + 8)
static uint8_t dma_buf[DMA_BUF_SIZE] __aligned(4);

CAP_DONE Polling:c

uint16_t tries = 1000;
do {
    if (tries-- == 0) {
        LOG_ERR("Capture timeout!");
        return -ETIMEDOUT;
    }
    k_msleep(5);
} while (!(arducam_mega_read_reg(&cfg->bus, ARDUCHIP_TRIG) & CAP_DONE_MASK));

FIFO Length:c

drv_data->fifo_length = arducam_mega_read_reg(&cfg->bus, FIFO_SIZE1);
drv_data->fifo_length |= (arducam_mega_read_reg(&cfg->bus, FIFO_SIZE2) << 8);
drv_data->fifo_length |= (arducam_mega_read_reg(&cfg->bus, FIFO_SIZE3) << 16);

Full Capture Function:c

static int arducam_mega_capture(const struct device *dev, uint32_t *length)
{
    const struct arducam_mega_config *cfg = dev->config;
    struct arducam_mega_data *drv_data = dev->data;
    uint16_t tries = 1000;
    int ret;

    cs_select();
    LOG_INF("CS asserted: t=%u ms", k_uptime_get_32());

    LOG_INF("Clearing FIFO...");
    ret = arducam_mega_write_reg(&cfg->bus, ARDUCHIP_FIFO, FIFO_CLEAR_ID_MASK);
    if (ret) {
        cs_deselect();
        LOG_ERR("Failed to clear FIFO");
        return ret;
    }
    k_busy_wait(10);

    LOG_INF("Starting capture...");
    ret = arducam_mega_write_reg(&cfg->bus, ARDUCHIP_FIFO, FIFO_START_MASK);
    if (ret) {
        cs_deselect();
        LOG_ERR("Failed to start capture");
        return ret;
    }
    k_busy_wait(10);

    do {
        if (tries-- == 0) {
            cs_deselect();
            LOG_ERR("Capture timeout! CAP_DONE never set");
            return -ETIMEDOUT;
        }
        LOG_DBG("Polling CAP_DONE: value=0x%02x", arducam_mega_read_reg(&cfg->bus, ARDUCHIP_TRIG));
        k_msleep(5);
    } while (!(arducam_mega_read_reg(&cfg->bus, ARDUCHIP_TRIG) & CAP_DONE_MASK));

    drv_data->fifo_length = arducam_mega_read_reg(&cfg->bus, FIFO_SIZE1);
    drv_data->fifo_length |= (arducam_mega_read_reg(&cfg->bus, FIFO_SIZE2) << 8);
    drv_data->fifo_length |= (arducam_mega_read_reg(&cfg->bus, FIFO_SIZE3) << 16);
    LOG_INF("FIFO length read: %u bytes", drv_data->fifo_length);

    drv_data->fifo_first_read = 1;
    *length = drv_data->fifo_length;
    cs_deselect();

    return 0;
}

 Journey and ChallengesThe port took over a year due to:Off-Tree Module Setup: Initial west.yml errors with duplicate remotes caused west update hangs. Fixed by using a single neal remote with unique paths.yaml

remotes:
  - name: neal
    url-base: https://github.com/NealRollins2022
projects:
  - name: my-arducam
    remote: neal
    path: modules/my-arducam

CMake Module Discovery: west zephyr-export didn’t include the off-tree module, requiring explicit target_include_directories.cmake

target_include_directories(app PRIVATE $ENV{ZEPHYR_BASE}/../modules/my-arducam/drivers/video)

Non-Secure Build: Used nrf5340dk_nrf5340_cpuapp_ns for OpenThread compatibility, with _ns overlay.
SPI Timing: CAP_DONE timeouts occurred until polling delays increased.
Manual vs Auto CS: Preview worked with auto CS on SPI1 (P0.25), but capture needed manual CS for large bursts.
DMA Buffers: Non-DMA-safe buffers caused corrupted captures. Fixed with aligned buffers.
Debugging: UART0 logs isolated issues. Partial captures confirmed data flow but revealed buffer sizing problems.

 Lessons LearnedPorting Drivers: Hardware differences require precise SPI and CS adjustments.
SPI/CS Handling: Manual CS (P0.25) is critical for large bursts; auto CS works for preview.
DMA/Memory: Aligned, DMA-safe buffers prevent transfer failures.
Zephyr Modules: Explicit CMake paths ensure off-tree integration.
Non-Secure Builds: Support OpenThread with non-secure configuration.
Debugging: UART0 logging is key for timing issues.

 Screenshots and Logs<!-- [SCREENSHOT PLACEHOLDER: Preview Mode]
*Save screenshot as `images/preview.png` and add: `![Preview Mode](images/preview.png)`.* -->
<!-- [SCREENSHOT PLACEHOLDER: Partial Capture]
*Save screenshot as `images/capture.png` and add: `![Partial Capture](images/capture.png)`.* -->
<!-- [SCREENSHOT PLACEHOLDER: Log Output]
*Save screenshot as `images/log.png` and add: `![Log Output](images/log.png)`.* -->

Sample Log (via UART0):plaintext

[14:04:53]: Port Connect Success
[14:04:54]: Command:55 FF AA, Send Success!
[14:04:56]: The camera is successfully initialized.
[14:04:59]: Video Open Success, The camera cannot be set in this mode
[14:04:59]: Command:55 02 03 AA, Send Success!
[14:05:46]: Command:55 10 AA, Send Success!
[14:05:58]: Correct data is not obtained

 TroubleshootingCAP_DONE Timeouts: Increase k_msleep(5) to 10ms or tries to 2000.
Partial Captures: Ensure video buffer size matches fifo_length.
CS Pin Issues: Verify P0.25 wiring and Device Tree configuration.
West Update Hangs: Delete .git/index.lock in zephyr/.
Rendering Issues: Save README.md in UTF-8 without BOM; check closing ```.

 ReferencesArducam Mega Host Communication Protocol
Zephyr RTOS v2.7.0 Documentation
nRF Connect SDK

