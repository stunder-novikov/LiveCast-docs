# Third-party licenses

LiveCast ships one vendored third-party library and otherwise builds against
system libraries and Unreal Engine modules. Everything here is compatible with
selling the plugin commercially. Two notices travel with it, both reproduced in
full below: the MIT notice for srs-librtmp, and the OpenSSL and SSLeay notices for
the engine's OpenSSL, which the engine links into the plugin's own binary.

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

**The vendored copy is modified.** One symbol is renamed: `gettimeofday` became
`srs_gettimeofday`. Windows has no such function, so both this library and
libwebsockets ship a compatibility shim for it — and chat makes the plugin depend on
the engine's WebSockets module, which links libwebsockets. A packaged game links
monolithically, the two definitions meet, and without the rename the game cannot be
built at all. The rename is applied offline with `llvm-objcopy`; the exact command is
recorded in `Source/LiveCast/LiveCast.Build.cs` beside the library's paths, so it can
be reproduced or undone. The MIT licence above permits modification and asks only that
the notice travel with the code, which it does.

---

## System libraries (no attribution required)

Linked from the Windows SDK; licensed to the developer as part of Windows. These
carry no redistribution obligation for the plugin.

| Library | Used for |
|---|---|
| `ws2_32.lib` | Winsock — the socket layer srs-librtmp and the RTMPS connection send on |
| `mfplat.lib`, `mfuuid.lib`, `wmcodecdspuuid.lib` | Media Foundation — the AAC audio encoder |

The AAC and H.264 codecs are provided by the operating system and the GPU driver
(Media Foundation and NVENC respectively), not bundled by this plugin, so their
patent licensing rests with those platform components, not with LiveCast.

---

## Unreal Engine modules

`Core`, `CoreUObject`, `ApplicationCore`, `Engine`, `Projects`, `RenderCore`, `RHI`,
`Slate`, `SlateCore`, `UMG`, `InputCore`, `DeveloperSettings`, `Sockets`,
`WebSockets`, `SSL`, `AVCodecsCore`, `AVCodecsCoreRHI`, `NVENC`, `AMFCodecs`, `Amf`,
`SignalProcessing`, `AudioCaptureCore`, the engine's third-party `OpenSSL` module and
the codec plugins (`NVCodecs`, `AMFCodecs`, `AudioCapture`) are part of Unreal Engine
and used under the Unreal Engine EULA. Their files are not redistributed by this plugin;
OpenSSL is the one whose code ends up inside the plugin's binary — see below.

Chat connects to Twitch over a secure WebSocket through the engine's `WebSockets`
module; the TLS it uses is the engine's own, so reading chat adds no third-party
library of this plugin's to the list above.

Broadcasting to an `rtmps://` address (new in 1.4) uses the same TLS: the engine's
OpenSSL, which the `WebSockets` module already links into every game that uses
LiveCast, and the engine's `SSL` module for the trusted root certificates. srs-librtmp
itself stays built without SSL — the encrypted connection is made beside it, not
inside it — so RTMPS adds no third-party library of this plugin's either.

---

## OpenSSL 1.1.1t — OpenSSL License and original SSLeay License

