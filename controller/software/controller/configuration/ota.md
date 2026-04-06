# Configuration: OTA Updates

Controllers and <Badge type="warning" text="TODO" /> Clients can have their firmware and UI updated over-the-air.  By default the device will check once per day, approximately at the time the device was booted, for new firmware.

If `OTA Disabled` is checked, no OTA configuration is sent for that device type.

Both `http` and `https` URLs are supported.  When `https` is used, the firmware validates the server certificate against a built-in bundle of root CAs (Amazon Root CA 1 and Starfield Services Root CA - G2) plus any certificates you have uploaded to the device.  No separate certificate selection is required.

You can configure the URL to include wildcards, which will be substituted at execution time.  The underlying library will URL encode as necessary.

| Wildcard | Example Value |
| -------- | ------------- |
| `$$pid$$` | `FFC3232-2305` |
| `$$app$$` | `FireFly Controller` |

::: info EEPROM Required
Using `$$pid$$` requires the external EEPROM to be present and programmed.
:::

Additional information about [OTA updates](/controller/support/ota_updates) can be found on the support page.

[![OTA](./ota.png)](https://raw.githubusercontent.com/BrentIO/FireFly-Docs/main/controller/software/controller/configuration/ota.png)