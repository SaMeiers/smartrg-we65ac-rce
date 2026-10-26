# SmartRG WE65ac command injection

The SmartRG WE65ac WiFi extender runs a web panel built on TBS Software Trunk
R2. The panel takes the uplink SSID and passphrase and passes them to
`system()` without quoting. An authenticated admin can put shell
metacharacters in either field and run commands as root.

## Affected

- SmartRG WE65ac, hardware V1.01, firmware V1.0.1 (build 2016-07-04).
- Other devices using the same TBS panel code may be affected. I have only
  tested the model above.

## Details

The values of these nodes are saved from the "connect to router" page:

```
:InternetGatewayDevice.LANDevice.1.X_TWSZ-COM_ApCliConfiguration.1.ConnectRootAp.SSID
:InternetGatewayDevice.LANDevice.1.X_TWSZ-COM_ApCliConfiguration.1.ConnectRootAp.PreSharedKey.1.KeyPassphrase
```

The SSID accepts 32 characters and the passphrase 63. Both end up in a shell
command line. A value like `;id;#` runs `id` as root; the final `#` comments out
the rest of the original command.

The panel listens on `http://<device>/cgi-bin/webproc`. The exploit logs in with
the admin credentials and sends a normal `obj-action=set` request for the
`advanced/conntorouter` page.

Execution is blind: the HTTP response does not include the command output. Use a
command with an observable effect. For example:

```
python3 exploit.py 192.168.1.1 admin PASSWORD "sleep 5"
```

takes about five seconds longer than a normal request when the injection works.
A reverse shell or a callback also works, as long as it fits in 63 characters.

Writing the passphrase sets `ConnectRootAp.Connect` to 1, which makes the device
look for the uplink and can change its IP address. The exploit writes the
original passphrase back and sets `Connect` to 0.

## Impact

Command execution as root on the device. The same panel also serves the running
configuration without authentication through
`/cgi-bin/webupg?name=downloadConfig`, and that file contains the admin password
in clear text. The two issues can be chained, so an attacker with access to the
panel port can go from an unauthenticated request to root. That config leak is a
separate bug and is not covered by the exploit here.

## Firmware

The full 8 MB flash and the extracted partitions are in the
[firmware-v1.0.1 release](https://github.com/SaMeiers/smartrg-we65ac-rce/releases/tag/firmware-v1.0.1):
complete flash, bootloader, kernel, rootfs partition, decompressed `vmlinux`,
extracted rootfs tarball and `SHA256SUMS`.

The bootloader block holds factory data for the unit in clear text, including
the serial number and the default web and WiFi credentials. Those are the
values shipped from the factory, not the ones in use on the network where the
device was tested.

## Fix

There is no fixed firmware for this model that I know of. The last build is from
2016. Mitigations:

- Keep the panel off untrusted networks.
- Change the default admin password.
- Do not reuse that password anywhere else.

## Timeline

- 2026-10-25: vulnerability found on a device owned by the author.
- 2026-10-25: published.
- CVE: not assigned yet.

## Credit

Sa Meiers

Use this only on devices you own or have permission to test.
