# Monlight Tizen WebRtc client

### Sunshine game streaming for Samsung Tizen TVs — straight from Sunshine

**Moonlight WebRTC streams games from your PC to a Samsung TV at up to 4K 60 FPS with HDR, HEVC, AV1, gamepad support, rumble and a TV-first interface. The TV talks directly to [Sunshine-Web-RTC](https://github.com/mlopezsegura/Sunshine-Web-RTC), a Sunshine build that serves WebRTC itself, so nothing else runs between the game and the TV.**

> [!NOTE]
> **This project is based on [Moonlight WebRTC by tsoas](https://github.com/tsoas/moonlight-webrtc-tizen).**
> The TV application, its interface and the WebRTC design come from that project. This version
> changes the architecture: the separate Windows Gateway is gone, and its job now happens inside
> Sunshine. See [What changed](#what-changed-from-the-original) below.

> [!IMPORTANT]
> **Moonlight WebRTC is in beta.** It is intended for testing on compatible Samsung Tizen TVs and may still contain bugs or device-specific limitations.

![Moonlight WebRTC application library](docs/images/apps.png)

**4K 60 FPS · HDR · HEVC Main10 · AV1 · H.264 · Opus audio · Gamepad + rumble · No video transcoding**

---

## What is Moonlight WebRTC?

Moonlight WebRTC lets you play games from a Sunshine-powered Windows PC directly on a Samsung Tizen TV, with an experience that aims to match Moonlight with Sunshine:

- the same applications, artwork and launch, resume and stop behaviour;
- pairing with a PIN in Sunshine's Web UI;
- every paired TV listed with your Moonlight clients;
- Sunshine's encoder, display and controller settings applied to the TV as to any Moonlight client.

```text
Gaming PC                                         Samsung TV
┌──────────────────────────────────┐             ┌──────────────────────┐
│ Sunshine-Web-RTC                 │   WebRTC    │ Moonlight WebRTC     │
│ capture → encode → WebRTC (RTP)  │ ──────────► │ hardware decode      │
│ GameStream for Moonlight clients │   your LAN  │ gamepad, remote      │
└──────────────────────────────────┘             └──────────────────────┘
```

Sunshine-Web-RTC is a fork of [Sunshine](https://github.com/LizardByte/Sunshine). It keeps everything
Sunshine does for Moonlight clients and adds a WebRTC server for the TV. Moonlight clients and the TV can
use the same PC.

---

## What changed from the original

The original Moonlight WebRTC put a **Windows Gateway** between Sunshine and the TV. The Gateway was a
Moonlight client towards Sunshine and a WebRTC server towards the TV:

```text
Original

Sunshine ── encode ── GameStream (RTP + FEC + encryption) ── loopback ──┐
                                                                        ▼
                              Gateway: receive, decrypt, reassemble each frame,
                              repacketize into WebRTC RTP
                                                                        │
TV ◄──────────────────────────── WebRTC ────────────────────────────────┘
```

Every frame Sunshine encoded therefore went through a full GameStream round trip on the same PC before the
TV saw it:

1. It was split into packets with forward error correction and encryption.
2. It was sent over the loopback network.
3. The Gateway received, decrypted and reassembled it.
4. The Gateway packetized it again for WebRTC.

That hop costs time on every frame. It also makes frame delivery uneven: a frame can only be forwarded once
the whole of it has been received, decrypted and reassembled, so frames left the PC at slightly irregular
intervals. The TV's WebRTC jitter buffer absorbs that unevenness by holding frames longer, or, when it
cannot, shows it as micro-stutter.

Sunshine-Web-RTC removes the hop entirely:

```text
Now

Sunshine-Web-RTC: capture → encode → WebRTC RTP → TV
```

- **Less latency.** Encoded frames go from Sunshine's encoder straight to the WebRTC packetizer, in the
  same process. Nothing is encrypted, decrypted, sent over loopback or reassembled on the way.
- **No micro-stutter from the bridge.** Each frame is sent as soon as the encoder produces it. Its RTP
  timestamp comes from the moment it was captured, not from when another process finished reassembling it.
  The TV therefore receives evenly paced frames with accurate timing, which keeps its jitter buffer small
  and playback smooth.
- **More of the bitrate for the picture.** There is no GameStream forward error correction to pay for on the
  PC, so the encoder gets the bitrate the TV asked for, less only audio and RTP overhead. WebRTC on the LAN
  repairs losses with retransmission and keyframe requests instead.
- **Faster recovery.** When the TV asks for a keyframe, the request goes directly to Sunshine's encoder.
- **Nothing extra to install or keep running.** There is no Windows service, tray application or second
  pairing. The TV pairs with Sunshine itself.

The goal is what Moonlight gives you with Sunshine on a PC or a phone: 4K 60 FPS HDR that feels local. The
difference is that it now runs on the TV with nothing in between.

## The server: Sunshine-Web-RTC and its patches

The server side is **[Sunshine-Web-RTC](https://github.com/mlopezsegura/Sunshine-Web-RTC)**, a fork of
[LizardByte/Sunshine](https://github.com/LizardByte/Sunshine). It is kept as a small set of patches on top of
upstream Sunshine, so it can follow new Sunshine releases.

| Patch | What it changes in Sunshine |
| --- | --- |
| `0001-core-per-session-packet-queues` | Lets a session receive its own encoded video and audio packets, so frames go from the encoder to WebRTC without passing through the GameStream sender. Moonlight sessions are unaffected. |
| `0002-core-webrtc-server-integration` | Starts the WebRTC server with Sunshine, adds the `webrtc_enabled` and `webrtc_port` options, and lists, pairs and manages TVs through Sunshine's existing PIN and client endpoints. |
| `0003-build-libdatachannel` | Builds [libdatachannel](https://github.com/paullouisageneau/libdatachannel) for WebRTC, SRTP and the RTP packetizers. |
| `0004-webrtc-module` | The TV server itself (`src/webrtc/`): this protocol, TV pairing, sessions, the media pump and the gamepad input bridge. |
| `0005-web-ui-tv-pairing-and-options` | TVs in the Web UI's PIN form, and the two options on the Network tab. |
| `0006-tests` | Unit tests and a headless end-to-end test that pairs, streams and measures like a TV. |
| `0007-docs` | The user guide and option documentation. |

- **[PATCHES.md](https://github.com/mlopezsegura/Sunshine-Web-RTC/blob/master/PATCHES.md)** explains every
  change file by file, the stream parameters, what has been verified and how to rebase onto a newer Sunshine.
- **[patches/](https://github.com/mlopezsegura/Sunshine-Web-RTC/tree/master/patches)** holds the patches.
  Applied to the upstream Sunshine commit named in PATCHES.md, they reproduce the fork exactly:

  ```sh
  git clone https://github.com/LizardByte/Sunshine.git && cd Sunshine
  git checkout 0594f62d   # the upstream base named in PATCHES.md
  git apply /path/to/Sunshine-Web-RTC/patches/*.patch
  ```

- **[docs/moonlight_webrtc_tizen.md](https://github.com/mlopezsegura/Sunshine-Web-RTC/blob/master/docs/moonlight_webrtc_tizen.md)**
  covers building, installing and configuring it.

---

# Features

## 🎮 Browse and launch your Sunshine library

Moonlight WebRTC retrieves your Sunshine applications and their artwork and presents them in a TV-friendly library.

From the TV you can:

- browse Sunshine applications;
- view application artwork;
- launch a game;
- resume an existing session;
- disconnect from the stream while leaving the game running;
- stop the Sunshine session when you actually want to close it;
- switch to another application.

The session lifecycle behaves like a normal Moonlight client rather than simply killing the host application whenever the TV disconnects.

---

## 📺 Up to 4K 60 FPS

Streams run at **60 FPS** in several resolution and codec combinations.

| Codec | 720p | 1080p | 1440p | 4K |
|---|:---:|:---:|:---:|:---:|
| H.264 | ✅ | ✅ | — | — |
| HEVC Main | ✅ | ✅ | 🧪 | ✅ |
| HEVC Main10 HDR | — | ✅ | 🧪 | ✅ |
| AV1 Main 8-bit ¹ | ✅ | ✅ | 🧪 | ✅ |
| AV1 Main 10-bit HDR ¹ | — | ✅ | 🧪 | ✅ |

¹ **AV1 is offered only when the PC's GPU can encode it and the TV can decode it over WebRTC.**

🧪 **1440p is experimental.**

Resolution, codec, HDR mode and bitrate are selectable directly from the TV.

![Moonlight WebRTC streaming settings](docs/images/settings.png)

---

## 🌈 HDR

Moonlight WebRTC streams **HEVC Main10 and AV1 Main 10-bit HDR** without tone mapping or transcoding. The
Main10 stream and its Rec.2020 signalling reach the Samsung TV as Sunshine encoded them.

If HDR was requested but the PC's display is not in HDR, the stream stops with an error instead of silently sending SDR.

---

## ⚡ No video transcoding

Sunshine encodes the game once, with your GPU's hardware encoder. That bitstream is what the TV decodes:

```text
Sunshine-Web-RTC encodes (H.264 / HEVC / AV1)
   ↓
WebRTC RTP over your LAN
   ↓
Samsung hardware decoder
```

There is no decode and re-encode anywhere, so there is no second generation of compression, no quality loss
from re-encoding and no transcoding delay.

---

## 🔊 Synchronized audio and video

Audio is delivered as **Opus stereo at 48 kHz** in 5 ms packets.

Audio and video share WebRTC's real-time media timeline. WebRTC handles:

- media timing;
- jitter buffering;
- audio/video synchronization;
- packet-loss feedback;
- stream timing adjustments.

Both streams are timestamped by Sunshine when they are captured, so the TV can keep them in sync without a second custom synchronization layer.

---

## 🎮 Gamepad support

Xbox-compatible controllers can be used directly with the Samsung TV.

Supported input includes analog sticks, triggers, D-pad, face buttons, shoulder buttons, stick buttons, menu/view controls and **controller rumble**.

Gamepad input travels to Sunshine over a **WebRTC DataChannel**. Sunshine treats the controller as a Moonlight controller, so its controller emulation settings apply. Holding Start toggles mouse emulation.

---

## 🛋️ Designed for the TV

The Samsung application is built around a remote-first interface rather than adapting a desktop UI to a television.

It includes:

- multiple saved PCs;
- online/offline status for each PC;
- Wake-on-LAN for a sleeping or powered-off PC;
- remote-friendly IPv4 editing;
- Sunshine application artwork;
- resolution selection;
- codec selection;
- HDR selection;
- bitrate configuration;
- stream controls;
- launch and resume progress feedback.

No keyboard is required for normal use. The app still labels each PC a "Gateway"; that name predates the
architecture change.

![Moonlight WebRTC PC selection](docs/images/gateways.png)

---

## ⏻ Wake-on-LAN

The TV can turn on a PC that is asleep or shut down.

Select an offline PC and the TV sends a Wake-on-LAN magic packet, waits for the PC to boot and opens its library once Sunshine answers. **Wake PC** is also available from the PC's menu (the controller's Menu button).

The TV learns the PC's network adapter address the first time it connects, so connect once while the PC is on. Wake-on-LAN also has to be enabled on the PC:

- in the BIOS/UEFI (often called *Wake on LAN*, *Power On By PCI-E* or *Resume by LAN*; waking from a full shutdown may also need *ErP* disabled);
- in Windows, under the network adapter's **Properties → Power Management** (*Allow this device to wake the computer*, *Only allow a magic packet to wake the computer*) and **Advanced** (*Wake on Magic Packet*).

Use wired Ethernet on the PC: most Wi-Fi adapters cannot wake a computer. The TV and PC must be on the same local network.

---

## 🔐 Pairing and TV management like Moonlight

Pairing works the way it does for Moonlight:

1. The TV shows a four-digit PIN.
2. You select the TV in Sunshine's Web UI and enter that PIN.
3. Only someone signed in to Sunshine's Web UI can approve a TV.

Once paired, the TV appears in Sunshine's client list next to your Moonlight clients. From there you can
disable it, which keeps it paired but refuses it until you enable it again, or unpair it.

---

# Quick start

You need:

1. **[Sunshine-Web-RTC](https://github.com/mlopezsegura/Sunshine-Web-RTC)** on your Windows gaming PC,
   in place of Sunshine.
2. **Moonlight WebRTC** on your Samsung TV.

## 1 — Install Sunshine-Web-RTC

Install Sunshine-Web-RTC and complete Sunshine's usual first-run setup. Its
[guide](https://github.com/mlopezsegura/Sunshine-Web-RTC/blob/master/docs/moonlight_webrtc_tizen.md) covers
building and installing it.

The TV connects to **TCP port 8000**. Keep Sunshine's `webrtc_port` option at its default of 8000, and make
sure nothing else, such as the retired Moonlight WebRTC Gateway service, uses that port.

## 2 — Install the Samsung TV application

Download `MoonlightWebRTC.wgt` from the [Releases page](https://github.com/mlopezsegura/moonlight-webrtc-tizen/releases)
and install it with Apps2Samsung, as described in [Installing on the TV](#installing-on-the-tv).

## 3 — Add your PC and pair

1. Open Moonlight WebRTC on the TV and select **Add Gateway**.
2. Enter the LAN IPv4 address of the PC running Sunshine-Web-RTC. The TV and the PC must be on the same
   local network.
3. The first time, the TV shows a four-digit PIN.
4. On the PC, open Sunshine's Web UI and go to **PIN**.
5. Select the TV in the list of devices waiting to pair (it appears as *Samsung TV* with its address).
6. Enter the PIN, optionally change the device name, and select **Send**.

Your Sunshine application library appears on the TV. Select an application and start streaming.

---

# Installing on the TV

The TV application is installed with **[Apps2Samsung](https://github.com/Apps2Samsung/Apps2Samsung)**, a
free tool for Windows, macOS, Linux and Android that side-loads apps onto Samsung Tizen TVs. It finds the
TV, creates and reuses the Samsung signing certificate, and installs the package, so **Tizen Studio is not
required**.

1. **Put the TV in Developer Mode**, as Apps2Samsung's
   [FAQ](https://github.com/Apps2Samsung/Apps2Samsung/wiki/FAQ#-how-to-enable-developer-mode-on-your-tv)
   explains, using the IP address of the computer or phone that will run Apps2Samsung.
2. **Install Apps2Samsung** from its [releases](https://github.com/Apps2Samsung/Apps2Samsung/releases), on a
   computer or Android phone on the same network as the TV.
3. **Download `MoonlightWebRTC.wgt`** from the latest release on this repository's
   [Releases page](https://github.com/mlopezsegura/moonlight-webrtc-tizen/releases).
4. **Open Apps2Samsung** and select your TV. It can find the TV on the network, or you can enter its IP
   address. The first time, sign in to your Samsung account when asked so that Apps2Samsung can create the
   certificate.
5. **Choose a custom `.wgt`**, select the downloaded `MoonlightWebRTC.wgt`, and install it.

Moonlight WebRTC then appears in the TV's app list.

To update, download the newer `MoonlightWebRTC.wgt` from Releases and install it the same way.

---

# Streaming controls

Moonlight WebRTC distinguishes between disconnecting from a stream and stopping the host application.

## Disconnect

Ends the stream to the TV. The application keeps running on the PC, so you can reconnect later and resume it.

## Stop

Asks Sunshine to close the running application. Use this when you actually want to stop the game on the PC.

## Launch another application

When switching to another application, Moonlight WebRTC ends the previous one before launching the new one.

---

# Current beta limitations

Moonlight WebRTC is still under active development.

Current known limitations include:

- streaming is currently fixed at **60 FPS**;
- PC auto-discovery is not implemented, so PCs are added by IPv4 address;
- the TV always connects to port 8000;
- 1440p support is experimental;
- automatic application updates are not provided.

---

# Troubleshooting

## The TV cannot connect to the PC

Check that:

- Sunshine-Web-RTC is running and its log shows `WebRTC: TV server listening on port 8000`;
- no other program uses port 8000, such as the retired Moonlight WebRTC Gateway service (stop or uninstall
  it);
- the TV and PC are on the same LAN and the IPv4 address entered on the TV is correct.

**Do not disable Windows Firewall as a troubleshooting step.**

## The TV says the PC runs the retired Gateway

The TV connected to the old Moonlight WebRTC Gateway instead of Sunshine-Web-RTC. Uninstall the Gateway,
or stop its service, and make sure Sunshine-Web-RTC listens on port 8000.

## The TV does not appear in Sunshine's PIN page

The TV asks to pair only while it shows the PIN. Make sure the PIN is on screen, then refresh the PIN page. A
PIN that is not entered within five minutes is replaced by a new one. **New PIN** on the TV asks again at
any time.

## The TV says it is disabled

The TV was disabled in Sunshine's client list under **Troubleshooting**. Enable it there; it does not need
to pair again.

## The PC does not wake up

Check that:

- the TV has connected to this PC at least once while it was on;
- Wake-on-LAN is enabled in the PC's BIOS/UEFI and network adapter settings (see [Wake-on-LAN](#-wake-on-lan));
- the PC is connected by Ethernet rather than Wi-Fi.

If the PC wakes from sleep but not from a shutdown, look for a BIOS option allowing Wake-on-LAN from a powered-off state (and disable *ErP*), or try turning off Windows **Fast startup**.

---

# Building from source

This repository contains the Samsung TV application (`tizen/`) and its tests (`tests/Tizen*.js`, run with
Node.js). The server side lives in [Sunshine-Web-RTC](https://github.com/mlopezsegura/Sunshine-Web-RTC).
The protocol between them is described in [docs/protocol.md](docs/protocol.md).

The TV package needs [Samsung's Emscripten SDK](https://developer.samsung.com/smarttv/develop/extension-libraries/webassembly/download.html) (1.39.4.7) to build its Wake-on-LAN WebAssembly module. `packaging/tizen/build-package.ps1` looks for it in `%USERPROFILE%\samsung-emscripten\emscripten-release-bundle\emsdk`; pass `-EmsdkRoot` or set `SAMSUNG_EMSDK` to use another location.

---

# Acknowledgements

- **[Moonlight WebRTC by tsoas](https://github.com/tsoas/moonlight-webrtc-tizen)**
  The original project this one is based on: the Samsung TV application, its remote-first interface, the
  WebRTC streaming design and the Gateway protocol all come from it. This version replaces its Windows
  Gateway with a WebRTC server built into Sunshine.

- **[BrightCraft / Moonlight Tizen](https://github.com/brightcraft/moonlight-tizen)**
  An important reference for the TV application. Parts of its Tizen-side code were reused and adapted, and
  its UI concepts and implementation ideas guided how to build a modern Moonlight experience for Samsung TVs.

- **[Sunshine](https://github.com/LizardByte/Sunshine)** and the **[Moonlight Game Streaming Project](https://github.com/moonlight-stream)**
  For the open-source game-streaming host and protocol that Sunshine-Web-RTC builds on.

- **Samsung Tizen / Samsung Developers**
  For the Tizen platform, WebRTC APIs and developer tooling that make the Samsung TV client possible.

---

# License

Moonlight WebRTC is licensed under the **GNU General Public License version 3**.

See:

- [LICENSE](LICENSE)
- [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)

for project licensing and third-party dependency information.
