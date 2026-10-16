# PASUPASASTRA bridge protocol (v2)

A bridge is the app on the phone or remote controller that is connected to the drone. **The bridge
opens a WebSocket to the GCS server** (`gcs-api` in
[pasupasastra-gcs-be](https://github.com/DarmawanWeb/pasupasastra-gcs-be)); the server never
connects to a bridge. One connection is one drone.

| Endpoint | For | Drone ID taken from |
|---|---|---|
| `ws://<server>:8080/ws/drone` | Autel and new bridges | `droneId` field of every message |
| `ws://<server>:8080/bridge/<droneId>` | A bridge with a fixed ID | path |
| `ws://<server>:8080/telemetry/<droneId>` | DJI bridge (DJI format) | path |
| `ws://<server>:8080/telemetry?drone=<droneId>` | DJI bridge (DJI format) | query, default `GCS_DJI_DEFAULT_DRONE_ID` |

When `GCS_BRIDGE_TOKEN` is set on the server, append `?token=<token>` to the URL (without it the
socket is closed with code 4401). The drone must already be registered on the **Drone** page of the
console with the same ID; data for an unknown ID is ignored and logged once in the event log.

## 1. Hello and capabilities

The first message after connecting, and again whenever the capabilities change (for example when
the pilot toggles the interlock):

```json
{"type":"hello","droneId":"x-001","vendor":"acme","capabilities":["telemetry","commands","mission"]}
```

Capabilities: `telemetry`, `commands`, `mission`, `drop`, `lens`. Without `hello`, a DJI bridge is
`telemetry,lens` and anything else is `telemetry`. The console disables the controls a bridge does
not support (tooltip "Bridge read-only …") and the server rejects those commands with the same
message. Announce `commands` and `mission` only while the bridge's **Izinkan kendali dari GCS**
(allow control from the GCS) interlock is on.

## 2. Telemetry

### Generic GCS format (use this for new bridges)

```json
{"type":"telemetry","droneId":"x-001",
 "data":{"timestamp":1790000000000,"lat":-6.2,"lon":106.84,"altM":42.1,"headingDeg":91.0,
         "groundSpeedMs":7.5,"climbRateMs":0.5,"satellites":16,"gpsFix":"3d","batteryPct":64,
         "armed":true,"flightMode":"AUTO","phase":"airborne","homeLat":-6.2,"homeLon":106.84}}
```

| Key | Unit or values |
|---|---|
| `timestamp` | epoch milliseconds (the server's time when missing) |
| `lat`, `lon` | degrees |
| `altM`, `altMslM` | metres above home, metres above mean sea level |
| `rollDeg`, `pitchDeg`, `headingDeg` | degrees; heading 0–360 |
| `groundSpeedMs`, `climbRateMs` | m/s; climb is positive going up |
| `satellites`, `gpsFix`, `hdop` | count; `none`, `2d`, `3d`, `dgps`, `rtk`; HDOP |
| `batteryPct`, `batteryV`, `batteryA` | %, volts, amperes |
| `rssiPct` | link quality, % |
| `armed`, `flightMode`, `phase` | boolean; `MANUAL`, `STABILIZE`, `ALT_HOLD`, `POSHOLD`, `LOITER`, `GUIDED`, `AUTO`, `RTL`, `LAND`; `landed`, `takeoff`, `airborne`, `landing` |
| `homeLat`, `homeLon`, `distanceToHomeM`, `flightTimeS` | degrees, metres, seconds |
| `missionSeq`, `missionTotal` | current and total waypoints |
| `payload` | `stowed`, `armed`, `released` |
| `gimbalPitchDeg` | degrees |

Send what the aircraft has; every key is optional and unknown keys are dropped. Do not put
`latitude`, `relativeAltitude`, `batteryPercentage` or `connected` in `data`: those mark the Autel
format below. 5–10 Hz is enough; the server broadcasts to the consoles at
`GCS_TELEMETRY_BROADCAST_HZ` (5 Hz) and records everything it receives.

### DJI Matrice 4T format (MSDK v5), 10 Hz

Bridge v2 adds `mode`, `motors` and `home{lat,lon}`, and sends `hello`, `ack` and `event` on the
same connection.

```json
{"pos":{"lat":-6.2,"lon":106.84,"alt":42.1},"att":{"pitch":-2.1,"roll":0.4,"yaw":-91.0},
 "vel":{"x":4.2,"y":-0.3,"z":-0.5},"heading":-91.0,"flying":true,"sats":19,"battery":78,"ts":1790000000000}
```

- `heading` and `yaw` in −180..180 are normalised to 0..360. Positive `vel.z` is down (NED).
- `flying` drives the flight phase; home is the last position before takeoff.
- Server to bridge (capability `lens`): `{"cmd":"set_lens","lens":"rgb"|"thermal"}`,
  `{"cmd":"set_display","mode":"split"|"single"}`.

### Autel EVO II V3 format (Autel SDK v1.5), 5 Hz

```json
{"type":"telemetry","droneId":"evo2-v3-001","timestamp":1790000000000,
 "data":{"connected":true,"latitude":-6.21,"longitude":106.85,"relativeAltitude":42.0,"altitudeMsl":50.1,
         "roll":2.0,"pitch":-4.0,"yaw":181.0,"heading":181.0,"groundSpeed":7.5,"velocityZ":-0.5,
         "satelliteCount":16,"gpsFixType":3,"batteryPercentage":64,"batteryVoltage":15.2,
         "homeLatitude":-6.2,"homeLongitude":106.84,"distanceToHome":1200,"flightMode":"GPS_FLIGHT",
         "rcSignalLost":false,"goHomePending":false,"gimbalPitch":-30.0}}
```

- Autel `flightMode` values map to GCS modes (`GPS_FLIGHT` → `POSHOLD`, `NORMAL_GO_HOME` → `RTL`,
  `LANDING` → `LAND`, …).
- The angle unit is set with `GCS_AUTEL_ATTITUDE_UNIT=deg|rad`, the vertical speed sign with
  `GCS_AUTEL_VZ_DOWN_POSITIVE`.
- `connected:false` makes the server ignore the position and log an "aircraft disconnected from the
  RC" event.

## 3. Commands and missions

Server to bridge:

```json
{"type":"command","id":"c-8f2a","command":{"type":"takeoff","altM":30}}
{"type":"mission","id":"m-11b0","plan":{"name":"…","takeoffAltM":30,"endAction":"rtl","waypoints":[…]}}
```

Bridge to server, matched by `id`, within 10 s for a command and 30 s for a mission:

```json
{"type":"ack","id":"c-8f2a","accepted":true,"message":"Lepas landas ke 30 m"}
```

`message` is shown to the operator as is, so bridges write it in Bahasa Indonesia. `command.type` is
one of `arm`, `disarm`, `takeoff{altM}`, `land`, `rtl`, `hold`, `set-mode{mode}`,
`goto{lat,lon,altM}`, `mission-start`, `mission-pause`, `payload-release`, `set-lens{lens}`. Every
command, accepted or rejected, is written to the `command_log` audit table.

Safety limits are checked twice, on the server and in the bridge against the latest GPS fix:
mission altitude 5–120 m and not above the drone's configured maximum, waypoints at most 1000 m from
the drone, speed at most 10 m/s, at most 99 waypoints.

## 4. Events

Things the operator must know (low battery, a mission finished, a rejected SDK call) go to the
console's event log:

```json
{"type":"event","droneId":"x-001","severity":"warning","message":"Baterai 20%"}
```

`severity` is `info`, `warning` or `critical`; the message is in Bahasa Indonesia.

## 5. Video

Video does not go over this socket. A bridge or the controller's own app publishes RTMP (or RTSP,
SRT) to MediaMTX at `rtmp://<server>:1935/<path>`, and the same URL is registered as the drone's
video source in the console.

## 6. Demo drones

A drone with the **Drone demo** switch is simulated on the server (no bridge needed), has every
capability and shows in purple with a DEMO label. Payload drops only execute on demo drones; drop
targets for real drones are still saved and shown on the map.

The operator's guide, in Indonesian, is the
[user manual](https://github.com/DarmawanWeb/Pasupasastra/blob/main/docs/PASUPASASTRA-Manual-Pengguna.pdf).
