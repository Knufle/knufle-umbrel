# Knufle Umbrel Apps

Custom Umbrel apps maintained for personal use and prepared for possible upstream
submission to `getumbrel/umbrel-apps`.

## TV stack

| App | Role |
| --- | --- |
| [`knufle-umbrel-tvheadend`](./knufle-umbrel-tvheadend/) | Sintonizes DVB/ISDB-T with the MyGica S270 and provides the channel/EPG source. |
| [`knufle-umbrel-dispatcharr`](./knufle-umbrel-dispatcharr/) | Organizes Tvheadend streams and exposes them to Plex as an emulated HDHomeRun tuner. |

The intended flow is:

```text
MyGica S270 (USB 187f:0600)
  -> Tvheadend
  -> Dispatcharr
  -> Plex HDHomeRun tuner
```

The official Umbrel Plex app runs with host networking. Dispatcharr therefore
publishes its HDHomeRun API on the Umbrel host at port `9191`. Use the Umbrel
LAN address so Plex stores channel-logo URLs that browser clients can also
reach:

```text
http://192.168.68.50:9191/hdhr
```

Dispatcharr reaches Tvheadend through Docker DNS:

```text
http://knufle-umbrel-tvheadend_server_1:9981
```

Do not configure Plex with `127.0.0.1`: Plex would save that loopback address in
the guide metadata and a browser on another computer would try to load its logos
from itself.

## Firmware

The Tvheadend app includes a one-shot firmware installer. It installs Debian's
official `firmware-siano` package version `20250410-2` only when
`sms1xxx-hcw-55xxx-isdbt-02.fw` is absent from the host, copies the file to
`/lib/firmware`, and attempts to rebind USB device `187f:0600` so the kernel
requests the firmware again.

Installing the firmware requires a writable `/lib/firmware` host mount,
privileged access to USB sysfs, and `/sys` access. These requirements are
documented because they matter for a future official-store review.

## Upstream submission

The custom-store IDs are intentionally prefixed. Before submitting upstream:

1. Rename `knufle-umbrel-tvheadend` to `tvheadend`.
2. Rename `knufle-umbrel-dispatcharr` to `dispatcharr`.
3. Update the generated container names in both READMEs and Dispatcharr's
   Tvheadend M3U/EPG URLs.
4. Replace the custom-store `submission` URL placeholders.
5. Add reviewed screenshots as gallery assets.
6. Discuss the Tvheadend firmware helper's host `/lib/firmware` mount and USB
   sysfs rebind with Umbrel maintainers.

See [MIGRATION-NOTES.md](./MIGRATION-NOTES.md) for the changes made during the
earlier Arcane-based experiment.
