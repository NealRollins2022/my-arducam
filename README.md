# Arducam Mega Driver Port to nRF5340 (Zephyr RTOS v2.7.0)

This repository contains a ported and modified driver for the [Arducam Mega](https://www.arducam.com/docs/arducam-mega/arducam-mega-getting-started/packs/HostCommunicationProtocol.html) camera module, adapted from its original Raspberry Pi implementation to run on the [nRF5340](https://www.nordicsemi.com/Products/nRF5340) using [Zephyr RTOS v2.7.0](https://docs.zephyrproject.org/2.7.0/). The project showcases low-level embedded system development, including SPI communication, DMA-safe buffer management, manual/automatic chip select (CS) handling, and off-tree Zephyr module integration.

This work demonstrates my ability to tackle complex driver porting challenges, debug hardware-specific issues, and optimize for real-time embedded systems, aligning with skills required for advanced wireless and SDR development roles (e.g., at Shared Spectrum Company).

## 🎯 Project Overview

The goal was to enable the Arducam Mega camera on the nRF5340, supporting both preview and high-resolution capture modes. The original Raspberry Pi driver relied on different SPI timing and automatic CS handling, which caused issues on the nRF5340 due to its stricter SPI controller and lack of persistent CS across transactions. Over a year, I faced challenges with:

- **SPI Timing and CS Handling**: The nRF5340 required manual CS control for high-resolution captures, unlike the Raspberry Pi’s automatic CS.
- **DMA Buffer Alignment**: Ensuring DMA-safe buffers for SPI bursts to prevent data corruption.
- **Zephyr Off-Tree Module Setup**: Managing custom modules in the Zephyr workspace without breaking the build system.
- **Capture Failures**: Debugging partial image captures, CAP_DONE timeouts, and FIFO length issues.

Through iterative debugging, I achieved stable preview mode and partial high-resolution capture, identifying key modifications to make capture reliable. This README captures both the technical solutions and the journey, serving as a demonstration of embedded systems expertise.

## 📸 Demo

- **Preview Mode**: Successfully streams low-resolution video (e.g., QVGA) using automatic CS, confirming SPI and DMA setup.
- **Capture Mode**: Achieved partial high-resolution captures (top half of image displayed), indicating correct data transfer but issues with buffer sizing or rendering.
- **Logs**: Detailed SPI transaction logs show initialization, command sequences, and CAP_DONE polling.

<!-- [SCREENSHOT PLACEHOLDER: Preview Mode]
*Insert screenshot of preview video running on nRF5340 here. Save the image as `images/preview.png` and add: `![Preview Mode](images/preview.png)` after taking the screenshot.* -->

<!-- [SCREENSHOT PLACEHOLDER: Partial Capture]
*Insert screenshot of partial high-resolution capture (showing top half of image) here. Save the image as `images/capture.png` and add: `![Partial Capture](images/capture.png)` after taking the screenshot.* -->

<!-- [SCREENSHOT PLACEHOLDER: Log Output]
*Insert screenshot of UART console or terminal showing SPI logs (e.g., initialization, CAP_DONE polling) here. Save the image as `images/log.png` and add: `![Log Output](images/log.png)` after taking the screenshot.* -->

**Sample Log** (from UART console):
```plaintext
[14:04:53]: Port Connect Success
[14:04:54]: Command:55 FF AA, Send Success!
[14:04:55]: Command:55 0F AA, Send Success!
[14:04:56]: The camera is successfully initialized.
[14:04:59]: Video Open Success, The camera cannot be set in this mode
[14:04:59]: Command:55 02 03 AA, Send Success!
[14:05:46]: Command:55 10 AA, Send Success!
[14:05:58]: Correct data is not obtained

 Hardware RequirementsMCU: nRF5340 (e.g., nRF5340 DK)
Camera: Arducam Mega (5MP or similar model)
Wiring:SPI: SCLK, MOSI, MISO connected to nRF5340 SPI3 peripheral.
CS: GPIO pin for manual control (e.g., P0.06).
Power: 3.3V supply, ensure stable current for camera.

Optional: Logic analyzer to verify SPI timing and CS behavior.

 Software RequirementsZephyr RTOS: v2.7.0 (via nRF Connect SDK)
Toolchain: nRF Connect SDK toolchain (e.g., GNU Arm Embedded Toolchain)
Dependencies:West (pip install west)
CMake 3.20+
Python 3.8+

Repositories:Primary driver: https://github.com/NealRollins2022/my-arducam.git
Optional second driver: https://github.com/NealRollins2022/arducam_AI_module.git

 Installation1. Set Up Zephyr WorkspaceCreate a new workspace:bash

mkdir -p ~/ncs/v2.7.0
cd ~/ncs/v2.7.0
west init -m https://github.com/nrfconnect/sdk-nrf --mr v2.7.0
west update
west zephyr-export

2. Add Arducam ModuleEdit ncs/v2.7.0/west.yml to include the driver as an off-tree module:yaml

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

Update workspace:bash

west update

3. Configure BuildCreate a sample application or use an existing one (e.g., samples/camera_demo).Add to CMakeLists.txt:cmake

find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(camera_demo)
target_sources(app PRIVATE src/main.c)
target_include_directories(app PRIVATE $ENV{ZEPHYR_BASE}/../modules/my-arducam/drivers/video)

Add Device Tree overlay (boards/nrf5340dk_nrf5340_cpuapp.overlay):dts

&spi3 {
    compatible = "nordic,nrf-spi";
    status = "okay";
    cs-gpios = <&gpio0 6 GPIO_ACTIVE_LOW>;
    arducam_mega0: arducam-mega@0 {
        compatible = "arducam,mega";
        status = "okay";
        reg = <0>;
        spi-max-frequency = <8000000>;
    };
};

4. Build and FlashBuild for nRF5340:bash

west build -b nrf5340dk_nrf5340_cpuapp path/to/sample
west flash

 Driver ModificationsThe original Raspberry Pi driver relied on automatic chip select and loose SPI timing, which didn’t work on the nRF5340 due to its stricter SPI controller and lack of persistent CS across transactions. Key changes:1. Manual Chip Select (CS) HandlingWhy: nRF5340 SPI doesn’t automatically hold CS low across multiple transactions, required for high-resolution capture bursts.
Change: Added cs_select() and cs_deselect() around capture sequences to ensure continuous CS during FIFO reads.
Code:c

cs_select();
arducam_mega_write_reg(&cfg->bus, ARDUCHIP_FIFO, FIFO_CLEAR_ID_MASK);
arducam_mega_write_reg(&cfg->bus, ARDUCHIP_FIFO, FIFO_START_MASK);
// ... poll CAP_DONE, read FIFO
cs_deselect();

2. DMA-Safe BufferWhy: High-resolution captures require DMA-safe buffers to handle large SPI bursts without corruption.
Change: Ensured main video buffer is DMA-safe and sized to hold full frames (e.g., 3–5 MB for 5MP JPEG).
Code:c

#define MAX_SPI_BURST 4096
#define DMA_BUF_SIZE (MAX_SPI_BURST + 8)
static uint8_t dma_buf[DMA_BUF_SIZE] __aligned(4);

3. CAP_DONE PollingWhy: Original polling was too fast, causing timeouts (CAP_DONE never set).
Change: Increased polling delay and tries to allow camera time to complete capture.
Code:c

uint16_t tries = 1000; // was 200
do {
    if (tries-- == 0) {
        LOG_ERR("Capture timeout!");
        return -ETIMEDOUT;
    }
    k_msleep(5); // was 2
} while (!(arducam_mega_read_reg(&cfg->bus, ARDUCHIP_TRIG) & CAP_DONE_MASK));

4. FIFO Length ReadsWhy: nRF5340 SPI required single-register reads instead of batched transactions for reliability.
Change: Used individual arducam_mega_read_reg() calls for FIFO_SIZE1/2/3.
Code:c

drv_data->fifo_length = arducam_mega_read_reg(&cfg->bus, FIFO_SIZE1);
drv_data->fifo_length |= (arducam_mega_read_reg(&cfg->bus, FIFO_SIZE2) << 8);
drv_data->fifo_length |= (arducam_mega_read_reg(&cfg->bus, FIFO_SIZE3) << 16);

5. Auto CS for PreviewWhy: Preview mode worked with automatic CS, but capture needed manual CS for large bursts.
Change: Reverted to auto CS for preview, keeping manual CS only for capture.
Code: Removed manual CS from arducam_mega_read_reg() and arducam_mega_write_reg() for non-capture operations.

Full Capture FunctionHere’s the final working capture function:c

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

 Journey and ChallengesThis port took over a year due to several complexities:Off-Tree Module Setup: Learning to configure west.yml for custom modules was non-trivial. Early attempts with duplicate remotes caused hangs during west update. Correcting to a single remote (neal) with unique project names/paths resolved this.yaml

remotes:
  - name: neal
    url-base: https://github.com/NealRollins2022
projects:
  - name: my-arducam
    remote: neal
    path: modules/my-arducam

SPI Timing: The Raspberry Pi’s loose timing didn’t work on nRF5340. CAP_DONE timeouts were common until polling delays were increased.
Manual vs Auto CS: Preview worked with auto CS, but capture required manual CS to hold low across large FIFO bursts. Initial manual CS attempts caused partial image transfers.
DMA Buffers: Ensuring DMA-safe, aligned buffers was critical. Early versions used non-aligned buffers, leading to corrupted or partial captures.
Debugging: Extensive logging of SPI transactions, CAP_DONE values, and FIFO lengths helped isolate issues. A partial capture (top half of image) confirmed data flow but highlighted buffer sizing problems.

 Lessons LearnedEmbedded Systems Porting: Adapting drivers across platforms requires deep understanding of hardware differences (e.g., nRF5340’s SPI vs Raspberry Pi).
SPI and CS Handling: Manual CS is essential for multi-transaction bursts, while auto CS suffices for smaller preview frames.
DMA and Memory: Proper buffer alignment and sizing prevent silent failures in high-speed transfers.
Zephyr Ecosystem: Off-tree modules require careful manifest management to avoid workspace conflicts.
Debugging Discipline: Logging timestamps, register values, and SPI responses is critical for isolating timing issues.

 Screenshots and Logs<!-- [SCREENSHOT PLACEHOLDER: Preview Mode]
*Insert screenshot of preview video running on nRF5340 here. Save the image as `images/preview.png` and add: `![Preview Mode](images/preview.png)` after taking the screenshot.* -->

<!-- [SCREENSHOT PLACEHOLDER: Partial Capture]
*Insert screenshot of partial high-resolution capture (showing top half of image) here. Save the image as `images/capture.png` and add: `![Partial Capture](images/capture.png)` after taking the screenshot.* -->

<!-- [SCREENSHOT PLACEHOLDER: Log Output]
*Insert screenshot of UART console or terminal showing SPI logs (e.g., initialization, CAP_DONE polling) here. Save the image as `images/log.png` and add: `![Log Output](images/log.png)` after taking the screenshot.* -->

Sample Log:plaintext

[00:02:18.909,484] <inf> mega_camera: Starting capture...
[00:02:18.909,515] <inf> mega_camera: Clearing FIFO...
[00:02:18.909,545] <inf> mega_camera: spi_write_dt returned: 0
[00:02:18.909,576] <inf> mega_camera: Starting capture...
[00:02:19.759,216] <inf> mega_camera: Polling CAP_DONE: rx_buf=00 ret=0
[00:02:19.759,216] <err> mega_camera: Capture timeout! CAP_DONE never set

 ReferencesArducam Mega Host Communication Protocol
Zephyr RTOS v2.7.0 Documentation
nRF Connect SDK

