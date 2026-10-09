---
title: LiveCast
description: Broadcast an Unreal Engine game to Twitch, YouTube or any RTMP or RTMPS server, from inside the engine.
---
# LiveCast

Broadcast your game straight from the engine. LiveCast grabs the game window, encodes it on the
GPU and publishes it over RTMP or RTMPS — to Twitch, YouTube, or your own server. No OBS, no second
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
- [Chat votes](#chat-votes)
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
| Network | An RTMP or RTMPS ingest URL and stream key (RTMPS uses outbound TCP 443 unless the address names another port). Reading chat also opens an outbound secure WebSocket to `irc-ws.chat.twitch.tv:443` |

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

Anything that accepts **RTMP or RTMPS**: Twitch, YouTube, Kick, your own nginx or SRS server, and
every multistreaming service — Restream, Castr, Streamlabs and the rest. They all take an RTMP URL
and a stream key from any encoder, so LiveCast needs no integration with them: paste the service's
address and key into the settings (over RTMPS, which Kick requires, see what is and is not yet
checked below). That is also the simplest way to reach several platforms at once — one stream
leaves your game, the service fans it out.

| Service | Ingest URL |
|---|---|
| Twitch | `rtmp://live.twitch.tv/app`, or encrypted `rtmps://live.twitch.tv:443/app` |
| YouTube | `rtmp://a.rtmp.youtube.com/live2`, or encrypted `rtmps://a.rtmps.youtube.com:443/live2` |
| Kick | `rtmps://…` — shown in your account; Kick accepts only RTMPS |
| Restream / Castr / others | shown in your account |

Plain RTMP to Twitch and YouTube is tested, not assumed: a full broadcast runs to each. RTMPS is
newer and checked less far — see below. Two things about YouTube are worth
knowing in advance. It **holds a stream key briefly after a disconnect** — reconnection may take two
or three attempts, which LiveCast handles by itself, but a manual restart within a few seconds will
be refused. And its **backup ingest (`b.rtmp.youtube.com`) will not accept a connection at all**: it
refuses at the RTMP handshake, whether or not a primary broadcast is live. That address is meant for
two independent encoders sending identical content, which is not what a game does — use the primary.

**RTMPS — new in 1.4.** An address that begins with `rtmps://` is broadcast over TLS: the
certificate is checked against the host name you wrote, before anything else is sent, with the
engine's trusted roots — its certificate bundle plus the Windows root store, and the project's `[SSL]`
root-certificate settings (the engine's public-key pinning settings are not applied). LiveCast puts
the engine's bundle (`Engine/Content/Certificates/ThirdParty/cacert.pem`, about 230 KB) into every
packaged game: the engine's own packaging copies it only for a project that sets `n.VerifyPeer`, and
without it a packaged game would trust only what the player's Windows root store happens to hold. A
project that packages its own `Content/Certificates/cacert.pem` keeps using that. The bundle is the
engine's, so once in the build it is what the game's other HTTPS — HTTP requests, WebSockets — trusts
as well, alongside the Windows store. The TLS is the engine's own
OpenSSL — the same library the engine's WebSockets module already links for chat — so it adds no
third-party library of ours; see `THIRD_PARTY_LICENSES.md`.
An `rtmps://` address is never sent as plain RTMP: one that cannot be used as written — a port that
is not a number, no `/app/key` — is refused before anything connects.

What is checked, and what is not yet:

- **Checked against Twitch's real ingest** at `rtmps://live.twitch.tv:443/app` and at its global
  address `rtmps://ingest.global-contribute.live-video.net/app`: TLS 1.3, certificate verified, the
  RTMP connection accepted. With a deliberately wrong key the session is cut, as it is over plain
  RTMP, and the plugin gives the same "check the key" diagnosis.
- **Checked from a packaged game**, not only from the editor: with the Windows root store switched
  off, the certificate bundle LiveCast puts into the build verified Twitch's ingest on its own, and
  with a bundle that does not hold Twitch's root the connection was refused, as it should be.
- **Checked end to end on a local RTMPS server**: a full broadcast, a dropped connection restored
  over a new TLS session, and the ways an encrypted connection fails — an untrusted certificate, a
  host name the certificate does not name, a port that does not speak TLS, and a server that speaks
  TLS but not RTMP — each named in the log, in **On Stream Error**, and in the self-test's verdict
  (`TLS REFUSED`, `TLS OK, BUT NOT AN RTMP INGEST`).
- **Not yet checked: a real broadcast with a real key to an RTMPS ingest.** Kick, YouTube and Twitch
  over RTMPS have not been watched live through this plugin yet.

**Plain `rtmp://` stays the default and is unchanged.** Some networks block TLS to an ingest while
letting plain RTMP through — measured from one network for YouTube and Facebook — and a TLS handshake
that never completes says so after ten seconds and suggests the `rtmp://` address. On such a network
a reconnect can take several minutes to give up, ten seconds per attempt. Facebook Live requires
RTMPS; it has not been tested. Certificates are not checked for revocation, and a system proxy is not
used for the broadcast — over RTMPS, as over plain RTMP, the connection goes direct.

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

### Upgrading from 1.3.1

- **Sync Chat To Stream Delay is now Chat Hold**, a choice of three: *Same as the broadcast delay*
  (what the checkbox did, and still the default), *Fixed moderation window* (new: **Chat Hold
  Seconds**, whatever the delay), and *Off*. A project that had the checkbox off is read as Off, so
  nothing behaves differently until you choose. A Blueprint or C++ class that used **Sync Chat To
  Stream Delay** (`bSyncChatToStreamDelay`) will not compile until that use is replaced — read
  **Chat Hold**, or call **Get Chat Hold Seconds**. See *Holding chat for moderation*.
- **Voters' names wait the chat hold when it is longer than the broadcast delay.** That happens only
  with a fixed window longer than the delay; otherwise names wait exactly as before.
- **New nodes**: **Get Chat Hold Seconds**, **Get Voter Name Hold Seconds** and **Get Measured
  Viewer Latency**.
- **Vote results gain First Answer Seconds**, and each vote with five or more of your viewers logs
  one more line: the floor your quickest viewers' latency puts under **Viewer Latency Seconds**. See
  *Chat votes*.
- **`LiveCast.Settings` prints the chat hold** on a line of its own.
- **In `DefaultGame.ini`** the setting is `ChatHold=BroadcastDelay`, `Fixed` or `Off`, with
  `ChatHoldSeconds=10`. An old `bSyncChatToStreamDelay=False` is still read, as Off, until `ChatHold`
  is written; an unknown `ChatHold` value is ignored with a warning. Project Settings → **Import** of
  a settings file exported from 1.3.1 does not carry the old checkbox over: set Chat Hold by hand
  after importing one.
- **Log lines changed**: the vote's "opened" line ends with "names after N s"; "the delay ended:
  dropped …" is now "the chat hold fell to zero: dropped …"; a Fixed hold with Chat Hold Seconds at
  0 warns at load (the editor may print it twice; it loads the settings twice). An unknown `ChatHold`
  value's warning names the choice used instead.
- **Stream keys are masked by the address's structure.** In 1.3.1 `rtmp://host/app/KEY/` was
  printed whole, in the log and in the self-test's reports; if you shared logs or reports from such a
  URL, regenerate the key. Now only the scheme, host, port and the first part of the path are kept
  (`rtmp://host/app/***`) — a nested application shows only its first part, a path of one part is
  hidden whole — and a query (`?…`), anything after a space, and a user name and password before the
  host are masked too. **`LiveCast.Settings`** now masks the ingest URL it prints, and
  **`LiveCast.TestUrl`** prints the key as `<key>`.
- **Make Stream URL drops a key's own leading and trailing slashes**, and warns when the Ingest URL
  has more than one part after the host — usually a stream key already in it.
- **An ingest that refuses the application path on the first connection** now reports **Connection
  Refused** ("…refused the application in the URL…"); 1.3.1 reported it as **Connection Lost**.
- **An `rtmps://` address that cannot be used as written** — a bad port, no `/app/key` — is refused
  at once with **Connection Refused** ("Not started: …") rather than tried.
- **`rtmps://` addresses broadcast over TLS**, where 1.3.1 refused them. `rtmp://` is unchanged. See
  *Where you can broadcast*. **Packaged games now carry the engine's certificate bundle** (about
  230 KB), staged by the plugin — and the game's other HTTPS, its HTTP requests and WebSockets, then
  trusts that bundle too, alongside the Windows root store.
- **Spaces around an address are ignored**: ` rtmp://…`, which 1.3.1 refused with **Connection
  Refused**, now broadcasts. The wrong-scheme message now names `rtmp:// or rtmps://`.
- **The self-test's report** masks the key as `***` (it was `<stream key hidden>`), and has new
  verdicts: `ENCODER UNAVAILABLE` (it used to blame the ingest), `LOCAL FAILURE`, `TLS REFUSED` and
  `TLS OK, BUT NOT AN RTMP INGEST`.
- **A Stop during a reconnect** returns at once instead of after the current wait (up to 15 s), and no
  longer logs "reconnecting" or "giving up".
- **Start Stream while your own broadcast runs** is refused with **Already Streaming** and leaves the
  broadcast as it was. 1.3.1 applied the new delay, then failed with **Encoder Unavailable** and cleared
  the delay of the broadcast still running.
- **First Answer Seconds is -1** for a vote during which the broadcast delay changed or a broadcast
  started or stopped, as well as for too few timed voters and a cancelled vote.
- **A chat hold too long for Max Queued Chat Messages** at the channel's rate now says so in the log
  (*Load*).

### Upgrading from 1.3

1.3.1 changes documentation and wording only. No behaviour, setting, Blueprint node or event
changed. What you will notice: the tooltips of **Sync Chat To Stream Delay**, **On Chat Event** and the
vote overlay's **Latest Ballot**, the plugin's description in the Plugins window, the example chat
overlay's status line — "held N s for moderation" where it said "held N s to match the stream" — and
one log line, for held messages dropped on a disconnect.

Earlier versions explained the chat hold as keeping chat "in step with the picture". That was wrong:
the hold gives moderators the delay to remove a line before it is drawn into the broadcast, and costs
your game one delay of reaction time. If you left **Sync Chat To Stream Delay** on to make reactions
line up with the picture, read *Holding chat for moderation* — for fast reactions to commands, set
**Chat Hold** to Off (1.4; in 1.3.1, turn the checkbox off).

### Upgrading from 1.2

Chat votes are new (see *Chat votes*) and nothing that existed is renamed or changed in meaning. What
a 1.2 game may notice:

- **The example controller opens a vote on Home**, and the example shows a vote overlay at the top of
  the screen while one runs. A Blueprint derived from the controller gets both, and the controller's
  binding takes Home before your pawn or level Blueprint sees it; set **Toggle Vote Key** to none, or
  **Vote Widget Class** to none, to leave them out.
- **The example's "ready" log line** names the vote key as well.
- **The example controller gained members**: `ToggleVote`, `Toggle Vote Key`, `Example Vote`,
  `Vote Widget Class`, `VoteWidget`, `HandleVoteEnded`, and an `EndPlay` override. A Blueprint or C++
  class derived from it that already has something by one of those names will not compile until it
  is renamed. The controller also plays the jump sound when a vote produces a winner.
- **Chat messages and channel events gain two fields**, **Is From Another Channel** and **Source
  Channel Id**, for Twitch Shared Chat (see *Shared Chat*). Nothing existing changed meaning: a
  partner's lines were delivered in 1.2 too, just without saying so.
- **Project Settings gain a Votes section** — one setting, **Viewer Latency Seconds**, written to
  `DefaultGame.ini` only if you change it.

### Upgrading from 1.1

Nothing is renamed: every Blueprint node, event, struct field, enum value and settings key from 1.1
is still there with the same meaning, so saved Blueprints and `DefaultGame.ini` load unchanged. What
a 1.1 game may notice is behaviour:

- **Channel events are new** — **On Chat Event**. They wait in the same queue as messages, so
  **Get Pending Chat Count**, **Get Dropped Chat Count** and **Max Queued Chat Messages** count them
  too, and a ban or a deletion can name an event's id.
- **The example overlay draws events**, so a subclass drawing **Lines** itself now finds event rows in
  it: **Is Channel Event** set, no author. A resub with words adds a second row with the viewer's name;
  both share one id.
- **`/me` arrives unwrapped**, without `\x01ACTION …\x01`, with **Is Action** set. A game that stripped
  the wrapper itself should read the flag instead.
- **A cheer** raises **On Chat Message** and then **On Chat Event** (Type Cheer, same id).
- **The broadcast delay ends with the broadcast** that set it, and what chat was still holding is
  dropped. In 1.1 it stayed in force after Stop Stream.
- **Connect Chat for the channel already joined returns true**, with no error. **Bad Request** now
  means only "a different channel while connected".
- **A rejoin Twitch is slow to confirm is retried** (Connection Lost) instead of being abandoned as a
  bad name (Channel Rejected), if that channel was joined earlier in the session.
- **Chinese, Japanese and Korean now draw** in the example overlay instead of turning into `?`; emoji
  still do. **Make Chat Text Drawable** now also works in a subclass that has no default layout.
- **Lines deleted while the overlay is hidden stay deleted** when it is shown again.
- **The hold is timed on the wall clock**, like the picture's delay, not on the engine's frame time.
- **Log lines changed**: "leaving #x after N message(s)" and "session ended" now also count events;
  the per-frame "releasing" line became one "released N item(s)" line at most every ten seconds.

---

## Project settings

**Edit → Project Settings → Plugins → LiveCast** is where a project decides once what a broadcast
looks like: server address, bitrate, framerate, encoded size, delay, microphone, and which hardware
encoder to prefer. Games usually have one answer to those questions and many places that start a
broadcast, so this saves repeating them at every call site.

Most of it is optional: a game that fills in the settings struct itself and calls **Start Stream**
takes its address, size, rate and delay from that struct. Six settings live only here, because they
are about the project rather than one broadcast. Every Start Stream reads the first two and chat
the next three. Every Start Vote reads the last, and Chat Hold with its seconds too, for how long
voters' names wait:

| Setting | Default | |
|---|---|---|
| **Adaptive Bitrate** | on | Lower the bitrate when the uplink cannot keep up, and restore it afterwards. |
| **Encoder** | Automatic | Which hardware encoder to prefer: Automatic, NVIDIA, AMD, or the engine's default. |
| **Chat Hold** | Same as the broadcast delay | How long each chat message and channel event waits before your game sees it, so moderators can remove a line before it is drawn into the broadcast: the broadcast delay, a **Fixed moderation window**, or **Off**. Every second of it is a second of reaction time; see *Holding chat for moderation*. |
| **Chat Hold Seconds** | 10 | 0–300. The window when Chat Hold is Fixed. |
| **Max Queued Chat Messages** | 500 | 16–10000. How many messages and events may wait out the hold; past that the oldest are discarded and counted. Keep it above messages per second × the hold, or a busy channel shows nothing — see *Load*. |
| **Viewer Latency Seconds** | 6 | 0–60. How far behind live viewers watch, not counting the broadcast delay — the streaming site's own latency. Votes stay open this much longer so viewers get their whole window. Your votes put a floor under it — see **First Answer Seconds** in *Chat votes*. |

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
| `Audio/A_LiveCastJump` | A short sound on **Space**. Someone watching the stream can tell at once whether a sharp noise lands with the movement that made it — which a droning loop cannot show. |
| `Blueprints/BP_LiveCastExampleController` | Starts and stops the broadcast on **F6**, mutes the microphone on **F10**, connects and disconnects chat on **F7**, opens and cancels a chat vote on **Home**, plays the jump sound on **Space** without taking the jump away from the pawn. Set **Chat Channel** on the controller first — it is empty on purpose, so nothing joins a stranger's channel by itself. |
| `Blueprints/GM_LiveCastExample` | The game mode that spawns that controller. |
| `Widgets/WBP_LiveCastStreamHealth` | The on-screen overlay. Redesign it freely — see below. |
| `Widgets/WBP_LiveCastChat` | The chat overlay, new in 1.1; draws channel events as well since 1.2. Stays empty until **Chat Channel** is set and F7 connects. Redesignable the same way — see *Reading chat*. |
| `LiveCast Vote` (C++ class `ULiveCastVoteWidget`, not an asset) | The vote overlay, new in 1.3: the question, a bar per option, the countdown, then the result. Hidden while no vote runs. The controller uses the C++ class directly; derive a Blueprint from it to redesign it — see *Chat votes*. |

Set a stream key first, in *Project Settings → Plugins → LiveCast (this machine)*, then open the map
and press Play and F6. That is the whole of it.

> The example uses **F6**, **F7**, **F10** and **Home** because the engine has already claimed the
> other function keys: `BaseInput.ini` binds F1-F5 to view modes and **F9 to a screenshot**, and F11 to
> fullscreen. Those bindings are live in every build that is not Shipping. During Play In Editor the
> editor also takes **F8** (Possess or Eject Player) before the game sees it, which is why votes are
> on Home. On a keyboard with no Home key it is Fn+Left; set **Toggle Vote Key** to anything else. F9 was the
> broadcast key until 2026-08-30, which meant every press also wrote a full-resolution PNG -
> a frame hitch at the exact moment a streaming plugin starts streaming.

A **packaged Shipping build** has no console at all, refuses this plugin's console commands as
cheats, and ignores `-ExecCmds` - the engine compiles that out. So the example takes both things it
needs from the command line instead, which is the only way to point a Shipping build at a
destination and a channel:

```
YourGame.exe -LiveCastKey=xxxx-xxxx-xxxx -LiveCastChat=somechannel
```

Then F6 broadcasts and F7 connects chat, exactly as in the editor. A key or channel set on the
controller wins; the command line is the fallback.


The ambient loop is there for a reason worth knowing: it is tonal and carries a soft marker every
two seconds, so a listener can hear at once whether audio is arriving and whether it is stuttering —
which broadband noise or silence cannot tell you. It also means a test broadcast proves game audio
and microphone mixing together rather than one at a time. Every asset here is synthesised, so
nothing in the example content carries a third-party licence.

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
the full RTMP URL — key included — and takes the broadcast's address, size, rate, delay and
microphone from that struct rather than from the settings page. The six project-wide settings
listed under *Project settings* still apply. **Get Default Stream Settings** sits between the two:
it returns the project's configured values so you can change one field and pass the rest through,
and **Make Stream URL** builds the address from the configured server and a key exactly as **Start
Stream With Key** does.

### Blueprint nodes

| Node | What it does |
|---|---|
| **Start Stream With Key** (key) | Begins broadcasting using Project Settings, with this key appended to the configured server. In the editor an empty key falls back to *LiveCast (this machine)*. |
| **Get Default Stream Settings** | The project's configured settings, with an empty URL — change a field and pass it to **Start Stream**. |
| **Make Stream URL** (key) | The configured server with this key appended — the address **Start Stream With Key** would use. Empty if the key is empty or the server is not an `rtmp://` or `rtmps://` address. |
| **Start Stream** (settings) | Begins broadcasting. Returns false only if it could not start at all; a connection that fails later reports through **On Stream Error**. |
| **Stop Stream** | Ends the broadcast and closes the connection. |
| **Is Streaming** | Whether a broadcast is running. |
| **Get Stream Stats** | A snapshot for a stream-health widget. See below. |
| **Set Stream Delay** (seconds) | Changes the anti-sniping delay while live. With **Chat Hold** at *Same as the broadcast delay*, chat follows it, and called with no broadcast running it still sets the hold, until the next broadcast starts or stops. A fixed window or Off does not follow it for chat; voters' names still wait at least the delay. |
| **Set Microphone Gain** (gain) | Changes microphone loudness while live. Zero is mute. |
| **Owns Running Stream** | Whether this game instance started the running broadcast (or nobody did — one started from the console). False only when another game instance in the same process owns it, which in practice means multiplayer testing in the editor. |
| **Connect Chat** (channel) | Starts reading a Twitch channel. The `#` is optional, case is ignored. Asking again for the channel already joined or joining returns true and does nothing. Returns false for a request that cannot be attempted: a name that is not a Twitch login (empty, over 25 characters, anything but letters, digits and `_` — a pasted channel address is recognised and trimmed), a different channel while chat is connected, or a WebSocket the engine could not create. Each raises **On Chat Error** saying which. |
| **Disconnect Chat** | Stops reading. Held messages are discarded rather than released. |
| **Is Chat Connected** | True once the channel was actually joined, not merely once the socket opened. |
| **Get Chat Channel** | The channel being read, lowercase and without `#`. Empty when not connected. |
| **Get Pending Chat Count** | How many messages and channel events are waiting out the chat hold. Worth showing on a debug overlay: it explains a screen that looks frozen while the stream is fine. |
| **Get Chat Hold Seconds** | How long chat is held right now before your game sees it, by **Chat Hold**: the broadcast delay, the fixed window, or 0. The example overlay's status line shows the same number. |
| **Get Dropped Chat Count** | Messages and events discarded because the hold queue filled up. Rising means a raid, not a fault. It does not count what is lost when a connection drops. |
| **Start Vote** (settings) | Opens a chat vote between options the game wrote. Returns the vote's id, or 0 if refused — another vote is running, or the options cannot make a fair vote — with the reason in **Why Refused**. See *Chat votes*. |
| **Cancel Vote** | Ends the running vote with nothing winning. **On Vote Ended** still fires, as Cancelled. False when nothing was running. |
| **Is Vote Running** | Whether a vote is open or still counting its last votes. |
| **Get Vote State** | The running vote for an overlay to draw: options, tallies, voters, the countdown to show, and whether chat is connected at all. |
| **Get Last Vote Result** | How the most recent vote ended — what **On Vote Ended** carried. Vote Id 0 before the first one. |
| **Get Voter Name Hold Seconds** | How long a voter's name from **On Vote Ballot** should wait before your own overlay draws it: the chat hold or the broadcast delay, whichever is longer. The default vote overlay waits this long. |
| **Get Measured Viewer Latency** (→ Seconds, Votes Measured) | The floor this session's votes put under **Viewer Latency Seconds**: the middle of the last five votes' **First Answer Seconds** (of fewer until five have run). False, Seconds -1, until a vote with five or more of your own viewers has ended. See *Chat votes*. |

### Stream settings

| Field | Default | Notes |
|---|---|---|
| **URL** | — | Full RTMP URL including the stream key, e.g. `rtmp://host/app/xxxx-xxxx` (or `rtmps://` for TLS) |
| **Bitrate Kbps** | 4000 | 500–20000 |
| **Framerate** | 30 | 10–120 |
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
| **On Chat Message** (message) | A viewer said something, and it is due to be shown. With a chat hold, this fires that long after the line arrived, once moderators have had that long to remove it. |
| **On Chat Message Deleted** (id) | A message or event already shown has been withdrawn — deleted by a moderator, or its author banned. Take that line off screen. Anything deleted while still held never fires this, because it was never shown. |
| **On Chat Event** (event) | The channel did something — a subscription, a gift, a raid, a cheer, an announcement, a milestone. Delivered in order with the messages around it. See *Channel events*. |
| **On Chat Connected** | The channel was joined and messages can now arrive. |
| **On Chat Disconnected** | Chat ended, whether asked for or not. A reconnect in progress raises this once, not per attempt. Not raised while the game itself is shutting down. |
| **On Chat Error** (error, message) | Chat failed, or an attempt to reconnect failed. Connection Failed and Connection Lost are retried with no attempt limit — treat them as a status. **Channel Rejected is final**: nothing retries it. See *Chat error values*. |
| **On Vote Ended** (result) | A vote is over — won, tied, empty or cancelled. Fires exactly once for every vote **Start Vote** accepted, except one still running when the game shuts down. Act on chat's choice here. |
| **On Vote Ballot** (ballot) | A viewer's vote was counted: who, and for what. Once per counted vote — a repeat does not raise it. Raised as the message arrives, before any chat hold — wait **Get Voter Name Hold Seconds** before drawing the name. |
| **On Vote Ballot Withdrawn** (ballot) | A counted vote stopped counting — its message deleted, or the viewer banned or timed out. Take down whatever you drew for it. |

Events are delivered on the game thread, so it is safe to touch UI directly from them.

**On Stream Started deliberately waits for the ingest.** It does not fire when *Start Stream*
returns — the handshake happens on a background thread, and a wrong stream key must never look
like a live broadcast.

### Error values

| Value | Meaning |
|---|---|
| **Connection Refused** | The first connection failed: the URL could not be parsed or used, the ingest could not be reached, it refused the application in the URL or the stream — usually a wrong or expired key — or, for `rtmps://`, TLS refused or the address is not an RTMP ingest. The message says which. |
| **Connection Lost** | The connection dropped and could not be restored within the retry budget — or, for `rtmps://`, LiveCast's own link on this machine failed ("could not be set up on this machine"): nothing was sent, try again. |
| **Encoder Unavailable** | No hardware encoder accepted the requested configuration. |
| **No Game World** | Streaming was requested while no game world was running. |
| **Already Streaming** | A broadcast is already running — your own (call Stop Stream first; the running one is left untouched), or another game instance's. LiveCast captures one at a time per process, which you meet when testing a multiplayer game in the editor with more than one client: each has its own subsystem and they share one capture. |

A failure *before* the first successful connection reports **Connection Refused** — that is the
"check your key" case. A failure *after* it reports **Connection Lost**, and so does an `rtmps://`
failure on this machine itself ("could not be set up on this machine"), first attempt or not: trying
again is the advice for both.

### Chat error values

| Value | Meaning | Retried? |
|---|---|---|
| **Bad Request** | Chat was asked to connect to a different channel while already connected. Disconnect first. | No — it is an answer to the call |
| **Connection Failed** | The WebSocket never opened: no network, or Twitch refused the handshake. | Yes, backing off to 12 s |
| **Connection Lost** | An open connection closed, or a rejoin went unconfirmed for 10 s. | Yes, backing off to 12 s |
| **Channel Rejected** | The channel itself is the problem: a name that is not a Twitch login, a channel Twitch refused, or a name Twitch never confirmed within 10 s on the first attempt. | **No — final.** Fix the name and connect again |

A channel that was joined once in a session is never rejected later for a slow confirmation: that is
Twitch or the network having a bad moment, and it is retried as a lost connection.

### Stream stats

**Get Stream Stats** returns: `Is Streaming`, `Is Connected`, `Elapsed Seconds`,
`Frames Per Second`, `Frames Encoded`, `Frames Dropped`, `Reconnect Count`, `Buffered Megabytes`,
`Bitrate Kbps`, `Requested Bitrate Kbps`, `Uplink Backlog Seconds`, `Microphone Level`,
`Microphone Active`, `Microphone Gain`.

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

New in 1.1. LiveCast can read a Twitch channel's chat and hand each message to your game. New in 1.2:
what the channel *does* as well — subscriptions, gifts, raids, cheers — see *Channel events*.

**No account, no token, nothing to keep secret.** Twitch allows an anonymous read-only connection and
that is what this uses: the client logs in as `justinfan<random>` with no password. You never
register an application, there is no OAuth screen, and nothing new has to be kept out of source
control. It reads only — LiveCast never sends a message.

### Turning it on

Call **Connect Chat** with a channel name; the `#` is optional and case does not matter. **On Chat
Message** then fires for each line, carrying the text, the display name, the login, the colour the
viewer chose, their badges, and whether they are a moderator, a subscriber or the broadcaster.

A `/me` message arrives as its words with **Is Action** set. Twitch sends it wrapped in control bytes
(`\x01ACTION waves\x01`); LiveCast takes the wrapper off, so a game that ignores the flag still shows
the right text and only loses the styling. Twitch's own clients draw an action without the colon and
in the author's colour, and the example overlay does the same, in italics. Worth handling if a channel
runs a bot: announcement bots use `/me`, and 688 of 344 673 recorded messages were actions. Before 1.2
the wrapper reached your game untouched.

`WBP_LiveCastChat` in the plugin's example content is a working overlay built on those events. Create
it and add it to the viewport and it works as it is; derive a Blueprint from it to replace the look
entirely while keeping the behaviour.

What it exposes, whether you use it as it is or derive from it:

| Property | Default | |
|---|---|---|
| **Max Lines** | 12 | How many lines stay on screen. The oldest scrolls off. A channel event takes one, or two when the viewer typed something with it. |
| **Font Size** | 14 | Message text size in the default layout. |
| **Show Status Line** | on | Prints the channel and how long chat is held (**Get Chat Hold Seconds**). Worth leaving on while building: a screen with no messages looks identical whether the channel is quiet, the hold is holding everything, or nothing ever connected. |
| **Replace Unsupported Characters** | on | Replaces characters the font cannot draw. **A memory fix rather than a cosmetic one** — see the note on emoji at the end of this section. |
| **Unsupported Character Replacement** | `?` | What such a character becomes. Empty removes it instead. Plain `?` on purpose: the replacement itself has to be drawable, and a cleverer choice is exactly the glyph a stripped-down font is also missing. |
| **Lines** | — | The lines on screen, oldest first, with their text as it arrived: messages, and channel events drawn as text. An event line has **Is Channel Event** set and no author — draw it without a name. A subclass rebuilds its own layout from this in **On Chat Updated**, which also fires when chat connects or disconnects. Lines a moderator withdraws leave it even while the overlay is hidden. |
| **Make Chat Text Drawable** | — | The same replacement as a function, for a subclass that draws `Lines` itself: the cost is paid by whoever puts the text on screen. In such a subclass it asks the engine's default text font. |

The rest of the Blueprint surface: **Disconnect Chat**, **Is Chat Connected**, **Get Chat Channel**,
**Get Pending Chat Count**, **Get Chat Hold Seconds**, **Get Dropped Chat Count**, and the events **On Chat Connected**,
**On Chat Disconnected**, **On Chat Message Deleted**, **On Chat Event** and **On Chat Error**.

In a development build the console does the same: `LiveCast.Chat.Join <channel>` while a game is
running. Console commands are development-only — they are refused in a Shipping build.

### Channel events

**On Chat Event** fires when the channel does something, as opposed to somebody saying something.
It comes over the same anonymous connection, so there is nothing more to set up, and it waits out the
chat hold **in the same queue as the messages**, so it keeps its place among the lines around it and
a ban or a deletion can still reach it. The wait does not make it land when viewers see it: they saw
the gift in Twitch's own chat the moment it happened, and the hold only delays your game's
celebration — see *Holding chat for moderation*.

Checked live: ten minutes on a busy channel, with a separate recorder listening alongside. All 29
channel events Twitch sent in that window reached the example overlay — matched one by one by id,
none missing and none extra — and the 58 cheers arrived as events too. That run had no delay. A
second ten-minute run, on another channel with a 10-second delay, held every one of its 7 events and
drew each 10.3–10.4 s after Twitch sent it: the delay plus the network.

Switch on **Type**:

| Type | What happened | Filled in |
|---|---|---|
| **Subscription** | A new subscription, a renewal, or an upgrade from Prime or from a gift | Months (not sent with an upgrade), Sub Tier, Is Prime, Streak Months (only when the viewer chose to share it) |
| **Subscription Gift** | One subscription given to one named viewer | Recipient Display Name, Recipient User Id, Gift Count (1), Sub Tier, Gift Bomb Id |
| **Gift Bomb** | Several subscriptions bought at once, not yet handed out | Gift Count, Sub Tier, Gift Bomb Id |
| **Raid** | Another channel arrived with its audience | Raid Viewers; Display Name is the raider |
| **Announcement** | The broadcaster or a moderator posted an announcement | Text holds the words |
| **Milestone** | A channel milestone, such as a viewer's watch streak | Category (`watch-streak`), Milestone Value |
| **Cheer** | Bits cheered | Bits, Text |
| **Unknown** | Something this version does not name | Kind, System Message |

Every event also carries **Kind** (Twitch's own id: `sub`, `resub`, `raid`, `viewermilestone`…),
**System Message**, **Display Name**, **User Name**, **User Id**, **Text** (anything the viewer typed
with it, usually empty), **Message Id** and **Sent At**.

**Draw System Message.** Twitch writes that sentence itself, in the channel's language, for every
kind — including kinds Twitch adds after this plugin shipped, which arrive as Unknown with the
sentence intact. That is the case this is built for: in six hours recorded across 18 channels the
second most common kind, `viewermilestone` (1 344 times, after 1 903 renewals), was one this feature
was not first designed around, and `modiversary` turned up once. The same recording found the two
cases where the sentence is not enough on its own:

- **An announcement carries no sentence at all.** The words a moderator announced are in Text; showing
  System Message alone puts a blank line on screen every time a channel announces anything.
- **A `modiversary` leaves the name out** ("has been a moderator for 78 months!"). Display Name is
  filled in; putting it in front is your call, because for every other kind that would say the name
  twice.

Four things that will otherwise surprise you:

- **A gift bomb arrives twice.** Twitch announces the batch as one Gift Bomb, then names each recipient
  in a Subscription Gift of its own — five gifts are six events. Every one of them carries the same
  **Gift Bomb Id**; a gift given on its own has none. Adding up Gift Count over both counts nearly
  everything twice: of 574 gifted subscriptions in the recording, 560 came out of one of 98 bombs, and
  every one arrived after its bomb. Count every Gift Bomb, and count a Subscription Gift unless you
  have seen the bomb its Gift Bomb Id names. "Count only gifts with no id" sounds the same and is
  not: a bomb can go missing — chat joined halfway through it, the hold queue overflowed in a raid, a
  reconnect dropped what was held — and its gifts would then count for nothing.
- **A cheer is a message and an event.** Bits come on an ordinary chat message, so **On Chat Message**
  fires for it exactly as before, and then **On Chat Event** with Type Cheer and the same Message Id.
  Show the words from one and celebrate from the other — handling both the same way shows them twice.
  A cheer a moderator deletes while it is still held raises neither.
- **Sub Tier is 1, 2 or 3 — and 0 whenever no paid tier was named.** That covers Prime, which costs
  the viewer nothing and is deliberately not tier 1, so a game rewarding generosity does not pay the
  same for it as for a bought subscription. But it also covers a viewer continuing a gifted
  subscription (`giftpaidupgrade`), which Twitch sends with no plan at all. **Is Prime** is the only
  way to tell Prime apart.
- **A hidden gifter has Is Anonymous set.** Twitch names a placeholder account then — User Name
  `ananonymousgifter`, Display Name `AnAnonymousGifter` — and every anonymous gift shares its User Id,
  so a leaderboard keyed on it would credit one fake viewer with all of them. Twitch is not consistent
  about hiding it in the sentence either: a single anonymous gift says "An anonymous user gifted…",
  but an anonymous gift bomb opens with the placeholder itself — "AnAnonymousGifter is gifting 5 Tier
  1 Subs…". The example overlay replaces it with Twitch's own "An anonymous user". 24 recorded gifts
  and bombs were anonymous, all of them ordinary `subgift` and `submysterygift`; the `anon…` kinds
  Twitch documents never appeared.

**How much of it there is:** 4 461 channel notices in those six hours against 344 673 messages — one
per 77 messages — most of them renewals (1 903) and watch streaks (1 344). Counting the 116 cheers,
which are events too, **On Chat Event** would have fired 4 577 times. Raids are rare, three in six
hours, which is exactly why they are worth making a fuss of.

Deletions, bans and timeouts reach events the way they reach messages: removing the viewer who raised
an event, or deleting the notice itself, withdraws it whether it is still held or already shown, and
**On Chat Message Deleted** names its Message Id. The deletion of a notice is handled and tested but
has not been seen on a real channel — none of the 41 recorded deletions targeted one.

**The example overlay** draws each event as a line with no author — nobody said it — using System
Message, with the name put in front only when Twitch left it out, and whatever the viewer typed on a
line of its own underneath, the way Twitch's own chat shows a resub. It does not draw cheers: the
message carrying the bits is already on screen.

### Holding chat for moderation

Chat reaches LiveCast the moment it is posted, and whatever your game draws from it goes into the
picture. A frame in the picture cannot be called back: deleting a line afterwards takes it off screen,
but the frames that showed it are already on their way to your viewers. A broadcast delay only
postpones when they arrive.

So the one way to keep a line that moderators remove out of the broadcast is to wait before handing
it to your game. That wait is the **chat hold**. LiveCast holds each message and channel event, and a
line deleted — or whose author is banned or timed out — while it waits never reaches your game at all
(see *Moderation*). A browser overlay cannot do this: it shows each line the moment it is posted. The
hold is timed on the wall clock, so a game whose frame time the engine clamps does not stretch it.

**Chat Hold** in Project Settings decides how long:

| Chat Hold | Holds for | Use it when |
|---|---|---|
| **Same as the broadcast delay** (default) | `Delay Seconds` — nothing while there is no delay | You want what 1.1–1.3.1 did. It follows the broadcast; see below. |
| **Fixed moderation window** | **Chat Hold Seconds** (default 10), delay or no delay | Your game draws viewers' words or names. Works when LiveCast is not the one broadcasting, too — a game captured by OBS. |
| **Off** | nothing | Your game reacts to commands and speed matters. |

**The window is the hold — nothing more.** A broadcast delay does not lengthen it: what decides
whether a removed line reaches the picture is how long moderators had before your game drew it, not
how long the picture then takes to reach viewers. And every second of hold is a second later that
your game answers chat. Following a 30-second delay gives moderators 30 seconds and makes a reply
reach the viewer who asked twice the delay, plus the site's own latency, after they typed. A fixed
10-second window gives moderators 10 seconds and costs 10, whatever the delay is.

**How long is enough?** Measured on 8.4 hours of recorded Twitch chat — twelve channels where
something was removed, 4 584 removals between them: deleted messages, matched by id, and bans and
timeouts (almost all timeouts), each counted by the last line its author wrote in the ten minutes
before it. 179 bans and timeouts whose author wrote nothing in those ten minutes are not counted.

| Removed within | 1 s | 5 s | 10 s | 15 s | 30 s | 60 s |
|---|---|---|---|---|---|---|
| All removals | 84% | 90% | 94% | 95% | 97% | 98% |
| Only those slower than a second — people, not bots | — | 35% | 61% | 69% | 79% | 88% |

A removal within a second is a moderation bot — nobody reads a line and clicks that fast — and any
hold catches those. Half of all removals came from one channel's bot; without that channel, 70% were
within a second. Counting everything slower as people, they took a median of about 7 seconds, and
ten seconds caught about six in ten. That is a middle, not a guarantee — raise it if your moderators
are human and your game can afford the wait, and see *Load* before raising it far. These are mostly
large channels; on a small one, removals may be rarer and slower still. With no moderators at all,
no hold helps.

What a hold protects is **the line that got its author removed**. A ban or timeout also takes back
everything else that viewer wrote, and their earlier lines, minutes old, were shown long ago: across
every line a removed viewer sent in the ten minutes before, a 10-second hold had caught 14%. LiveCast
withdraws those it still remembers — the last 256 lines shown, which on a busy channel is well under
ten minutes — through **On Chat Message Deleted**, but the frames that showed them have gone out.
Lines AutoMod holds back are not in these figures either: Twitch shows them only once a moderator
approves them, already reviewed.

Choose by what your game does with chat:

- **It draws viewers' words or names** — a chat overlay, a name over a character: use a fixed window.
- **It reacts to commands and speed matters**: turn the hold **Off**, and lines reach your game the
  moment they are posted. A deletion can then only withdraw a line already shown — **On Chat Message
  Deleted** still reports it — and the frames that showed it before the deletion are already on their
  way to your viewers.
- **Votes** are unaffected either way: they are counted as messages arrive and timed on the viewers'
  clock. Voters' names wait the chat hold or the broadcast delay, whichever is longer, before the
  default overlay draws them (see *Chat votes*).

Things to know before relying on it:

- **Same as the broadcast delay follows the broadcast.** **Start Stream** applies the delay and the
  broadcast ending clears it — by Stop Stream, an error, or the game closing — so after a stop chat
  arrives live again. Setting `Delay Seconds` in Project Settings and then connecting chat without
  streaming holds nothing; **Set Stream Delay** called while nothing is streaming sets the hold until
  the next broadcast starts or stops. Only a delay your game set is cleared this way; one typed at the
  console or kept in an ini (`LiveCast.DelaySeconds`) stays until changed. A **fixed window** does
  none of this: it holds whenever chat is connected.
- **When the hold falls to nothing, what was held is dropped, not released** — the broadcast ending,
  the delay set to zero, Chat Hold turned Off, Chat Hold Seconds set to 0, or Chat Hold switched from a
  fixed window to the broadcast delay while there is none. Handing it over at once
  would be a burst of up to 500 lines in one frame. **Shortening** the hold is different: whatever has
  already waited the new, shorter hold is released at once, in one batch — every line of it has had
  the moderators' window, but a game drawing chat sees a burst.
- **The site's own latency is never part of the hold.** It delays only the picture — several seconds
  of ingest, transcode and player buffer — and nothing inside an encoder can measure it.
- **Get Chat Hold Seconds** says how long chat is being held right now; the example overlay's status
  line shows the same number.

### Moderation, including before a message is ever shown

With a chat hold, a moderator deleting a message inside the hold means the deletion reaches
LiveCast **while the message is still waiting**. Your game never sees it at all. Every external overlay put that
message on screen the moment it was posted and can only take it down afterwards.

A message already shown can still be withdrawn: **On Chat Message Deleted** carries its id so you can
take that line off screen. A ban or a timeout reaches backwards over both halves — everything that
user has in flight is dropped, and everything of theirs among the last few hundred messages shown is
withdrawn, one id at a time. The remembered window is 256 lines, messages and events together, so a
deletion older than that finds nothing.

Measured on a live channel: over two hours, 42 bans withdrew 516 already-shown messages, the largest
single ban taking back 68.

This covers chat messages and events. **Votes are counted as they arrive, not after the hold**, but
voters' names are held: the default vote overlay and a result's **Winning Voter Names** name only
voters whose message has waited out the chat hold or the broadcast delay, whichever is longer. An
overlay of your own that draws names straight from **On Vote Ballot** should wait the same way —
**Get Voter Name Hold Seconds** says how long; see *Chat votes*.

### When chat stops

**Held messages and events are dropped, not flushed.** If the connection drops, if Twitch asks the
client to move, if you disconnect, if a moderator clears the whole chat (`/clear`), or if the chat
hold falls to zero — the broadcast that set the delay ended, the hold was turned Off or set to 0
(see *Holding chat for moderation*) — whatever was waiting is discarded. This is deliberate. While the
connection is down no deletion or ban can reach them, so part of their moderation window is blind,
and once the broadcast has ended there is nothing left for them to wait for. Releasing them could
dump a minute of backlog into the game at once, which reads as a bug.

A dropped connection reconnects on its own, backing off 1, 2, 4, 8 and then 12 seconds, and keeps
trying for as long as chat is connected — **there is no attempt limit**. Every failed attempt raises
**On Chat Error**, so treat that event as a status to display rather than a fault to give up on.
Reconnecting yourself from **On Chat Disconnected** or **On Chat Error** is safe: asking for the
channel already being joined simply returns true, the automatic retry stands down when it finds a
connection you made, and a Connect Chat made from the handler of an error that Connect Chat itself
raised — a name that is not a channel, a connection refused on the spot — is ignored rather than
repeated (a bad name stays bad; a refused connection is already being retried). A Connect Chat made
from inside **On Chat Message** or **On Chat Event** opens its connection on the next frame.

Twitch's own `RECONNECT` request — sent before it restarts a server — closes the socket at once and
opens a new one after the first step of the backoff, a second later, on Twitch's schedule rather
than after an unpredictable hang-up. It raises **On Chat
Disconnected** and takes held messages with it, and it is not reported as an error, because it is not
one.

### Load

`Max Queued Chat Messages` (default 500) bounds how many messages and events may wait out the hold.
Past that the **oldest** are discarded and counted — during a raid the recent lines are the ones you
are reacting to. **Get Dropped Chat Count** reports the running total; note it counts only what was
lost to a full queue, not what was discarded when a connection dropped.

**It must be larger than messages per second times the hold.** A line pushed out before it has waited
its hold is never shown, so a queue too small for the hold shows **nothing at all** while the channel
is that busy: at five messages a second, a 120-second hold needs more than 600. The log says so when
it happens ("too small for a … s chat hold"). With the default 10-second window, 500 lasts up to fifty
messages a second.

That limit is also what bounds the worst frame *inside LiveCast*. Measured on 1.2, which parses
channel events and `/me` as well: 10 000 messages through the parser and the queue in 43–47 ms over
three runs, and the largest possible single-frame release — the whole 500-message queue at once —
costs about 0.2 ms in the queue itself. What your own overlay then does with
500 messages is your cost, not that figure. A busy channel is a few messages a second.

### Shared Chat

Twitch lets streamers who play together merge their chats for a while. During such a session every
line and event from the partner's chat also arrives on your channel's connection, and LiveCast hands
them over like any other — which is what Twitch's own client shows. Each carries **Is From Another
Channel** and **Source Channel Id**, so your game can tell the two audiences apart.

Check the flag wherever it matters which audience did something. On **On Chat Event** above all: a
partner's gift sub or raid is relayed here too (mostly as Kind `sharedchatnotice`), and a game that
celebrates every event celebrates somebody else's subscriber. Votes skip the partner's viewers by
default — see *Chat votes*. A partner's event arrives with Kind `sharedchatnotice` and Twitch names
the real kind in another tag, so **Type** is that real kind — a partner's resub is a Subscription with
its months filled in — and the flag says whose it was.

**Read as documented, not yet seen live.** The flag comes from Twitch's `source-room-id` tag, the real
kind from `source-msg-id`; no Shared Chat session has been recorded for this plugin, so both are tested
against the documented shape. How Twitch relays a *deletion or ban* in a partner's chat is not in its
documentation at all — whether removing a partner's line reaches your copy of it is unverified.

### What is deliberately not here

No sending, no emote images, and no filtering. No follows and no channel-point redemptions either:
Twitch does not send follows to an anonymous chat connection at all, and a redemption shows up only
when its reward asks the viewer for text — and then as an ordinary message. Message text is handed to
your game exactly as it arrived and drawn as plain text; what to show is your decision, and the badges
and moderator flags are there to decide with. A filter that fails is worse than no filter.

Two consequences of handing the text over untouched, both of which you will meet on a real channel:

- **A Twitch emote arrives as its code, not as a picture.** On the wire an emote is ordinary text —
  `Kappa`, or whatever a channel calls it — and Twitch's own client swaps in the image using a
  separate tag that says which characters to replace. LiveCast does not read that tag, so your game
  receives the word.
- **Unicode emoji arrive intact, and the example overlay replaces the ones its font cannot draw.**
  The engine's default font has no emoji glyphs. Left alone such a character draws as an empty box
  *and* makes Slate log a warning every time the line is shaped — once per row per message for a
  scrolling overlay, about 0.6 KB of memory each that never comes back. Measured on a live channel:
  180 000 warnings and 102 MB over two hours. So **Replace Unsupported Characters** is on by
  default and they arrive on screen as `?`. Everything the font family can draw is untouched —
  Cyrillic and the other alphabets, and Chinese, Japanese and Korean, which the engine font draws
  through its fallback typeface without a warning. (Until 1.2 those were replaced too: the check
  asked the main typeface only. Seen on screen replaying recorded lines, and fixed.) Turn it off if
  your own font really does cover emoji. Either way your game
  receives the original text: the replacement happens where the overlay draws, not where the
  message arrives.

## Chat votes

New in 1.3. Let chat choose what happens next — spawn a monster or drop a medkit, go left or right —
by typing a word. Like reading chat, it needs no account and no token: a vote is counted from the
same anonymous connection.

**Chat suggests, the game decides, LiveCast counts.** That division is the whole design, and it is
what lets a vote exist without breaking your game's balance:

- **Your game writes the options**, and decides what each one does. Chat picks *which*, never *what*
  or *how much*: a monster chosen by three viewers and one chosen by three thousand are the same
  monster.
- **Your game decides when to ask.** Open a vote at a moment that suits it — a lull between fights, a
  fork in the road — and the number of votes you open is the number of times chat gets to intervene,
  however many people are watching.
- **Your game acts on the result**, in **On Vote Ended**, and can still say not now: a medkit chat
  chose while the player is at full health can wait, a monster chosen during a cutscene can come
  later. Chat voted on a picture that was already seconds old, so check the present before acting.
- **LiveCast never spawns, heals or changes anything.** It counts, times and reports.

```
Start Vote (Prompt "What comes next?", Options [monster, medkit], Window Seconds 30)
    ...
On Vote Ended (Result)  →  Result.Winning Index ≥ 0 ?  →  yes: make that option happen
                                                      →  no:  show why, from Result.Outcome
```

Act on **Winning Index**, not on Outcome alone: a tie (**Decided By Lot**) and a fallback pick
(**Picked At Random**) have a winner too, and a game that only handles **Decided** drops them.

**Chat must be connected first** — **Connect Chat** to the streamer's own channel. A vote opened
without it runs, says in red that nobody can vote, and ends with no votes. The example connects on
F7 once **Chat Channel** is set. And **the overlay is yours to add**: outside the example, create a
**LiveCast Vote** widget and add it to the viewport, or nothing on screen shows the vote.

**Leave a gap between votes** — at least the latency plus the broadcast delay plus a few seconds after
a result, if the next vote uses the same keywords. Viewers see the result late and cheer it in chat
("MONSTER!"), and those lines, typed before anyone saw the next question, would count as first votes
in a vote that opened too soon.

**A vote runs on the wall clock.** Pausing the game does not pause it: the countdown keeps going and
the result arrives on time, because viewers' time keeps going too.

### What viewers type

The option's keyword as the **first word** of a message: `monster`, `#monster`, `!Monster`,
`monster pls` all count; `the monster again?` does not, because a keyword in the middle of a sentence
is conversation. A Twitch **reply** counts too: its text starts with `@name`, and that one word is
skipped. Autocorrect's `…`, a Spanish `¡medkit!`, and the fullwidth `！` and digits a Japanese or
Chinese keyboard types are all read as the plain characters.

**Numbers are off by default.** With **Accept Option Numbers** on, `1` votes for the first option —
and the default overlay then numbers the options. Off, because a bare "1" or "2" is ordinary chat
and would quietly tilt every vote towards whatever you listed first. A keyword that is itself a
number turns them off regardless. Keywords are one word each and must differ from one another, ignoring case — in any
script: `Монстр` counts for `монстр`, which matters because a phone capitalises the first word.
Keywords come back in results as they are matched — trimmed of `#`, `!` and trailing punctuation — so
act on **Winning Index** rather than comparing **Winning Keyword** with your own string.

**One viewer, one vote, and the first one counts.** Votes are counted by Twitch's user id, which
survives a rename. Typing the keyword forty times is one vote, and so is changing your mind: the
first stands, so nobody can swing a vote by repeating themselves at the last second.

**What moderators take back stops counting** — however long ago in the vote it was cast. A deleted
message takes its vote with it, and a viewer banned or timed out loses theirs; **On Vote Ballot
Withdrawn** says whose. The viewer may vote again after one deleted message, since a deletion is not
a ban. Once the result is in, it stands: a ban after that does not change it.

**Votes are counted the moment they arrive; names are not shown until the wait is over.** The chat
hold keeps a deleted message from ever being *shown* (see *Moderation*). A vote is counted at once —
the bars move — but the default overlay waits the chat hold or the broadcast delay, whichever is
longer, before it draws "Dave voted monster" — with the chat hold Off it still waits the delay — and
a ban or deletion inside that time means the name is never drawn. **On Vote Ballot** itself fires at
once: a game drawing names from it should wait **Get Voter Name Hold Seconds**. A **result** names
only voters whose message had waited that long by the time the result came — see *Winning Voter
Names* below.

### The window is on the viewers' clock

Viewers watch the stream late — by your broadcast delay, and by however long the streaming site takes
to deliver the video. A vote opened and closed on the game's clock would be over before most of them
had seen the question. So LiveCast:

- **Counts a message only once it could be an answer** — from one broadcast delay after the vote
  opened, because nobody can see it sooner. Anything earlier was typed before the vote existed: late
  votes for a previous vote, for one. Reactions to a previous *result* arrive later than that — see
  *Leave a gap between votes* above.
- **Keeps the vote open until viewers have had the whole window**: the broadcast delay, plus **Viewer
  Latency Seconds** from Project Settings, plus **Window Seconds**.
- **Announces the result one second later**, for the last messages to cross the network.

Votes are read as each message **arrives**, not when the chat overlay shows it. Chat waits out the
chat hold before it is shown, and a vote that waited too would lose votes two ways — the queue's
limit (**Max Queued Chat Messages**) drops the oldest lines first in a burst, and in a vote the
oldest are people's first votes — and the result would wait out the hold on top of the delay. So
with a chat hold, a vote is counted and **On Vote Ballot** fires before the line it came from
reaches the chat overlay.

**Votes typed before the counting starts are ignored without a word in the log**, because there are
usually many and they are not mistakes. With a broadcast delay set — even one from an ini, with no
broadcast running — that is the first *delay* seconds of every vote. The "opened" log line says when
counting starts ("counting from +30.0 s"); look there first when testing with `LiveCast.Vote.Cast`
seems to count nothing.

The countdown on the vote overlay simply runs from Window Seconds down to zero on the game's clock,
and that is exact rather than naive: the overlay is part of the picture, so it reaches viewers as
late as everything else, and reads zero for them when their time is up — provided **Viewer Latency
Seconds** matches your platform; see below. After zero the overlay says
**Counting the last votes…** until the result is in: the broadcast delay plus the latency plus one
second — with the default latency and no delay about seven seconds, with a 30-second delay about 37.
**Get Vote State** gives you both numbers for your own overlay.

**Viewer Latency Seconds defaults to 6, and that is a cautious guess, not a measurement.** Too low
cuts off viewers who vote in their last seconds; too high makes the result arrive a little later and
also accepts votes typed after a viewer's countdown showed zero.

**Your votes put a floor under it.** A vote reaches your broadcast one delay after it opens —
LiveCast knows that moment exactly — and your viewers see it the site's latency later. So when the
first answers arrive says how far behind your *quickest* viewers watch. Every result carries **First
Answer Seconds**: how long after the vote reached the broadcast the first answers came, taken where the
first fifth of voters had answered but never earlier than the third voter, so one viewer who typed the
keyword early does not read as no latency at all. After each vote the log says it next to the setting
(one line in the log):

```
[vote] #3: the first answers came 9.4 s after the vote reached the broadcast (the quickest viewers'
latency plus reaction); over 3 vote(s) 9.1 s, Viewer Latency Seconds is 6.0 - further above the
setting than reaction time explains: raise it, above this number
```

**Get Measured Viewer Latency** gives the middle of this session's last five such votes, or of fewer
until five have run; it is forgotten when chat joins another channel and when the game closes, and
LiveCast never changes the setting itself. The log advises raising the setting only when the number is
more than two seconds above it — the reaction a correct setting still reads as. Within those two
seconds it says to time your slowest viewers before changing anything, and below the setting, to keep
the setting above the number. Know what the number is:

- **A floor, not the answer.** It is the latency of your quickest viewers — low-latency players, fast
  typists — plus their reaction. Slower viewers, on phones, TVs or the normal player, watch later
  still, and **Viewer Latency Seconds** has to cover them. A setting well below this number — by more
  than a reaction takes — is very likely too low; a setting above it is not thereby enough. The number
  holds the quickest viewers' reaction as well, so a right setting may sit a couple of seconds under
  it; the slowest viewers decide the setting, and only a clock measures them.
- **Only from your own channel, and only with five or more voters.** Fewer, and the result says -1
  and nothing is logged. Shared Chat partners' viewers are left out — they watch a different stream.
  So are votes read late after the game froze, and cancelled votes, which keep only the quickest; a
  game running below about four frames a second reads every vote as after a freeze and measures
  nothing. A vote during which the broadcast delay changed, or a broadcast started or stopped, is
  not timed at all — it reached viewers at a moment nobody recorded.
- **Counted from when the vote reached the broadcast** — its opening plus the broadcast delay. If
  your game shows the question later than it calls Start
  Vote, the measurement includes that gap; a question that takes long to read adds reading time.
  Answers that do not come from watching — a chat bot's "vote now", leftovers of the previous vote
  on the same keywords — pull it down; leave a gap between votes.
- **The way to the slowest viewers is still the clock:** put one on screen, watch your stream on a
  phone or the site's normal player, and subtract, leaving the broadcast delay out.

**The delay and the latency are read when the vote opens and kept until it ends.** Changing the
broadcast delay halfway through cannot move a countdown that has already been broadcast. Lowering it
can cost votes: the delay buffer drains, viewers see the question sooner than the vote expects, and
what they type before the delay frozen at the start has passed is not counted. The delay is the one in
force when the vote opens — so **open votes while live**: a vote opened just before Start Stream
counts its viewers as if there were no delay at all. Either way the vote is not timed.

**A hitch at the very end does not cost votes.** Messages are timed by when this machine read them,
and a game that freezes reads them late — so a message read more than a quarter of a second after the
previous frame is credited to the moment the freeze began (or to the start of counting, if the freeze
began before it), and the result waits one extra frame after
it is due so that a long frame's messages are read first. This errs towards counting a vote that
arrived a little late rather than losing one that was on time.

**Cancelling during "Counting the last votes…" throws those votes away.** **Is Vote Running** stays
true until the result, so a "cancel" key pressed after the countdown still cancels.

### How a vote ends

Always on time, and always with **On Vote Ended** — whether anybody voted or not, whether chat is even
connected. A vote that waits for something can hang, and a hung vote on a live stream looks like a
broken game.

| Outcome | Meaning |
|---|---|
| **None** | No vote has ended yet: what **Get Last Vote Result** says before the first. Never carried by **On Vote Ended**. |
| **Decided** | One option had the most votes. |
| **Decided By Lot** | Options tied for the most votes and one was drawn at random, each tied option equally likely. Ties are never settled in favour of the first-listed option — that would hand every close vote to whatever you happened to list first. |
| **No Winner** | Fewer than **Minimum Voters** took part, and **When Too Few Voters** is Skip. |
| **Picked At Random** | Fewer than Minimum Voters took part, and When Too Few Voters is Pick At Random. |
| **Cancelled** | **Cancel Vote** was called. |

**Minimum Voters is 1 by default, deliberately.** Most channels are small, and a vote that needs
three voters in a channel watched by two never resolves. **Skip** is the right fallback for a vote
that makes something happen — chat staying quiet should not summon a monster — and **Pick At
Random** for one that chooses between things that will happen anyway, where the show has to go on.
Either way, say it on screen: "no votes — skipped" reads as a working feature, silence reads as a
broken one.

**Winning Voter Names** lists who voted for the winning option, earliest first, up to ten — after
**Picked At Random**, whoever among the few voters had chosen the option drawn — **but only voters
whose message had already waited out the chat hold or the broadcast delay, whichever is longer, when
the result came.** A name is chat text, and it is held so a moderator can remove it first, even with
the chat hold Off as long as there is a delay; a name in a result keeps that rule. Usually it costs
nothing, because the earliest voters are the ones named and they have waited longest. With a delay
or a hold as long as the window or longer, few or no names can qualify, and the result says
"thanks" to nobody — the votes still count. A ban after the result does not change it. Use them:
the monster can wear the first voter's name, the medkit can say who sent it. On a small channel that
recognition is most of why anybody votes. They are names viewers chose — draw them the way you draw
chat, through **Make Vote Text Drawable** on the vote overlay if your font may lack their characters.

### The vote overlay

`ULiveCastVoteWidget` (**LiveCast Vote** in the widget palette) draws the question, a bar per option
with its count, the countdown, then the result for **Result Seconds** with the winner highlighted and
the first three winning voters thanked. Lines wrap rather than run off the frame. While chat is not
connected it says so in red, under the countdown — a
vote nobody can take part in otherwise looks exactly like one nobody wanted to. Viewers' names go
through the same missing-glyph filter as the chat overlay, so they cannot bring back the warning leak
described under *What is deliberately not here*; **Show Voter Names** turns the names off, and
**Replace Unsupported Characters** the filter, for a font that really covers them. A redesigned overlay
gets the same filter as the **Make Vote Text Drawable** node.

Derive a Blueprint from it to redesign it: as with the other overlays, the default layout is only
built when the widget tree is empty, and **State**, **Result**, **Showing Result**, **Latest Ballot**
and **On Vote Updated** keep coming.

### What is deliberately not here

- **One vote at a time.** A second **Start Vote** is refused while one runs.
- **No effects, budgets or cooldowns.** LiveCast does not know what a monster is, so it cannot know
  how often one is too often. Your game decides how often it asks.
- **No veto window.** The result goes straight to your game, which can refuse it — say so on screen
  when it does. A silent refusal reads as contempt; a visible "the streamer saved that one for the
  boss" reads as a show.
- **No weighted votes and no subscriber bonus**, by design: one viewer, one vote. **No channel points
  and no Twitch polls** — those need a Twitch token, which LiveCast does not use.
- **No messages back to chat.** Reading is all this plugin does.
- **Not measured at scale.** The counting rules hold for any number of viewers, and one-vote-per-id
  is checked against a set, not a list. What has not been run is a vote against a real crowd of
  thousands, so no number is claimed for it.
- **Clearing the whole chat does not void votes.** `/clear` is not aimed at anyone; bans and
  deletions are, and those do.
- **Votes are chat lines, with everything that brings.**
  - `!monster` meant as a command for another chat bot also counts as a vote, and lines from bots
    and from the streamer vote like anyone else's.
  - The channel's modes apply: in emote-only mode nobody can type a keyword, in subscriber- or
    follower-only mode only those can vote, and a line AutoMod holds arrives when approved —
    possibly after the vote has closed.
  - Lines that arrive while chat is reconnecting are lost, votes among them; the vote still ends on
    time.
  - A timeout, however short, voids the viewer's vote, and they may vote again once it ends.
- **Shared Chat partners do not vote by default.** While two chats are merged, the partner's viewers
  type into yours but watch somebody else's stream. Set **Count Shared Chat** in the vote settings for
  a joint event where both audiences decide together. See *Shared Chat*.

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

Over `rtmps://`, a certificate refused on a reconnect is not retried either: the broadcast ends with
**Connection Lost** and the certificate's reason.

### Delay buffer (anti stream-sniping)

`Delay Seconds` holds the broadcast behind the game so viewers cannot watch your screen in real
time. It buffers **encoded** packets, so 15 seconds at the default 4000 kbps costs about 7.5 MB —
not gigabytes of frames.
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

Your stream key is a password, and log files end up in bug reports. In every address LiveCast
writes — the log, the self-test's reports, `LiveCast.Settings` — it keeps only what names the server:
the scheme, the host, the port and the first part of the path, as in `rtmps://live.twitch.tv:443/app/***`.
Everything after that first part is masked, so a key pasted twice is masked twice; so is a path of
one part, which may be the key itself, anything after a `?`, a `#` or a space, and a user name or
password written before the host. `LiveCast.TestUrl` shows the key it joins as `<key>`.

Note that Unreal's *own* console echo does print whatever you type, and so does its record of the
command line at startup. LiveCast cannot mask those — they are written before the plugin sees
anything. So a key typed into `LiveCast.Stream` is in that log for good, and that log is what gets
attached to a bug report.

The two commands that broadcast therefore accept **`file`** instead of a URL (`LiveCast.TestRtmp`
and `LiveCast.TestUrl` take their key on the command line only):

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
programmatically. The `LiveCast.<Name> <value>` lines without a verb — `DelaySeconds`, `Microphone`,
`MicrophoneGain`, `AdaptiveBitrate`, `ThrottleKbps`, `MirrorToFile`, `ForceEncoder` — are console
*variables*, not commands: they exist in every build, because the Blueprint functions and Project
Settings work by writing them.

```
LiveCast.Stream "rtmp://host/app/key" [kbps] [fps] [width] [height]   rtmps:// too
LiveCast.Stream file [kbps] [fps] [width] [height]   read the URL from a file, keep it out of the log
LiveCast.Start [kbps] [fps]      record to a file instead of streaming
LiveCast.Stop
LiveCast.TestRtmp "rtmp://host/app/key"   is the key accepted? connects, publishes nothing; plain RTMP only
LiveCast.DelaySeconds 15
LiveCast.Microphone 1            set before starting, read at start
LiveCast.MicrophoneGain 1.5      live; zero is mute
LiveCast.AdaptiveBitrate 0       hold the configured bitrate whatever the uplink does
LiveCast.ThrottleKbps 1200       pretend the uplink is this slow — watch your UI react; 0 removes it
LiveCast.DropConnection          force a reconnect, to test your UI
LiveCast.MirrorToFile 1          keep a copy of the broadcast on disk (diagnostic)
LiveCast.ForceEncoder nvenc      auto | nvenc | amf | default — amf verified on one Radeon card,
                                 see Requirements

LiveCast.Chat.Join <channel>     read a channel through the subsystem, so Blueprint events and
                                 chat widgets see it; needs a running game
LiveCast.Chat.Leave              stop that chat, and any replay feeding it
LiveCast.Chat.Connect <channel>  a separate, console-only chat that prints messages to the log and
LiveCast.Chat.Status             reaches no Blueprint event and no widget - for checking a channel
LiveCast.Chat.Disconnect         with no game running. Chat.Join is the one your game sees
LiveCast.Chat.DropConnection [s] break the chat connection as the network would, now or in N
                                 seconds, and let the reconnect run
LiveCast.Chat.Inject <per second> [wide glyphs 0/1]
                                 feed the overlay synthetic messages at a fixed rate, no network;
                                 rate 0 stops. For load, not for looks — see below
LiveCast.Chat.Replay <file|folder> [speed] [channel]
                                 replay a recording of real chat into the overlay, with no network
                                 and no channel; `stop` ends it. See below
LiveCast.After <seconds> <command>
                                 run another console command later. `-ExecCmds` fires everything at
                                 startup, which is the wrong moment for most of what is worth
                                 watching: chat cannot be broken on the frame it connects, and a
                                 screenshot taken before anything arrives shows an empty overlay
LiveCast.Vote.Start <keyword> <keyword> [...] [window=<seconds>] [random] [numbers]
                                 open a vote through the subsystem - the overlay, events and log
                                 are the real ones; `random` picks an option when nobody votes,
                                 `numbers` counts 1, 2... as votes too
LiveCast.Vote.Cancel             cancel it
LiveCast.Vote.Cast <name> <message...>
                                 say a chat line as a named viewer, through the real chat path, with
                                 no channel - for trying votes alone. The same name is the same
                                 viewer, so a second line from it tests that repeats do not count.
                                 Prints the message id
LiveCast.Vote.Delete <message id>
                                 delete that message as a moderator would
LiveCast.Settings                print what Project Settings currently says
LiveCast.TestUrl "rtmp://…"      show how an ingest URL and a key are joined, without connecting
LiveCast.SelfTest "rtmp://…"     a scripted ten-minute broadcast; rtmps:// too; see below
```

### Driving chat without a channel

Two commands put chat on your overlay with nothing connected. They answer different questions and
are not interchangeable.

**`LiveCast.Chat.Inject <per second>`** generates messages at a rate you choose. Use it for load:
how does your layout behave at fifty messages a second, does your game still hold its frame rate,
does anything grow that should not. The second argument widens the character repertoire — without
it every message is English words the font has already cached, which measures almost nothing, and
that mistake cost a real measurement here.

What it cannot do is look like chat. The text is generated, the names are `Viewer1`, and deletions
are not modelled at all — a moderator's deletion names the id of a message that really arrived, and
nothing synthetic can produce one.

**`LiveCast.Chat.Replay <file>`** plays back a recording of a real channel instead. Same overlay,
same parser, same queue, same Blueprint events — the socket is the only thing missing. So a
subscription arrives where a subscription arrived, a deletion lands on the message it landed on, and
the text is what people actually typed, in the scripts they actually type in.

```
LiveCast.Chat.Replay D:/captures/chat-20260907-20.jsonl          as recorded
LiveCast.Chat.Replay D:/captures 30 somechannel                  a folder, 30× speed, one channel
LiveCast.Chat.Replay stop
```

- A folder plays its `.jsonl` files in name order, so an hour-per-file recording runs end to end.
- **Speed** multiplies the recorded pacing. At 1 it is real time.
- **Channel** keeps only that channel's lines. A recording usually holds several at once, and
  replaying all of them into one overlay produces something no viewer has ever seen.
- Lines are retargeted onto whatever channel the game is on, so the recording need not be of yours.
- **Only what the channel said and did is replayed**: messages, channel events, deletions and bans.
  A recording also holds the recorder's own conversation with the server — its login, the join
  confirmation, keepalives, notices, a server `RECONNECT` — and none of that is played: a replayed
  join confirmation would tell your chat it had joined a channel it never connected to. Those lines
  are counted in the summary as skipped.
- **The chat hold is honoured.** Replayed lines wait out whatever **Chat Hold** holds for — the
  broadcast delay by default, so with `LiveCast.DelaySeconds` set — and a deletion can land while its
  message is still held, which is the configuration worth replaying most, and until 1.2 the one
  where a replay showed nothing at all.

**No channel is needed and no network is touched**, which is the point: it runs behind a firewall,
it runs in CI, and it plays the same lines in the same order every time. That last one matters more
than it sounds — a measurement paced by how busy somebody else's channel happened to be cannot be
compared with the one before it. The pacing follows your frame rate, so which frame a line lands on
can differ between runs; what arrives, and in what order, does not.

### The recording format

One JSON object per line, UTF-8, written in arrival order:

```json
{"t": 1788800923.251, "line": "@badge-info=…;display-name=… :a!a@a.tmi.twitch.tv PRIVMSG #chan :hi"}
```

`t` is seconds since 1970 and only the differences matter. `line` is the raw IRC line exactly as it
came off the wire. A line whose bytes were not valid UTF-8 is written as `{"t": …, "b64": "…"}`
instead and skipped on replay — a recording that quietly mangles text would be worse than none.

Anything that can write that file will do; no recorder ships with the plugin. The one used to build
this plugin's own test material is a standard-library Python script — anonymous, read-only,
and deliberately not built on this plugin's parser, because a recording made by the code it is used
to test proves nothing. If you write your own, keep that last property.

**One limit worth knowing before you make a timing claim from a recording:** the timestamp is taken
once per network read — up to 64 KB, which can hold a backlog — and written onto every line that read
returned; in the six-hour recording up to 44 lines share one reading. Order is exact; spacing inside a
read is not recorded and replays as simultaneous. Over minutes this is invisible. Inside a raid it is
not.

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
malformed URL. Test it with `LiveCast.TestRtmp` before blaming the stream — or, for an `rtmps://`
address, with `LiveCast.SelfTest`, which is a real ten-minute broadcast to that channel and stops
early on a refusal. If the message begins "The encrypted (rtmps://) connection to the ingest failed",
TLS refused before anything was sent, and the rest of the message says why: a certificate this
machine does not trust, a host name the certificate does not name, or a handshake that never
completed — usually a port that serves plain RTMP, or a network that blocks TLS, where `rtmp://` may
work. "No root certificate loaded here vouches for the authority that issued it" is most often a root
missing on this machine rather than an attack: a game packaged with LiveCast 1.4 carries the engine's
bundle, so check the build first. Rarely the cause is this machine itself — "the engine has no
trusted root certificates loaded". "answered over TLS with a valid certificate, but it is not an RTMP
ingest" means the address is a working HTTPS server and the wrong place: check the host and port.

**On Stream Error → Connection Lost, "could not be set up on this machine".** For `rtmps://` only:
LiveCast's own link between the RTMP library and the TLS connection failed, or Windows would not give
it a socket or a thread. Nothing was sent; try again. The self-test calls this `LOCAL FAILURE`.

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
- RTMP and RTMPS — no SRT, no WebRTC.
- The encoded resolution is fixed for the duration of a broadcast.
- One broadcast at a time.
- Chat is Twitch only, read-only and anonymous. No sending, no follows, no channel-point redemptions,
  no emote images — an emote arrives as its code word — and no filtering. A withdrawal reaches only
  the last 256 messages and events shown.
- Votes: one at a time, by keyword (or option number, when switched on), counted from chat — no Twitch polls, no channel points, no
  weighting. Not measured against a crowd of thousands.

---

## Licensing

LiveCast bundles one third-party library, **srs-librtmp**, under the MIT licence. The engine's OpenSSL,
used for `rtmps://`, is linked into the plugin's binary, so its notice travels with the plugin too, and
packaged games receive the engine's certificate bundle. See `THIRD_PARTY_LICENSES.md` for the full
notices and for what is used from the system and the engine.

---
