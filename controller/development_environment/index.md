# Controller Development Environment
This guide covers the tools and configuration required to build and test FireFly Controller firmware locally.

## Installing and Configuring arduino-cli

Using arduino-cli is more flexible and reliable than the IDE.

### Install Arduino CLI

Instructions are available at https://arduino.github.io/arduino-cli/latest/. Essentially:
```bash
brew update
brew install arduino-cli
```

### Post Installation

Initialize the installation:
```bash
arduino-cli config init
```


Enable unsafe installation so that local zip files can be installed:
```bash
arduino-cli config set library.enable_unsafe_install true
```

### Updating Arduino CLI

To update an existing installation of Arduino CLI:
```bash
brew update
brew upgrade arduino-cli
```


## Installing and Updating ESP Core

### Install

Add the ESP32 board manager packages:
```bash
arduino-cli config set board_manager.additional_urls https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
```

Update the index:
```bash
arduino-cli core update-index
```

Install ESP32 core, version 3.3.7:
```bash
arduino-cli core install esp32:esp32@3.3.7
```

Verify the installation was successful, and optionally remove any other cores:
```bash
arduino-cli core list
```

Expect:


| ID | Installed | Latest | Name |
| --- | --- | --- | --- |
| esp32:esp32 | 3.3.7 | 3.3.7 | esp32 |


### Updating
 Uninstall the current ESP core:
 ```bash
 arduino-cli core uninstall esp32:esp32
 ```

Specify the version number to upgrade the ESP to 3.3.7:
```bash
arduino-cli core install esp32:esp32@3.3.7
```

## Installing and Configuring Libraries
Check the libraries to ensure no libraries are installed:
```bash
arduino-cli lib list
```

Expect:
```
No libraries installed.
```
If there are any installed libraries, uninstall them before proceeding.
  

#### Download required libraries
Download each library below as a zip file or download from GitHub.

| Library | Version | URL |
|---------|---------|-----|
| Adafruit_BusIO | 1.17.4 | https://github.com/adafruit/Adafruit_BusIO |
| Adafruit-GFX-Library | 1.12.5 | https://github.com/adafruit/Adafruit-GFX-Library |
| Adafruit_SSD1306 | 2.5.16 | https://github.com/adafruit/Adafruit_SSD1306 |
| ArduinoJson | 7.4.1 | https://github.com/bblanchon/ArduinoJson |
| ArduinoStreamUtils | 1.9.2 | https://github.com/bblanchon/ArduinoStreamUtils |
| AsyncTCP | 3.4.9 | https://github.com/ESP32Async/AsyncTCP |
| ESPAsyncWebServer | 3.9.6 | https://github.com/ESP32Async/ESPAsyncWebServer |
| BrentIO_esp32FOTA | 2026.4.1 | https://github.com/BrentIO/esp32FOTA |
| BrentIO_PCA95x5 | 2023.10.2 | https://github.com/BrentIO/PCA95x5 |
| BrentIO_PCT2075 | 2023.10.3 | https://github.com/BrentIO/PCT2075 |
| TBPubSubClient | v2.12.1 | https://github.com/thingsboard/pubsubclient |
| Ethernet | 2.0.2 | https://github.com/arduino-libraries/Ethernet |
| LinkedList | 1.3.3 | https://github.com/ivanseidel/LinkedList |
| NTPClient | 3.2.1 | https://github.com/arduino-libraries/NTPClient |
| PCA9685_RT | 0.7.3 | https://github.com/RobTillaart/PCA9685_RT |
| Regexp | 0.1.1 | https://github.com/nickgammon/Regexp |

Install each library above using the following command:
```bash
arduino-cli lib install --zip-path /my/downloads/directory/library_name.zip
```

> [!INFO]  
> Versions must also be changed in the GitHub actions.

## Add Symlink for boards.local.txt

The `boards.local.txt` file must be placed adjacent to the other `boards.txt` file provided by the ESP32 core. It is generated from `boards.local.txt.template` at CI build time, but for local development you must create it manually first.

1. Generate `boards.local.txt` from the template, substituting the bootloader address and app partition size for your target hardware (values are in `devices.yaml`):
```bash
sed \
  -e "s|{{BOOTLOADER_ADDR}}|0x1000|g" \
  -e "s|{{APP_PARTITION_MAX_SIZE}}|6553600|g" \
  ./FireFly-Controller/boards.local.txt.template \
  > ./FireFly-Controller/boards.local.txt
```

