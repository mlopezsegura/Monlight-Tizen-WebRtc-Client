# Moonlight WebRTC TV protocol

The Samsung TV application talks to [Sunshine-Web-RTC](https://github.com/mlopezsegura/Sunshine-Web-RTC),
a Sunshine build with a built-in WebRTC server. The protocol was designed for the original project's Windows
Gateway (see the [README](../README.md)). Sunshine-Web-RTC speaks it with two changes:

- the TV shows the pairing PIN;
- paired TVs are managed in Sunshine's client list.

Protocol version 2 is a JSON protocol carried by a signaling WebSocket on TCP port 8000. Sunshine's
`webrtc_port` option sets the port, but the TV always uses 8000. Every message contains `"version": 2` and a
`type`. The WebSocket carries configuration, lifecycle state and WebRTC signaling only. Audio, video and
realtime gamepad snapshots travel over WebRTC.

## TV authentication

Every TV must pair once before Sunshine serves it.

- An unauthenticated connection can only authenticate or ask to pair. Any other request is answered with a
  `not-authenticated` error.
- An unauthenticated connection does not displace the TV that is streaming.
- Up to eight unauthenticated connections wait at a time; the oldest is dropped beyond that.

On every connection Sunshine first sends a single-use 256-bit nonce:

```json
{"version":2,"type":"auth-required","nonce":"<64 hex digits>","pairing":"client-pin","macAddress":"2C:F0:5D:7B:E6:D0","sunshineAvailable":true}
```

- **`pairing`** is `"client-pin"`: the TV shows the PIN (see below). The retired Gateway did not send this
  field, so the TV reports such a server as the retired Gateway and does not pair with it.
- **`macAddress`** is the Wake-on-LAN address described below. It is offered before authentication because
  the LAN already sees it through ARP. This lets the TV's reachability probe learn how to wake the PC; the
  probe never authenticates, so it cannot displace a streaming TV.
- **`sunshineAvailable`** is always true, since the server is Sunshine itself. The TV keeps the probe
  connection open while it shows the PC, and marks the PC offline when that connection drops.

A paired TV answers with its client ID and an HMAC-SHA256, keyed with its secret, over the ASCII label
`moonlight-webrtc-tv-auth-v1:` followed by the nonce in lower-case hex:

```json
{"version":2,"type":"authenticate","clientId":"<32 hex digits>","proof":"<64 hex digits>"}
```

Sunshine compares the proof in constant time and replies with `authenticated`, then the startup messages
below. The secret never crosses the network again, and a captured proof cannot be replayed against another
nonce. If the client is unknown or the proof is wrong, Sunshine returns an `error` with `requestType`
`authenticate` and code `authentication-failed`; the TV then forgets its credentials and asks to pair.

### Pairing

Pairing works like Moonlight's. An unpaired TV picks a random four-digit PIN, shows it, and sends:

```json
{"version":2,"type":"request-pairing","pin":"0421","clientName":"Samsung TV"}
```

Sunshine lists the TV in its Web UI under **PIN**, beside any Moonlight clients waiting to pair. The entry
shows the TV's name and address and carries an unguessable request ID. A request lasts five minutes.

When the user selects the TV and enters the PIN, Sunshine does three things:

1. It issues a random 128-bit client ID and 256-bit secret.
2. It stores them in `webrtc_tv_clients.json` in Sunshine's configuration directory, under the name typed in
   the Web UI.
3. It replies:

```json
{"version":2,"type":"paired","clientId":"<32 hex digits>","clientSecret":"<64 hex digits>"}
```

The startup messages follow. A request that does not pair ends with an `error` whose `requestType` is
`request-pairing` and whose code is one of:

- `incorrect-pin` — a wrong PIN was entered; this ends the request;
- `pairing-cancelled` — the request was declined in the Web UI;
- `pairing-expired`;
- `pairing-failed`.

After any of these the TV can send a new request. The TV app does so at once with a new PIN, except after a
cancellation, where it waits until the user chooses **New PIN**. Only a user signed in to the Web UI can
approve a PIN, so there is no attempt limit to guess against. Sunshine keeps at most 32 paired TVs, dropping
the oldest.

TV apps from before this change sent `pair-client` with a PIN shown on the PC. Sunshine answers that with
the error code `unsupported-pairing`.

### Managing paired TVs

Paired TVs appear in Sunshine's client list under **Troubleshooting**, next to Moonlight clients, and can be
disabled or unpaired there.

- **Unpairing** removes the TV's credentials and disconnects it. Its next `authenticate` fails with
  `authentication-failed`, and the TV pairs again.
- **Disabling** disconnects the TV but keeps it paired. Its `authenticate` fails with code
  `client-disabled`; the TV keeps its credentials, reports that it is disabled, and works again once it is
  enabled.

The WebSocket itself is not encrypted, so the secret is visible to a passive observer on the LAN during that
one pairing exchange. Authentication stops other devices on the network from using or interrupting the
stream; it does not make the stream confidential.

## Startup

Opening the WebSocket does not start a stream. Once the TV is authenticated, Sunshine sends:

```json
{"version":2,"type":"gateway-status","gatewayName":"Sunshine-PC","sunshineDetected":true,"sunshinePaired":true,"sessionActive":false,"runningAppId":"7","macAddress":"2C:F0:5D:7B:E6:D0"}
```

- `gatewayName` is Sunshine's name.
- `sunshineDetected` and `sunshinePaired` are always true.
- `runningAppId` is present while an application runs on the host.
- `macAddress` is the Wake-on-LAN address of the PC: the MAC address of the local adapter that routes to
  this TV, in upper-case colon form. It is omitted when that adapter has no Ethernet-style address, such as
  loopback. The TV stores it with the saved PC and later wakes the PC itself, since Sunshine is asleep too.

### Wake-on-LAN

A Tizen web application cannot send UDP from JavaScript. The TV therefore sends the magic packet from a small
WebAssembly module (`tizen/wasm/wake-on-lan.c`), built with Samsung's Emscripten fork, whose Tizen Sockets
extension provides POSIX sockets. Those sockets are refused on the browser main thread, so each request runs
on its own pthread.

Each request sends the standard 102-byte magic packet to UDP port 9 at three destinations:

- the IPv4 limited broadcast `255.255.255.255`;
- the PC's last known IPv4 address;
- the IPv6 all-nodes group `ff02::1`.

The request succeeds when at least one of those sends does.

Selecting an offline PC whose address is known wakes it, as does **Wake PC** in its menu. The TV then keeps
reconnecting, with 5-second attempts, for up to two minutes while the PC boots, and opens the application
library once `gateway-status` arrives.

### Capabilities

Sunshine also sends `capabilities`, which lists explicit `videoModes` so the TV never has to infer a
resolution and codec combination. Each mode contains:

- `width`, `height` and `fps`;
- `codecs`, `hdrCodecs` and `defaultCodec`;
- `defaultBitrateKbps`;
- `experimental`, `hdrSupported` and `hdrExperimental`.

`hdrCodecs` lists the codecs of `codecs` that may be combined with HDR in that mode.

| Mode | Codecs | HDR | Default | Default bitrate | Experimental |
| --- | --- | --- | --- | ---: | --- |
| 1280x720 @ 60 | H.264, HEVC, AV1 | Off | H.264 | 12000 kbps | No |
| 1920x1080 @ 60 | H.264, HEVC, AV1 | Off, On | H.264 | 20000 kbps | HDR only |
| 2560x1440 @ 60 | H.264, HEVC, AV1 | Off, On | HEVC | 30000 kbps | Yes |
| 3840x2160 @ 60 | HEVC, AV1 | Off, On | HEVC | 50000 kbps | HDR only |

AV1 depends on the GPU encoder:

- AV1 is listed only when Sunshine's encoder probe found an AV1 encoder.
- AV1 HDR is listed in `hdrCodecs` only when that encoder also supports Main 10-bit.

Sunshine checks the encoders every five seconds and sends `capabilities` again when they change, for example
when a hardware encoder appears after the first probe. The TV further hides AV1 when
`RTCRtpReceiver.getCapabilities("video")` does not list `video/AV1`.

The selectable bitrates are 10000, 12000, 15000, 20000, 25000, 30000, 40000, 50000, 60000, 80000 and 100000
kbps. HDR defaults to off. Audio is stereo Opus at 48 kHz and the frame rate is fixed at 60 fps. The whole
1440p mode, including HDR, is experimental because Samsung does not list it in the official Cloud Gaming
resolution table.

## Applications

Client request:

```json
{"version":2,"type":"get-apps"}
```

Response, with the entries of Sunshine's application list:

```json
{"version":2,"type":"apps","apps":[{"id":"0","title":"Desktop","artworkAvailable":false,"running":false}]}
```

`id` is Sunshine's application ID. `running` marks the application running on the host.
`artworkAvailable` only reports whether the artwork is already cached, so the list stays small and renders
without waiting for images.

Artwork is fetched lazily, one asset at a time:

```json
{"version":2,"type":"get-app-artwork","appId":"0"}
```

Sunshine returns the application's cover image when it is PNG, JPEG or WebP:

```json
{"version":2,"type":"app-artwork","appId":"0","available":true,"mimeType":"image/jpeg","data":"...base64..."}
```

Missing or unsupported artwork returns `"available":false`, and the TV keeps its fallback card.

## Start a session

```json
{
  "version": 2,
  "type": "start-session",
  "appId": "0",
  "video": {
    "width": 3840,
    "height": 2160,
    "fps": 60,
    "codec": "hevc",
    "bitrateKbps": 50000,
    "hdr": true
  },
  "audio": {"channels": 2}
}
```

`codec` is exactly `"h264"`, `"hevc"` or `"av1"`.

| Codec | `hdr: false` | `hdr: true` |
| --- | --- | --- |
| HEVC | Main profile, 8-bit SDR, Rec.709 | Main10 HDR, Rec.2020; 1920x1080, 2560x1440 or 3840x2160 only |
| AV1 | Main 8-bit SDR, Rec.709 | Main 10-bit HDR, Rec.2020; same resolutions as HEVC HDR |
| H.264 | 8-bit SDR | Rejected |

720p HDR is rejected. Sunshine validates every field and rejects unsupported settings; there is no silent
codec or SDR fallback. A session asking for AV1 or AV1 HDR fails when the encoder cannot produce it.

Sunshine launches the application, or resumes it when it is already running, and prepares the display as it
would for a Moonlight client. It then captures, encodes and sends the stream over WebRTC.

Status transitions use `session-status` with one of `idle`, `starting`, `starting-webrtc`,
`connecting-sunshine`, `starting-moonlight`, `streaming`, `stopping` or `error`. The intermediate states
are kept so the TV shows the same progress as before.

If HDR was requested and the host display is not in HDR, the session ends with an error. If the application
exits on the host, the session ends with an error too.

### SDP

For HEVC HDR, the SDP format parameters explicitly request Main10 (`profile-id=2`), Main tier, and level
4.1 at 1080p60, level 5.0 at 1440p60 or level 5.1 at 4K60. Sunshine rejects the session if the Tizen answer
does not preserve that profile, rather than sending Main10 under a Main 8-bit negotiation.

The standard WebRTC RTP colour-space extension is offered for HDR but never transmitted. On this Samsung
runtime the TV decoded no frames when it was present, and the HEVC VUI/SEI or AV1 sequence header already
carry the colour description.

### Feedback

Each started session receives a positive `sessionId`. WebRTC `offer`, `answer` and `candidate` messages
contain that ID, so late signaling from a stopped session can be ignored.

When the TV requests a keyframe (RTCP PLI), Sunshine's encoder produces one directly.

## Stop a session

```json
{"version":2,"type":"stop-session"}
```

Sunshine disconnects the controller, stops capture, closes the PeerConnection, keeps the WebSocket open and
returns to `idle`. This is a client disconnect only: the application keeps running and the next
`start-session` for it resumes it.

If the same TV reconnects while it is streaming, for example after a network drop, the new connection
replaces the old one and the old connection's stream is stopped.

## Stop or switch a running application

The TV asks Sunshine to stop the host-side application with:

```json
{"version":2,"type":"stop-host-session"}
```

Sunshine first closes any stream to this TV, then terminates the application and reports
`host-session-status` with `"state":"stopped"`.

Switching is one ordered operation. Sunshine validates the target, stops the current application and only
then starts the new one:

```json
{
  "version":2,
  "type":"switch-session",
  "appId":"7",
  "video":{"width":1920,"height":1080,"fps":60,"codec":"hevc","bitrateKbps":20000,"hdr":false},
  "audio":{"channels":2}
}
```

`host-session-status` carries the transition state (`stopping`, `switching`, `resuming`, `starting`,
`stopped` or `failed`) and optional running and target application IDs.

## Errors

```json
{"version":2,"type":"error","requestType":"start-session","code":"unsupported-settings","message":"Unsupported resolution"}
```

Errors contain no credential or pairing secret.
