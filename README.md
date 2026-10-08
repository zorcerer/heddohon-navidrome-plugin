# The Heddohon plugin for Navidrome

A plugin for [Navidrome](https://www.navidrome.org/) that tells
[Heddohon](https://github.com/zorcerer/heddohon), a self-hosted music player
for Navidrome, Subsonic and Jellyfin, what is played in every other app.

Navidrome tells this plugin of every track a user finishes, whichever app
played it, and the plugin tells Heddohon. With it:

- **Recently played and Your listening cover every app.** A play in Symfonium,
  DSub or Navidrome's own player is noted in Heddohon's history beside the
  plays made in Heddohon. A play made in Heddohon is noted once.
- **"Import play history" brings in every play.** Navidrome keeps a scrobble
  history that only a plugin can read. Without the plugin the import has one
  play per track, at the date it was last played.
- **The Devices button lists your other apps**, each with the track it is on,
  and "Continue here" plays that track in Heddohon from where the app has got
  to. An app that reports its position, which few do so far, is picked up at
  the second. For the rest the time is where the track would be had it played
  without a pause, and the dialog says so. The app goes on playing: Navidrome
  cannot pause a client.
- **Listening now can include what you play in other apps**, for an account
  that turns on "Include what I play in other apps" under Settings, Account.
  Off until it does.

It needs Navidrome 0.64 or later, and Heddohon 0.6.1 or later with
`HEDDOHON_NAVIDROME_PLUGIN_TOKEN` set.

The Heddohon side of it is in the main repository,
[zorcerer/heddohon](https://github.com/zorcerer/heddohon): the route the plugin
posts to (`src/routes/api/plugin/navidrome`), what is done with a request
(`src/lib/server/plugin.ts`), and the tests that post what the plugin would.
Its [Configuration guide](https://github.com/zorcerer/heddohon/wiki/Configuration#the-navidrome-plugin)
has the same steps as below from Heddohon's side.

## Install

1. **Choose a token**, at least 32 characters: `openssl rand -hex 32`. Set it
   as `HEDDOHON_NAVIDROME_PLUGIN_TOKEN` on the Heddohon server and restart it.
   It must not be the value of `HEDDOHON_SECRET`.
2. **Put `heddohon.ndp` in Navidrome's plugins folder**, which is `plugins`
   inside its data folder unless `Plugins.Folder` says otherwise. The file is
   attached to each of this repository's
   [releases](https://github.com/zorcerer/heddohon-navidrome-plugin/releases).
   Keep the name: it is the plugin's id. Restart Navidrome, or run
   `navidrome plugin rescan`.
3. **In Navidrome, under Plugins, open Heddohon** and fill in:
   - **Heddohon address:** where Navidrome reaches Heddohon, such as
     `http://192.168.1.10:3000` or `https://music.example.com`.
   - **Plugin token:** the token from step 1.
   - **Heddohon's public address:** only where the address above is not the
     one Heddohon is opened at in a browser. Then it is the value of `ORIGIN`
     on the Heddohon server. Heddohon refuses a write that does not name its
     own address, and logs `cross-origin-blocked` with the address it expects.
4. **Choose the users** the plugin may act for, or all of them, and enable it.

From the command line the last two steps are:

```
navidrome plugin edit heddohon --all-users \
  --config '{"url":"http://192.168.1.10:3000","token":"<the token>"}'
navidrome plugin enable heddohon
```

Under Settings, Listening history, Heddohon then shows "Plays from other apps"
and says when the plugin has not been heard from in the last minute.

## Updating, and when nothing arrives

Navidrome turns a plugin off when its file changes. After replacing
`heddohon.ndp` with a newer one, enable it again under Plugins; its settings
are kept. A change made with `navidrome plugin` on the command line takes
effect when Navidrome restarts.

The plugin writes to Navidrome's log when Heddohon does not take a play, with
the status Heddohon answered:

- **401:** the token is not the value of `HEDDOHON_NAVIDROME_PLUGIN_TOKEN`.
- **403:** the address is not the one Heddohon knows itself by. Fill in
  "Heddohon's public address". Heddohon's log names the address it expects, in
  a `cross-origin-blocked` line, at `HEDDOHON_LOG_LEVEL=warn`.
- **404:** `HEDDOHON_NAVIDROME_PLUGIN_TOKEN` is not set on that Heddohon, the
  Heddohon is 0.6.0 or earlier, or the address is another server's.
- **301, 302, 308:** the address redirects, usually from http to https. Give
  the address it redirects to; a redirect is not followed.

Navidrome keeps the plays meanwhile and sends them once the setting is right,
after up to 4 minutes.

## What it sends, and where

Only to the address in its settings, and it does not follow a redirect from
there:

- for each play Navidrome reports: the user name, the track's id and the time;
- for what a client is playing now, as a track starts and whenever the client
  says where it is: the user name, the track's id, whether it plays or is
  paused, the position, and the client's id and name. Heddohon's own reports
  are not sent back to it;
- every 15 seconds, a request with nothing in it, which Heddohon answers with
  the user names whose scrobble history it is waiting for;
- for such a user, that history: the track id and the time of each scrobble,
  1000 a request.

Heddohon looks each track up itself, as the account the play belongs to, and
drops a play for a user who has never signed in to it. A play Heddohon cannot
take while it is down is kept by Navidrome and sent again.

The permissions it asks Navidrome for are in `manifest.json`, each with its
reason. HTTP is asked for any host, since Heddohon is usually on a private
address, which Navidrome otherwise refuses a plugin.

What Heddohon does with a request, and what the token allows, is in its
[`SECURITY.md`](https://github.com/zorcerer/heddohon/blob/main/SECURITY.md#plays-from-the-navidrome-plugin).
A vulnerability in the plugin is reported as that file says.

## The messages

The plugin and Heddohon are released apart, so what passes between them is
versioned. Every message is a `POST` to `/api/plugin/navidrome` on the address
configured, with `Authorization: Bearer <token>`, an `Origin` header naming
Heddohon's own address, and a JSON body `{ "v": 1, "type": ... }`:

| `type` | With | When |
| --- | --- | --- |
| `plays` | `plays: [{ username, songId, at }]` | A play, as Navidrome reports the scrobble |
| `poll` | nothing | Every 15 seconds |
| `history` | `username`, `plays: [{ songId, at }]`, `more`, `failed` | A page of a user's scrobble history, oldest first, for a user the last answer named |
| `playback` | `username`, `songId`, `state`, `positionMs`, `player`, `playerName` | What a client is playing: `state` is `starting`, `playing`, `paused`, `stopped` or `expired` |

`at` is in Unix seconds. Every answer is
`{ "wanted": [{ "username", "from" }] }`: the users whose history Heddohon is
waiting for, each from a time in Unix seconds. Navidrome makes a new instance
of a plugin for every call, so the plugin keeps nothing between two of them,
and where an import has got to is held by Heddohon.

Heddohon answers a `v` it does not know with 400, and a `type` it does not
know likewise, which an older Heddohon does for a newer plugin's message. A
change that an older Heddohon could misread takes a new `v`, which Heddohon
then answers beside the old one. `protocol` in `main.go` and `PROTOCOL` in
Heddohon's route change together.

## Build

```
./build.sh
```

writes `heddohon.ndp` beside it, using TinyGo where it is installed and the
`tinygo/tinygo` image through Docker otherwise. `VERSION=1.2.3` sets the
version in the manifest; without it the build is `0.0.0-dev`.

Every push builds the package and keeps it as an artifact of the run
(`.github/workflows/build.yml`). A pushed tag `v1.2.3` publishes a release
with `heddohon.ndp`, version 1.2.3, attached (`release.yml`).

Nothing here runs the plugin: that takes a Navidrome. Heddohon's suites post
to its route what the plugin would post, and the plugin itself is tried
against a real Navidrome before a release.

## Licence

MIT, as Heddohon is. See `LICENSE`.