2. Create a symlink adjacent to the main boards file. Example for ESP Core version 3.3.7:
```bash
ln -s ./FireFly-Controller/boards.local.txt ~/Library/Arduino15/packages/esp32/hardware/esp32/3.3.7/boards.local.txt
```


## Compile Flags

The following flags must be passed to the compiler regardless of build method (CI or local):

**`ASYNCWEBSERVER_REGEX`** Enables regex path matching in the async web server. Required for the URL routing patterns used throughout the application; must be set as a compile flag (not just a header define) so the library's own source files are also compiled with regex support enabled.

**`PRODUCT_HEX`** The hardware product ID expressed as a hexadecimal. Required — the compiler will error if omitted. Use the `0x`-prefixed value from `devices.yaml` for the target hardware (e.g. `-DPRODUCT_HEX=0x08062305`).

**`ESP32`** Required for Adafruit libraries to configure correctly. Without it, expect errors such as `fatal error: util/delay.h: No such file or directory`.

**`CORE_DEBUG_LEVEL`** Controls debug output verbosity:
- `0` = None
- `1` = Error
- `2` = Warn
- `3` = Info
- `4` = Debug
- `5` = Verbose

**`DISABLE_ALL_LIBRARY_WARNINGS`** Suppresses diagnostic messages from the FOTA library.

The repo root must also be added to the include path (e.g. `-I/path/to/FireFly-Controller`) so that headers in `common/` can be resolved. Abbreviated paths using `~` will not work.

## Partitions

Each hardware model has its own partition layout defined in `devices.yaml`. The `partitions.csv` file is generated automatically at CI build time — there is no static file committed to the repository.

See [Partitions](/controller/support/partitions) for the full partition layout and details on how the table is generated.

## Adding a new hardware version

Hardware configurations are abstracted from the main applications to allow for compilation with minimal hardware-specific design considerations. Each hardware model's pin mappings and hardware constants are defined in `hardware.h`.

Peripheral information and build metadata are defined in `devices.yaml` at the repository root. Adding the product HEX, product ID, `inputs_count`, `outputs_count`, `bootloader_addr`, and `partition_scheme` to `devices.yaml` will include the model in the CI build matrix when its status is `ACTIVE`, and will populate the Product ID drop-down in the Hardware Registration and Configuration application.

## Filter Large JSON documents

The Controller [filters large JSON documents](./configuration_json_filtering.md) in order to conserve memory and protect future upgradeability.

## Dockerfile for ACT

The Dockerfile is maintained in the FireFly-Controller repository at [`.act/dockerfile`](https://github.com/BrentIO/FireFly-Controller/blob/main/.act/dockerfile). Build the image from the `.act/` directory:

Usage for Intel CPU: `docker build --no-cache --platform=linux/amd64 -t act-arduino-ubuntu-24-04:latest .`

Usage for Apple Silicon: `docker build --no-cache --platform=linux/arm64 -t act-arduino-ubuntu-24-04:latest .`

::: info
Be sure to map runner setting `ubuntu-24.04` = `act-arduino-ubuntu-24-04`

On Apple Silicon, Options setting `pull` = `false` is required.
:::

::: info
Be sure to set the Option `artifact-server-path` to a local directory to retrieve the binaries.
:::

## Repository Secrets

The following secrets must be configured in the FireFly-Controller repository under **Settings → Secrets and variables → Actions** for CI workflows to function correctly.

### `FIREFLY_DOCS_TOKEN`

A fine-grained personal access token that allows the FireFly-Controller production CI workflow to automatically open a pull request on the FireFly-Docs repository when the library list changes.

**To create the token:**

1. Go to GitHub → **Settings** → **Developer settings** → **Personal access tokens** → **Fine-grained tokens**
2. Click **Generate new token**
3. Set the resource owner to `BrentIO`
4. Under **Repository access**, select **Only select repositories** and choose `BrentIO/FireFly-Docs`
5. Under **Permissions → Repository permissions**, grant:
   - **Contents**: Read and write
   - **Pull requests**: Read and write
6. Generate the token and copy it immediately

**To install the token:**

1. Go to the FireFly-Controller repository → **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Name: `FIREFLY_DOCS_TOKEN`
4. Value: paste the token generated above