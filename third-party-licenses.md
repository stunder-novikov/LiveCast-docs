# Third-party licenses

LiveCast ships one vendored third-party library and otherwise builds against
system libraries and Unreal Engine modules. Everything here is compatible with
selling the plugin commercially. The only attribution obligation is the MIT
notice for srs-librtmp, reproduced in full below.

---

## srs-librtmp — MIT License

The RTMP transport. A prebuilt static library (`Source/ThirdParty/srs_librtmp/`),
vendored because it is unmaintained upstream. Built without SSL, so it uses the
simple handshake and pulls in no OpenSSL dependency.

> The MIT License (MIT)
>
> Copyright (c) 2013-2015 SRS(ossrs)
>
> Permission is hereby granted, free of charge, to any person obtaining a copy of
> this software and associated documentation files (the "Software"), to deal in
> the Software without restriction, including without limitation the rights to
> use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of
> the Software, and to permit persons to whom the Software is furnished to do so,
> subject to the following conditions:
>
> The above copyright notice and this permission notice shall be included in all
> copies or substantial portions of the Software.
>
> THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
> IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS
> FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR
> COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER
> IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
> CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

The original license file is kept alongside the library at
`Source/ThirdParty/srs_librtmp/LICENSE`.

---

## System libraries (no attribution required)

Linked from the Windows SDK; licensed to the developer as part of Windows. These
carry no redistribution obligation for the plugin.

| Library | Used for |
|---|---|
| `ws2_32.lib` | Winsock — the socket layer srs-librtmp sends on |
| `mfplat.lib`, `mfuuid.lib`, `wmcodecdspuuid.lib` | Media Foundation — the AAC audio encoder |

The AAC and H.264 codecs are provided by the operating system and the GPU driver
(Media Foundation and NVENC respectively), not bundled by this plugin, so their
patent licensing rests with those platform components, not with LiveCast.

---

## Unreal Engine modules

`Core`, `Engine`, `RenderCore`, `RHI`, `AVCodecsCore`, `AVCodecsCoreRHI`,
`NVENC`, `AMFCodecs`, `Amf`, `DeveloperSettings`, `UMG`, `InputCore`,
`SignalProcessing`, `AudioCaptureCore` and the codec plugins (`NVCodecs`,
`AMFCodecs`, `AudioCapture`) are part of Unreal Engine and used
under the Unreal Engine EULA. They are not redistributed by this plugin.

---

## Example content — no third party involved

Everything in `Content/Example/` was made for this plugin: the map and its
Blueprints, and the ambient audio loop, which is synthesised arithmetically
rather than sampled or sourced. Nothing in it carries an obligation of any kind.
