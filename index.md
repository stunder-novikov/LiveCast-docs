---
title: LiveCast
description: Broadcast an Unreal Engine game to Twitch, YouTube or any RTMP server, from inside the engine.
---
# LiveCast

Broadcast your game straight from the engine. LiveCast grabs the game window, encodes it on the
GPU and publishes it over RTMP — to Twitch, YouTube, or your own server. No OBS, no second
application, no capture card, and nothing for your player to install.

Start a stream from Blueprint in one node.

---


## On this page

- [Requirements](#requirements)
- [Installation](#installation)
- [Project settings](#project-settings)
- [The example](#the-example)
- [Quick start](#quick-start)
- [Reading chat](#reading-chat)
- [How it behaves](#how-it-behaves)
- [Performance](#performance)
- [Console commands (development builds)](#console-commands-development-builds)
- [Troubleshooting](#troubleshooting)
- [Limitations](#limitations)
- [Licensing](#licensing)

## Requirements

| | |
|---|---|
| Engine | Unreal Engine 5.8 |
| Platform | Windows 64-bit |
| GPU | NVIDIA with NVENC, or AMD with AMF |
| Network | An RTMP ingest URL and stream key |

Intel QuickSync is not available: Unreal's codec layer ships no QuickSync video encoder on Windows.

**LiveCast is built on Unreal's codec plugins, and Epic marks them Experimental.** Enabling LiveCast
enables `AVCodecsCore`, `NVCodecs` and `AMFCodecs` with it — they ship with the engine, so there
is nothing for you to download, and `AudioCapture`, the fourth dependency, is not experimental.
The label is Epic's own and it is theirs to change: if a future engine release drops or
reworks these plugins, LiveCast's support for that release depends on what replaces them. They are
the only route to hardware H.264 from inside the engine on Windows, which is why the plugin uses
them instead of shipping an encoder of its own.

**Which card was actually tested, stated exactly.** NVIDIA is what this plugin was developed and
measured on throughout. AMD was verified on **one card**: a **Radeon RX 6700 XT**, driver Adrenalin
26.5.2, Windows 11, D3D12. It ran the same ten-minute self-test the NVIDIA numbers in this document
come from, and passed every check — AMF chosen by name, decoder parameters repeated so late viewers
get a picture, the encoder reconfigured rather than rebuilt when the bitrate changed, adaptive
bitrate down to a quarter and back twice, a deliberately dropped connection recovered on its own,
18 351 frames with none dropped at 30.3 fps.

Other Radeon models are expected to behave identically and have not been tested. That distinction is
kept deliberately: everything else in this document was measured before it was written down, and one
card is what one card proves. If yours misbehaves, the support link is the fastest route to a fix.

One measured difference worth knowing, because it affects nobody's picture but explains a number:
changing bitrate costs about **24 MB of committed memory per change on NVIDIA**, inside the driver
and never returned, while on the Radeon it cost **nothing measurable across sixteen changes**. The
bitrate controller is deliberately unhurried partly because of that NVIDIA cost.

### Where you can broadcast

Anything that accepts **RTMP**: Twitch, YouTube, your own nginx or SRS server, and every
multistreaming service — Restream, Castr, Streamlabs and the rest. They all take a plain RTMP URL
and a stream key from any encoder, so LiveCast needs no integration with them: paste the service's
address and key into the settings and it works. That is also the simplest way to reach several
platforms at once — one stream leaves your game, the service fans it out.

| Service | Ingest URL |
|---|---|
| Twitch | `rtmp://live.twitch.tv/app` |
| YouTube | `rtmp://a.rtmp.youtube.com/live2` |
| Restream / Castr / others | shown in your account |

Both are tested, not assumed: a full broadcast runs to each. Two things about YouTube are worth
knowing in advance. It **holds a stream key briefly after a disconnect** — reconnection may take two
or three attempts, which LiveCast handles by itself, but a manual restart within a few seconds will
be refused. And its **backup ingest (`b.rtmp.youtube.com`) will not accept a connection at all**: it
refuses at the RTMP handshake, whether or not a primary broadcast is live. That address is meant for
two independent encoders sending identical content, which is not what a game does — use the primary.

**RTMPS is not supported — only plain RTMP.** The transport is built without OpenSSL on purpose:
encryption would drag a licence tail into a plugin that is sold, and every platform above accepts
unencrypted RTMP. One platform does not: **Facebook Live requires RTMPS**, so broadcast to it
through a multistreaming service, which accepts RTMP from you and delivers RTMPS onward.

**Unreal Engine 5.8 only.** Earlier engine versions are not supported at release — LiveCast is built
on the engine's own codec layer, which moves between releases. If you need an earlier version, ask
through the support link; demand is what decides whether it gets built.

**Your project needs no configuration changes.** LiveCast adapts to whatever the host project uses —
it resamples audio to the rate RTMP requires and converts back-buffer formats it is handed,
including 10-bit HDR ones.

---

## Installation

1. Copy the `LiveCast` folder into your project's `Plugins/` directory.
2. Restart the editor. Enable **LiveCast** under *Edit → Plugins → Media* if it is not on already.
3. That is all. Nothing about your project has to change for LiveCast to work.

---

## Project settings

**Edit → Project Settings → Plugins → LiveCast** is where a project decides once what a broadcast
looks like: server address, bitrate, framerate, encoded size, delay, microphone, and which hardware
encoder to prefer. Games usually have one answer to those questions and many places that start a
broadcast, so this saves repeating them at every call site.

Everything here is optional. A game that fills in the settings struct itself and calls **Start
Stream** never has to open this page.

> **If you write `DefaultGame.ini` by hand — a build script, a per-platform override — put the ingest
> URL in quotes:**
>
> ```ini
> [/Script/LiveCast.LiveCastSettings]
> IngestUrl="rtmp://live.twitch.tv/app"
> ```
>
> Unreal's ini parser treats an unquoted `//` as the start of a comment, so `rtmp://host/app` reads
> back as `rtmp:` and the plugin refuses it. The Project Settings page never has this problem — the
> engine quotes such values when it writes them — so this only matters if you edit the file yourself.

### Where the stream key lives, and why not here

**The stream key is deliberately not on that page.** Project settings are written to
`Config/DefaultGame.ini`, which belongs to your project and goes into source control — and a stream
key committed once is a stream key to regenerate. So the page holds the *server address*, which is
public (`rtmp://live.twitch.tv/app` and its equivalents), and never the key.

For development there is a second page, **LiveCast (this machine)**, holding one field: a stream key
stored in your own machine's configuration, outside the project and outside source control. It saves
pasting a key every time you test. **Start Stream With Key** falls back to it when given an empty key
— in the editor only. A packaged build has no such fallback and must be given a key, which is the
correct behaviour: a shipped game asks its player for one and stores it however that game stores
such things.

---

## The example

`LiveCast Content/Example/` ships a working broadcast you can run before writing anything:

| Asset | What it is |
|---|---|
| `Maps/L_LiveCastExample` | A small scene with the example game mode already attached. Open it and press Play. |
| `Audio/A_LiveCastAmbient` | The looping backdrop, so a test broadcast proves game audio and voice at the same time. |
| `Blueprints/BP_LiveCastExampleController` | Starts and stops the broadcast on **F6**, mutes the microphone on **F10**, connects and disconnects chat on **F7**. Set **Chat Channel** on the controller first — it is empty on purpose, so nothing joins a stranger's channel by itself. |
| `Blueprints/GM_LiveCastExample` | The game mode that spawns that controller. |
| `Widgets/WBP_LiveCastStreamHealth` | The on-screen overlay. Redesign it freely — see below. |
| `Widgets/WBP_LiveCastChat` | The chat overlay, new in 1.1. Stays empty until **Chat Channel** is set and F7 connects. Redesignable the same way — see *Reading chat*. |

Set a stream key first, in *Project Settings → Plugins → LiveCast (this machine)*, then open the map
and press Play and F6. That is the whole of it.

> The example uses **F6**, **F7** and **F10** because the engine has already claimed the
> others: `BaseInput.ini` binds F1-F5 to view modes and **F9 to a screenshot**, and F11 to
> fullscreen. Those bindings are live in every build that is not Shipping. F9 was the
> broadcast key until 2026-08-30, which meant every press also wrote a full-resolution PNG -
> a frame hitch at the exact moment a streaming plugin starts streaming.

A **packaged Shipping build** has no console at all, refuses this plugin's console commands as
cheats, and ignores `-ExecCmds` - the engine compiles that out. So the example takes both things it
needs from the command line instead, which is the only way to point a Shipping build at a
destination and a channel:

```
LiveCastDemo.exe -LiveCastKey=xxxx-xxxx-xxxx -LiveCastChat=somechannel
```

Then F6 broadcasts and F7 connects chat, exactly as in the editor. A key or channel set on the
controller wins; the command line is the fallback.


The ambient loop is there for a reason worth knowing: it is tonal and carries a soft marker every
two seconds, so a listener can hear at once whether audio is arriving and whether it is stuttering —
which broadband noise or silence cannot tell you. It also means a test broadcast proves game audio
and microphone mixing together rather than one at a time. Every asset here is synthesised, so
nothing in this plugin carries a third-party licence.

Plugin content is hidden in the Content Browser by default: turn on **Settings → Show Plugin
Content** to see these assets.

### The overlay is worth a paragraph

The stream-health overlay is not decoration. Adaptive bitrate makes the picture soften on purpose
when the uplink cannot keep up, and without something on screen saying so, that looks exactly like a
bug — to the streamer, and to whoever they complain to. The overlay shows the state in one word,
along with bitrate, framerate, dropped frames, reconnects and microphone level.

`ULiveCastStreamHealthWidget` builds a plain default layout in C++, so it works with no design work.
Derive a Blueprint from it and lay out your own widgets: the default is only built when the widget
tree is empty, so a designed subclass replaces the appearance entirely while keeping every value —
`Stats`, `Status Text`, `Status Colour`, `Bitrate Reduced` — and the `On Stats Updated` event.

### Why C++ and not a Blueprint graph

The example's logic lives in C++ rather than in Blueprint nodes for one practical reason: **a
Shipping build has no console**, so a broadcast there can only be started by the game itself. This
controller is the smallest honest demonstration of that, and it is what the plugin's own Shipping
builds are tested with. Everything it does is a handful of calls to the same Blueprint API you have
— derive from it, or read it and write your own.

---

## Quick start

Everything is on one Blueprint subsystem. From any Blueprint:

```
Get Game Instance Subsystem (LiveCast)  →  Start Stream With Key
```

Pass the player's stream key. Everything else comes from Project Settings. To end the broadcast,
call **Stop Stream**. If the player quits or the level ends, the stream stops itself.

If you would rather decide everything in Blueprint, **Start Stream** takes a settings struct with
the full RTMP URL — key included — and ignores the settings page entirely. **Get Default Stream
Settings** sits between the two: it returns the project's configured values so you can change one
field and pass the rest through.

### Blueprint nodes

| Node | What it does |
|---|---|
| **Start Stream With Key** (key) | Begins broadcasting using Project Settings, with this key appended to the configured server. In the editor an empty key falls back to *LiveCast (this machine)*. |
| **Get Default Stream Settings** | The project's configured settings, with an empty URL — change a field and pass it to **Start Stream**. |
| **Start Stream** (settings) | Begins broadcasting. Returns false only if it could not start at all; a connection that fails later reports through **On Stream Error**. |
| **Stop Stream** | Ends the broadcast and closes the connection. |
| **Is Streaming** | Whether a broadcast is running. |
| **Get Stream Stats** | A snapshot for a stream-health widget. See below. |
| **Set Stream Delay** (seconds) | Changes the anti-sniping delay while live. |
| **Set Microphone Gain** (gain) | Changes microphone loudness while live. Zero is mute. |
| **Connect Chat** (channel) | Starts reading a Twitch channel. The `#` is optional, case is ignored. Returns false only for a request that cannot be attempted — an empty name, or chat already connected. |
| **Disconnect Chat** | Stops reading. Held messages are discarded rather than released. |
| **Is Chat Connected** | True once the channel was actually joined, not merely once the socket opened. |
| **Get Chat Channel** | The channel being read, lowercase and without `#`. Empty when not connected. |
| **Get Pending Chat Count** | How many messages are waiting out the broadcast delay. Worth showing on a debug overlay: it explains a screen that looks frozen while the stream is fine. |
| **Get Dropped Chat Count** | Messages discarded because the hold queue filled up. Rising means a raid, not a fault. It does not count messages lost when a connection drops. |

### Stream settings

| Field | Default | Notes |
|---|---|---|
| **URL** | — | Full RTMP URL including the stream key, e.g. `rtmp://host/app/xxxx-xxxx` |
| **Bitrate Kbps** | 4000 | 500–20000 |
| **Framerate** | 30 | 10–120 |
| **Sync Chat To Stream Delay** | on | Hold each chat message for the broadcast delay, so it appears beside the picture the audience is watching. Off hands messages over the moment they arrive. |
| **Max Queued Chat Messages** | 500 | 16–10000. How many may wait out the delay; past that the oldest are discarded and counted. |
| **Resolution** | 1280×720 | The encoded size. Leave at zero to stream the window as it is. |
| **Delay Seconds** | 0 | Holds the broadcast behind the game, up to 300 s. See *Delay* below. |
| **Capture Microphone** | false | Mixes the default input device into the broadcast. Decided at start. |
| **Microphone Gain** | 1.0 | 0–4. Zero is mute. |

### Events

| Event | When it fires |
|---|---|
| **On Stream Started** | The ingest **accepted** the connection and the broadcast is live. |
| **On Stream Stopped** | The broadcast ended — by request, by the game ending, or by an unrecoverable error. |
| **On Reconnecting** | The connection dropped and is being restored. Show a warning; do not stop. |
| **On Reconnected** | The connection came back. |
| **On Stream Error** (error, message) | Something failed. The typed error says what. |
| **On Chat Message** (message) | A viewer said something, and it is due to be shown. With a delay configured this fires when the audience reaches that moment, not when the line arrived. |
| **On Chat Message Deleted** (id) | A message already shown has been withdrawn — deleted by a moderator, or its author banned. Take that line off screen. A message deleted while still held never fires this, because it was never shown. |
| **On Chat Connected** | The channel was joined and messages can now arrive. |
| **On Chat Disconnected** | Chat ended, whether asked for or not. A reconnect in progress raises this once, not per attempt. |
| **On Chat Error** (error, message) | Chat failed, or an attempt to reconnect failed. There is no attempt limit, so treat it as a status rather than something final. |

Events are delivered on the game thread, so it is safe to touch UI directly from them.

**On Stream Started deliberately waits for the ingest.** It does not fire when *Start Stream*
returns — the handshake happens on a background thread, and a wrong stream key must never look
like a live broadcast.

### Error values

| Value | Meaning |
|---|---|
| **Connection Refused** | The URL could not be parsed, or the ingest refused it — usually a wrong or expired key. |
| **Connection Lost** | The connection dropped and could not be restored within the retry budget. |
| **Encoder Unavailable** | No hardware encoder accepted the requested configuration. |
| **No Game World** | Streaming was requested while no game world was running. |

A failure *before* the first successful connection reports **Connection Refused** — that is the
"check your key" case. A failure *after* it reports **Connection Lost**.

### Stream stats

**Get Stream Stats** returns: `Is Streaming`, `Is Connected`, `Elapsed Seconds`,
`Frames Per Second`, `Frames Encoded`, `Frames Dropped`, `Reconnect Count`, `Buffered Megabytes`,
`Bitrate Kbps`, `Uplink Backlog Seconds`, `Microphone Level`, `Microphone Active`.

Two of these deserve a widget of their own:

- **Bitrate Kbps** is what is being encoded *right now*. Below what you asked for means the
  adaptive controller traded quality to keep up — show it, so a softer picture is not mistaken
  for a bug.
- **Uplink Backlog Seconds** is how far behind the uplink is. Anything above a second or two is
  congestion.

**Microphone Level** is measured *after* gain, so a muted microphone reads as silent — which is the
question a level meter exists to answer.

---

## Reading chat

New in 1.1. LiveCast can read a Twitch channel's chat and hand each message to your game.

**No account, no token, nothing to keep secret.** Twitch allows an anonymous read-only connection and
that is what this uses: the client logs in as `justinfan<random>` with no password. You never
register an application, there is no OAuth screen, and nothing new has to be kept out of source
control. It reads only — LiveCast never sends a message.

### Turning it on

Call **Connect Chat** with a channel name; the `#` is optional and case does not matter. **On Chat
Message** then fires for each line, carrying the text, the display name, the login, the colour the
viewer chose, their badges, and whether they are a moderator, a subscriber or the broadcaster.

`WBP_LiveCastChat` in the plugin's example content is a working overlay built on those events. Create
it and add it to the viewport and it works as it is; derive a Blueprint from it to replace the look
entirely while keeping the behaviour.

What it exposes, whether you use it as it is or derive from it:

| Property | Default | |
|---|---|---|
| **Max Lines** | 12 | How many messages stay on screen. The oldest scrolls off. |
| **Font Size** | 14 | Message text size in the default layout. |
| **Show Status Line** | on | Prints the channel and how far chat is held back. Worth leaving on while building: a screen with no messages looks identical whether the channel is quiet, the delay is holding everything, or nothing ever connected. |
| **Replace Unsupported Characters** | on | Replaces characters the font cannot draw. **A memory fix rather than a cosmetic one** — see the note on emoji at the end of this section. |
| **Unsupported Character Replacement** | `?` | What such a character becomes. Empty removes it instead. Plain `?` on purpose: the replacement itself has to be drawable, and a cleverer choice is exactly the glyph a stripped-down font is also missing. |
| **Lines** | — | The messages on screen, oldest first, with their text as it arrived. A subclass rebuilds its own layout from this in **On Chat Updated**. |
| **Make Chat Text Drawable** | — | The same replacement as a function, for a subclass that draws `Lines` itself: the cost is paid by whoever puts the text on screen. |

The rest of the Blueprint surface: **Disconnect Chat**, **Is Chat Connected**, **Get Chat Channel**,
**Get Pending Chat Count**, **Get Dropped Chat Count**, and the events **On Chat Connected**,
**On Chat Disconnected**, **On Chat Message Deleted** and **On Chat Error**.

In a development build the console does the same: `LiveCast.Chat.Join <channel>` while a game is
running. Console commands are development-only — they are refused in a Shipping build.

### Chat in step with the picture

Chat reaches you live, but your viewers are watching a picture that is `Delay Seconds` behind. So a
message posted "now" is a reaction to something your audience has not seen yet, and an in-game
response to it runs ahead of them.

LiveCast owns both sides, so it holds each message for the delay you configured and releases it on
the first frame after that delay has elapsed. Change the delay mid-broadcast and the hold follows.

Three things to know before relying on it:

- **The hold follows a running broadcast.** It is applied by **Start Stream** and by **Set Stream
  Delay**. Setting `Delay Seconds` in Project Settings and then connecting chat without streaming
  holds nothing.
- **`Sync Chat To Stream Delay` turns it off.** On by default; with it off, chat is handed over the
  moment it arrives.
- **Your platform's own latency is not compensated and cannot be.** Twitch's ingest, transcode and
  player buffer add several seconds of their own, and nothing inside an encoder can measure them.
  With `Delay Seconds` at its default of zero the hold is zero and chat is exactly as far out of step
  as any browser overlay.

### Moderation, including before a message is ever shown

A moderator deleting a message inside the delay window means the deletion reaches LiveCast **while
the message is still waiting**. Your game never sees it at all. Every external overlay put that
message on screen the moment it was posted and can only take it down afterwards.

A message already shown can still be withdrawn: **On Chat Message Deleted** carries its id so you can
take that line off screen. A ban or a timeout reaches backwards over both halves — everything that
user has in flight is dropped, and everything of theirs among the last few hundred messages shown is
withdrawn, one id at a time. The remembered window is 256 messages, so a deletion older than that
finds nothing.

Measured on a live channel: over two hours, 42 bans withdrew 516 already-shown messages, the largest
single ban taking back 68.

### When chat stops

**Held messages are dropped, not flushed.** If the connection drops, if Twitch asks the client to
move, or if you disconnect, whatever was waiting is discarded. This is deliberate: those messages
exist to appear beside a particular picture, and by the time chat is back their moment has passed.
Releasing them would dump a minute of backlog into the game at once, which reads as a bug.

A dropped connection reconnects on its own, backing off 1, 2, 4, 8 and then 12 seconds, and keeps
trying for as long as chat is connected — **there is no attempt limit**. Every failed attempt raises
**On Chat Error**, so treat that event as a status to display rather than a fault to give up on.

Twitch's own `RECONNECT` request — sent before it restarts a server — closes the socket at once and
opens a new one, on Twitch's schedule rather than after an unpredictable hang-up. It raises **On Chat
Disconnected** and takes held messages with it, and it is not reported as an error, because it is not
one.

### Load

`Max Queued Chat Messages` (default 500) bounds how many messages may wait out the delay. Past that
the **oldest** are discarded and counted — during a raid the recent lines are the ones you are
reacting to. **Get Dropped Chat Count** reports the running total; note it counts only messages lost
to a full queue, not those discarded when a connection drops.

That limit is also what bounds the worst frame *inside LiveCast*. Measured: 10 000 messages through
the parser and the queue in 31 ms, and the largest possible single-frame release — the whole
500-message queue at once — costs 0.14 ms in the queue itself. What your own overlay then does with
500 messages is your cost, not that figure. A busy channel is a few messages a second.

### What is deliberately not here

No sending, no channel points, no follower or subscriber events, no emote images, and no filtering.
Message text is handed to your game exactly as it arrived and drawn as plain text; what to show is
your decision, and the badges and moderator flags are there to decide with. A filter that fails is
worse than no filter.

Three consequences of handing the text over untouched, all of which you will meet on a real channel:

- **A Twitch emote arrives as its code, not as a picture.** On the wire an emote is ordinary text —
  `Kappa`, or whatever a channel calls it — and Twitch's own client swaps in the image using a
  separate tag that says which characters to replace. LiveCast does not read that tag, so your game
  receives the word.
- **Unicode emoji arrive intact, and the example overlay replaces the ones its font cannot draw.**
  The engine's default font has no emoji glyphs. Left alone such a character draws as an empty box
  *and* makes Slate log a warning every time the line is shaped — once per row per message for a
  scrolling overlay, about 0.6 KB of memory each that never comes back. Measured on a live channel:
  180 000 warnings and 102 MB over two hours. So **Replace Unsupported Characters** is on by
  default and they arrive on screen as `?`. Cyrillic and other alphabets are covered by the font
  and are untouched. Turn it off if your own font really does cover emoji. Either way your game
  receives the original text: the replacement happens where the overlay draws, not where the
  message arrives.
- **A `/me` message arrives in Twitch's raw form**, `\x01ACTION waves\x01`, control bytes and all.
  Strip that wrapper yourself if you want it drawn as an action.

## How it behaves

### What ends up in the broadcast

**The frame the engine rendered, and nothing else.** LiveCast reads the game's back buffer at
present time, before the operating system composes windows onto the screen. So anything that is not
part of your game cannot reach the broadcast, however it sits on screen:

- windows on top of the game, including messengers and browsers
- system notifications and pop-ups
- a second monitor
- whatever is in front while you are alt-tabbed away

This is the opposite of how screen and window capture behave, and it is worth knowing if the reason
you are streaming from inside the game is that you would rather not broadcast your desktop by
accident.

**The exception, and it matters:** overlays that inject themselves *into* the game draw into the
very same frame and therefore do appear — Steam's overlay, Discord's, hardware monitors such as MSI
Afterburner. So does the engine's own console and anything a `stat` command puts on screen. The rule
is not "only the game window", it is **"only what the game itself drew"**.

**The mouse cursor is not in the broadcast.** Unreal uses the system cursor unless a project turns
on `[CursorControl] bAllowSoftwareCursor`, and the system draws it over the window rather than into
it. Measured, not assumed: sweeping the cursor across a still scene for fifteen seconds changed the
encoded size by 4%, which is codec noise. If your game does use a software cursor, it will appear —
because at that point your game is the one drawing it.

**A minimised window freezes the broadcast.** There is nothing to capture: a minimised window stops
presenting frames, so encoding all but stops (measured: 220 KB/s down to 0.7 KB/s) and viewers see
the last frame held. The connection stays up and everything resumes by itself the moment the window
is restored — no reconnect, no restart. Worth telling your players if they can minimise mid-stream;
if you need to keep broadcasting regardless, keep the window on screen and let it lose focus instead,
which does not affect capture at all.

### Reconnection

A dropped connection does not end the broadcast. LiveCast retries with a backoff of
**1, 2, 4, 8, 15 seconds, up to ten attempts**, and asks the encoder for an immediate keyframe when
it gets back — a fresh session has no decoder state, so without one viewers would see nothing.

**The very first connection deliberately does not retry.** A bad stream key should fail loudly and
at once, not after a minute of silent attempts.

### Delay buffer (anti stream-sniping)

`Delay Seconds` holds the broadcast behind the game so viewers cannot watch your screen in real
time. It buffers **encoded** packets, so 15 seconds costs about 5 MB — not gigabytes of frames.
Audio and video share one queue and stay in step. It can be changed mid-broadcast with
**Set Stream Delay**; raising it stretches the buffer, lowering it lets the buffer drain, and
neither interrupts the stream.

### Adaptive bitrate

On by default. When the uplink cannot keep up, LiveCast lowers the bitrate rather than letting the
stream fall behind, then restores it once the connection recovers.

It falls fast and climbs slowly, on purpose: every viewer notices lateness at once, and nobody
notices ten extra seconds of caution. The floor is a quarter of what you asked for, never below
300 kbps, and it never climbs above your requested rate.

The signal is how long packets have waited *beyond* the delay you configured — not queue size,
because a configured delay makes a deep queue perfectly normal.

### Window resizing

The game window may be resized freely while streaming. The picture is **scaled** into the encoded
size with the aspect ratio preserved, and any remainder is filled with black bars — the window is
never cropped.

The encoded resolution is fixed when the stream starts and cannot change mid-broadcast: every
viewer's decoder would have to restart.

### Microphone

`Capture Microphone` mixes the default input device into the broadcast. It is decided **when the
stream starts** — opening a capture device mid-broadcast is not worth the risk.

To mute afterwards, set the gain to zero with **Set Microphone Gain**. The device stays open, so
unmuting is instant and resumes at the present moment.

Voice is mixed into the game audio *before* conversion, so voice and game go through exactly the
same processing and cannot drift apart.

That has one consequence worth knowing in advance: **on a machine with no audio device, the
microphone is silent too**, even though it was asked for. Voice is mixed into the game's audio
buffers, and with no device there are no buffers to mix into — the broadcast carries video and no
audio track at all. This is correct by construction rather than a fault, and it is easy to meet
without expecting it: a server or a virtual machine with sound disabled, with a headset plugged into
it, is exactly that shape. The log says so plainly when it happens — `audio: no audio device for
this world, the stream will be silent` — and nothing else about the broadcast is affected.

### Stream keys are never logged

Your stream key is a password, and log files end up in bug reports. LiveCast masks everything after
the last slash of the URL in every message it writes.

Note that Unreal's *own* console echo does print whatever you type, and so does its record of the
command line at startup. LiveCast cannot mask those — they are written before the plugin sees
anything. So a key typed into `LiveCast.Stream` is in that log for good, and that log is what gets
attached to a bug report.

Both commands that take a key therefore accept **`file`** instead of a URL:

```
LiveCast.Stream   file      reads Saved/LiveCastStreamUrl.txt
LiveCast.SelfTest file      reads Saved/LiveCastSelfTestUrl.txt
```

Each file is deleted the moment it is read — before connecting, so a run that crashes does not leave
your key on disk either. Use this form whenever a log might be shared. A shipped game does not face
the question at all: it starts streams through the Blueprint API, which never goes near the console.

---

## Performance

Encoding shares the GPU with your game, so the cost depends on how much headroom the card has, not
on the plugin. Measured at 1280×720 on a GTX 960 — a deliberately modest card:

| Scene | Encoder thread per frame | Output |
|---|---|---|
| Empty level | ~15 ms | 28.8 fps of 30 |
| Sky, atmosphere and volumetric fog (packaged build) | 41–45 ms | 22–27 fps of 30 |

The pipeline itself tops out far higher; what limits a heavy scene is the same GPU rendering the
game and running the encoder. On a card with headroom this is not a factor. If you measure lower
output than you expect, measure on a simple scene first.

Capture is held to the framerate you requested; frames skipped to hold that rate are counted
separately from real drops, so a rising **Frames Dropped** is the number that matters.

---

## Console commands (development builds)

Useful while building your game. **Development builds only** — every command below is registered as
a cheat, so a Shipping build refuses it even if something in your game reaches the console
programmatically. Settings changed from C++ or Blueprint are unaffected.

```
LiveCast.Stream "rtmp://host/app/key" [kbps] [fps] [width] [height]
LiveCast.Stream file [kbps] [fps] [width] [height]   read the URL from a file, keep it out of the log
LiveCast.Start [kbps] [fps]      record to a file instead of streaming
LiveCast.Stop
LiveCast.TestRtmp "rtmp://host/app/key"   is the key accepted? connects, publishes nothing
LiveCast.DelaySeconds 15
LiveCast.Microphone 1            set before starting, read at start
LiveCast.MicrophoneGain 1.5      live; zero is mute
LiveCast.AdaptiveBitrate 0       hold the configured bitrate whatever the uplink does
LiveCast.ThrottleKbps 1200       pretend the uplink is this slow — watch your UI react; 0 removes it
LiveCast.DropConnection          force a reconnect, to test your UI
LiveCast.MirrorToFile 1          keep a copy of the broadcast on disk (diagnostic)
LiveCast.ForceEncoder nvenc      auto | nvenc | amf | default — amf is written but unverified

LiveCast.Chat.Join <channel>     read a channel through the subsystem, so Blueprint events and
                                 chat widgets see it; needs a running game
LiveCast.Chat.Leave              stop that chat
LiveCast.Chat.DropConnection [s] break the chat connection as the network would, now or in N
                                 seconds, and let the reconnect run
LiveCast.Settings                print what Project Settings currently says
LiveCast.TestUrl "rtmp://…"      show how an ingest URL and a key are joined, without connecting
LiveCast.SelfTest "rtmp://…"     a scripted ten-minute broadcast; see below
```

### `LiveCast.SelfTest`

Runs a broadcast on a fixed schedule — two untouched minutes, a window resize, two rounds of
congestion and recovery, a forced disconnect — and writes `Saved/LiveCastSelfTest.log`: a table of
what happened at each step, and a verdict per question in plain language.

It answers, on hardware neither of us has, the things only real hardware can: whether the encoder
opened and which one, whether decoder parameters repeat (so late viewers see a picture), whether the
encoder survives a bitrate change rather than being rebuilt, whether the broadcast recovers from a
cut connection, and what a bitrate change costs in memory.

**It is a Development-build tool**, like everything in this section — it is a console command, and
Shipping has no console. If you hit a problem in a Shipping build, reproduce it in a Development
build and send that report.

**Quote the URL.** The console treats an unquoted `//` as a comment and will silently truncate the
address to `rtmp:`.

On a Russian keyboard layout the console opens with `ё` — the key in the tilde position.

---

## Troubleshooting

**Nothing appears on the ingest, and no error.** Check the broadcast dashboard rather than the
public channel page; ingests preview with a delay of up to a minute.

**On Stream Error → Connection Refused straight away.** Wrong or expired stream key, or a
malformed URL. Test it with `LiveCast.TestRtmp` before blaming the stream.

**Viewers who join late see nothing.** Not an issue here — LiveCast repeats the H.264 parameter
sets before every keyframe so a decoder can start at any point. If you see this, the ingest is
transcoding.

**The picture is softer than expected.** Check `Bitrate Kbps` in the stats: the adaptive controller
may have lowered it because the uplink is congested. `Uplink Backlog Seconds` confirms it.

**Frames Dropped keeps rising.** The GPU has no headroom left. Lower the resolution, the framerate,
or the scene cost.

**Testing in the editor:** run PIE as **New Editor Window**. LiveCast captures the game window, and
in other PIE modes the only window holding the game is the editor frame itself.

---

## Limitations

- Unreal Engine 5.8 only.
- Windows 64-bit only.
- Only the game is broadcast. There is no way to include a desktop, a webcam or a browser source —
  by design, but it does mean LiveCast replaces a capture tool only for streaming the game itself.
- A minimised window freezes the picture until it is restored. Losing focus is fine; being minimised
  is not, because the window stops rendering.
- H.264 video (NVENC on NVIDIA, AMF on AMD) and AAC audio. AMD is verified on one card — see the
  requirements above.
- RTMP only — no SRT, no WebRTC.
- The encoded resolution is fixed for the duration of a broadcast.
- One broadcast at a time.
- Chat is Twitch only, read-only and anonymous. No sending, no channel points, no follower or
  subscriber events, no emote images — an emote arrives as its code word — and no filtering. A
  withdrawal reaches only the last 256 messages shown.

---

## Licensing

LiveCast bundles one third-party library, **srs-librtmp**, under the MIT licence. See
`THIRD_PARTY_LICENSES.md` for the full notice and for what is used from the system and the engine.

---
