# Hardware Registration and Configuration

The Hardware Registration and Configuration application is intended to be used to test hardware and perform the initial configuration of the device.  This application is used at the factory to configure the device prior to being shipped, or by a developer building their own hardware.

## :white_check_mark: What this application does
- Sets up an HTTP server using the on-board Ethernet or WiFi (depending on hardware compile options)
- Sets up the [OLED display](/controller/support/OLED_screens/) (where supported by the hardware) to display basic information and provide a user interface button
- Displays hardware MAC addresses for the MCU
- Provides a user interface to configure an identity for the device, including the device unique ID, product ID, and secret key
- <Badge type="warning" text="TODO" /> Registers the device with the cloud service for configuration backup
- Verifies the peripherals, such as the inputs, outputs, temperature sensors, and OLED display are online and functional
- Confirms the [partition table](/controller/support/partitions) matches the [expected configuration](/controller/development_environment/#adding-a-new-hardware-version)
- Displays a more robust [event log and error log](/controller/support/event_and_error_logs) inclusive of the event time and type with additional entries

## VDD_SDIO eFuse Configuration

On supported hardware variants (FFC0806-2305, FFC0806-2505, FFC3232-2505, FFC3232-2603), the Hardware Registration and Configuration application automatically burns three ESP32 eFuses on first boot to permanently lock the VDD_SDIO power rail at 3.3V.

**Why this is necessary:** These hardware variants use the W5500 Ethernet chipset, whose MISO signal is connected to GPIO12. GPIO12 is also the ESP32's MTDI bootstrap pin — if it reads HIGH at reset (which occurs when the W5500 is active on the SPI bus), the ESP32 defaults VDD_SDIO to 1.8V instead of 3.3V. The PSRAM module requires 3.3V and will fail to initialize at 1.8V. Burning the eFuses removes this dependency on the GPIO12 state at reset.

**What is burned:**

| eFuse | Effect |
| ----- | ------ |
| `XPD_SDIO_FORCE` | Forces VDD_SDIO regulator on, overriding the GPIO12 bootstrap |
| `XPD_SDIO_REG` | Enables the internal VDD_SDIO regulator |
| `VDD_SDIO_TIEH` | Sets VDD_SDIO output to 3.3V |

::: danger Irreversible
eFuse burns are permanent and cannot be undone, even by reflashing the firmware. This operation happens automatically on the first boot of the Hardware Registration and Configuration application on supported hardware — no user action is required.
:::

## :no_entry_sign: What this application does not do
- Configures the user-programmable logic for high voltage switching
- Exposes the port and channel configuration
- Executes actions based on inputs from physical buttons or API/MQTT requests. See [Controller Application](/controller/software/controller/).