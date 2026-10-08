# Third-party notices

## Original project

This project is based on [Moonlight WebRTC by tsoas](https://github.com/tsoas/moonlight-webrtc-tizen),
distributed under GPL-3.0. The Samsung TV application in this repository derives from it. Its Windows
Gateway, which incorporated `moonlight-common-c`, is no longer part of this repository: the server side is
now [Sunshine-Web-RTC](https://github.com/mlopezsegura/Sunshine-Web-RTC), a GPL-3.0 fork of Sunshine.

## GPL-3.0 source availability

Moonlight WebRTC is licensed under GPL-3.0; the full licence text is at [LICENSE](LICENSE). The
corresponding source of every release is this public repository at the release's tag.

## BrightCraft / Moonlight Tizen

Parts of the Tizen-side implementation of Moonlight WebRTC were reused or adapted from the open-source
[BrightCraft Moonlight Tizen](https://github.com/brightcraft/moonlight-tizen) project and from
implementation ideas developed by that project.

BrightCraft's Moonlight Tizen project is distributed under GPL-3.0. Moonlight WebRTC is also distributed
under GPL-3.0.

We are grateful to BrightCraft and the wider Moonlight Tizen community for their work on bringing Moonlight
game streaming to Samsung televisions and for the implementation experience that helped inform this project.

Any original copyright and licence notices present in reused or adapted source files must remain intact.

## Samsung Tizen WGT

The WGT contains the project's HTML, JavaScript, CSS, assets and normal package-signature metadata.

It also contains the Wake-on-LAN WebAssembly module (`wasm/wake-on-lan.*`). The module is built from
`tizen/wasm/wake-on-lan.c` with Samsung's Emscripten fork, and includes that toolchain's generated JavaScript
runtime and statically linked system libraries:

| Component | Version | Licence / notice source |
| --- | --- | --- |
| Emscripten (Samsung Tizen fork), runtime and system libraries | 1.39.4.7 | MIT or University of Illinois/NCSA; [Emscripten licence](https://github.com/emscripten-core/emscripten/blob/main/LICENSE) |
| musl libc (inside Emscripten) | bundled with 1.39.4.7 | [MIT](https://git.musl-libc.org/cgit/musl/tree/COPYRIGHT) |

## Release maintenance

Publish `LICENSE` and this notice with each release. Re-audit this list whenever the Emscripten toolchain
changes.
