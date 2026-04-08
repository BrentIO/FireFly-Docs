# API Testing

Automated REST API test suites are available for both the Controller and Hardware Registration and Configuration applications.  Tests run from your local machine inside a Docker container and connect to a physical device over the network.

> [!NOTE]
> A physical FireFly device flashed with the appropriate application firmware must be reachable on your network before running tests.

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed and running on your local machine
- A physical FireFly device running the Controller or Hardware Registration and Configuration firmware
- The device's IP address

## Building the Docker Image

The test image is shared between both suites and only needs to be built once.  From the root of the `FireFly-Controller` repository:

```bash
docker compose -f tests/docker-compose.yaml build
```

Rebuild the image any time `tests/requirements.txt` or `tests/Dockerfile` changes.

## Running the Controller Test Suite

```bash
DEVICE_IP=<device-ip> docker compose -f tests/docker-compose.yaml run controller
```

The test runner will start and prompt you to enter the Visual Token displayed on the device's OLED screen:

```
Enter visual token from device OLED:
```

Look at the device display, type the token, and press Enter.  The token is promoted to a long-term authorization (valid for 1 hour) before any tests execute, so there is no time pressure once you have entered it.  When the run completes it will delete the container.

## Running the Hardware Registration and Configuration Test Suite

```bash
DEVICE_IP=<device-ip> docker compose -f tests/docker-compose.yaml run --rm hw-reg
```

The same interactive token prompt applies.

> [!WARNING]
> The `test_identity.py` tests write and then delete device identity.  Run these tests against a device that has not yet been provisioned, or one you are prepared to re-provision afterward.

> [!WARNING]
> The `test_mcu.py` reboot test will restart the device.  No further tests should be expected to succeed after the device reboots within the same test session.

## Test Results

After each run, two output locations are populated:

- **Terminal** — live pass/fail output appears during the run
- **HTML report** — a self-contained report is written to `tests/results/` in the repository, with an epoch timestamp in the filename so successive runs do not overwrite each other:
  - `tests/results/controller-<timestamp>.html` — Controller suite results
  - `tests/results/hw-reg-<timestamp>.html` — Hardware Registration and Configuration suite results

Open either file in any browser to review the results.

## Running Without Authentication (DEBUG Firmware)

When testing against a firmware built with `CORE_DEBUG_LEVEL=4`, authentication is bypassed by the device.  Pass `NO_AUTH=true` to skip the token prompt:

```bash
DEVICE_IP=<device-ip> NO_AUTH=true docker compose -f tests/docker-compose.yaml run controller
```
