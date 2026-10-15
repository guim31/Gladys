# Music: play any media, from any source, on any player

> **Living specification — source of truth.** This document specifies how Gladys Assistant describes music players and music sources, how "play *this* on *that*" is represented and routed, where the "now playing" state lives, and the contracts integrations (internal, MQTT, external) and the frontend rely on. **Rule: any PR that changes a music behavior or contract modifies this file in the same diff** — spec first, code second.
>
> Status: **draft, for review.** Nothing in sections A–H is implemented. Section "Precedents" describes the code as of **v5.2.1** (`ff59ad8`, upstream `master` at the time of writing); nothing cited here exists only on an unreleased branch.
>
> Designed under `docs/specs/device-feature-categories.md` (section I audits it against that checklist) and modelled on `docs/specs/camera-ptz-control.md`.

## Context

Community request ([forum topic 11260](https://community.gladysassistant.com/t/11260), "Gladys Music"): ask Gladys to play **any music** — a track, an album, an artist, a playlist, a radio — from **any source** (Subsonic/Navidrome, Plex, Jellyfin, a DLNA server, web radios, Spotify…) on **any speaker** (Sonos, AirPlay, Google Cast, DLNA renderers, Denon/Marantz amplifiers, MusicCast, LinkPlay/WiiM, Lyrion/Squeezebox, Bose SoundTouch, MPD, VLC, Echo…), by voice (the Gladys AI), from a scene and from a dashboard widget.

Today the `music` category can change the volume, play, pause, skip, report a playback state and play a notification URL. It cannot say "play *this*": there is no notion of a media, of a source, or of what is currently playing, and the AI has no music tool at all.

The maintainer's answer (quoted in the forum thread) frames the work: it is **core** work, because sources and speakers are spread across many integrations and linking them is precisely the core's job; the existing **Music** widget must be adapted rather than duplicated; and the spec must settle five questions before any code:

| # | Question | Settled in |
|---|---|---|
| Q1 | How a player is described and what it can do — separate features or `supported_options` | A.1–A.3 |
| Q2 | How "play this" is represented, when a device command is a single value | A.4, C |
| Q3 | `play_media` or an extended `play_notification` | D |
| Q4 | Where the "now playing" state (title, artist, artwork) lives | E |
| Q5 | How a source declares itself, in the core and from external integrations | B |

### Design goals

1. **Players stay devices, sources are services.** A speaker is a thing in a room with features (volume, playback state); a catalog is not. A player is a device of the `music` category; a source is a service — internal or external — that implements a small search/resolve interface, exactly as a weather provider implements `weather.get`.
2. **Nothing that works today breaks.** Every existing `music` type, every scene, every MQTT setup and every external integration keeps working unchanged. New capabilities are additive.
3. **One scalar value on the wire wherever a generic consumer sees it.** What is played is named by a **media URI string** (`navidrome://album/al-42`). The generic device path, the HTTP value route, MQTT topics and scene editors keep handling one value per feature; the core does the expansion into something a speaker can play.
4. **Capabilities are data.** What a player can play is declared on one feature through `supported_options` (the `camera.move` precedent), not by multiplying feature types.
5. **The core never streams audio and never fetches a URL it was handed.** Sources hand playable URLs to players; artwork travels as bytes through the owning service. No transcoding, no stream proxy, no SSRF surface (G).
6. **A few dozen lines for an integration author.** Becoming a source is three SDK handlers; becoming a player is one more feature type in an existing `onSetValue` (B.4, F.4).

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
| `sonos` (`@svrooij/sonos`) | `play`, `pause`, `previous`, `next` (1/1), `volume` (0–100), `playback_state` (read-only), `play_notification` — all `has_feedback: false`, `keep_history: false` | `sonosDevice.PlayNotification({ trackUri, onlyWhenPlaying: false, volume: options?.volume \|\| 45, timeout: 20, delayMs: 700 })` (`lib/sonos.setValue.js`); the library advertises restoring the previous playback afterwards | UPnP events: `AVTransport` → `playback_state`, `Volume` → `volume` (`sonos.onAvTransportEvent.js`, `sonos.onVolumeEvent.js`). Track metadata is in the event but **ignored** |
| `google-cast` (`castv2-client`) | `play_notification` only | Launches `DefaultMediaReceiver`, `load({ contentId: url, contentType: 'audio/mp3' })`, restores the volume on `FINISHED` (`lib/google_cast.setValue.js`). Launching the receiver **kills** whatever app was casting: nothing is resumed | none |
| `airplay` (`@lox-audioserver/node-airplay-sender`) | `play_notification` only | `ffmpeg -re -i <url>` decoded to PCM and pushed to the sender, capped at 5 minutes (`MAX_NOTIFICATION_DURATION_MS`, `lib/airplay.setValue.js`) | none |

No service reports a title, an artist or an artwork anywhere.

**The value paths**

- `device.setValue(device, deviceFeature, value, options = {})` calls `service.device.setValue(device, deviceFeature, value, options)`, with no type/min/max validation in the core; it saves the state only when `!has_feedback` (`saveState` for numbers, `saveStringState` for `text`/`select`).
- Both HTTP value routes (`post /api/v1/device/:device_selector/:feature_category/:feature_type/value`, `post /api/v1/device_feature/:device_feature_selector/value`, `server/api/controllers/device.controller.js`) emit `ACTION.TRIGGERED` with a `device.set-value` action, and that action converts every value to a number on any feature that is not `text`/`select`, aborting with `ACTION_VALUE_NOT_A_NUMBER` otherwise. **A URL cannot be sent to a music feature over HTTP today**, and no `options` can be passed.
- `saveStringState` writes `last_value_string` (TEXT column, no history) and emits the WebSocket event `device.new-string-state` `{ device, device_feature, last_value_string, last_value_changed }`. Inbound states carrying `text` (MQTT `.../feature/:id/text`, external `POST /api/integration/v1/state` with `text`) go through it **for any feature** (`server/lib/device/device.newStateEvent.js`).
- MQTT (`server/services/mqtt/lib/setValue.js`) publishes `value.toString()` on `gladys/device/{device_external_id}/feature/{feature_external_id}/state`; `options` are not part of the topic contract.
- External integrations: the proxy service (`server/lib/external-integration/externalIntegration.registerProxyService.js`) forwards `external-integration.device.set-value` `{ message_id, device: { external_id, selector, params }, device_feature: { external_id, category, type }, value }` and **drops `options`** — an external speaker never receives the notification volume today.

**`supported_options`**

- Table `t_device_feature_supported_option` (`migrations/20260619120000-create-device-feature-supported-option.js`, plus `value_string` in `20260814000000-…`): `{ value, label, sort_order }`, synced by `device.syncFeatureSupportedOptions` on create/update, validated by `server/utils/normalizeSupportedOptions.js` (integer values everywhere; string values only on `text`/`select`), loaded with every device through `getStandardDeviceIncludes()`.
- `camera.move` (`CAMERA_MOVE`) is the model: one feature, canonical integer values in the core, per-device subset in `supported_options` (`docs/specs/camera-ptz-control.md`). `AC_MODE`, `WATER_HEATER_MODE`, `SIREN_MODE`, the thermostat preset follow the same pattern.

**Providers and images in the external-integration contract**

- The core enumerates weather providers by duck typing: `server/lib/weather/weather.get.js` scans `stateManager.getAllKeys('service')` for `service.weather.get`. External integrations join through the proxy service, which wraps `sendCommand` (request/response over WebSocket, `message_id` + `external-integration.command-result`, 5 s default, 15 s for weather and widgets) and normalizes every payload ("the payload is never trusted").
- A new contract that the core does not consume through a dedicated type is a **capability field** usable by every type (`docs/specs/external-integrations/capabilities/provider-type.md`; `CAPABILITY_MANIFEST_FIELDS = ['widgets', 'scene_triggers', 'scene_actions', 'energy_contracts']` in `server/lib/external-integration/constants.js`).
- **Images**: the core never lets the browser load a third-party URL and never fetches an integration-supplied URL; content carries **keys**, the integration returns the bytes on demand (`widget.get-image`, raw base64, PNG/JPEG/WebP, ≤ 300 KB, ≤ 4096 px), and the core serves them from its own origin with a cache (`docs/specs/external-integrations/capabilities/dashboard-widgets.md` §6). The same section lists "media control with seek and volume" as a **core** widget matter.

**The AI**

- Tools live in `server/services/mcp/lib/buildSchemas.js` (`getAllTools`): `{ intent, config: { title, description, categories, inputSchema (zod) }, cb }`, exposed as `function` tools to the Gladys Plus chat (`server/lib/gateway/gateway.forwardMessageToAiChat.js`) after a category classification (`AI_CHAT_TOOL_CATEGORIES`: scenes, device_control, device_query, weather, web_and_time, other). Devices and rooms are matched by `findBySimilarity` (exact, then Levenshtein ≤ 3).
- **No music tool exists**: music devices only appear in the `scene.create` schema, for `music.play-notification`.

**The widget**

- `front/src/components/boxs/music/MusicBox.jsx`: card titled with the device name, three buttons (previous, play/pause, next) and a volume range, driven by the `play`/`pause`/`previous`/`next`/`volume`/`playback_state` features, live through `device.new-state`. `EditMusicBox.jsx` lists `music` devices that have a `play` feature ("some music devices like the Nest Mini only expose the notification for now"). Box config: `{ type: 'music', device }`.

**Prior art outside Gladys** (what we keep and what we drop is argued in "Alternatives considered")

- **Home Assistant `media_player`**: one entity per player, a `supported_features` bitmask (PLAY_MEDIA, BROWSE_MEDIA, SEARCH_MEDIA, MEDIA_ENQUEUE, MEDIA_ANNOUNCE, GROUPING…), `play_media(media_content_id, media_content_type, enqueue: add|next|play|replace, announce)`, attributes `media_title`, `media_artist`, `media_album_name`, `media_duration`, `media_position` + `media_position_updated_at` (clients extrapolate the position), artwork served through `/api/media_player_proxy/<entity>?token=…` because it often lives on the LAN. `media-source://` URIs are resolved by the core into a plain URL before reaching URL-only players. Since 2025.6 the `HassMediaSearchAndPlay` intent searches then plays the first result, and is exposed as an LLM tool.
- **Music Assistant**: music providers vs player providers; URIs `<provider>://<media_type>/<item_id>`; `search` returns tracks/albums/artists/playlists/radios; server-side queues; announcements snapshot and resume the previous queue; MA **streams every source itself** to every player.
- **Matter**: the *Speaker* device type is On/Off (mute) + Level Control (volume); the *Media Playback* cluster has transport commands and a `CurrentState` (Playing, Paused, NotPlaying, Buffering); the *Content Launcher* cluster has `LaunchContent` (a search) and `LaunchURL`. There is no Matter model of a music catalog.

## A. Data model

### A.1 What a player is (Q1)

**A player is a device with features of the `music` category.** What it can do is read from *which* features it has and, for the new `play-media` feature, from that feature's `supported_options`.

**Decision on Q1 — no churn on the transport, `supported_options` for everything new:**

- The existing transport types (`play`, `pause`, `previous`, `next`, `volume`, `playback_state`, `play_notification`) are **kept unchanged**. They are already one feature per command, they are in production on three internal services and in external integrations, and users' scenes reference them by feature selector. Replacing them by a single `transport` feature would break every one of those scenes for no user-visible gain (rejected for v1, see Alternatives; kept as open question 1).
- **No new one-command-per-type feature is added.** Every new enumerable capability of a player is one feature with canonical values in the core and the per-device subset in `supported_options` — the `camera.move` model the maintainer pointed to. The first is `play-media` (A.3); the later ones (repeat, phase 3) follow the same rule.
- What a player can **play** — any stream, or only its own service's content, with or without a queue — is the one thing a speaker list must know to route a request (C.2). It is declared on `play-media`'s `supported_options`, not as more feature types.

### A.2 New types and enum changes

`DEVICE_FEATURE_TYPES.MUSIC` gains two types (`server/utils/constants.js`):

| Type | Constant | Role | Value | `read_only` | `min`/`max` | `has_feedback` | `keep_history` |
|---|---|---|---|---|---|---|---|
| `play-media` | `PLAY_MEDIA` | command: start playing a media | **media URI string** (A.4) | `false` | — / — | `false` | `false` |
| `now-playing` | `NOW_PLAYING` | state: what the player is playing | **JSON string** in `last_value_string` (E) | `true` | — / — | `false` | `false` |

**Naming:** the new types use kebab-case (rule 7 of `device-feature-categories.md`), next to the legacy underscored siblings (`play_notification`, `playback_state`), which are not renamed — renaming is the breaking mistake rule 7 warns about. The mixed spelling inside one category is ugly; open question 2 asks the maintainer to pick.

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

**New capability enum for `play-media`:**

```js
const MUSIC_PLAY_MEDIA_CAPABILITY = {
  STREAM: 1, // plays HTTP(S) audio URLs, hence media from ANY streamable source (C.2)
  ENQUEUE: 2, // honours enqueue = 'next' | 'add' instead of always replacing the queue
};
```

**New request enum (not stored, used by the API, the scene action and the AI tool):**

```js
const MUSIC_ENQUEUE = {
  REPLACE: 'replace', // default: stop what plays, play this
  NEXT: 'next', // insert after the current item
  ADD: 'add', // append at the end of the queue
};
```

**New media-type list (search filter, URI segment, item `type`):** `track`, `album`, `artist`, `playlist`, `radio`. Append-only (`podcast`, `audiobook` are expected later). Lowercase singular, the Music Assistant vocabulary.

**No schema migration.** Types are Sequelize ENUMs generated from the constants (TEXT in SQLite), `last_value_string` already exists, `t_device_feature_supported_option` is unchanged.

### A.3 The `play-media` feature and its `supported_options`

A player declares **one** `play-media` feature. Its `supported_options` list the `MUSIC_PLAY_MEDIA_CAPABILITY` values it supports (integer values, canonical i18n labels, like `camera.move`):

| Player | Options | Meaning |
|---|---|---|
| Sonos, Google Cast, LinkPlay/WiiM, MPD, DLNA renderer, VLC… | `STREAM`, `ENQUEUE` | plays any streamable source, with a native queue |
| AirPlay (Gladys pushes PCM), a renderer with no queue | `STREAM` | plays any streamable source; the integration plays the items in order, `enqueue` other than `replace` is refused (C.3) |
| Spotify Connect device (owned by the Spotify integration) | none | plays only media of its own service (C.2) |

**No fallback.** `play-media` is new, so no legacy feature exists without options: a feature with **no** option means "only my own service's media", never "everything" — unlike the AC-mode/`camera.move` fallback, which exists for features that predate `supported_options`. Assuming `STREAM` by default would send Navidrome URLs to a Spotify Connect speaker.

**State.** The feature has no state of its own (`has_feedback: false`, a string on a non-text feature is not saved, `device.setValue` already behaves so). What plays is in `now-playing` (E).

### A.4 The media URI (Q2)

A media is named by **one string**:

```
<source>://<type>/<id>
```

- `<source>` — the **source key** (B.1): the service's key in the state manager (`radio`, `ext-navidrome`, `ext-spotify`). Lowercase, `[a-z0-9-]`, ≤ 64 chars.
- `<type>` — one of the media types of A.2.
- `<id>` — opaque to the core, chosen by the source, URL-encoded, ≤ 512 chars. It must **not** carry credentials (G.3).

Examples: `ext-navidrome://album/al-42` (an external Navidrome integration), `radio://radio/fip` (the internal `radio` source), `ext-spotify://playlist/37i9dQZF1DXcBWIGoYBM5M`.

**Direct URLs.** `http://…` and `https://…` are accepted as a media URI of implicit type `track` with no source: "play this stream". They exist for scenes and power users (a web radio URL pasted in a scene), are playable only on `STREAM` players, and are refused from the AI tool (G.1).

Why a string: it is the one shape every generic Gladys consumer already carries — a feature value, a scene field, an MQTT payload, a chat tool argument, a dashboard box setting. The structure a speaker needs (one URL per track, titles, artwork keys) is produced **by the core at play time** (C.1), never stored and never typed by a user.

## B. Sources (Q5)

### B.1 What a source is

A **source** is a service that can search a catalog and turn a media URI into something playable. It is identified by its **source key** = its state-manager service key (the same key `weather.get` iterates). One service is at most one source. A service can be a source **and** own players (Spotify, Plex/Plexamp, Lyrion, MPD, Music Assistant): nothing forbids it, and C.2 relies on it.

A source declares, once, its **descriptor**:

```json
{
  "name": "Navidrome",
  "media_types": ["track", "album", "artist", "playlist"],
  "streamable": true
}
```

- `name` — display name (≤ 40 chars).
- `media_types` — the subset of A.2 types it can search and resolve.
- `streamable` — `true` when `resolve` returns HTTP(S) URLs any `STREAM` player can fetch; `false` for a source whose content only plays on its own players (Spotify: no stream URL exists outside Spotify Connect).

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
    getDescriptor: () => ({ name: 'Radios', media_types: ['radio'], streamable: true }),
    search: ({ query, limit, offset }) => radioHandler.search(query, limit, offset),
    resolve: ({ uri }) => radioHandler.resolve(uri),
    getImage: (key) => radioHandler.getImage(key),
  },
};
```

`server/lib/music/` lists sources the way `weather.get.js` lists providers: `stateManager.getAllKeys('service')`, keep those with `service.music && typeof service.music.search === 'function'`, sorted by key.

**Reference source shipped with the feature: `radio`** (new internal service, phase 1). The user saves web radios (name, stream URL, optional logo uploaded as a dashboard asset) from its integration page; search matches names; `resolve` returns the one URL. It has no network dependency, exercises the whole chain (search → play → now-playing → widget) on Sonos and Cast from day one, and covers "joue FIP dans la cuisine" — the single most common voice request in the forum thread. A public directory (radio-browser.info) is a phase-3 option of that service, not a core concern.

### B.4 External sources (store integrations)

**Manifest — new capability field `music_source`** (added to `CAPABILITY_MANIFEST_FIELDS`, usable by every type: a `device` integration that also owns players, or a `provider` integration that is a catalog only):

```json
{
  "manifest_version": 1,
  "type": "provider",
  "name": "Navidrome",
  "music_source": {
    "media_types": ["track", "album", "artist", "playlist"],
    "streamable": true
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

- Source images: `GET /api/v1/music/source/:source/image/:image_key` → `music.getImage(key)` on that source.
- Player images (the artwork in `now-playing`, E.2): `GET /api/v1/music/player/:device_selector/image/:image_key` → `music.getImage(key)` on **the service owning the device** (an internal Sonos service fetches `/getaa?…` from the speaker it already talks to; an external player answers `music.get-image`). A player service therefore implements `getImage` even when it is not a source — that is the only source function a player needs.
- Same validation, limits, error codes and cache policy as `dashboard-widgets.md` §6 (≤ 300 KB, ≤ 4096 px, 1 h cache per `(service, key)`, ≤ 100 entries per service LRU, ≤ 4 in flight per service, concurrent requests for one key share one command). Keys are immutable for the life of the cache.
- Allowlist: a key is served only if it appeared in that source's last 200 search/resolve items, or in the device's current `now-playing`. Any other key `404`s without a command sent.

## C. "Play this" (Q2)

### C.1 The play pipeline

One core function, `gladys.music.play({ device, uri, enqueue = 'replace', volume })`, used by the HTTP route, the scene action, the AI tool and the widget:

1. **Player.** `device` is a device with a `play-media` feature, else `400 MUSIC_NOT_A_PLAYER`.
2. **Parse** the URI (A.4), else `400 MUSIC_INVALID_URI`. Find the source by key, else `404 MUSIC_SOURCE_NOT_FOUND`.
3. **Compatibility** (C.2), else `409 MUSIC_PLAYER_INCOMPATIBLE` (with the reason: `not_streamable` / `stream_unsupported`). `enqueue` other than `replace` on a player without `ENQUEUE` → `409 MUSIC_ENQUEUE_UNSUPPORTED`.
4. **Resolve.** *Native* route (same service): nothing to resolve. *Stream* route: `source.music.resolve({ uri, player })` → `PlayableItem[]` (empty → `404 MUSIC_MEDIA_NOT_FOUND`). A direct `http(s)` URI resolves to one item `{ url: uri }` without any call.
5. **Volume.** If `volume` (0–100) is given and the player has a `volume` feature, `device.setValue(volumeFeature, volume)` first — the existing path, no new contract.
6. **Command.** `device.setValue(device, playMediaFeature, uri, { media })` where

```json
{
  "media": {
    "uri": "ext-navidrome://album/al-42",
    "route": "stream",
    "enqueue": "replace",
    "item": { "type": "album", "title": "\"Heroes\"", "subtitle": "David Bowie · 1977", "image": "cover-al-42" },
    "source": "ext-navidrome",
    "items": [ { "url": "http://…", "title": "Beauty and the Beast", "artist": "David Bowie", "album": "\"Heroes\"", "duration": 216, "image": "cover-al-42", "uri": "ext-navidrome://track/tr-1" } ]
  }
}
```

`route: "native"` carries no `items`: the player's own service knows what its URI means. `item` is the `MediaItem` the user picked when the caller has it (widget, AI), else built from the first resolved item; players may use it for their own display.

7. **Answer** `{ device, uri, route, items_count }` — never the URLs.

The **value** is still the scalar URI (design goal 3): logs, scene definitions and MQTT topics stay readable, and a player that only needs the URI (native route) ignores `options`. The expansion rides in the existing 4th argument, `options`, which already carries `volume` for `play_notification`.

### C.2 Choosing a compatible player

A player `P` (feature `play-media`, options `O`) can play a media of source `S` iff:

- **native**: `P`'s device belongs to `S`'s service (`device.service_id` = the source's service), **or**
- **stream**: `S.streamable` and `STREAM ∈ O`, **or**
- the URI is a direct `http(s)` URL and `STREAM ∈ O`.

Native wins when both hold (a Lyrion player playing the Lyrion library natively keeps gapless playback and the server's own queue).

This is the whole answer to **"a source that only plays at home" (Spotify Connect)**: the Spotify integration declares `music_source` with `streamable: false` **and** publishes its Connect devices as players with a `play-media` feature and no `STREAM` option. Spotify media are compatible with those devices only; a Navidrome album is compatible with the Sonos and not with the Spotify devices. Nothing Spotify-specific exists in the core. The same mechanism covers Echo devices driven by an Alexa integration, Plexamp, Lyrion, Music Assistant (which can declare `streamable: false` and route every request to its own players).

When only a **room** (or nothing) is given — AI and scenes —, `gladys.music.choosePlayer({ uri, room?, user? })`:

1. candidates = players compatible with the media (above); if a room is given, those in that room;
2. none → `MUSIC_NO_COMPATIBLE_PLAYER` (the AI explains which players exist and why they do not fit);
3. one → it;
4. several → the one currently `PLAYING` (the user means "this speaker"), then the one that played most recently (`now-playing` `updated_at`), then **ambiguous**: the AI asks back with the list, the scene editor never gets there (a scene always names a device).

### C.3 What a player receives

**Internal service:** `service.device.setValue(device, deviceFeature, uri, options)` with `deviceFeature.type === 'play-media'` and `options.media` as above. Sonos: `route: stream` → clear queue (`replace`) / `AddURIToQueue` (`next`, `add`) then `Play`; `route: native` does not occur (Sonos is not a source in v1).

**External integration:** `external-integration.device.set-value` gains an **optional, additive `options` field** — `{ message_id, device, device_feature, value, options }`. For `play-media` it carries `{ media }`; for `play_notification` it carries `{ volume }`, which fixes today's dropped notification volume as a side effect. An old SDK ignores the field. SDK: `onSetValue(device, feature, value, options)` (4th argument).

**MQTT (generic device):** `gladys/device/{device_external_id}/feature/{feature_external_id}/state` — for `play-media` only, the payload is the **JSON** `{ "uri", "route", "enqueue", "items": [{ "url", "title", "artist", "album", "duration" }] }` (a bridge cannot call the core back to resolve). Every other feature keeps its plain-string payload. This is the one documented exception to "payload = value as a string", justified because the alternative is an MQTT player that can play nothing but direct URLs. Image keys are dropped from the MQTT payload (an MQTT bridge cannot answer `get-image`).

A player integration MUST acknowledge only once playback has been **accepted** by the speaker (queue loaded), not when the track ends, within the 5 s command timeout; an album longer than the timeout is never an error. Failure → `success: false` with a human-readable `error`, shown by the widget and spoken by the AI.

### C.4 HTTP API

All routes are user-authenticated (standard API auth, any role — see open question 5).

| Route | Body / query | Answer |
|---|---|---|
| `GET /api/v1/music/source` | — | `[{ source, name, media_types, streamable, status }]`; `status` = `ok` / `not_connected` (external integration down) |
| `GET /api/v1/music/search` | `query`, `types` (csv), `source` (csv, default all), `limit`, `offset` | `{ sources: [{ source, name, status: 'ok' \| 'timeout' \| 'error', error?, items, has_more }] }` — fan-out in parallel, 8 s per source, a slow source never blocks the others' results |
| `GET /api/v1/music/player` | `uri` (optional), `room` (optional) | players with `{ device_selector, name, room, capabilities, playback_state, now_playing, compatible?, reason? }`; with `uri`, each player says whether it can play it (the speaker picker of H.3) |
| `POST /api/v1/music/play` | `{ device_selector, uri, enqueue?, volume? }` | C.1; `202` once the player acknowledged |
| `GET /api/v1/music/source/:source/image/:image_key` | — | B.5 |
| `GET /api/v1/music/player/:device_selector/image/:image_key` | — | B.5 |

**The generic value route also works**: `POST /api/v1/device_feature/:selector/value { value: "<uri>" }` on a `play-media` feature is routed by the `device.set-value` scene action to `gladys.music.play` with the string kept (instead of `Number()`-converting it and aborting with `ACTION_VALUE_NOT_A_NUMBER`). Generic clients — Node-RED, the scene `device.set-value` action, the existing MQTT `gladys/master/...` paths — can therefore "play this" with one scalar. `enqueue` and `volume` are only available on `/music/play`.

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
      "read_only": false, "has_feedback": false, "keep_history": false,
      "supported_options": [
        { "value": 1, "label": "Stream", "sort_order": 0 },
        { "value": 2, "label": "Enqueue", "sort_order": 1 }
      ] },
    { "name": "Now playing", "external_id": "ext:wiim:a1b2c3:now-playing", "category": "music", "type": "now-playing",
      "read_only": true, "has_feedback": false, "keep_history": false }
  ]
}
```

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
  "type": "external-integration.device.set-value",
  "payload": {
    "message_id": "4f0c…",
    "device": { "external_id": "ext:wiim:a1b2c3", "selector": "wiim-pro-kitchen", "params": [] },
    "device_feature": { "external_id": "ext:wiim:a1b2c3:play-media", "category": "music", "type": "play-media" },
    "value": "ext-navidrome://album/al-42",
    "options": {
      "media": {
        "uri": "ext-navidrome://album/al-42",
        "route": "stream",
        "enqueue": "replace",
        "source": "ext-navidrome",
        "item": { "type": "album", "title": "\"Heroes\"", "subtitle": "David Bowie · 1977", "image": "cover-al-42" },
        "items": [
          { "url": "http://192.168.1.20:4533/rest/stream?id=tr-1&…", "mime_type": "audio/flac",
            "title": "Beauty and the Beast", "artist": "David Bowie", "album": "\"Heroes\"",
            "duration": 216, "image": "cover-al-42", "uri": "ext-navidrome://track/tr-1" }
        ]
      }
    }
  }
}
```

## D. `play-media` vs `play_notification` (Q3)

**Decision: two features, two semantics. `play_notification` is not extended.**

| | `play_notification` (existing) | `play-media` (new) |
|---|---|---|
| Purpose | a short sound that **interrupts** (TTS, doorbell, alarm chime) | a media that **replaces** (or joins) what plays |
| Value | one audio URL | a media URI, expanded by the core |
| After it ends | the previous playback **resumes** | the queue continues |
| Queue | untouched | replaced or extended (`enqueue`) |
| Volume option | temporary, restored after | persistent (`volume` = a plain `volume` write) |
| Typical caller | `music.play-notification` scene action, alarm | AI "play X", `music.play` scene action, widget |

Merging them (Home Assistant's `play_media(…, announce: true)`) would make every caller pass a mode flag through the scalar path, and every player implement both behaviors behind one feature, while `play_notification` already exists and works. Keeping two features keeps each contract one sentence long.

**The resume contract, made explicit** (it was implicit until now):

- A player that has both features SHOULD restore after a notification what `play-media` was playing: same queue, same item, position within a few seconds, same volume, same paused/playing state. Sonos does it through `PlayNotification`. Google Cast currently does **not** (launching the default receiver kills the running app): phase 2 of this spec makes the Cast service snapshot its media session (queue, current item, position) before the notification and reload it afterwards when the session was Gladys' own; a session owned by another app (YouTube, Spotify) is not restorable and is documented as such.
- A notification sent while `playback_state` is `PAUSED`/`STOPPED` leaves the player in that state afterwards.
- A `play-media` received while a notification plays cancels the pending resume: the newest explicit request wins.

## E. "Now playing" (Q4)

### E.1 Where it is stored

**Decision: one read-only `now-playing` feature per player, whose state is a JSON document in `last_value_string`, normalized by the core.**

- It reuses the plumbing that exists for every transport: external integrations push it with `POST /api/integration/v1/state` `{ device_feature_external_id, text }` (SDK `publishState(id, { text })`), MQTT bridges on `gladys/master/device/{id}/feature/{id}/text`, internal services through the `device.new-state` event with `text`. The frontend receives the existing `device.new-string-state` WebSocket event. **No new transport, no new WebSocket event.**
- It survives a reboot (`last_value_string` is persisted) and the device API already returns it — the widget renders immediately on load.
- No history (`keep_history: false`, and `last_value_string` has none anyway): a track list is not a time series.
- Cost: one row update per track change, comparable to a camera image refresh.

`device.newStateEvent` gains one step for `music/now-playing`: parse, normalize (whitelisted keys, lengths, enums; an invalid document is refused and logged with the device selector, the previous state is kept), re-serialize. The core never stores what it did not validate.

### E.2 The document

```json
{
  "state": "playing",
  "uri": "ext-navidrome://track/tr-1",
  "source": "ext-navidrome",
  "type": "track",
  "title": "Beauty and the Beast",
  "artist": "David Bowie",
  "album": "\"Heroes\"",
  "image": "cover-al-42",
  "duration": 216,
  "position": 37,
  "position_updated_at": "2026-10-15T18:42:07.120Z",
  "queue_index": 0,
  "queue_length": 10
}
```

- `state` — mirrors `playback_state` as a word (`playing`, `paused`, `stopped`, `buffering`) so the document is self-sufficient; `playback_state` stays the feature scenes trigger on.
- `uri`/`source`/`type` — present when the player plays something Gladys started (or a native media of its own source); absent for content started elsewhere (the Spotify app, a line-in): the widget then shows title/artist with no source badge.
- `title` (required unless `state` is `stopped`), `artist`, `album` (≤ 200 each).
- `image` — an image key served by the **player's** service (B.5). A player relaying the source's key (the WiiM above playing a Navidrome item) answers `get-image` itself; the convenient implementation is to keep the artwork bytes it was given — open question 9 discusses letting the core route the key to the source instead.
- `duration` (s, absent for live streams), `position` (s), `position_updated_at` (ISO date) — the Home Assistant model: the player publishes on **changes** (track, seek, play/pause), never every second; the widget extrapolates while `state` is `playing`.
- `queue_index`/`queue_length` — optional, the widget shows "3 / 10".

Total ≤ 4 KB serialized. Rate: a player publishes at most on each change, and the existing `POST /state` limit (300/min per integration) applies.

### E.3 Fixture — a "now playing" event

What an external integration sends:

```json
POST /api/integration/v1/state
{ "states": [ { "device_feature_external_id": "ext:wiim:a1b2c3:now-playing",
  "text": "{\"state\":\"playing\",\"uri\":\"ext-navidrome://track/tr-1\",\"source\":\"ext-navidrome\",\"type\":\"track\",\"title\":\"Beauty and the Beast\",\"artist\":\"David Bowie\",\"album\":\"\\\"Heroes\\\"\",\"image\":\"cover-al-42\",\"duration\":216,\"position\":0,\"position_updated_at\":\"2026-10-15T18:41:30.000Z\"}" } ] }
```

What the frontend receives (existing event, unchanged shape):

```json
{
  "type": "device.new-string-state",
  "payload": {
    "device": "wiim-pro-kitchen",
    "device_feature": "wiim-pro-kitchen-now-playing",
    "last_value_string": "{\"state\":\"playing\",\"uri\":\"ext-navidrome://track/tr-1\",…}",
    "last_value_changed": "2026-10-15T18:41:30.020Z"
  }
}
```

SDK sugar: `gladys.publishNowPlaying(featureExternalId, doc)` serializes and validates in dev mode.

## F. Integration contracts — summary and migration

### F.1 What changes in the core

- `server/utils/constants.js`: `PLAY_MEDIA`, `NOW_PLAYING` in `DEVICE_FEATURE_TYPES.MUSIC` (with the inline scope comment rule 8 requires), `MUSIC_PLAYBACK_STATE` extended, `MUSIC_PLAY_MEDIA_CAPABILITY`, `MUSIC_ENQUEUE`, `MUSIC_MEDIA_TYPES`, `ACTIONS.MUSIC.PLAY`, `AI_CHAT_TOOL_CATEGORIES.MUSIC`, WebSocket types `external-integration.music.{search,resolve,get-image}`.
- `server/lib/music/` (new): source registry, `search`, `play`, `choosePlayer`, image cache, normalizers.
- `server/lib/device/device.newStateEvent.js`: `now-playing` normalization (E.1).
- `server/lib/scene/scene.actions.js`: `device.set-value` keeps a string for `play-media` and routes to `music.play` (C.4); new `music.play` action (H.4).
- `server/lib/external-integration/`: `music_source` manifest field, `music.*` on the proxy service, `options` forwarded in `device.set-value`.
- `server/services/mcp/`: music tools (H.5).
- `device.setValue` is **unchanged**.

### F.2 Internal players

| Service | Phase | Change |
|---|---|---|
| `sonos` | 1 | `play-media` (`STREAM`, `ENQUEUE`: `RemoveAllTracksFromQueue` + `AddURIToQueue` + `Play`, `AddURIToQueue` with `EnqueuedAsNext` for `next`), `now-playing` from the `AVTransport` event's `CurrentTrackMetaData` it already receives (title, creator, album, `albumArtURI` → image key `art-<hash>` served by `getImage`, fetched from the speaker), `playback_state` gains `STOPPED`/`BUFFERING` from `TransportState`. Nothing removed |
| `google-cast` | 1 | `play-media` (`STREAM`, `ENQUEUE` via the default receiver's `queueLoad`/`queueInsert`), `now-playing` from media status, `play`/`pause`/`next`/`previous`/`volume`/`playback_state` (the Nest Mini finally appears in the Music widget); phase 2: resume after notification (D) |
| `airplay` | 3 | `play-media` (`STREAM` only: the service plays items sequentially through its ffmpeg pipeline, without the 5-minute notification cap), `pause` (stops the pipeline), `now-playing` from what it sends. AirPlay has no queue of its own and no remote metadata to read |

Existing devices gain the new features **on the next scan**: `device.create` matches existing features by `external_id` (`server/lib/device/device.create.js`), so a re-created Sonos device keeps its features, selectors and the scenes that use them, and gets the new ones. No `device.migrate` entry: no feature is renamed or removed, and no dashboard box field references a new feature (the Music box references a device).

### F.3 MQTT

- Generic devices can declare `play-media` and `now-playing` in the MQTT setup UI (catalog entries from the i18n keys). `play-media` gets two checkboxes (stream, enqueue) like `camera.move`'s movement checkboxes; `now-playing` shows the JSON format in the feature's help text.
- Outbound: JSON payload for `play-media` only (C.3). Inbound: `.../feature/{id}/text` for `now-playing` (existing handler).

### F.4 External integrations

- **Existing integrations keep working**: `music_source` is optional, `options` is an additive field, no type changes.
- **To become a player:** publish the `play-media` (+ `now-playing`) features, handle `feature.type === 'play-media'` in `onSetValue` reading `options.media`, implement `onMusicGetImage` if `now-playing` carries images.
- **To become a source:** declare `music_source`, implement `onMusicSearch`, `onMusicResolve` (if `streamable`), `onMusicGetImage`.
- Credentials (Navidrome password, Spotify OAuth) stay in the integration's `config_schema` (`secret`, `oauth2`) — the core never sees them (G.3).

### F.5 Backward compatibility

- No type, value, route or event changes meaning. `MUSIC_PLAYBACK_STATE` is extended append-only; a scene condition `playback_state = 0` still means paused.
- A player with no `play-media` stays exactly as today in the widget and in scenes; the widget simply hides the search and artwork areas for it.
- Existing `music` boxes (`{ type: 'music', device }`) render with the new defaults (H.1) without a dashboard migration.
- `play_notification` gains nothing mandatory; the resume behavior (D) is a SHOULD.

## G. Security

### G.1 Arbitrary URLs

- The core **never fetches** a media or stream URL: it hands URLs to players. The only bytes it fetches are images, and only through the owning service by key (B.5) — the widget-image doctrine, so no SSRF from the core.
- A direct `http(s)` URI makes **the speaker** fetch it. Speakers already fetch arbitrary URLs through `play_notification`; the risk is a LAN device requesting a LAN URL, with an audio decoder as the only consumer of the response. Schemes other than `http`/`https` (`file:`, `smb:`, `rtsp:`…) are refused at parse time.
- The **AI tool never accepts a URL** (`uri` must come from a search result in the same conversation turn or name a source key): web pages fetched through `web.fetch` must not be able to make Gladys play an attacker-chosen stream by prompt injection. Direct URLs remain available to users through scenes and the HTTP API.

### G.2 Untrusted payloads

- Search/resolve answers, `now-playing` documents and images from integrations are normalized and bounded (B.4, E.1, B.5): whitelisted keys, lengths, item caps, URI source check (a source cannot vend another source's URIs), image magic numbers and pixel bounds.
- Text is rendered as text, never HTML (titles come from file tags anybody can write).

### G.3 Sources that need credentials

- Credentials live in the integration (config `secret`, `oauth2`, `account_link`), never in the core, never in a URI (`<id>` must not carry credentials).
- Resolved stream URLs often embed tokens (Subsonic `t`/`s` salt, Plex `X-Plex-Token`). They travel core → player only: **no API returns a `PlayableItem`**, the play answer carries a count, `now-playing` carries URIs not URLs, and the core does not log `items[].url` (it logs the URI and the item count). MQTT players do receive the URLs — publishing them on a broker is the MQTT user's trust boundary, documented in F.3.
- **Per-user accounts** (two Spotify users in a household) are out of v1: a source is house-wide. The calendar type's per-user `account_schema` is the precedent if needed (open question 6).

### G.4 Authorization and abuse

- All routes require an authenticated user. Search is rate-limited per user (10/s) and fans out with per-source timeouts; images obey the per-service in-flight cap.
- Any user who can use the dashboard can play music, as any user can press the existing play button today (open question 5 asks whether to restrict sources to admins).

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
- `search` (default `true`, ignored when no source exists or the player has no `play-media`) — shows the "Play something…" button.

**Rendering rules** — the UI derives from the features, like the camera PTZ overlay:

| The player has… | The widget shows |
|---|---|
| `now-playing` | artwork (image key via B.5, placeholder note icon otherwise), title, artist · album, source badge, a progress bar extrapolated from `position`/`position_updated_at` when `duration` is set, `LIVE` badge otherwise |
| `play`/`pause`/`previous`/`next` | the transport buttons, each only if the feature exists (today's behavior) |
| `volume` | the volume slider (today's behavior) |
| `play-media` | the "Play something…" button and the speaker picker; idle state "Nothing playing" with that button when `state` is `stopped` |
| only `play_notification` | not listed (unchanged from today) |

Live updates: `device.new-state` (volume, playback state — today) **and** `device.new-string-state` for `now-playing`. Reconnection refresh: unchanged (`websocket.connected`).

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
- each row: artwork thumbnail (lazy-loaded), title, subtitle, duration for tracks, a play button, and a menu "Play next" / "Add to queue" when the selected player has `ENQUEUE`;
- items not playable on the selected player are dimmed with the reason ("Spotify plays only on Spotify Connect devices") — the client computes it from the source descriptor and the player capabilities, the same rule as C.2;
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

- `device` — a device selector with `play-media` (the `music.play-notification` convention); `uri` — picked with the search modal of H.2 (or typed: a direct URL is allowed here); `label` — the title shown in the editor, stored so the scene stays readable when the source is offline; `volume` / `evaluate_volume` — same validation as `music.play-notification`.
- Executes `gladys.music.play`. A failure aborts the scene with the music error code (`MUSIC_PLAYER_INCOMPATIBLE`…), like other actions.
- `music.play-notification` is unchanged.
- The MCP `scene.create` schema gains the action (with `uri` restricted per G.1: from a prior search or a source URI).

### H.5 AI tools (MCP)

New tool category `AI_CHAT_TOOL_CATEGORIES.MUSIC` (the classifier learns "joue", "mets", "play", "musique", "radio", "volume"…). Three tools, registered only when at least one player with `play-media` exists:

| Intent → tool | Input | Behavior |
|---|---|---|
| `music.search` → `music_search` | `{ query, types?: MediaType[], source?: enum(source names) }` | the C.4 search, flattened to the top 10 items with `uri`, type, title, subtitle, source name — lets the model pick among homonyms or answer "which albums of X do I have?" |
| `music.play` → `music_play` | `{ query?, type?: MediaType, uri?, device?: enum(player names), room?: enum(room names), enqueue?, volume? }` | if `uri` is absent, searches `query` (filtered by `type`) and takes the **first playable result compatible with the target** (the `HassMediaSearchAndPlay` model — one tool call for "joue l'album Heroes de Bowie dans le salon"); then `choosePlayer` (C.2) and `music.play`. Returns what was played and where, or the ambiguity (list of players / results) for the model to ask back |
| `music.control` → `music_control` | `{ action: 'pause' \| 'resume' \| 'next' \| 'previous' \| 'volume', volume?, device?, room? }` | writes the existing transport features; without a target, acts on the players currently `PLAYING` |

Device and room matching reuse `findBySimilarity`; `requireDeviceTargeting` is **not** set on `music_play` (no target = `choosePlayer`). `uri` must be a URI returned by a search in the conversation or a source URI — never a raw URL (G.1).

"Qu'est-ce qui joue dans la cuisine ?" needs no new tool: `device.get-state` learns to render a `now-playing` feature as "title — artist".

## I. Compliance with `device-feature-categories.md`

- **Capability, not brand (rule 1):** `play-media`/`now-playing` describe what any networked player does; Spotify, Sonos, Navidrome appear only as examples. The source model is service-level, outside the taxonomy.
- **Reuse before creating (rule 4):** no new category; the `music` category is extended with two types. `text/text` was considered for "now playing" and rejected: it would put a playback document in a category meant for user-visible free text, lose the category grouping in the device UI, and give the widget no way to find it among other text features.
- **Standards (rule 3):** Matter's Media Playback `CurrentState` (Playing/Paused/NotPlaying/Buffering) drives the `MUSIC_PLAYBACK_STATE` extension; Matter's Content Launcher (`LaunchContent` with a search, `LaunchURL`) has the same split as our search + play. Matter has no catalog model, so the source contract is Gladys' own, designed from Home Assistant and Music Assistant (multi-vendor, multi-protocol) rather than from one API.
- **Full set + `supported_options` (rule 6):** `MUSIC_PLAY_MEDIA_CAPABILITY` is the full set; each player declares its subset. No fallback, justified in A.3.
- **Naming (rule 7):** kebab-case new types; legacy underscored siblings kept (open question 2).
- **Plumbing (rule 8):** inline comments on the constants; i18n `deviceFeatureCategory.music.{play-media,now-playing}` and `deviceFeatureValue` for the new playback states and capabilities in every locale file; `DeviceFeatureCategoriesIcon` (`fe-play-circle`, `fe-music`); not added to `SUPPORTED_FEATURE_TYPES_BY_CATEGORY` (devices-in-room keeps no music rows — the Music widget is the surface); MQTT defaults (`read_only` true for `now-playing`, no min/max); no history grouping (no history).

## J. Testing expectations

Per repository policy (100 % patch coverage server-side): `server/test/lib/music/*` for the registry, search fan-out (ok/timeout/error per source, pagination passthrough), compatibility matrix (native / stream / URL / incompatible / enqueue unsupported), `choosePlayer` ordering, normalizers (every refusal), image allowlist and cache; `device.newStateEvent` with valid and invalid `now-playing`; the `device.set-value` string routing; `music.play` scene action; external-integration manifest validation and proxy messages; `options` forwarding; Sonos and Cast `setValue`/event mappings; MCP tools with fake sources. SDK: handlers, typings, `gladys.fake.music*`. Front: eslint, `compare-translations`, build; a Cypress spec playing a radio from the widget on a fake player.

## K. Phases

| Phase | Content | Deliverable |
|---|---|---|
| **1 — core & contract** | Constants; `server/lib/music` (sources, search, play, compatibility); HTTP API; `now-playing` normalization; `device.set-value` string routing; external `music_source`, `music.*` messages, `options` in `device.set-value` (+ SDK release); `radio` internal source; Sonos and Google Cast `play-media` + `now-playing` | "Play FIP on the Sonos" works through the API and from an external source (a Navidrome integration can be built against the SDK) |
| **2 — surfaces** | Music widget (full/compact, search, picker); `music.play` scene action; AI tools and the `MUSIC` tool category; Cast resume after notification | The three entry points of the forum request: voice, scene, widget |
| **3 — breadth** *(not committed)* | AirPlay `play-media`; browse (a tree: artist → albums → tracks, HA `browse_media` model) for sources that declare it; transport extras through `supported_options` (`repeat` enum, `shuffle`, `seek`, `stop`, `mute`); grouping / multi-room; radio-browser.info in the `radio` source; per-user source accounts; native sources inside player services (Sonos favorites) | — |

## Alternatives considered (rejected)

- **One `transport` feature replacing `play`/`pause`/`previous`/`next`** (a `MUSIC_TRANSPORT` enum with `supported_options`, the pure `camera.move` model). Cleanest on paper, and what we would do from scratch. Rejected for v1: the separate features are deployed on three internal services and in external integrations, users' scenes target them by feature selector, and the MQTT setups declare them — a migration with `device.migrate` and a compatibility window would cost more than the extra rows it removes. New capabilities follow the `supported_options` rule instead, so the debt does not grow. Open question 1 leaves the final call to the maintainer.
- **A structured value in `setValue` everywhere** (the media object as the feature value). Rejected for the reasons `camera-ptz-control.md` gave: every generic consumer (scene `device.set-value`, HTTP route, MQTT, the external WS message, MCP) speaks one scalar per feature. The URI is that scalar; the structure travels in the `options` argument that already exists, only between the core and the player.
- **The player resolves the URI itself** (calls the core or the source back). Rejected: every player integration would have to implement a source client; the core already knows both ends.
- **A core-managed queue feeding one URL at a time** (watching `playback_state` to send the next track). Rejected: gaps between tracks, fragile end-of-track detection, double logic with every native queue (Sonos, Cast, MPD, LinkPlay all have one).
- **The core streams or transcodes** (Music Assistant's model: every source → core → every player). Rejected: CPU and bandwidth on a Raspberry Pi, and against the "the core never fetches integration URLs" doctrine. Players fetch from sources directly; a source that needs a relay (a cloud service with no direct stream) is a `streamable: false` source with its own players. Using Music Assistant itself is welcome — as an external integration that is both a source and a set of players (C.2).
- **Extending `play_notification`** (`announce` flag, Home Assistant style). Rejected in D: two semantics, two features, each one sentence long.
- **"Now playing" as one feature per field** (`track-title`, `artist`, `album`, `artwork`, `duration`, `position`). Rejected: six features per player, six writes per track change, inconsistent intermediate states (new title with old artwork), and no history value to justify them.
- **"Now playing" only in memory with a new WebSocket event** (`music.now-playing-updated`). Lighter on the database, but needs a new push path for external integrations, a new MQTT topic, and a reload API; and the widget would be empty after a restart until the next track. The `last_value_string` reuse wins on simplicity.
- **Artwork as URLs loaded by the browser.** Rejected: LAN-only and `http:` URLs break on Gladys Plus and in HTTPS pages, third-party CDNs learn the user's IP, authenticated covers (Subsonic) would leak tokens. Keys + bytes on demand, the widget doctrine.
- **A `music` manifest type.** Rejected per `provider-type.md`: the dual-role integrations (Spotify, Plex, Lyrion, MPD) are devices *and* sources; a capability field fits every type.
- **Declaring the accepted URI schemes as string `supported_options`** (`spotify`, `http`…). Rejected: string option values are reserved to `text`/`select` dynamic lists, and "same service = native" expresses the native case without any declaration.
- **Browse in v1** (HA `browse_media` tree). Deferred to phase 3: search covers the three requested entry points (voice, scene, widget), and a tree navigator is a larger UI than this feature needs to be useful.
- **Global ranking across sources.** Rejected: relevance scores of two catalogs are not comparable; grouped results with per-source pagination are honest and simpler.

## Open questions (for review)

1. **Transport model.** Keep `play`/`pause`/`previous`/`next` as separate features forever, or schedule a migration to a single `transport` feature with `supported_options` (and when)?
2. **Naming.** `play-media`/`now-playing` (kebab-case, rule 7) next to `play_notification`/`playback_state`, or `play_media`/`now_playing` for consistency within the category?
3. **`now-playing` storage.** Is a JSON document in `last_value_string`, normalized by the core, acceptable, or does the maintainer prefer the in-memory store + dedicated event?
4. **MQTT JSON payload** for `play-media` (C.3) — acceptable exception, or should MQTT players be limited to direct URLs?
5. **Who may play?** Any authenticated user (as today's play button), or restrict search/play of some sources to admins?
6. **Per-user source accounts** (Spotify per family member) — needed in v1? If yes, reuse the calendar `account_schema` model.
7. **Default player.** When the AI gets no room and several compatible players exist: ask back (current proposal), or use the room of the user's last interaction / a per-user preferred player?
8. **Built-in `radio` source.** Core service (proposal: zero dependency, reference implementation) or a store integration?
9. **Image routing for relayed media.** Should a player relaying a source item be allowed to publish `image` as `source:<key>` so the core serves it from the source (no bytes kept by the player)?
10. **Grouping / multi-room** (Sonos groups, Cast groups): model as a phase-3 feature (`group-members` with dynamic options) or as player devices that integrations publish for each group?
11. **`play-media` on the generic value route** (C.4): keep it (Node-RED, scenes) or force `/music/play` only?
12. **Resume after notification**: SHOULD (proposal) or MUST for players that have both features?
