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

Firmware preparation:

1. an Umbrel `pre-start` hook runs on the host before Compose;
2. the hook installs `firmware-siano=20250410-2` if either required file is
   missing;
3. it verifies `sms1xxx-hcw-55xxx-isdbt-02.fw` and `isdbt_rio.inp`;
4. it attempts to unbind and bind the S270's USB driver;
5. only then does Umbrel start Tvheadend.

On Umbrel installations where the host root is reset during reboot, the
`pre-start` hook repopulates the Debian package on every app start instead of
assuming an earlier host-level installation remains available. There is no
firmware sidecar: one hook prepares and rebinds the device before the Tvheadend
container starts.

If the device did not exist during startup or the sysfs rebind is refused,
unplug/replug the S270 once or restart Umbrel.

## Tvheadend setup

1. Open the app and complete the wizard.
2. Select the `ISDB-T` network.
3. Use the predefined Brazilian São Paulo mux list.
4. Edit the network at Expert level and set **Character set** to
   `ISO-8859-1`. Brazilian ISDB-T EIT data from these muxes is decoded through
   ISO-6937 incorrectly when it remains on `AUTO`, producing text such as
   `FØ`, `notŦcias`, and `regiªo`.
5. Scan in small batches. Do not start all available frequencies at once when
   using one USB tuner.
6. Map only services with usable video. Do not blindly map every `{PMT:...}`
   fallback service.
7. Create a limited Tvheadend user for Dispatcharr with permission to:
   - stream;
   - read the channel grid;
   - download M3U/XMLTV endpoints.

Suggested playlist URL for Dispatcharr:

```text
http://knufle-umbrel-tvheadend_server_1:9981/playlist/channels.m3u?profile=pass
```

Suggested XMLTV URL:

```text
http://knufle-umbrel-tvheadend_epg_1:9983/xmltv.xml
```

The bridge endpoint is preferred over Tvheadend's legacy
`/xmltv/channels` export. Tvheadend displays events collected by OTA EIT, but
its legacy XMLTV exporter can omit events without a `title`. The bridge reads
`/api/epg/events/grid`, fills a safe title from title/subtitle/description, and
exports all enabled channels to Dispatcharr. Channels with no OTA EPG receive
two-hour placeholder programmes so clients such as Plex continue to identify
the current channel instead of showing `Unknown Airing`. Channel icons use the
stable container DNS name so Dispatcharr can cache them after Tvheadend moves to
a different internal IP.

If authentication is required, embed or supply the limited Dispatcharr
credentials in Dispatcharr's M3U account. Do not use the Tvheadend admin user.

## Why no `mknod`

Arcane created character nodes inside a second Docker daemon because it could
not directly see the host `/dev/dvb`. This Umbrel-native app is attached to the
Umbrel host Docker daemon and starts with privileged device access, so no fixed
major/minor `mknod` values are required.

## Permissions

Tvheadend is privileged and the pre-start hook accesses host sysfs because:

- host kernel firmware lives outside the container;
- USB driver rebind uses host sysfs;
- DVB nodes can change names and minors between host boots.

This is the main item to discuss with Umbrel maintainers before an official
store submission.

The official App Store linter reports five intentional warnings for this app:

- the hook needs host access to restore `/lib/firmware` and rebind the USB
  driver;
- `/lib/firmware` is mounted because kernel firmware requests are resolved by
  the host, not by the container filesystem;
- `/sys` is mounted to locate and rebind `187f:0600`;
- Tvheadend is privileged so it sees the actual DVB device nodes without
  hard-coding volatile character-device major/minor numbers.

These warnings are narrowly tied to USB/DVB hardware support. The package does
not mount the Docker socket, the Umbrel home directory, or other apps' data.
