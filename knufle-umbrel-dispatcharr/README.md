# Dispatcharr as the Tvheadend-to-Plex bridge

Dispatcharr replaces the experimental `tvhproxy` role while Tvheadend remains
responsible for DVB hardware, mux scanning, and over-the-air reception.

## Stable URLs

Umbrel web UI:

```text
http://knufle-umbrel-dispatcharr_server_1:9191
```

Tvheadend source from inside Dispatcharr:

```text
http://knufle-umbrel-tvheadend_server_1:9981
```

Plex's HDHomeRun source:

```text
http://127.0.0.1:9191/hdhr/all
```

`127.0.0.1` is intentional. The official Umbrel Plex app uses host networking,
so it reaches Dispatcharr through the published port without requiring a fixed
Umbrel LAN address. For a Plex server on another machine, use the Umbrel LAN
hostname or address explicitly instead.

## Initial setup

1. Create the Dispatcharr administrator.
2. Open **M3U & EPG Manager**.
3. Add a standard M3U account with the Tvheadend playlist:

   ```text
   http://knufle-umbrel-tvheadend_server_1:9981/playlist/channels.m3u?profile=pass
   ```

4. Add an XMLTV source with Tvheadend's endpoint:

   ```text
   http://knufle-umbrel-tvheadend_epg_1:9983/xmltv.xml
   ```

   This Tvheadend app endpoint reads the same EPG events shown in the
   Tvheadend UI. It is preferred to the legacy `/xmltv/channels` endpoint,
   which can omit OTA events that contain only subtitle/description fields.
   It also generates clearly identified placeholder blocks for channels whose
   OTA signal provides no guide data, avoiding Plex's `Unknown Airing`.

5. If Tvheadend requires authentication, use a dedicated streaming user rather
   than the administrator and enter its credentials in the M3U account.
6. In Dispatcharr **Channels**, enable only real TV streams. Exclude radio,
   one-segment duplicates, data PIDs, and technical `{PMT:...}` fallbacks.
7. Create or select the `all` channel profile.
8. Copy the HDHomeRun URL for that profile.

## Plex setup

In Plex:

1. Remove or disable the old tvhproxy tuner during the migration.
2. Choose **Set Up Plex Tuner**.
3. Enter the tuner URL manually:

   ```text
   http://127.0.0.1:9191/hdhr/all
   ```

4. Use Dispatcharr's XMLTV output if Plex cannot map the guide automatically.
5. Verify playback and recording before deleting the Arcane project.

## Important limitation

Dispatcharr cannot scan DVB hardware and cannot recover a channel name that
Tvheadend never received in the broadcast's SDT metadata. It can rename,
normalize, group, match EPG entries, and proxy the resulting streams, but the
DVB acquisition layer remains Tvheadend.

## Why AIO

The official Dispatcharr AIO mode bundles its Redis and PostgreSQL dependencies
in one pinned container. It is appropriate for the current Umbrel use case and
matches the upstream quick-start. A modular deployment would require four
containers and is unnecessary until independent scaling or external database
security is needed.
