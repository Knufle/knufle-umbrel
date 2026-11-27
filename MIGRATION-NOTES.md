# Migration notes from the Arcane experiment

This file records direct changes from the earlier MyGica S270 experiment. It
does not contain passwords or generated credentials.

## Host-level changes

- Debian package installed on the Umbrel host:
  `firmware-siano` version `20250410-2`.
- The package added the required firmware file:
  `/lib/firmware/sms1xxx-hcw-55xxx-isdbt-02.fw`.
- The Siano board also requests `/lib/firmware/isdbt_rio.inp` from the same
  package. Umbrel reboots can remove the earlier host-level installation of
  both files.
- The tuner identified by the host was:
  - USB vendor: `187f`
  - USB product: `0600`
  - driver family: Siano `smsusb` / `smsmdtv` / `smsdvb`
  - DVB frontend: `Siano Mobile Digital MDTV Receiver`
- The new Tvheadend app repopulates the full `firmware-siano` package on the host
  when either required file is absent. Its persistent helper is healthchecked
  and Tvheadend starts only after both files exist; do not rely on a one-time
  host package installation surviving an Umbrel reboot.

## Arcane application

The installed Arcane Umbrel app's own Compose file was **not modified**.

An Arcane project named `tvheadend` was created at:

```text
/home/umbrel/umbrel/app-data/arcane/data/projects/tvheadend/compose.yaml
```

The project contained the following custom Docker configuration:

- `lscr.io/linuxserver/tvheadend:latest` (unpinned, experimental only)
- `prengineer/tvhproxy:latest` (unpinned, experimental only)
- `cap_add: MKNOD`
- a `c 212:* rwm` device cgroup rule
- startup creation of DVB nodes because Arcane's Docker-in-Docker environment
  did not expose them directly:
  - `/dev/dvb/adapter0/frontend0`: `212,2`
  - `/dev/dvb/adapter0/demux0`: `212,0`
  - `/dev/dvb/adapter0/dvr0`: `212,1`
- persistent named volumes for Tvheadend config and recordings
- a protected `proxy.secret` read from the Tvheadend config volume
- a Tvheadend superuser compatibility record for the proxy
- the published Tvheadend proxy port `5004:5004`

The proxy container also had runtime source patches:

- corrected `device.xml` Jinja variables from `data.*` to `device.*`
- replaced missing/repeated channel number `0` / `00` with channel IDs
- added an authenticated stream relay
- changed advertised stream URLs from an internal hostname to
  `http://192.168.68.50:5004`
- tolerated unknown EPG genre IDs
- changed startup caching behavior during later debugging

These are reason-specific workarounds from the experiment. They must not be
copied blindly into the new Umbrel-native apps.

## Tvheadend configuration

The experimental volume contains:

- the completed setup wizard
- São Paulo muxes
- mapped services/channels
- normal user/access records
- internal proxy credentials
- EPG/channel configuration

None of the generated credentials are included in this repository.

When switching to the Umbrel-native Tvheadend app, either:

1. configure a clean installation, preferred for a reproducible baseline; or
2. make a backup of the Arcane Tvheadend config volume and restore only its
   channel/mux state after reviewing access-control and password files.

## Plex

No direct modification was made to the official Plex Umbrel app's Compose file
or image. Plex settings were changed through its UI:

- a tuner at `http://192.168.68.50:5004`
- XMLTV from `http://192.168.68.50:5004/epg.xml`
- channels and guide mapping
- manual public port `22669` for Remote Access during troubleshooting

The router was later given a persistent TCP forwarding rule:

```text
WAN 22669 -> 192.168.68.50:32400
```

The new setup intentionally uses `127.0.0.1:9191` inside Umbrel so Plex does
not depend on that LAN address for Live TV.

## Shutdown sequence

Before testing the new apps:

1. Stop the Arcane `tvheadend` project.
2. Keep Arcane itself installed unless it is no longer needed.
3. Remove the experimental Plex tuner or keep it disabled while testing.
4. Install and verify `knufle-umbrel-tvheadend`.
5. Install and verify `knufle-umbrel-dispatcharr`.
6. Add Dispatcharr to Plex only after its own HDHomeRun and stream tests pass.
