# Configuration: Certificates

The firmware includes built-in root CAs (Amazon Root CA 1 and Starfield Services Root CA - G2) that cover most HTTPS OTA servers out of the box.  Additional certificates can be uploaded here if your OTA server uses a different certificate authority.  All uploaded certificates are combined with the built-in root CAs into a single bundle used for all HTTPS OTA validation.

See more information about [certificate management](/controller/support/certificate_management).


[![Certificates](./certificates.png)](https://raw.githubusercontent.com/BrentIO/FireFly-Docs/main/controller/software/controller/configuration/certificates.png)