# Tvheadend for MyGica S270

This app packages Tvheadend for the Umbrel host rather than Arcane's
Docker-in-Docker daemon. That distinction matters: DVB device nodes and firmware
must ultimately be visible to the Umbrel host kernel.

## Stable endpoints

Through the Umbrel internal Docker network:

```text
Web/API:   http://knufle-umbrel-tvheadend_server_1:9981
HTSP:      knufle-umbrel-tvheadend_server_1:9982
```

The app uses no fixed Umbrel LAN IP for app-to-app connections.

## Device

The package targets:

```text
Vendor/product: 187f:0600
Device: MyGica S270 / Siano Mobile Digital TV Receiver
System: ISDB-T
Firmware: sms1xxx-hcw-55xxx-isdbt-02.fw + isdbt_rio.inp
Debian package: firmware-siano 20250410-2
```

The firmware helper:

1. checks the host `/lib/firmware`;
2. installs the exact Debian package when either required file is missing;
3. copies every file owned by `firmware-siano` into the host, including
   `sms1xxx-hcw-55xxx-isdbt-02.fw` and `isdbt_rio.inp`;
4. attempts to unbind and bind the S270's USB driver to request firmware again.

On Umbrel installations where the host root is reset during reboot, this helper
repopulates the package files on each app start instead of assuming an earlier
host-level Debian install remains available.

If the device did not exist during startup or the sysfs rebind is refused,
unplug/replug the S270 once or restart Umbrel.

## Tvheadend setup

1. Open the app and complete the wizard.
2. Select the `ISDB-T` network.
3. Use the predefined Brazilian São Paulo mux list.
4. Scan in small batches. Do not start all available frequencies at once when
   using one USB tuner.
5. Map only services with usable video. Do not blindly map every `{PMT:...}`
   fallback service.
6. Create a limited Tvheadend user for Dispatcharr with permission to:
   - stream;
   - read the channel grid;
   - download M3U/XMLTV endpoints.

Suggested playlist URL for Dispatcharr:

```text
http://knufle-umbrel-tvheadend_server_1:9981/playlist/channels.m3u?profile=pass
```

Suggested XMLTV URL:

```text
http://knufle-umbrel-tvheadend_server_1:9981/xmltv/channels
```

If authentication is required, embed or supply the limited Dispatcharr
credentials in Dispatcharr's M3U account. Do not use the Tvheadend admin user.

## Why no `mknod`

Arcane created character nodes inside a second Docker daemon because it could
not directly see the host `/dev/dvb`. This Umbrel-native app is attached to the
Umbrel host Docker daemon and starts with privileged device access, so no fixed
major/minor `mknod` values are required.

## Permissions

The firmware helper and Tvheadend are privileged because:

- host kernel firmware lives outside the container;
- USB driver rebind uses host sysfs;
- DVB nodes can change names and minors between host boots.

This is the main item to discuss with Umbrel maintainers before an official
store submission.

The official App Store linter reports five intentional warnings for this app:

- the firmware service is one-shot and intentionally uses `restart: "no"`;
- the firmware helper needs privileged sysfs access to rebind the USB driver;
- `/lib/firmware` is mounted because kernel firmware requests are resolved by
  the host, not by the container filesystem;
- `/sys` is mounted to locate and rebind `187f:0600`;
- Tvheadend is privileged so it sees the actual DVB device nodes without
  hard-coding volatile character-device major/minor numbers.

These warnings are narrowly tied to USB/DVB hardware support. The package does
not mount the Docker socket, the Umbrel home directory, or other apps' data.
