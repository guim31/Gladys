# Music: play any media, from any source, on any player

> **Living specification — source of truth.** This document specifies how Gladys Assistant describes music players and music sources, how "play *this* on *that*" is represented and routed, where the "now playing" state lives, and the contracts integrations (internal, MQTT, external) and the frontend rely on. **Rule: any PR that changes a music behavior or contract modifies this file in the same diff** — spec first, code second.
>
> Status: **draft, revision 2, for review.** Nothing in sections A–H is implemented. Section "Precedents" describes the code as of **v5.2.1** (`ff59ad8`, upstream `master` at the time of writing); nothing cited here exists only on an unreleased branch.
>
> Designed under `docs/specs/device-feature-categories.md` (section I audits it against that checklist) and modelled on `docs/specs/camera-ptz-control.md`.
>
> **Revision 2** (after the maintainer's review, [forum 11260/4](https://community.gladysassistant.com/t/11260/4)):
>
> - "Now playing" is no longer a JSON document in a feature's `last_value_string`: it is a **record owned by the core**, held in memory, with one entry point, a strict schema and typed transports in and out (E) — the maintainer's model, adopted as proposed and detailed.
> - The same rule is applied to the command: "play this" is a record the core builds, so it travels on **typed music transports** (internal call, `external-integration.music.play`, MQTT `…/music/play`) instead of riding in `setValue`'s `options` (C.3). The generic device path, the `external-integration.device.set-value` message and the MQTT state topics are untouched: "payload = the value as a string" holds everywhere.
> - `play-media` carries **no `supported_options`** anymore: its presence means "plays media by URL" and native playback follows from ownership (Matter's Content Launcher split, C.2); queueing is part of every player's contract.
> - Closed: former open questions 3 (now playing storage), 4 (MQTT JSON payload), 9 (artwork of relayed media), 11 (`play-media` on the generic value route).

## Context

Community request ([forum topic 11260](https://community.gladysassistant.com/t/11260), "Gladys Music"): ask Gladys to play **any music** — a track, an album, an artist, a playlist, a radio — from **any source** (Subsonic/Navidrome, Plex, Jellyfin, a DLNA server, web radios, Spotify…) on **any speaker** (Sonos, AirPlay, Google Cast, DLNA renderers, Denon/Marantz amplifiers, MusicCast, LinkPlay/WiiM, Lyrion/Squeezebox, Bose SoundTouch, MPD, VLC, Echo…), by voice (the Gladys AI), from a scene and from a dashboard widget.

Today the `music` category can change the volume, play, pause, skip, report a playback state and play a notification URL. It cannot say "play *this*": there is no notion of a media, of a source, or of what is currently playing, and the AI has no music tool at all.

The maintainer's answer (quoted in the forum thread) frames the work: it is **core** work, because sources and speakers are spread across many integrations and linking them is precisely the core's job; the existing **Music** widget must be adapted rather than duplicated; and the spec must settle five questions before any code:

| # | Question | Settled in |
|---|---|---|
| Q1 | How a player is described and what it can do — separate features or `supported_options` | A.1–A.3, C.2 |
| Q2 | How "play this" is represented, when a device command is a single value | A.4, C |
| Q3 | `play_media` or an extended `play_notification` | D |
| Q4 | Where the "now playing" state (title, artist, artwork) lives | E |
| Q5 | How a source declares itself, in the core and from external integrations | B |

### Design goals

1. **Players stay devices, sources are services.** A speaker is a thing in a room with features (volume, playback state); a catalog is not. A player is a device of the `music` category; a source is a service — internal or external — that implements a small search/resolve interface, exactly as a weather provider implements `weather.get`.
2. **Nothing that works today breaks.** Every existing `music` type, every scene, every MQTT setup and every external integration keeps working unchanged. New capabilities are additive.
3. **Features carry values; music records are core objects.** A feature holds one atomic value that every generic consumer can handle — scene triggers, the device page, the MQTT state topics, the AI's `device.get-state` (`device-feature-categories.md` rule 5). What the music domain needs beyond that — what is playing (E), and the request "play this" (C) — are **records** whose fields the core must understand. They live in `server/lib/music/`, behind one entry point with a strict schema, and travel on **typed transports** in both directions (a direct call for internal services, a dedicated host-API endpoint or WebSocket message for external integrations, a dedicated MQTT topic). The generic device path is not touched. A media itself is named by one string, the **media URI** (`ext-navidrome://album/al-42`): an identifier any consumer can carry.
4. **Capabilities follow the standard, without flags.** Matter's Content Launcher separates "play a URL" (`UrlPlayback`) from "play my own content" (`ContentSearch`). So does this spec: a `play-media` feature means "plays media by URL", and native playback follows from ownership — a source plays its own media on its own devices. No capability flag hides in `supported_options` (C.2).
5. **The core never streams audio and never fetches a URL it was handed.** Sources hand playable URLs to players; artwork travels as bytes through the owning service. No transcoding, no stream proxy, no SSRF surface (G).
6. **A few dozen lines for an integration author.** Becoming a source is three SDK handlers; becoming a player is one feature declaration and one `onMusicPlay` handler; reporting what plays is one `publishNowPlaying` call (B.4, F.4).

## Precedents (verified in the code, v5.2.1)

Paths are relative to the repository root.

**The music category today**

- `DEVICE_FEATURE_CATEGORIES.MUSIC = 'music'` and `DEVICE_FEATURE_TYPES.MUSIC` = `volume`, `play`, `pause`, `previous`, `next`, `playback_state`, `play_notification` (`server/utils/constants.js`, the `MUSIC` blocks). The types use underscores, unlike the kebab-case convention of `device-feature-categories.md` rule 7.
- `MUSIC_PLAYBACK_STATE = { PLAYING: 1, PAUSED: 0 }` is the only music enum.
- `ACTIONS.MUSIC.PLAY_NOTIFICATION = 'music.play-notification'` is the only music scene action (`server/lib/scene/scene.actions.js`): it compiles `action.text` with Handlebars, asks Gladys Plus for a TTS URL (`gateway.getTTSApiUrl`, plan-gated), then calls `device.setValue(device, playNotificationFeature, url, { volume })`. It targets a **device selector**, not a feature selector.
- `play_notification` is the only place where a **string** reaches a non-text feature: `server/lib/device/device.setValue.js` documents it ("a string is only a state on a text feature; on any other feature it is a one-shot command (a music notification URL for example): nothing to save").

**The three internal players** (`server/services/*`)

| Service | Features | Notification | Feedback |
|---|---|---|---|
| `sonos` (`@svrooij/sonos`) | `play`, `pause`, `previous`, `next` (1/1), `volume` (0–100), `playback_state` (read-only), `play_notification` — all `has_feedback: false`, `keep_history: false` | `sonosDevice.PlayNotification({ trackUri, onlyWhenPlaying: false, volume: options?.volume \|\| 45, timeout: 20, delayMs: 700 })` (`lib/sonos.setValue.js`); the library advertises restoring the previous playback afterwards | UPnP events: `AVTransport` → `playback_state`, `Volume` → `volume` (`sonos.onAvTransportEvent.js`, `sonos.onVolumeEvent.js`). The event's `CurrentTrackMetaData` is already parsed by the library into a `Track` (`Title`, `Artist`, `Album`, `AlbumArtUri`, `Duration`, `TrackUri`) but **ignored**; the position (`RelTime`) is read with `AVTransport.GetPositionInfo` |
| `google-cast` (`castv2-client`) | `play_notification` only | Launches `DefaultMediaReceiver`, `load({ contentId: url, contentType: 'audio/mp3' })`, restores the volume on `FINISHED` (`lib/google_cast.setValue.js`). Launching the receiver **kills** whatever app was casting: nothing is resumed | none: **no persistent connection** — a client is created per notification, after a 5 s bonjour scan that only keeps `name → IP` |
| `airplay` (`@lox-audioserver/node-airplay-sender`) | `play_notification` only | `ffmpeg -re -i <url>` decoded to PCM and pushed to the sender, capped at 5 minutes (`MAX_NOTIFICATION_DURATION_MS`, `lib/airplay.setValue.js`) | none |

No service reports a title, an artist or an artwork anywhere.

**The value paths**

- `device.setValue(device, deviceFeature, value, options = {})` calls `service.device.setValue(device, deviceFeature, value, options)`, with no type/min/max validation in the core; it saves the state only when `!has_feedback` (`saveState` for numbers, `saveStringState` for `text`/`select`).
- Both HTTP value routes (`post /api/v1/device/:device_selector/:feature_category/:feature_type/value`, `post /api/v1/device_feature/:device_feature_selector/value`, `server/api/controllers/device.controller.js`) emit `ACTION.TRIGGERED` with a `device.set-value` action, and that action converts every value to a number on any feature that is not `text`/`select`, aborting with `ACTION_VALUE_NOT_A_NUMBER` otherwise. **A URL cannot be sent to a music feature over HTTP today**, and no `options` can be passed.
- `saveStringState` writes `last_value_string` (TEXT column, no history) and emits the WebSocket event `device.new-string-state` `{ device, device_feature, last_value_string, last_value_changed }`. Inbound states carrying `text` (MQTT `.../feature/:id/text`, external `POST /api/integration/v1/state` with `text`) go through it **for any feature** (`server/lib/device/device.newStateEvent.js`).
- **Inbound states are fire-and-forget.** `POST /api/integration/v1/state` answers `200 { success: true }` as soon as the batch shape is valid: the controller does not await `saveStates`, which emits one `device.new-state` event per state; `device.newStateEvent` then logs and drops whatever it cannot save (unknown feature at `info`, save errors at `debug`). An integration never learns that a value was refused — the reason a record cannot be validated on this path.
- MQTT (`server/services/mqtt/lib/setValue.js`) publishes `value.toString()` on `gladys/device/{device_external_id}/feature/{feature_external_id}/state` (QoS 0, not retained); `options` are not part of the topic contract. Inbound, the core subscribes to `gladys/master/#` (`handler/handleGladysMessage.js`): `device/create`, `device/:id/feature/:id/{state,text,historical}`, `scene/:selector/start` — topics outside the feature pattern already exist. Retained messages get no special treatment: they are processed like fresh ones at each (re)subscription.
- External integrations: the proxy service (`server/lib/external-integration/externalIntegration.registerProxyService.js`) forwards `external-integration.device.set-value` `{ message_id, device: { external_id, selector, params }, device_feature: { external_id, category, type }, value }` and **drops `options`** — an external speaker never receives the notification volume today.

**`supported_options`**

- Table `t_device_feature_supported_option` (`migrations/20260619120000-create-device-feature-supported-option.js`, plus `value_string` in `20260814000000-…`): `{ value, label, sort_order }`, synced by `device.syncFeatureSupportedOptions` on create/update, validated by `server/utils/normalizeSupportedOptions.js` (integer values everywhere; string values only on `text`/`select`), loaded with every device through `getStandardDeviceIncludes()`.
- `camera.move` (`CAMERA_MOVE`) is the model: one feature, canonical integer values in the core, per-device subset in `supported_options` (`docs/specs/camera-ptz-control.md`). `AC_MODE`, `WATER_HEATER_MODE`, `SIREN_MODE`, the thermostat preset follow the same pattern.
- **Options are values a user sends.** The scene editor builds a value dropdown from the `supported_options` of **any** feature when no static labels exist (`front/src/utils/deviceFeatureValueOptions.js`, used by the `device.set-value` action, the device-state trigger and the condition); the dashboard renders them as buttons through `AdaptiveOptionControl`. A capability flag stored there would be offered as a value to send.
- The same spec **rejects structured payloads in `setValue`** ("Extending `setValue` with a structured `options` payload", `camera-ptz-control.md`, Alternatives): every consumer of the device-control path speaks one scalar per feature.

**Core objects kept in memory**

- External integrations' connection status: `connectionStatuses` (a `Map` in `server/lib/external-integration/index.js`), pushed with `external-integration.connection-status-updated`, served by `GET /api/v1/external_integration/:selector` — "kept in memory only (lost at restart, republished by the integration)" (`externalIntegration.setConnectionStatus.js`). The discovered devices of external integrations, Zigbee2mqtt and the MQTT Home Assistant discovery follow the same pattern (memory + event + read route).
- **A dedicated endpoint for a payload that is not a feature value**: `POST /api/integration/v1/camera/image` `{ device_external_id, image }` → `gladys.device.camera.setImage`, `404` if the device does not belong to the integration, `400` if it has no camera feature, its own rate limit (12 images/min per device) outside `POST /state` (`docs/specs/external-integrations/contracts/host-api-endpoints.md`).
- Host-API errors: a `BadParameters` thrown by the core becomes `400 { status, code: 'BAD_REQUEST', message }` (`errorMiddleware`); Joi-validated contracts turn the first failure into `BadParameters('<path>: <message>')` (`server/lib/energy-contract/contract.validate.js`). Rate limits are in-memory fixed windows per endpoint (`server/lib/external-integration/constants.js`).
- The SDK (`@gladysassistant/integration-sdk` 0.15.0) re-authenticates after a WebSocket drop, re-reads `GET /device` and `GET /config`, emits `connected`, and **replays nothing**: the template republishes its discovered devices and transports from that event. Messages sent while disconnected are dropped.

**Providers and images in the external-integration contract**

- The core enumerates weather providers by duck typing: `server/lib/weather/weather.get.js` scans `stateManager.getAllKeys('service')` for `service.weather.get`. External integrations join through the proxy service, which wraps `sendCommand` (request/response over WebSocket, `message_id` + `external-integration.command-result`, 5 s default, 15 s for weather and widgets) and normalizes every payload ("the payload is never trusted").
- A new contract that the core does not consume through a dedicated type is a **capability field** usable by every type (`docs/specs/external-integrations/capabilities/provider-type.md`; `CAPABILITY_MANIFEST_FIELDS = ['widgets', 'scene_triggers', 'scene_actions', 'energy_contracts']` in `server/lib/external-integration/constants.js`).
- **Images**: the core never lets the browser load a third-party URL and never fetches an integration-supplied URL; content carries **keys**, the integration returns the bytes on demand (`widget.get-image`, raw base64, PNG/JPEG/WebP, ≤ 300 KB, ≤ 4096 px), and the core serves them from its own origin with a cache (`docs/specs/external-integrations/capabilities/dashboard-widgets.md` §6). The same section lists "media control with seek and volume" as a **core** widget matter.

**The AI**

- Tools live in `server/services/mcp/lib/buildSchemas.js` (`getAllTools`): `{ intent, config: { title, description, categories, inputSchema (zod) }, cb }`, exposed as `function` tools to the Gladys Plus chat (`server/lib/gateway/gateway.forwardMessageToAiChat.js`) after a category classification (`AI_CHAT_TOOL_CATEGORIES`: scenes, device_control, device_query, weather, web_and_time, other). Devices and rooms are matched by `findBySimilarity` (exact, then Levenshtein ≤ 3).
- **No music tool exists**: music devices only appear in the `scene.create` schema, for `music.play-notification`. `device.get-state` renders `last_value` only, for sensor, light, switch, AC and shutter features: music and text features are not covered (`server/services/mcp/lib/formatValue.js`, `selectFeature.js`).

**The widget**

- `front/src/components/boxs/music/MusicBox.jsx`: card titled with the device name, three buttons (previous, play/pause, next) and a volume range, driven by the `play`/`pause`/`previous`/`next`/`volume`/`playback_state` features, live through `device.new-state`. `EditMusicBox.jsx` lists `music` devices that have a `play` feature ("some music devices like the Nest Mini only expose the notification for now"). Box config: `{ type: 'music', device }`.

**Prior art outside Gladys** (what we keep and what we drop is argued in "Alternatives considered")

- **Home Assistant `media_player`**: one entity per player, a `supported_features` bitmask (PLAY_MEDIA, BROWSE_MEDIA, SEARCH_MEDIA, MEDIA_ENQUEUE, MEDIA_ANNOUNCE, GROUPING…), `play_media(media_content_id, media_content_type, enqueue: add|next|play|replace, announce)`, attributes `media_title`, `media_artist`, `media_album_name`, `media_duration`, `media_position` + `media_position_updated_at` (clients extrapolate the position), artwork served through `/api/media_player_proxy/<entity>?token=…` because it often lives on the LAN. `media-source://` URIs are resolved by the core into a plain URL before reaching URL-only players. Since 2025.6 the `HassMediaSearchAndPlay` intent searches then plays the first result, and is exposed as an LLM tool.
- **Music Assistant**: music providers vs player providers; URIs `<provider>://<media_type>/<item_id>`; `search` returns tracks/albums/artists/playlists/radios; server-side queues; announcements snapshot and resume the previous queue; MA **streams every source itself** to every player.
- **Matter** (checked against `@matter/types`, spec 1.6): the *Speaker* device type is On/Off (mute) + Level Control (volume). The *Media Playback* cluster has **one command per transport action** (`Play`, `Pause`, `Stop`, `StartOver`, `Previous`, `Next`, `SkipForward`, `SkipBackward`, `Seek`), a `CurrentState` attribute (Playing, Paused, NotPlaying, Buffering) and, with the `AdvancedSeek` feature, `Duration` and `SampledPosition { updatedAt, position }`. The *Content Launcher* cluster has two features: `UrlPlayback` (command `LaunchURL`) and `ContentSearch` (command `LaunchContent`, the device's own catalog). Matter has no title/artist/artwork attributes and no model of a music catalog.

## A. Data model

### A.1 What a player is (Q1)

Two notions, both read from the device's features and owner — no new flag:

- A **music device** is a device with at least one feature of the `music` category. It shows in the Music widget, carries the transport features it has, and can report what it plays (E). An amplifier, a TV or a Bluetooth speaker that only reports its current track is a music device.
- A **media player** is a music device that can receive "play this" (C): it has a `play-media` feature (A.3: it plays media by URL, from any streamable source), **or** it belongs to the service of a source that declares native playback (C.2) — the Spotify integration's Connect devices, a Lyrion player.

**Decision on Q1 — the transport stays one feature per command; `supported_options` stay for values a user picks.**

- The existing transport types (`play`, `pause`, `previous`, `next`, `volume`, `playback_state`, `play_notification`) are **kept unchanged**. They are deployed on three internal services and in external integrations, users' scenes reference them by feature selector, and they already are Matter's model: the Media Playback cluster has one command per transport action next to one `CurrentState` attribute. A missing command (`stop`, phase 3) is added the same way, as one more type mapping one Matter command.
- `camera.move` groups the values of **one** quantity — the directions of one movement, released by `STOP = 0` — into one feature. Transport commands are not values of one quantity, and the standard keeps them apart. A single `transport` feature would also break every scene that uses the current ones (see Alternatives; the final call is open question 1).
- `supported_options` keep their meaning: **values a user sends**, rendered as controls (a future `repeat` feature: off / one / all). They never carry capability flags: what a player can play follows from the presence of `play-media` and from ownership (C.2).

### A.2 New types and enum changes

`DEVICE_FEATURE_TYPES.MUSIC` gains **one** type (`server/utils/constants.js`):

| Type | Constant | Role | Value | `read_only` | `min`/`max` | `has_feedback` | `keep_history` |
|---|---|---|---|---|---|---|---|
| `play-media` | `PLAY_MEDIA` | declaration: the player plays media by URL (A.3) | none — never written through `setValue` | `false` | — / — | `false` | `false` |

What is playing is **not** a feature type (E).

**Naming:** the new type uses kebab-case (rule 7 of `device-feature-categories.md`), next to the legacy underscored siblings (`play_notification`, `playback_state`), which are not renamed — renaming is the breaking mistake rule 7 warns about. The mixed spelling inside one category is ugly; open question 2 asks the maintainer to pick.

**`MUSIC_PLAYBACK_STATE` is extended, append-only:**

```js
const MUSIC_PLAYBACK_STATE = {
  PAUSED: 0, // unchanged
  PLAYING: 1, // unchanged
  STOPPED: 2, // new: nothing loaded, or playback ended (Matter NotPlaying)
  BUFFERING: 3, // new: loading, about to play (Matter Buffering)
};
```

`0` and `1` keep their meaning, so every existing scene condition and the current widget are unaffected. A player that cannot tell `STOPPED` from `PAUSED` keeps publishing `0`. The widget uses `STOPPED` to show its idle state (H.1).

**New request enum (not stored, used by the API, the scene action, the AI tool and the play request of C.3):**

```js
const MUSIC_ENQUEUE = {
  REPLACE: 'replace', // default: stop what plays, play this
  NEXT: 'next', // insert after the current item
  ADD: 'add', // append at the end of the queue
};
```

**New media-type list (search filter, URI segment, item `type`):** `track`, `album`, `artist`, `playlist`, `radio`. Append-only (`podcast`, `audiobook` are expected later). Lowercase singular, the Music Assistant vocabulary.

**No schema migration.** Types are Sequelize ENUMs generated from the constants (TEXT in SQLite), `t_device_feature_supported_option` is unchanged, and nothing music-related is persisted outside the existing tables (E.5).

### A.3 The `play-media` feature

A player that plays media **by URL** declares one `play-media` feature — Matter's Content Launcher `UrlPlayback`. Its presence is the whole declaration: the player accepts HTTP(S) streams, hence media from any streamable source and direct URLs (C.2).

| Player | `play-media` | Plays |
|---|---|---|
| Sonos, Google Cast, LinkPlay/WiiM, MPD, DLNA renderer, VLC, AirPlay… | yes | any streamable source, direct URLs |
| Lyrion player (its integration is also a native source) | yes | any streamable source, and the Lyrion library natively |
| Spotify Connect device (owned by the Spotify integration) | **no** | Spotify media only, natively (C.2) |
| A TV or an amplifier that only reports its track | no | nothing — a music device, not a media player |

- **No `supported_options`.** There is no value to pick. `normalizeSupportedOptions`, which already scopes option domains by feature type, refuses options on `play-media`. Queueing is not a capability either: `replace`, `next` and `add` belong to every media player's contract (C.3).
- **Never written through the generic path.** The only writer is `gladys.music.play` (C.1), and it does not call `device.setValue`: the request is a record and travels on the music transports (C.3). `device.setValue` refuses a `play-media` feature (`BadParameters`: "play-media is driven by the music API"), and no generic control offers it: the scene `device.set-value` picker, the device-state trigger and the condition exclude it, and it stays out of `SUPPORTED_FEATURE_TYPES_BY_CATEGORY`.
- **No state.** What plays is a core record (E), not the state of this feature. A feature that holds no value is the one place where this model bends the feature mechanism; it is preferred to a flag hidden in `supported_options` because it is typed, per device, persisted, and offered to nobody as a value (open question 3).

### A.4 The media URI (Q2)

A media is named by **one string**:

```
<source>://<type>/<id>
```

- `<source>` — the **source key** (B.1): the service's key in the state manager (`radio`, `ext-navidrome`, `ext-spotify`). Lowercase, `[a-z0-9-]`, ≤ 64 chars.
- `<type>` — one of the media types of A.2.
- `<id>` — opaque to the core, chosen by the source, URL-encoded, ≤ 512 chars. It must **not** carry credentials (G.3).

Examples: `ext-navidrome://album/al-42` (an external Navidrome integration), `radio://radio/fip` (the internal `radio` source), `ext-spotify://playlist/37i9dQZF1DXcBWIGoYBM5M`.

**Direct URLs.** `http://…` and `https://…` are accepted as a media URI of implicit type `track` with no source: "play this stream". They exist for scenes and power users (a web radio URL pasted in a scene), are playable only on players with `play-media`, and are refused from the AI tool (G.1).

Why a string: a media is **named**, and a name is the one thing every consumer can carry — a scene field, a chat tool argument, a dashboard box setting, a log line. The URI is an identifier, opaque outside `server/lib/music/`. The structure a speaker needs (one URL per track, titles, artwork keys) is a separate record, built **by the core at play time** (C.1), never stored and never typed by a user.

## B. Sources (Q5)

### B.1 What a source is

A **source** is a service that can search a catalog and turn a media URI into something playable. It is identified by its **source key** = its state-manager service key (the same key `weather.get` iterates). One service is at most one source. A service can be a source **and** own players (Spotify, Plex/Plexamp, Lyrion, MPD, Music Assistant): nothing forbids it, and C.2 relies on it.

A source declares, once, its **descriptor**:

```json
{
  "name": "Navidrome",
  "media_types": ["track", "album", "artist", "playlist"],
  "streamable": true,
  "native": false
}
```

- `name` — display name (≤ 40 chars).
- `media_types` — the subset of A.2 types it can search and resolve.
- `streamable` — `true` when `resolve` returns HTTP(S) URLs any player with `play-media` can fetch; `false` for a source whose content has no stream outside its own players (Spotify: no stream URL exists outside Spotify Connect).
- `native` — `true` when the source's media can be played **natively on the music devices of its own service** (Matter's `ContentSearch`): the Spotify integration on its Connect devices, Lyrion on its players, Music Assistant on its players. `false` (default) for a catalog without players (Navidrome, a radio list).
- At least one of the two is `true` (validation error otherwise: such a source could play nowhere).

### B.2 The source interface

Every source implements three functions — duck-typed on internal services exactly like `service.weather.get`, and forwarded over WebSocket for external integrations (B.4):

| Function | Input | Output | Timeout |
|---|---|---|---|
| `music.search(query)` | `{ query, types[], limit, offset, language }` | `{ items: MediaItem[], has_more }` | 8 s |
| `music.resolve(request)` | `{ uri, player: { selector, service } }` | `{ items: PlayableItem[] }` — **only for `streamable` sources** | 8 s |
| `music.getImage(key)` | `image_key` | raw bytes (PNG/JPEG/WebP, ≤ 300 KB, ≤ 4096 px) | 15 s |

Plus the descriptor: `music.getDescriptor()` for internal services, the manifest field for external ones.

**`MediaItem`** — what a search returns, and the only media shape the frontend and the AI ever see:

```json
{
  "uri": "ext-navidrome://album/al-42",
  "type": "album",
  "title": "\"Heroes\"",
  "subtitle": "David Bowie · 1977",
  "artist": "David Bowie",
  "album": null,
  "duration": null,
  "image": "cover-al-42",
  "playable": true
}
```

- `uri` (required) — A.4, and its `<source>` must equal the answering source's key (the core rejects others: a source cannot vend another source's URIs).
- `type` (required), `title` (required, ≤ 200), `subtitle` (≤ 200, free line shown under the title), `artist`/`album` (≤ 200, optional, used by the AI to disambiguate), `duration` (seconds, tracks only).
- `image` — an **image key** (`^[a-z0-9][a-z0-9-]{0,63}$`, the widget-image rule), never a URL. The front loads it from the core (B.5).
- `playable` — `false` for an item that can only be expanded (a genre, phase 2 browse).

**`PlayableItem`** — what `resolve` returns, handed to the player and **never** returned to a client (it can contain stream tokens, G.3):

```json
{
  "url": "http://192.168.1.20:4533/rest/stream?id=tr-1&u=gladys&t=…&s=…",
  "mime_type": "audio/flac",
  "title": "Beauty and the Beast",
  "artist": "David Bowie",
  "album": "\"Heroes\"",
  "duration": 216,
  "image": "cover-al-42",
  "uri": "ext-navidrome://track/tr-1"
}
```

`url` is `http:`/`https:` only; `mime_type` optional (the player sniffs otherwise); `duration` absent for a live radio. A resolve returns **at most 500 items** — an artist resolves to the source's choice (top tracks, shuffled discography…): the source knows its catalog better than the core. Ordering is the play order.

**Search rules.** `query` is free text (≤ 200 chars, trimmed, non-empty). `types` empty = all the source's `media_types`. `limit` 1–50 (default 20), `offset` ≥ 0 — **pagination is per source**: the aggregated API (C.4) passes `limit`/`offset` to each source and returns each source's `has_more`; there is no global ranking across sources (relevance scores of two catalogs are not comparable). Each source returns items ordered by its own relevance; the core keeps that order.

### B.3 Internal sources

An internal service becomes a source by exposing `service.music`:

```js
// server/services/radio/index.js — sketch
return {
  start, stop,
  music: {
    getDescriptor: () => ({ name: 'Radios', media_types: ['radio'], streamable: true, native: false }),
    search: ({ query, limit, offset }) => radioHandler.search(query, limit, offset),
    resolve: ({ uri }) => radioHandler.resolve(uri),
    getImage: (key) => radioHandler.getImage(key),
  },
};
```

The same `service.music` object carries `play(device, request)` when the service owns media players (C.3) — a service can be a source, a player host, or both. `server/lib/music/` lists sources the way `weather.get.js` lists providers: `stateManager.getAllKeys('service')`, keep those with `service.music && typeof service.music.search === 'function'`, sorted by key.

**Reference source shipped with the feature: `radio`** (new internal service, phase 1). The user saves web radios (name, stream URL, optional logo uploaded as a dashboard asset) from its integration page; search matches names; `resolve` returns the one URL. It has no network dependency, exercises the whole chain (search → play → now playing → widget) on Sonos and Cast from day one, and covers "joue FIP dans la cuisine" — the single most common voice request in the forum thread. A public directory (radio-browser.info) is a phase-3 option of that service, not a core concern.

### B.4 External sources (store integrations)

**Manifest — new capability field `music_source`** (added to `CAPABILITY_MANIFEST_FIELDS`, usable by every type: a `device` integration that also owns players, or a `provider` integration that is a catalog only):

```json
{
  "manifest_version": 1,
  "type": "provider",
  "name": "Navidrome",
  "music_source": {
    "media_types": ["track", "album", "artist", "playlist"],
    "streamable": true,
    "native": false
  }
}
```

`name` defaults to the manifest `name`. The source key is the integration's selector (`ext-navidrome`). Validation errors name their rule, as `validateManifest` does today.

**WebSocket messages** (core → integration, `message_id` + `external-integration.command-result` ack, the `<domain>.<action>` naming convention):

| Type | Payload | `command-result` `data` | Timeout |
|---|---|---|---|
| `external-integration.music.search` | `{ message_id, query, types, limit, offset, language }` | `{ items, has_more }` | 8 s |
| `external-integration.music.resolve` | `{ message_id, uri, player: { selector, service } }` | `{ items }` | 8 s |
| `external-integration.music.get-image` | `{ message_id, image_key }` | `{ image }` (raw base64, no data-URI prefix) | 15 s |

Every answer goes through a core normalizer (whitelisted keys, lengths, URI source check, `url` scheme check, item cap); an invalid item is dropped and logged at warn level with the integration selector, the rest of the answer is kept. An integration that does not know a type ignores it (existing protocol rule), so old SDKs time out cleanly.

**SDK (`@gladysassistant/integration-sdk`)** — three handlers, mirroring `onWeatherGet` / `onWidgetGetImage`:

```js
gladys.onMusicSearch(async ({ query, types, limit, offset, language }) => {
  const r = await subsonic.search3(query, { limit, offset });
  return {
    items: r.albums.map((a) => ({
      uri: gladys.musicUri('album', a.id), // → 'ext-navidrome://album/<id>'
      type: 'album',
      title: a.name,
      subtitle: `${a.artist} · ${a.year}`,
      artist: a.artist,
      image: `cover-${a.coverArt}`,
      playable: true,
    })),
    has_more: r.albums.length === limit,
  };
});

gladys.onMusicResolve(async ({ uri }) => ({
  items: (await subsonic.tracksOf(uri)).map((t) => ({
    url: subsonic.streamUrl(t.id), title: t.title, artist: t.artist,
    album: t.album, duration: t.duration, image: `cover-${t.coverArt}`,
  })),
}));

gladys.onMusicGetImage(async (key) =>
  (await sharp(await subsonic.coverArt(key.slice(6))).resize(300).webp().toBuffer()).toString('base64'),
);
```

SDK side each handler is one `on…` setter, one `case` in `_handleMessage` dispatching to `_runCommand`, typings, a `gladys.fake.music*` trigger in `lib/testing.js` and a dev-mode validator (the `widget-content.js` pattern).

**Spec placement.** At implementation time, the external part of this section moves to `docs/specs/external-integrations/capabilities/music-source.md` (one file per capability, `docs/specs/external-integrations/README.md`), with a row in the capability table of `provider-type.md`, and this file links to it — in the same diff, per the living-spec rule.

### B.5 Images

The images of search results and of "now playing" follow the widget-image doctrine verbatim: **keys only, bytes on demand, served from Gladys' origin.**

**A key belongs to whoever named the media.**

- Source images: `GET /api/v1/music/source/:source/image/:image_key` → `music.getImage(key)` on that source. This covers search results **and** the artwork of a now playing record that carries a `uri` (E.7): the record's `image` is then a key of the URI's source, so a player relaying a Navidrome album just echoes the key it received and keeps no bytes — an MQTT player included.
- Player images: `GET /api/v1/music/player/:device_selector/image/:image_key` → `music.getImage(key)` on **the service owning the device**, only for a record **without** `uri` (content started outside Gladys: the Spotify app, a line-in, the Sonos app). An internal Sonos service fetches the `AlbumArtUri` from the speaker it already talks to; an external player answers `music.get-image`. A player that never reports such content implements no image handler.
- Same validation, limits, error codes and cache policy as `dashboard-widgets.md` §6 (≤ 300 KB, ≤ 4096 px, 1 h cache per `(service, key)`, ≤ 100 entries per service LRU, ≤ 4 in flight per service, concurrent requests for one key share one command). Keys are immutable for the life of the cache.
- Allowlist: a source key is served only if it appeared in that source's last 200 search/resolve items or in a current now playing record whose `uri` belongs to that source; a player key only if it is in that device's current record. Any other key `404`s without a command sent.

## C. "Play this" (Q2)

### C.1 The play pipeline

One core function, `gladys.music.play({ device, uri, enqueue = 'replace', volume })`, used by the HTTP route, the scene action, the AI tool and the widget — the **only** way to make a player play a media:

1. **Player.** `device` is a media player (A.1), else `400 MUSIC_NOT_A_PLAYER`.
2. **Parse** the URI (A.4), else `400 MUSIC_INVALID_URI`. Find the source by key, else `404 MUSIC_SOURCE_NOT_FOUND`.
3. **Compatibility** (C.2), else `409 MUSIC_PLAYER_INCOMPATIBLE` with the reason (`not_streamable`: the source has no stream and the player is not its own; `no_url_playback`: the player has no `play-media`).
4. **Resolve.** *Native* route: nothing to resolve. *Stream* route: `source.music.resolve({ uri, player })` → `PlayableItem[]` (empty → `404 MUSIC_MEDIA_NOT_FOUND`). A direct `http(s)` URI resolves to one item `{ url: uri }` without any call.
5. **Volume.** If `volume` (0–100) is given and the player has a `volume` feature, `device.setValue(volumeFeature, volume)` first — the existing path, no new contract.
6. **Request.** Build the play request (C.3) and send it to **the service owning the device** on the music transport of its kind (internal call, `external-integration.music.play`, MQTT `…/music/play`).
7. **Answer** `{ device, uri, route, items_count }` — never the URLs.

**Why not `setValue`.** Revision 1 sent the URI as the value of the `play-media` feature and the expansion in `setValue`'s 4th argument, `options`. That is a record riding in the generic device path: every transport of that path would have needed a side channel (an `options` field in `external-integration.device.set-value`, a JSON exception on the MQTT state topic), whose shape depends on the feature type — the structured payload `camera-ptz-control.md` rejected, and the command-side twin of the now playing problem (E.1). "Play this" is a core command, like "now playing" is a core record: one entry point, a record built and validated by the core, typed transports. The generic path keeps carrying values only.

### C.2 Choosing a compatible player

A media player `P` can play a media `M` of source `S` according to two facts only — does `P` have `play-media`, and does it belong to `S`'s service:

| | `S.native` and `P` belongs to `S`'s service | otherwise |
|---|---|---|
| `P` has `play-media` | **native** (wins) | **stream** if `S.streamable` or `M` is a direct URL, else incompatible |
| `P` has no `play-media` | **native** | incompatible |

Native is Matter's `ContentSearch` (the device plays its own catalog), stream is Matter's `UrlPlayback`. Native wins when both hold: a Lyrion player playing the Lyrion library natively keeps gapless playback and the server's own queue.

This is the whole answer to **"a source that only plays at home" (Spotify Connect)**: the Spotify integration declares `music_source` with `streamable: false, native: true` and publishes its Connect devices as music devices **without** `play-media`. Spotify media are compatible with those devices only; a Navidrome album is compatible with the Sonos (it has `play-media`) and not with the Spotify devices (they have none). Nothing Spotify-specific exists in the core, and no capability flag hides in a feature: the source says `native` once, in its descriptor. The same rule covers Echo devices driven by an Alexa integration, Plexamp, Music Assistant (`streamable: false, native: true`: every request goes to its own players).

- A device of the source's service that cannot play a given media natively answers the request with an error (C.3): the core does not model per-device exceptions to `native`.
- One speaker exposed by two integrations (a Sonos speaker seen by the Sonos service and as a Spotify Connect device) is two music devices; the speaker picker shows both, each with what it can play.

When only a **room** (or nothing) is given — AI and scenes —, `gladys.music.choosePlayer({ uri, room?, user? })`:

1. candidates = players compatible with the media (above); if a room is given, those in that room;
2. none → `MUSIC_NO_COMPATIBLE_PLAYER` (the AI explains which players exist and why they do not fit);
3. one → it;
4. several → the one currently `PLAYING` (the user means "this speaker"), then the one whose now playing record changed most recently (E), then **ambiguous**: the AI asks back with the list, the scene editor never gets there (a scene always names a device).

### C.3 What a player receives

**The play request** — a record built by the core, identical on every transport:

```json
{
  "uri": "ext-navidrome://album/al-42",
  "route": "stream",
  "enqueue": "replace",
  "item": { "type": "album", "title": "\"Heroes\"", "subtitle": "David Bowie · 1977", "image": "cover-al-42" },
  "items": [
    { "url": "http://…", "mime_type": "audio/flac", "title": "Beauty and the Beast", "artist": "David Bowie",
      "album": "\"Heroes\"", "duration": 216, "image": "cover-al-42", "uri": "ext-navidrome://track/tr-1" }
  ]
}
```

- `route: "native"` carries no `items`: the player's own service knows what its URI means.
- `item` is the `MediaItem` the user picked when the caller has it (widget, AI), else built from the first resolved item.
- `image` keys are the source's (B.5); a player echoes them in its now playing record (E.7) and keeps no bytes.
- The record evolves by adding optional fields only; players ignore the fields they do not know.

**Enqueue belongs to every player's contract.** A player that plays an album has a queue — its own (Sonos, Cast, MPD, LinkPlay, LMS) or the one its integration keeps to feed the items one by one (AirPlay, a DLNA renderer without `SetNextAVTransportURI`). Inserting after the current item (`next`) or appending (`add`) is therefore always possible. A player whose protocol truly cannot (a cloud-driven speaker) answers with an error, shown by the widget and spoken by the AI — no capability flag announces it in advance.

**Transports** — one per kind of integration, the mirror of the now playing transports (E.4):

| Player | Receives | Acknowledgement |
|---|---|---|
| Internal service | `service.music.play(device, request)` | the returned promise |
| External integration | WebSocket `external-integration.music.play` `{ message_id, device: { external_id, selector, params }, request }`; SDK `gladys.onMusicPlay(async (device, request) => …)` | `external-integration.command-result`, 10 s |
| MQTT device | topic `gladys/device/{device_external_id}/music/play`, payload = the request as JSON, QoS 1, not retained | none (MQTT has no return channel) |

- The MQTT state topics are untouched: `gladys/device/…/feature/…/state` keeps carrying `value.toString()` for every feature. The music topic is a documented message on a topic of its own, like the inbound now playing topic (E.4).
- `external-integration.device.set-value` is untouched (no `options` field). An integration that does not know `music.play` ignores it (existing protocol rule) and the core reports a timeout.
- Sonos: `route: stream` → `RemoveAllTracksFromQueue` (`replace`) or `AddURIToQueue` (`EnqueuedAsNext` for `next`), then `Play`. `route: native` does not occur (Sonos is not a source in v1).

A player MUST acknowledge once playback has been **accepted** by the speaker (queue loaded), not when the track ends: an album longer than the timeout is never an error. Failure → `success: false` with a human-readable `error`.

### C.4 HTTP API

All routes are user-authenticated (standard API auth, any role — see open question 4).

| Route | Body / query | Answer |
|---|---|---|
| `GET /api/v1/music/source` | — | `[{ source, name, media_types, streamable, native, status }]`; `status` = `ok` / `not_connected` (external integration down) |
| `GET /api/v1/music/search` | `query`, `types` (csv), `source` (csv, default all), `limit`, `offset` | `{ sources: [{ source, name, status: 'ok' \| 'timeout' \| 'error', error?, items, has_more }] }` — fan-out in parallel, 8 s per source, a slow source never blocks the others' results |
| `GET /api/v1/music/player` | `uri` (optional), `room` (optional) | music devices with `{ device_selector, name, room, play_media, native_sources, playback_state, now_playing, compatible?, reason? }` (`play_media`: has the feature; `native_sources`: keys of the sources that play natively on it); with `uri`, each says whether it can play it (the speaker picker of H.3) |
| `GET /api/v1/music/player/:device_selector` | — | the same object for one device — what the widget loads (H.1) |
| `POST /api/v1/music/play` | `{ device_selector, uri, enqueue?, volume? }` | C.1; `202` once the player acknowledged |
| `GET /api/v1/music/source/:source/image/:image_key` | — | B.5 |
| `GET /api/v1/music/player/:device_selector/image/:image_key` | — | B.5 |

**No generic value route.** `play-media` is not written through `POST /api/v1/device_feature/:selector/value` or the `device.set-value` scene action (A.3): generic clients (Node-RED, HTTP scripts) use `POST /api/v1/music/play`, scenes use the `music.play` action (H.4). One entry point, one validation.

### C.5 Fixture — a player device (external WiiM integration)

```json
{
  "name": "WiiM Pro — Kitchen",
  "external_id": "ext:wiim:a1b2c3",
  "features": [
    { "name": "Play", "external_id": "ext:wiim:a1b2c3:play", "category": "music", "type": "play",
      "min": 1, "max": 1, "read_only": false, "has_feedback": false, "keep_history": false },
    { "name": "Pause", "external_id": "ext:wiim:a1b2c3:pause", "category": "music", "type": "pause",
      "min": 1, "max": 1, "read_only": false, "has_feedback": false, "keep_history": false },
    { "name": "Next", "external_id": "ext:wiim:a1b2c3:next", "category": "music", "type": "next",
      "min": 1, "max": 1, "read_only": false, "has_feedback": false, "keep_history": false },
    { "name": "Previous", "external_id": "ext:wiim:a1b2c3:previous", "category": "music", "type": "previous",
      "min": 1, "max": 1, "read_only": false, "has_feedback": false, "keep_history": false },
    { "name": "Volume", "external_id": "ext:wiim:a1b2c3:volume", "category": "music", "type": "volume",
      "min": 0, "max": 100, "read_only": false, "has_feedback": true, "keep_history": false },
    { "name": "Playback state", "external_id": "ext:wiim:a1b2c3:playback-state", "category": "music",
      "type": "playback_state", "min": 0, "max": 3, "read_only": true, "has_feedback": false, "keep_history": false },
    { "name": "Notification", "external_id": "ext:wiim:a1b2c3:play-notification", "category": "music",
      "type": "play_notification", "min": 1, "max": 1, "read_only": false, "has_feedback": false, "keep_history": false },
    { "name": "Play media", "external_id": "ext:wiim:a1b2c3:play-media", "category": "music", "type": "play-media",
      "read_only": false, "has_feedback": false, "keep_history": false }
  ]
}
```

`play-media` without options: the WiiM plays media by URL. What it plays is not a feature: the integration publishes it with `publishNowPlaying` (E.4).

### C.6 Fixture — search

`GET /api/v1/music/search?query=heroes&types=album,track&limit=2`

```json
{
  "sources": [
    {
      "source": "ext-navidrome",
      "name": "Navidrome",
      "status": "ok",
      "has_more": true,
      "items": [
        { "uri": "ext-navidrome://album/al-42", "type": "album", "title": "\"Heroes\"",
          "subtitle": "David Bowie · 1977", "artist": "David Bowie", "image": "cover-al-42", "playable": true },
        { "uri": "ext-navidrome://track/tr-7", "type": "track", "title": "\"Heroes\"",
          "subtitle": "David Bowie — \"Heroes\"", "artist": "David Bowie", "album": "\"Heroes\"",
          "duration": 367, "image": "cover-al-42", "playable": true }
      ]
    },
    { "source": "radio", "name": "Radios", "status": "ok", "has_more": false, "items": [] },
    { "source": "ext-spotify", "name": "Spotify", "status": "timeout", "has_more": false, "items": [] }
  ]
}
```

### C.7 Fixture — "play this"

`POST /api/v1/music/play`

```json
{ "device_selector": "wiim-pro-kitchen", "uri": "ext-navidrome://album/al-42", "enqueue": "replace", "volume": 30 }
```

What the WiiM integration receives over WebSocket:

```json
{
  "type": "external-integration.music.play",
  "payload": {
    "message_id": "4f0c…",
    "device": { "external_id": "ext:wiim:a1b2c3", "selector": "wiim-pro-kitchen", "params": [] },
    "request": {
      "uri": "ext-navidrome://album/al-42",
      "route": "stream",
      "enqueue": "replace",
      "item": { "type": "album", "title": "\"Heroes\"", "subtitle": "David Bowie · 1977", "image": "cover-al-42" },
      "items": [
        { "url": "http://192.168.1.20:4533/rest/stream?id=tr-1&…", "mime_type": "audio/flac",
          "title": "Beauty and the Beast", "artist": "David Bowie", "album": "\"Heroes\"",
          "duration": 216, "image": "cover-al-42", "uri": "ext-navidrome://track/tr-1" }
      ]
    }
  }
}
```

What an MQTT player receives for the same request: topic `gladys/device/mqtt:kitchen-esp32/music/play`, payload = the `request` object above.

## D. `play-media` vs `play_notification` (Q3)

**Decision: two features, two semantics. `play_notification` is not extended.**

| | `play_notification` (existing) | `play-media` (new) |
|---|---|---|
| Purpose | a short sound that **interrupts** (TTS, doorbell, alarm chime) | a media that **replaces** (or joins) what plays |
| Transport | `setValue` with one audio URL — a value | the play request record on the music transports (C.3) |
| After it ends | the previous playback **resumes** | the queue continues |
| Queue | untouched | replaced or extended (`enqueue`) |
| Volume option | temporary, restored after | persistent (`volume` = a plain `volume` write) |
| Typical caller | `music.play-notification` scene action, alarm | AI "play X", `music.play` scene action, widget |

Merging them (Home Assistant's `play_media(…, announce: true)`) would make every caller pass a mode flag, and every player implement both behaviors behind one entry, while `play_notification` already exists and works as a plain value. Keeping them apart keeps each contract one sentence long.

**The resume contract, made explicit** (it was implicit until now):

- A player that has both features SHOULD restore after a notification what `play-media` was playing: same queue, same item, position within a few seconds, same volume, same paused/playing state. Sonos does it through `PlayNotification`. Google Cast currently does **not** (launching the default receiver kills the running app): phase 2 of this spec makes the Cast service snapshot its media session (queue, current item, position) before the notification and reload it afterwards when the session was Gladys' own; a session owned by another app (YouTube, Spotify) is not restorable and is documented as such.
- A notification sent while `playback_state` is `PAUSED`/`STOPPED` leaves the player in that state afterwards.
- A play request received while a notification plays cancels the pending resume: the newest explicit request wins.
- **A notification never changes the now playing record.** The core does not write it when it sends a notification, and a player does not publish the notification as its now playing (E.6): the record keeps describing the media the notification interrupted.

## E. "Now playing" (Q4)

### E.1 A core record, not a feature state

**Decision — the maintainer's model ([forum 11260/4](https://community.gladysassistant.com/t/11260/4)): what a player is playing is a record owned by the core, held in memory by `server/lib/music/`, with one entry point, a strict schema and typed transports in and out.** It is not a feature: no `now-playing` type, nothing in `last_value_string`, no branch in `device.newStateEvent`.

Why revision 1's JSON document in a text feature's `last_value_string` was the wrong model: `last_value_string` holds an **opaque value** — a text, the option picked in a select, a camera image. A document whose fields the core must understand is a record, not a value, and storing it as a string costs more than looks:

- **The format cannot be enforced.** Validation would be a special branch in `newStateEvent`, a generic path that only logs what it drops, after `POST /state` has already answered `200` (Precedents): the integration author never sees the error, and every integration ends up with its own dialect.
- **Every generic consumer sees a blob.** The `device.new-state` trigger compares strings; the device page, the MQTT state topic and the AI's `device.get-state` would show raw JSON unless each grew a special case.
- **No versioning**: a new field becomes every integration's and the front's problem.
- **A duplicate**: the document's `state` repeated `playback_state` — two sources of truth for the transport state, bound to diverge.
- **A precedent**: the next record (a vacuum map, a queue, a schedule) would do the same.

It also broke rule 5 of `device-feature-categories.md` (one atomic value per feature).

### E.2 The entry point

```js
await gladys.music.setNowPlaying(device, nowPlaying); // nowPlaying: object | null
gladys.music.getNowPlaying(device); // object | null
```

Every rule lives here, once:

1. **Music devices only** (A.1): a device with at least one `music` feature, else `BadParameters`. A player does not need `play-media` to report what it plays.
2. **Ownership**, checked by each transport before the call: an external integration publishes for its own devices only (`404` otherwise, like `POST /camera/image`), the MQTT handler for `mqtt` devices only, an internal service for its own.
3. **`null` means nothing is loaded.**
4. **Strict schema** (Joi, E.3, `convert: false`, unknown keys refused). A violation is a `BadParameters` returned to the caller with the path and the rule — `now_playing.position: must be less than or equal to ref:duration` — never a warning in a log.
5. **`position_updated_at` is set by the core** to the reception time when the record has a `position` and no `position_updated_at`.
6. **Nothing changed, nothing sent**: a record deep-equal to the stored one emits no event, so the replay of E.5 does not make the widget blink.
7. **A notification does not touch it**: the core never calls `setNowPlaying` when it plays a notification (D, E.6).

On change the core emits the WebSocket event **`music.now-playing-updated`** `{ device: <device selector>, now_playing }` to all users (`<domain>.<kebab-event>`, like `thermostat.config-updated`; forwarded through Gladys Plus like every event).

### E.3 The record

```json
{
  "title": "Beauty and the Beast",
  "artist": "David Bowie",
  "album": "\"Heroes\"",
  "image": "cover-al-42",
  "duration": 216,
  "position": 37,
  "position_updated_at": "2026-10-15T18:42:07.120Z",
  "uri": "ext-navidrome://track/tr-1",
  "queue_index": 0,
  "queue_length": 10
}
```

| Field | Rule |
|---|---|
| `title` | **required**, string 1–200. Nothing to name → publish `null` |
| `artist`, `album` | optional, string ≤ 200 |
| `image` | optional, image key `^[a-z0-9][a-z0-9-]{0,63}$`, owned by the URI's source, or by the player's service when there is no `uri` (E.7) |
| `duration` | optional, seconds, > 0; absent for a live stream |
| `position` | optional, seconds, ≥ 0, ≤ `duration` when `duration` is present |
| `position_updated_at` | optional ISO 8601 date, at most 5 s in the future, only with `position`; set by the core when `position` comes without it |
| `uri` | optional, a **source** media URI (A.4), never an `http(s)` URL (it may carry stream tokens, G.3). Present when the player knows which media it plays: Gladys started it, or it is native content of the player's own source |
| `queue_index`, `queue_length` | optional integers, `0 ≤ queue_index < queue_length` |

- **No `state`**: `playback_state` is the only transport state, with `STOPPED` and `BUFFERING` added (A.2). Scenes keep triggering on it.
- **No `source`, no `type`**: they are the first two segments of `uri` — the same "one source of truth" rule.
- Matter alignment: `duration`, `position` and `position_updated_at` are the Media Playback cluster's `Duration` and `SampledPosition { updatedAt, position }`. Matter has no title, artist or artwork attribute; the field names follow Home Assistant (`media_title`, `media_artist`, `media_album_name`).
- The field limits bound the record (about 1.5 KB serialized).

### E.4 Transports

| From | In | On error |
|---|---|---|
| Internal service | `await gladys.music.setNowPlaying(device, record)` | the thrown `BadParameters` |
| External integration | `POST /api/integration/v1/music/now_playing` `{ device_external_id, now_playing }` → `200 { success: true }`, **awaited** (unlike `POST /state`). SDK: `await gladys.publishNowPlaying(deviceExternalId, record \| null)` | `400 BAD_REQUEST` naming the path and the rule; `404` if the device is not the integration's; `429` beyond 60 records/min per device, a counter of its own outside `POST /state` |
| MQTT | `gladys/master/device/{device_external_id}/music/now_playing`, payload = the record as JSON, or `null` (or an empty payload) for nothing, published **retained** | logged at `warn` with the rule, and published on `gladys/device/{device_external_id}/music/now_playing/error` (payload = the message), so a bridge author can subscribe to their errors |

Out: the `music.now-playing-updated` event (E.2), `GET /api/v1/music/player[/:device_selector]` (C.4) and the AI's `device.get-state` (H.5). The record is never part of the device object (`GET /api/v1/device`): it is not a feature state.

The shape mirrors `POST /api/integration/v1/camera/image` → `gladys.device.camera.setImage` (Precedents): `device_external_id` + payload, ownership `404`, a rate limit of its own. The MQTT handler (`handleGladysMessage`) routes `gladys/master/device/:id/music/…` before its feature branch — today such a topic would be parsed as a numeric state of a feature named `now_playing` and dropped, so an older core ignores it harmlessly.

**SDK — `publishNowPlaying(deviceExternalId, record)`:**

- stamps `position_updated_at` itself when the record has a `position` without it: the integration knows when it read the position, and a replay must not move it backwards;
- sends records only for devices created in Gladys (it knows them from `GET /device` and `device-created`) and holds the others until their device is created — an integration publishes for every device it discovers;
- keeps the last record **accepted** for each device and **replays them after every re-authentication**, before emitting `connected`: every external integration gets the restart behavior of E.5 without a line of code. A record refused with `400` is not kept;
- throws `GladysApiError` on `400`, `404` and `429`, like the other publishers.

### E.5 Memory only: restarts and other lifecycle events

No table, no migration, no `last_value_string`. The data is volatile by nature: after a Gladys restart, a persisted value would almost always be wrong — the player kept playing or stopped meanwhile. Each transport already has the right way to fill the gap:

| Player | After a Gladys restart |
|---|---|
| Sonos | the UPnP subscription made at start receives the speaker's initial event (full `AVTransport` state, `CurrentTrackMetaData` included); the service also calls `GetPositionInfo` at start, which gives the position and makes it independent of that first event |
| Google Cast | a media status request at connection — for which the service must keep one connection per device, which it does not today (F.2) |
| External integrations | the SDK replays the last record of each device at reconnection (E.4) |
| MQTT | the broker re-delivers the retained message when the core subscribes |

The record is cleared (event with `null`) when:

- the device is destroyed;
- the internal service owning the device stops;
- the external integration's WebSocket closes — the moment the core marks it `DEGRADED`. A widget must not show a track for a player nobody can reach; the replay restores the record at reconnection.

The precedent is the external integrations' connection status, "kept in memory only (lost at restart, republished by the integration)" (Precedents).

### E.6 What a player must do

- **Publish on changes only**: track change, seek, pause, resume, stop — never every second. On pause and on resume, republish with the current `position`: the front extrapolates the position only while `playback_state` is `PLAYING`, from the last `position` and `position_updated_at`.
- **Never publish a notification as the now playing.** The core sees a notification start, not end; only the player knows when the previous media is back. A player whose protocol reports the notification as a track (Sonos `AVTransport` events during `PlayNotification`) filters it out until the previous media is restored.
- Set `uri` only for a media a source can name, and `image` according to E.7.
- Publish `null` when nothing is loaded anymore (queue cleared, input switched to a TV with no metadata).

### E.7 Artwork: the key belongs to whoever named the media

- **With a `uri`**: `image` is a key of the URI's **source**, served by `GET /api/v1/music/source/:source/image/:key` (B.5). A player relaying a source item echoes the `image` of the `PlayableItem` it received (C.3): it keeps no bytes and implements no image handler — an MQTT player gets artwork too.
- **Without a `uri`** (content started elsewhere): `image` is a key of the **player's own service**, served by `GET /api/v1/music/player/:device_selector/image/:key`.
- The front picks the route from the presence of `uri`. This settles revision 1's open question 9.

### E.8 Evolution and limits

- **Versioning.** The record evolves by **adding optional fields** only. The core is strict on what it receives; the front and the AI ignore what they do not know. A field added in Gladys X.Y is refused by an older core, so an integration that sends it raises the `gladys_version` range of its manifest — the store already marks an integration incompatible with an older Gladys and disables its installation.
- **No history** of what played: a listening history would be a separate feature.
- **Scenes** trigger on `playback_state`, as today. A `music.track-changed` trigger with typed fields (title, artist, source) can be built later on the record (phase 3); it compares fields, never a JSON string.

### E.9 Fixtures

What an external integration sends, and the error it gets back:

```
POST /api/integration/v1/music/now_playing
{ "device_external_id": "ext:wiim:a1b2c3",
  "now_playing": { "title": "Beauty and the Beast", "artist": "David Bowie", "album": "\"Heroes\"",
    "image": "cover-al-42", "duration": 216, "position": 0, "position_updated_at": "2026-10-15T18:41:30.000Z",
    "uri": "ext-navidrome://track/tr-1", "queue_index": 0, "queue_length": 10 } }
→ 200 { "success": true }

POST /api/integration/v1/music/now_playing
{ "device_external_id": "ext:wiim:a1b2c3", "now_playing": { "title": "\"Heroes\"", "position": 400, "duration": 367 } }
→ 400 { "status": 400, "code": "BAD_REQUEST", "message": "now_playing.position: must be less than or equal to ref:duration" }
```

What the frontend receives:

```json
{
  "type": "music.now-playing-updated",
  "payload": {
    "device": "wiim-pro-kitchen",
    "now_playing": { "title": "Beauty and the Beast", "artist": "David Bowie", "album": "\"Heroes\"",
      "image": "cover-al-42", "duration": 216, "position": 0, "position_updated_at": "2026-10-15T18:41:30.000Z",
      "uri": "ext-navidrome://track/tr-1", "queue_index": 0, "queue_length": 10 }
  }
}
```

## F. Integration contracts — summary and migration

### F.1 What changes in the core

- `server/utils/constants.js`: `PLAY_MEDIA` in `DEVICE_FEATURE_TYPES.MUSIC` (with the inline scope comment rule 8 requires), `MUSIC_PLAYBACK_STATE` extended, `MUSIC_ENQUEUE`, `MUSIC_MEDIA_TYPES`, `ACTIONS.MUSIC.PLAY`, `AI_CHAT_TOOL_CATEGORIES.MUSIC`, WebSocket types `music.now-playing-updated` and `external-integration.music.{search,resolve,get-image,play}`.
- `server/lib/music/` (new): source registry, `search`, `play`, `choosePlayer`, the now playing store (`setNowPlaying`, `getNowPlaying`, clearing hooks), the Joi schemas of the records, image cache, normalizers.
- `server/lib/device/device.setValue.js`: refuses a `play-media` feature (A.3) — a guard, no new path.
- `server/utils/normalizeSupportedOptions.js`: refuses options on `play-media`.
- `server/lib/scene/scene.actions.js`: new `music.play` action (H.4). `device.set-value` is unchanged.
- `server/lib/external-integration/`: `music_source` manifest field; `music.*` messages on the proxy service; `POST /api/integration/v1/music/now_playing`; clearing of the integration's records when its WebSocket closes.
- `server/services/mqtt/`: route `gladys/master/device/:id/music/now_playing` in `handleGladysMessage`; publish the play request on `gladys/device/:id/music/play`.
- `server/services/mcp/`: music tools, `device.get-state` for music devices (H.5).
- **Unchanged:** `device.newStateEvent`, `device.saveStringState`, the `external-integration.device.set-value` message, the MQTT state topics, `POST /state`.

### F.2 Internal players

| Service | Phase | Change |
|---|---|---|
| `sonos` | 1 | `play-media` feature; `service.music.play`: `RemoveAllTracksFromQueue` (`replace`) or `AddURIToQueue` (`EnqueuedAsNext` for `next`), then `Play`. Now playing from the `AVTransport` event's `CurrentTrackMetaData` it already receives (`Title`, `Artist`, `Album`, `Duration`), the position from `GetPositionInfo` on each transport change and at start (E.5); for a media Gladys started, the service maps the stream URL of the current track back to the `PlayableItem` it sent (its `uri` and source `image`), otherwise `AlbumArtUri` → a player image key `art-<hash>` served by its `getImage`. `playback_state` gains `STOPPED`/`BUFFERING` from `TransportState`. Filters the notification track out of the now playing (E.6). Nothing removed |
| `google-cast` | 1 | **Keeps one connection per device** (new: today a client is created per notification) to read the media status — needed for the now playing at start (E.5) and for the transport features; `play-media` feature, `service.music.play` through the default receiver's `queueLoad`/`queueInsert`; `play`/`pause`/`next`/`previous`/`volume`/`playback_state` (the Nest Mini finally appears in the Music widget); phase 2: resume after notification (D) |
| `airplay` | 3 | `play-media` feature; `service.music.play` plays the items one by one through its ffmpeg pipeline, without the 5-minute notification cap, and keeps the queue itself (C.3); `pause` (stops the pipeline); now playing from what it sends. AirPlay has no queue of its own and no remote metadata to read |

Existing devices gain the new feature **on the next scan**: `device.create` matches existing features by `external_id` (`server/lib/device/device.create.js`), so a re-created Sonos device keeps its features, selectors and the scenes that use them, and gets `play-media`. No `device.migrate` entry: no feature is renamed or removed, and no dashboard box field references a new feature (the Music box references a device).

### F.3 MQTT

- Generic devices can declare `play-media` in the MQTT setup UI (catalog entry from the i18n key): no option editor, no min/max; the catalog preview explains that requests arrive on `gladys/device/{device_external_id}/music/play`.
- Outbound: the play request on `gladys/device/{device_external_id}/music/play` (C.3). The state topics are unchanged.
- Inbound: the now playing record on `gladys/master/device/{device_external_id}/music/now_playing`, retained; refusals on `gladys/device/{device_external_id}/music/now_playing/error` (E.4). The MQTT device page documents both topics.
- The play request carries stream URLs, tokens included: publishing them on a broker is the MQTT user's trust boundary (G.3).

### F.4 External integrations

- **Existing integrations keep working**: `music_source` is optional, every new message is one an old SDK ignores, no message changes shape.
- **To become a player by URL:** publish a `play-media` feature and implement `onMusicPlay(device, request)`.
- **To report what plays** (any music device): call `publishNowPlaying(deviceExternalId, record | null)` on changes (E.6); implement `onMusicGetImage` only for content without `uri` (E.7).
- **To become a source:** declare `music_source`, implement `onMusicSearch`, `onMusicResolve` (if `streamable`), `onMusicGetImage`; with `native: true`, implement `onMusicPlay` for its own devices (`route: "native"`).
- SDK (`@gladysassistant/integration-sdk`): `onMusicSearch`, `onMusicResolve`, `onMusicGetImage`, `onMusicPlay`, `publishNowPlaying` (with the stamping, holding and replay of E.4), typings, `gladys.fake.music*` triggers in `lib/testing.js`, a dev-mode validator for the records it receives and sends.
- Credentials (Navidrome password, Spotify OAuth) stay in the integration's `config_schema` (`secret`, `oauth2`) — the core never sees them (G.3).

### F.5 Backward compatibility

- No type, value, route, message or event changes meaning. `MUSIC_PLAYBACK_STATE` is extended append-only; a scene condition `playback_state = 0` still means paused.
- A music device with no now playing record and no `play-media` looks exactly as today in the widget and in scenes; the widget hides the search and artwork areas for it.
- Existing `music` boxes (`{ type: 'music', device }`) render with the new defaults (H.1) without a dashboard migration.
- `play_notification` gains nothing mandatory; the resume behavior (D) is a SHOULD.
- Out of scope, noted on the way: `external-integration.device.set-value` drops `setValue`'s `options`, so an external speaker never receives the notification volume today. Revision 1 fixed it as a side effect of its `options` field; it is now a separate fix.

## G. Security

### G.1 Arbitrary URLs

- The core **never fetches** a media or stream URL: it hands URLs to players. The only bytes it fetches are images, and only through the owning service by key (B.5) — the widget-image doctrine, so no SSRF from the core.
- A direct `http(s)` URI makes **the speaker** fetch it. Speakers already fetch arbitrary URLs through `play_notification`; the risk is a LAN device requesting a LAN URL, with an audio decoder as the only consumer of the response. Schemes other than `http`/`https` (`file:`, `smb:`, `rtsp:`…) are refused at parse time.
- The **AI tool never accepts a URL** (`uri` must come from a search result in the same conversation turn or name a source key): web pages fetched through `web.fetch` must not be able to make Gladys play an attacker-chosen stream by prompt injection. Direct URLs remain available to users through scenes and the HTTP API.

### G.2 Untrusted payloads

- Search/resolve answers and images from integrations are normalized and bounded (B.4, B.5): whitelisted keys, lengths, item caps, URI source check (a source cannot vend another source's URIs), image magic numbers and pixel bounds. Now playing records are refused, not repaired, when they break the schema (E.2).
- Text is rendered as text, never HTML (titles come from file tags anybody can write).

### G.3 Sources that need credentials

- Credentials live in the integration (config `secret`, `oauth2`, `account_link`), never in the core, never in a URI (`<id>` must not carry credentials).
- Resolved stream URLs often embed tokens (Subsonic `t`/`s` salt, Plex `X-Plex-Token`). They travel core → player only: **no API returns a `PlayableItem`**, the play answer carries a count, a now playing record carries a source URI and the schema refuses an `http(s)` one (E.3), and the core does not log `items[].url` (it logs the URI and the item count). MQTT players do receive the URLs — publishing them on a broker is the MQTT user's trust boundary, documented in F.3.
- **Per-user accounts** (two Spotify users in a household) are out of v1: a source is house-wide. The calendar type's per-user `account_schema` is the precedent if needed (open question 5).

### G.4 Authorization and abuse

- All routes require an authenticated user. Search is rate-limited per user (10/s) and fans out with per-source timeouts; images obey the per-service in-flight cap.
- Any user who can use the dashboard can play music, as any user can press the existing play button today (open question 4 asks whether to restrict sources to admins).

## H. Surfaces

### H.1 The Music widget, adapted

`front/src/components/boxs/music/` is extended, not replaced (maintainer's request). Mockups: `docs/specs/music/` — PNGs rendered from `music/mockups.html`, which uses the real Gladys theme (`@gladysassistant/theme-optimized@2.1.0`, the version `front/package.json` pins, loaded from jsDelivr) and, for the dark-mode frame, the app's own `front/src/style/dark-mode.css`. Each `.frame` of the HTML is one PNG.

**Box config** — additive fields, lazily defaulted:

```json
{ "type": "music", "device": "living-room-sonos", "display": "full", "player_switch": true, "search": true }
```

- `device` (existing) — the default player.
- `display` — `"full"` (default, artwork + transport + volume + search) or `"compact"` (one line: thumbnail, title/artist, play/pause, next; a tap opens the full player in a modal).
- `player_switch` (default `true`) — the header shows the current player as a dropdown; switching is local to the widget instance and to the session (not saved), so a wall tablet always comes back to its room's speaker.
- `search` (default `true`, ignored when no source exists or the device is not a media player) — shows the "Play something…" button.

**Data**: the widget loads the device (`GET /api/v1/device/:selector`, today) and its music object (`GET /api/v1/music/player/:selector`: now playing record, media player or not). **Rendering rules** — the UI derives from the features and the record, like the camera PTZ overlay:

| The player has… | The widget shows |
|---|---|
| a now playing record (E) | artwork (image key via B.5 and E.7, placeholder note icon otherwise), title, artist · album, source badge when `uri` is set, a progress bar extrapolated from `position`/`position_updated_at` while `playback_state` is `PLAYING` and `duration` is set, `LIVE` badge when there is no `duration` |
| `play`/`pause`/`previous`/`next` | the transport buttons, each only if the feature exists (today's behavior) |
| `volume` | the volume slider (today's behavior) |
| `play-media`, or native playback from a source (C.2) | the "Play something…" button and the speaker picker; idle state "Nothing playing" with that button when the record is `null` or `playback_state` is `STOPPED` |
| only `play_notification` | not listed (unchanged from today) |

Live updates: `device.new-state` (volume, playback state — today) **and** `music.now-playing-updated` for the record. Reconnection refresh: both loads again on `websocket.connected` (today's handler).

**Mockups:**

| Screen | File |
|---|---|
| Desktop dashboard — full and compact players, legacy player, today's widget for comparison | `music/desktop-dashboard.png` |
| Search ("what") — query, type filter, results by source, "play on" and enqueue mode | `music/search-desktop.png` |
| Speaker picker ("where") — players by room, incompatibility reasons | `music/player-picker.png` |
| Mobile — compact widget, full-screen search, picker bottom sheet | `music/mobile.png` |
| Dark mode — full and compact | `music/dark-mode.png` |
| Scene editor — the `music.play` action | `music/scene-action.png` |

![Desktop dashboard](music/desktop-dashboard.png)
![Search](music/search-desktop.png)
![Speaker picker](music/player-picker.png)
![Mobile](music/mobile.png)
![Dark mode](music/dark-mode.png)
![Scene action](music/scene-action.png)

### H.2 Search ("what")

A modal (desktop) / full-screen sheet (mobile) shared by the widget and the scene editor:

- query field (debounced 300 ms, ≥ 2 chars), type chips (All, Tracks, Albums, Artists, Playlists, Radios — only the types some source supports), source filter when there is more than one source;
- results grouped by **source** (no cross-source ranking, B.2), each group with its own "more" button (`offset`), a source in `timeout`/`error` shows a one-line notice instead of hiding silently;
- each row: artwork thumbnail (lazy-loaded), title, subtitle, duration for tracks, a play button, and a menu "Play next" / "Add to queue" (every media player honours them, C.3);
- items not playable on the selected player are dimmed with the reason ("Spotify plays only on Spotify Connect devices") — the client computes it from the source descriptor (`streamable`, `native`) and the player's `play_media` / `native_sources` (C.4), the same rule as C.2;
- footer: **Play on [player ▾]** (H.3).

### H.3 Speaker picker ("where")

`GET /api/v1/music/player?uri=…` → players grouped by room, the current one first; each line shows the playback state and what it plays; incompatible ones are disabled with their reason. Selecting one sets the target of the search modal (and of the widget when `player_switch` is on).

### H.4 Scene action `music.play`

```json
{
  "type": "music.play",
  "device": "living-room-sonos",
  "uri": "radio://radio/fip",
  "label": "FIP",
  "enqueue": "replace",
  "volume": 25
}
```

- `device` — the selector of a media player (A.1; the `music.play-notification` convention); `uri` — picked with the search modal of H.2 (or typed: a direct URL is allowed here); `label` — a display snapshot of the title, stored so the scene stays readable when the source is offline, never used to play; `volume` / `evaluate_volume` — same validation as `music.play-notification`.
- Executes `gladys.music.play`. A failure aborts the scene with the music error code (`MUSIC_PLAYER_INCOMPATIBLE`…), like other actions.
- `music.play-notification` is unchanged.
- The MCP `scene.create` schema gains the action (with `uri` restricted per G.1: from a prior search or a source URI).

### H.5 AI tools (MCP)

New tool category `AI_CHAT_TOOL_CATEGORIES.MUSIC` (the classifier learns "joue", "mets", "play", "musique", "radio", "volume"…). Three tools, registered only when at least one media player exists:

| Intent → tool | Input | Behavior |
|---|---|---|
| `music.search` → `music_search` | `{ query, types?: MediaType[], source?: enum(source names) }` | the C.4 search, flattened to the top 10 items with `uri`, type, title, subtitle, source name — lets the model pick among homonyms or answer "which albums of X do I have?" |
| `music.play` → `music_play` | `{ query?, type?: MediaType, uri?, device?: enum(player names), room?: enum(room names), enqueue?, volume? }` | if `uri` is absent, searches `query` (filtered by `type`) and takes the **first playable result compatible with the target** (the `HassMediaSearchAndPlay` model — one tool call for "joue l'album Heroes de Bowie dans le salon"); then `choosePlayer` (C.2) and `music.play`. Returns what was played and where, or the ambiguity (list of players / results) for the model to ask back |
| `music.control` → `music_control` | `{ action: 'pause' \| 'resume' \| 'next' \| 'previous' \| 'volume', volume?, device?, room? }` | writes the existing transport features; without a target, acts on the players currently `PLAYING` |

Device and room matching reuse `findBySimilarity`; `requireDeviceTargeting` is **not** set on `music_play` (no target = `choosePlayer`). `uri` must be a URI returned by a search in the conversation or a source URI — never a raw URL (G.1).

"Qu'est-ce qui joue dans la cuisine ?" needs no new tool: `device.get-state` learns to render a music device — its `playback_state` and its now playing record as "title — artist" (music features are not covered today, Precedents). It reads the record from `gladys.music.getNowPlaying`, not from a feature.

## I. Compliance with `device-feature-categories.md`

- **Capability, not brand (rule 1):** `play-media` describes what any networked player does; Spotify, Sonos, Navidrome appear only as examples. The source model is service-level, outside the taxonomy.
- **Standards (rule 3):** Matter drives the model where it has one. Media Playback: one command per transport action and one `CurrentState` (the existing features, kept; `STOPPED`/`BUFFERING` added from `NotPlaying`/`Buffering`), `Duration` and `SampledPosition { updatedAt, position }` (the record's `duration`, `position`, `position_updated_at`). Content Launcher: `UrlPlayback` is `play-media`, `ContentSearch` is native playback (C.2). Matter has no metadata attributes and no catalog model, so the record's fields and the source contract are Gladys' own, designed from Home Assistant and Music Assistant (multi-vendor, multi-protocol) rather than from one API.
- **Reuse before creating (rule 4):** no new category; the `music` category gains one type.
- **Atomic values (rule 5):** what plays is a record of ten fields, so it is not a feature (E.1); `play-media` holds no value at all (A.3). Every music feature keeps one atomic value.
- **Full set + `supported_options` (rule 6):** no enumerable value is added, so no option set: `play-media` refuses `supported_options` (A.3), and capabilities come from presence and ownership (C.2). A future enumeration (`repeat`: off / one / all) follows rule 6.
- **Naming (rule 7):** kebab-case new type; legacy underscored siblings kept (open question 2).
- **Plumbing (rule 8):** inline comment on `PLAY_MEDIA`; i18n `deviceFeatureCategory.music.play-media` and `deviceFeatureValue` for the new playback states in every locale file; `DeviceFeatureCategoriesIcon` (`fe-play-circle`); not added to `SUPPORTED_FEATURE_TYPES_BY_CATEGORY` and excluded from the scene value pickers (A.3); MQTT defaults (no min/max); no history grouping (no history).

## J. Testing expectations

Per repository policy (100 % patch coverage server-side): `server/test/lib/music/*` for the registry, search fan-out (ok/timeout/error per source, pagination passthrough), compatibility matrix (native / stream / direct URL / incompatible, with and without `play-media`, `native` true/false), `choosePlayer` ordering, normalizers (every refusal), image allowlist and cache (source keys from records with `uri`, player keys without); the now playing store: every schema rule and its message, ownership, `position_updated_at` stamping, no event on an equal record, clearing on device destroy / service stop / integration disconnect; the `device.setValue` guard and the `normalizeSupportedOptions` refusal on `play-media`; `music.play` scene action; external-integration manifest validation (`music_source`, `native`/`streamable`), proxy messages (`music.play` acknowledgement and timeout), `POST /music/now_playing` (`200`/`400`/`404`/`429`); MQTT routing of `…/music/now_playing` (record, `null`, empty, invalid → error topic) and publication on `…/music/play`; Sonos and Cast play and now playing mappings (notification filtered out); MCP tools with fake sources and `device.get-state` on a music device. SDK: handlers, `publishNowPlaying` (stamping, holding until `device-created`, replay after re-authentication, no replay of a refused record), typings, `gladys.fake.music*`. Front: eslint, `compare-translations`, build; a Cypress spec playing a radio from the widget on a fake player.

## K. Phases

| Phase | Content | Deliverable |
|---|---|---|
| **1 — core & contract** | Constants; `server/lib/music` (sources, search, play, compatibility, now playing store); HTTP API; the typed transports — internal `service.music`, external `music.*` messages and `POST /music/now_playing` (+ SDK release with the replay), MQTT `…/music/play` and `…/music/now_playing`; `radio` internal source; Sonos `play-media` + now playing; Google Cast persistent connection, `play-media` + now playing | "Play FIP on the Sonos" works through the API and from an external source (a Navidrome integration can be built against the SDK), and the record of what plays is right after a restart |
| **2 — surfaces** | Music widget (full/compact, search, picker); `music.play` scene action; AI tools and the `MUSIC` tool category, `device.get-state` on music devices; Cast resume after notification | The three entry points of the forum request: voice, scene, widget |
| **3 — breadth** *(not committed)* | AirPlay `play-media`; browse (a tree: artist → albums → tracks, HA `browse_media` model) for sources that declare it; transport extras — `stop` as one more type (a Matter command), `seek`, `shuffle`, `mute` as features, a `repeat` enumeration with `supported_options`; a `music.track-changed` scene trigger on the record's fields; grouping / multi-room; radio-browser.info in the `radio` source; per-user source accounts; native sources inside player services (Sonos favorites) | — |

## Alternatives considered (rejected)

- **One `transport` feature replacing `play`/`pause`/`previous`/`next`** (a `MUSIC_TRANSPORT` enum with `supported_options`, the pure `camera.move` model). Rejected for v1: it is not the standard's model — Matter keeps one command per transport action, while `camera.move` groups the values of one quantity —, the separate features are deployed on three internal services and in external integrations, users' scenes target them by feature selector, and the MQTT setups declare them: a migration with a compatibility window would cost more than the rows it removes. New transport commands are added as types, new enumerations through `supported_options`. Open question 1 leaves the final call to the maintainer.
- **The play request in `setValue`** — as the feature value, or as revision 1 did, the URI as the value and the expansion in `options`. Rejected: a record riding in the generic device path needs a side channel on every transport of that path (an `options` field in `external-integration.device.set-value`, a JSON exception on the MQTT state topic) whose shape depends on the feature type — the structured payload `camera-ptz-control.md` rejected. "Play this" is a core command with typed transports (C.1, C.3).
- **MQTT players limited to direct URLs** (to keep the state topic scalar without a music topic). Rejected: an MQTT player could then play nothing from a source; a dedicated topic carries the record without bending the state topics.
- **The player resolves the URI itself** (calls the core or the source back). Rejected: every player integration would have to implement a source client, an MQTT bridge cannot call back, and the core already knows both ends.
- **A core-managed queue feeding one URL at a time** (watching `playback_state` to send the next track). Rejected: gaps between tracks, fragile end-of-track detection, double logic with every native queue (Sonos, Cast, MPD, LinkPlay all have one).
- **The core streams or transcodes** (Music Assistant's model: every source → core → every player). Rejected: CPU and bandwidth on a Raspberry Pi, and against the "the core never fetches integration URLs" doctrine. Players fetch from sources directly; a source that needs a relay (a cloud service with no direct stream) is a `streamable: false, native: true` source with its own players. Using Music Assistant itself is welcome — as an external integration that is both a source and a set of players (C.2).
- **Extending `play_notification`** (`announce` flag, Home Assistant style). Rejected in D: two semantics, two contracts, each one sentence long.
- **"Now playing" as a JSON document in a text feature's `last_value_string`** (revision 1). Rejected after review, for the reasons of E.1: unenforceable on the fire-and-forget state path, a blob for every generic consumer, no versioning, a duplicate of `playback_state`, a precedent for every future record.
- **"Now playing" persisted** (a table, or a copy in a feature next to the memory). Rejected: after a restart a stored value is almost always wrong, every transport already restores the truth (E.5), and a table means a migration for data nobody needs once stale.
- **"Now playing" as one feature per field** (`track-title`, `artist`, `album`, `artwork`, `duration`, `position`). Rejected: six features per player, six writes per track change, inconsistent intermediate states (new title with old artwork), and no history value to justify them.
- **Capability flags in `play-media`'s `supported_options`** (revision 1: `STREAM`, `ENQUEUE`). Rejected after review: options are values a user sends — the scene editor builds a value list from any feature's options and would offer "Stream" as a value to send. `ENQUEUE` is implied by the queue every album-playing player has (C.3); `STREAM` is the presence of `play-media` (C.2).
- **"Accepts URLs" declared elsewhere** — a device param, or a per-integration manifest field. Rejected: a param is an untyped string; a per-integration flag is too coarse (a Home Assistant bridge exposes speakers that accept URLs next to Spotify entities that do not). The presence of a feature is per device and typed.
- **Native playback by ownership alone, without the source's `native` flag.** Rejected: every music device of a catalog integration (the clients a Plex integration reports) would be listed as a target it may not be. The source says once whether its own devices play its media.
- **Declaring the accepted URI schemes as string `supported_options`** (`spotify`, `http`…). Rejected: string option values are reserved to `text`/`select` dynamic lists, and ownership plus the source's `native` express the native case.
- **Artwork as URLs loaded by the browser.** Rejected: LAN-only and `http:` URLs break on Gladys Plus and in HTTPS pages, third-party CDNs learn the user's IP, authenticated covers (Subsonic) would leak tokens. Keys + bytes on demand, the widget doctrine.
- **A `music` manifest type.** Rejected per `provider-type.md`: the dual-role integrations (Spotify, Plex, Lyrion, MPD) are devices *and* sources; a capability field fits every type.
- **Browse in v1** (HA `browse_media` tree). Deferred to phase 3: search covers the three requested entry points (voice, scene, widget), and a tree navigator is a larger UI than this feature needs to be useful.
- **Global ranking across sources.** Rejected: relevance scores of two catalogs are not comparable; grouped results with per-source pagination are honest and simpler.

## Open questions (for review)

1. **Transport model.** Proposal: keep `play`/`pause`/`previous`/`next` as separate features and add `stop` the same way later, as Matter does (A.1). Or schedule a migration to a single `transport` feature with `supported_options` (and when)? This choice is as definitive as the record's.
2. **Naming.** `play-media` (kebab-case, rule 7) next to `play_notification`/`playback_state`, or `play_media` for consistency within the category?
3. **A feature without a value.** `play-media` declares URL playback per device, persisted with the device and visible in the MQTT setup — the role of a Matter feature bit — but it holds no value (A.3). Acceptable, or is there a better home in Gladys for a per-device capability?
4. **Who may play?** Any authenticated user (as today's play button), or restrict search/play of some sources to admins?
5. **Per-user source accounts** (Spotify per family member) — needed in v1? If yes, reuse the calendar `account_schema` model.
6. **Default player.** When the AI gets no room and several compatible players exist: ask back (current proposal), or use the room of the user's last interaction / a per-user preferred player?
7. **Built-in `radio` source.** Core service (proposal: zero dependency, reference implementation) or a store integration?
8. **Grouping / multi-room** (Sonos groups, Cast groups): model as a phase-3 feature (`group-members` with dynamic options) or as player devices that integrations publish for each group?
9. **Resume after notification**: SHOULD (proposal) or MUST for players that have both features?
10. **Clearing on disconnect** (E.5): immediately, at the moment the integration turns `DEGRADED` (proposal), or after a grace period that would hide short reconnections?
11. **MQTT refusals** (E.4): publish them on `…/music/now_playing/error` (proposal), or only log them?