The engine's copy (`Engine/Source/ThirdParty/OpenSSL`), not one of ours, used for
`rtmps://`. The engine links it statically, so its code is inside the plugin's own
binary (`UnrealEditor-LiveCast.dll` in the editor, the game's executable once packaged),
and that binary is distributed. Its licence therefore travels with the plugin:

**This product includes software developed by the OpenSSL Project for use in the
OpenSSL Toolkit (http://www.openssl.org/).** **This product includes cryptographic
software written by Eric Young (eay@cryptsoft.com).**

1.1.1 is the version the engine carries; its upstream support ended in 2023, and the
engine, not this plugin, decides when it moves. The full notice, as the engine ships it
in `Engine/Source/ThirdParty/Licenses/OpenSSL_License.txt`:

```
  LICENSE ISSUES
  ==============

  The OpenSSL toolkit stays under a double license, i.e. both the conditions of
  the OpenSSL License and the original SSLeay license apply to the toolkit.
  See below for the actual license texts.

  OpenSSL License
  ---------------

/* ====================================================================
 * Copyright (c) 1998-2019 The OpenSSL Project.  All rights reserved.
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions
 * are met:
 *
 * 1. Redistributions of source code must retain the above copyright
 *    notice, this list of conditions and the following disclaimer.
 *
 * 2. Redistributions in binary form must reproduce the above copyright
 *    notice, this list of conditions and the following disclaimer in
 *    the documentation and/or other materials provided with the
 *    distribution.
 *
 * 3. All advertising materials mentioning features or use of this
 *    software must display the following acknowledgment:
 *    "This product includes software developed by the OpenSSL Project
 *    for use in the OpenSSL Toolkit. (http://www.openssl.org/)"
 *
 * 4. The names "OpenSSL Toolkit" and "OpenSSL Project" must not be used to
 *    endorse or promote products derived from this software without
 *    prior written permission. For written permission, please contact
 *    openssl-core@openssl.org.
 *
 * 5. Products derived from this software may not be called "OpenSSL"
 *    nor may "OpenSSL" appear in their names without prior written
 *    permission of the OpenSSL Project.
 *
 * 6. Redistributions of any form whatsoever must retain the following
 *    acknowledgment:
 *    "This product includes software developed by the OpenSSL Project
 *    for use in the OpenSSL Toolkit (http://www.openssl.org/)"
 *
 * THIS SOFTWARE IS PROVIDED BY THE OpenSSL PROJECT ``AS IS'' AND ANY
 * EXPRESSED OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
 * IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR
 * PURPOSE ARE DISCLAIMED.  IN NO EVENT SHALL THE OpenSSL PROJECT OR
 * ITS CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
 * SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT
 * NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES;
 * LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION)
 * HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT,
 * STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
 * ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED
 * OF THE POSSIBILITY OF SUCH DAMAGE.
 * ====================================================================
 *
 * This product includes cryptographic software written by Eric Young
 * (eay@cryptsoft.com).  This product includes software written by Tim
 * Hudson (tjh@cryptsoft.com).
 *
 */

 Original SSLeay License
 -----------------------

/* Copyright (C) 1995-1998 Eric Young (eay@cryptsoft.com)
 * All rights reserved.
 *
 * This package is an SSL implementation written
 * by Eric Young (eay@cryptsoft.com).
 * The implementation was written so as to conform with Netscapes SSL.
 *
 * This library is free for commercial and non-commercial use as long as
 * the following conditions are aheared to.  The following conditions
 * apply to all code found in this distribution, be it the RC4, RSA,
 * lhash, DES, etc., code; not just the SSL code.  The SSL documentation
 * included with this distribution is covered by the same copyright terms
 * except that the holder is Tim Hudson (tjh@cryptsoft.com).
 *
 * Copyright remains Eric Young's, and as such any Copyright notices in
 * the code are not to be removed.
 * If this package is used in a product, Eric Young should be given attribution
 * as the author of the parts of the library used.
 * This can be in the form of a textual message at program startup or
 * in documentation (online or textual) provided with the package.
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions
 * are met:
 * 1. Redistributions of source code must retain the copyright
 *    notice, this list of conditions and the following disclaimer.
 * 2. Redistributions in binary form must reproduce the above copyright
 *    notice, this list of conditions and the following disclaimer in the
 *    documentation and/or other materials provided with the distribution.
 * 3. All advertising materials mentioning features or use of this software
 *    must display the following acknowledgement:
 *    "This product includes cryptographic software written by
 *     Eric Young (eay@cryptsoft.com)"
 *    The word 'cryptographic' can be left out if the rouines from the library
 *    being used are not cryptographic related :-).
 * 4. If you include any Windows specific code (or a derivative thereof) from
 *    the apps directory (application code) you must include an acknowledgement:
 *    "This product includes software written by Tim Hudson (tjh@cryptsoft.com)"
 *
 * THIS SOFTWARE IS PROVIDED BY ERIC YOUNG ``AS IS'' AND
 * ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
 * IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
 * ARE DISCLAIMED.  IN NO EVENT SHALL THE AUTHOR OR CONTRIBUTORS BE LIABLE
 * FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
 * DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS
 * OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION)
 * HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT
 * LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY
 * OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF
 * SUCH DAMAGE.
 *
 * The licence and distribution terms for any publically available version or
 * derivative of this code cannot be changed.  i.e. this code cannot simply be
 * copied and put under another distribution licence
 * [including the GNU Public Licence.]
 */
```

---

## Root certificates — the engine's `cacert.pem` (Mozilla CA certificate list)

To check an ingest's certificate in a packaged game, LiveCast stages the engine's
certificate bundle, `Engine/Content/Certificates/ThirdParty/cacert.pem`, into every
packaged game (about 230 KB). It is engine content, unchanged, derived from Mozilla's
CA certificate list and distributed by Epic with the engine under the Mozilla Public
License 2.0 (https://www.mozilla.org/MPL/2.0/); the engine's own packaging copies the same file into any project that sets
`n.VerifyPeer`. A project that packages its own `Content/Certificates/cacert.pem`
uses that instead. Once in the build, the bundle is what all of the game's HTTPS
trusts, alongside the Windows root store.

---

## Example content — no third party involved

Everything in `Content/Example/` was made for this plugin: the map and its
Blueprints, and the ambient audio loop, which is synthesised arithmetically
rather than sampled or sourced. Nothing in it carries an obligation of any kind.

