# Configuration: Certificates

By default, HTTPS OTA validation uses the ESP32 core's built-in Mozilla root CA bundle (~130 CAs, maintained by Espressif and updated with each firmware build).  If you upload one or more certificates here, those uploaded certificates are used exclusively for OTA validation instead of the built-in bundle.  If your OTA server requires a CA not in the Mozilla bundle, upload that root CA here.

See more information about [certificate management](/controller/support/certificate_management).


[![Certificates](./certificates.png)](https://raw.githubusercontent.com/BrentIO/FireFly-Docs/main/controller/software/controller/configuration/certificates.png)