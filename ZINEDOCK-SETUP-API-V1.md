# Zinedock computer setup API, version 1

Zinedock's **Set up from a computer** page talks to Zinedock on the Zidoo through a small HTTP API. This page describes
that API so that a script or an AI assistant can use it. It applies to Zinedock on the Z9X 8K and other Zidoos running
the same app, from version 1.2.8.

## In short

Zinedock answers plain HTTP on your home network, with JSON. It changes Zinedock's own setup only: Home rows, layout
and collections. It cannot change your films, Home Theater or the Zidoo.

**With the code on the TV** (Setup > Computer > Set up from a computer shows an address and a six-digit code):

```sh
ZD=http://192.168.1.50:41234                      # the address on the TV
curl -s -c zd.jar -d pin=123456 "$ZD/login"       # log in with the code
curl -s -b zd.jar "$ZD/api/state"                 # read the setup
curl -s -b zd.jar -H 'X-Zinedock: 1' -H 'Content-Type: application/json' -d '{"q":"night"}' "$ZD/api/search"
```

**With a key** (turn on Setup > Computer > Paired computers, then pair once from the setup page):

```sh
curl -s -H "Authorization: Bearer $KEY" http://192.168.1.50:9463/api/state
```

Every `POST` needs the header `X-Zinedock: 1` and a JSON body (`{}` if the call takes nothing).

| Call | What it does |
|---|---|
| `GET /api/version` | This copy's version and the calls it answers |
| `GET /api/state` | The whole setup: Home rows and layout |
| `GET /api/export` | Save the setup as a file |
| `POST /api/import` | Load a setup file. Replaces the whole setup |
| `POST /api/undo-restore` | Bring back the setup from before the last import |
| `POST /api/rows` | Order, show, hide and rename Home rows |
| `POST /api/layout` | Spotlight, trailers, background, posters, top menu |
| `POST /api/collections` | List your collections |
| `POST /api/search` | Find films and series by name |
| `POST /api/col-get` | One collection, with its titles in order |
| `POST /api/col-save` | Make or change a collection: hand-picked, nested or rule-based |
| `POST /api/col-delete` | Delete a collection |
| `POST /api/onhome` | Put a row on Home or take it off |
| `POST /api/suggested`, `suggested-add` | Ready-made collections that match the library; add one |
| `POST /api/smart-choices`, `smart-preview` | What rules can choose from; what a set of rules matches |
| `POST /api/people-search` | Find a director, actor or producer |
| `POST /api/build` | Follow a rule-based collection while it builds |
| `POST /api/tmdb-save`, `tmdb-remove`, `tmdb-status` | Your TMDB key: save or remove it; it is never sent back |
| `POST /api/mdb-save`, `mdb-remove`, `mdb-status` | The same for your MDBList key |
| `POST /api/pair-status`, `pair` | Whether pairing is on; pair this computer (code login only) |

**Answers.** `{"ok": true, ...}` when done. `{"ok": false, "error": "why, in words", "code": "a_stable_word"}` when
refused; decide on `code`. Common codes: `not_logged_in`, `session_ended`, `key_invalid`, `key_removed`, `unknown_call`,
`refused`, `tv_not_on_home`.

**Promise.** A version 1 call never changes what it means. New calls may be added.

Everything below is the full reference: every field, limit and error.

---

# Reference

Version 1 is a promise:

- A version 1 call never changes what it means. New calls, and new fields in an answer, may be added.
- If a call ever has to change, it becomes API version 2, version 1 stays alongside it for at least two releases, and
  the release notes say so.
- `GET /api/version` says which API version a copy of Zinedock speaks and lists every call it answers.

Read every answer as JSON and ignore fields you do not know.

## What the API can and cannot change

It changes **Zinedock's own setup** only: Home rows, layout, trailers, the top menu, and your collections. It cannot
change your films, your files, Home Theater's library or the Zidoo itself. See [What Home Theater owns](#what-home-theater-owns-and-what-zinedock-adds).

**`POST /api/import` replaces the whole setup.** Every choice and every collection you made in Zinedock is replaced by
what is in the file; a collection that is not in the file is gone. **Save a setup file first** (`GET /api/export`)
before importing, and before any large change. `POST /api/undo-restore` brings back the setup from before the last
import, once.

The TMDB and MDBList key calls are **write-only**: a key can be saved or removed, and `status` says only whether one is
saved. No call ever sends a saved key back.

## Connecting

There are two ways in. **The code on the TV** (below) is for setting up from a browser now: a new address and code each
time, and it closes by itself. **A paired key** ([Pairing a computer once](#pairing-a-computer-once)) is for a script or
an AI assistant that comes back: a fixed address and a key that lasts until you remove it on the TV.

1. On the TV, in Zinedock: **Setup > Computer > Set up from a computer**. The TV shows an address such as
   `192.168.1.50:41234` and a six-digit code.
2. Log in with the code. You get a cookie, `zs`. Send it with every call.
3. Every `POST` also needs the header `X-Zinedock: 1` and a JSON body.

```sh
ZD=http://192.168.1.50:41234          # the address on the TV, exactly as shown

# log in with the code on the TV; the cookie is kept in zd.jar
curl -s -i -c zd.jar -d pin=123456 "$ZD/login"
# HTTP/1.1 303 See Other ... Set-Cookie: zs=...; Path=/; HttpOnly; SameSite=Strict

curl -s -b zd.jar "$ZD/api/version"
curl -s -b zd.jar "$ZD/api/state"
curl -s -b zd.jar -o zinedock-setup.json "$ZD/api/export"           # save a setup file first

curl -s -b zd.jar -H 'X-Zinedock: 1' -H 'Content-Type: application/json' \
     -d '{"q":"night"}' "$ZD/api/search"
```

The session, measured from the code:

| | |
|---|---|
| Port | A new random port each time the TV opens the page. Read it from the TV. |
| Code | Six digits. Must be entered within **5 minutes** of the TV showing it. |
| Attempts | Five login attempts per session, right or wrong. The **sixth** answers 429 and the session ends. |
| Ends | After **10 minutes** without a call, **30 minutes** after it opened whatever happens, when **Close computer setup** is pressed on the TV, or when Zinedock closes. |
| After it ends | When its time runs out (the code not entered in 5 minutes, 10 minutes without a call, or 30 minutes in all) the address keeps answering, with **401 `session_ended`** only. When Close is pressed, Zinedock closes, the TV opens the page again, or after the sixth login attempt, the old address stops answering (connection refused). |
| Zinedock must be running | The Zidoo stops Zinedock while a disc plays, so nothing answers then. |
| Network | Plain HTTP on your home network only. The connection is not encrypted. |

After a session ends, every call answers 401 `session_ended`, or the address stops answering (see the table). Either
way: open **Set up from a computer** on the TV again for a new address and code, and log in again. Do not retry on the
old address.

### Rules every request must follow

- `Host` must be exactly the address on the TV (`IP:port`). Use the IP address, not a name for the box.
- If `Origin` is sent it must be `http://IP:port`. `Sec-Fetch-Site: cross-site` is refused. Requests from other web
  pages are therefore refused; a script or `curl` sends neither header.
- `POST /api/*`: header `X-Zinedock: 1`; a `Content-Length` header; a body of **2 to 262,144 bytes** (send `{}` when a
  call takes no fields); no `Transfer-Encoding`.
- Each header at most once, all headers together under 8,192 bytes. A request that breaks these, or a `POST` without a
  readable `Content-Length`, is closed with no answer.
- One request per connection (`Connection: close`). Every answer has `Cache-Control: no-store`.

### Login

`POST /login`, `Content-Type: application/x-www-form-urlencoded`, body `pin=<six digits>` (at most 128 bytes). This is
the HTML form the browser uses, so its answers are pages, not JSON; read the status code.

| Status | Meaning |
|---|---|
| 303, `Location: /`, `Set-Cookie: zs=<32 hex>` | Logged in. |
| 400 | Wrong code (counts as an attempt), or `Content-Type` is not exactly `application/x-www-form-urlencoded`. |
| 413 | Body over 128 bytes. |
| 429 | Too many attempts. The session has ended. |

A successful login replaces any earlier cookie for this session.

## Pairing a computer once

Pair a computer once and it keeps a key. With the key, a script or an AI assistant can use every call on this page
without a code from the TV, for as long as Zinedock is running. It is **off until you turn it on**.

1. On the TV, in Zinedock: **Setup > Computer > Paired computers**, and turn it on. The TV first says what a paired
   computer can do.
2. Log in with the code as in [Connecting](#connecting), then open **Paired computers** on the setup page, type a name
   for the computer and press **Pair this computer** (or send `POST /api/pair`).
3. The key is shown **once**. Keep it private. The TV now lists the computer by its name.
4. From then on, call the Zidoo at the address pairing gave you (**port 9463**, see below) and send the key with every
   request: `Authorization: Bearer <key>`.

```sh
ZD=http://192.168.1.50:9463           # the address pairing gave you
KEY=zdk_...                           # the key, shown once when you paired

curl -s -H "Authorization: Bearer $KEY" "$ZD/api/version"
curl -s -H "Authorization: Bearer $KEY" -o zinedock-setup.json "$ZD/api/export"
curl -s -H "Authorization: Bearer $KEY" -H 'X-Zinedock: 1' -H 'Content-Type: application/json' \
     -d '{"q":"night"}' "$ZD/api/search"
```

| | |
|---|---|
| Port | **9463**. If another app on the Zidoo already uses 9463, Zinedock takes the next free port, up to 9472. It chooses once and keeps that port from then on. `POST /api/pair` and `POST /api/pair-status` answer the address, and the TV shows it under Setup > Computer. |
| Key | `zdk_` and 64 hexadecimal characters. Shown once, when pairing. Zinedock keeps only a SHA-256 of it, so it cannot show it again: if you lose it, remove the computer on the TV and pair again. |
| Lasts | Until you remove it on the TV. A key does not expire. |
| What a key can do | Every call on this page except `POST /api/pair`, including `import` (which replaces the whole setup) and saving or removing the TMDB and MDBList keys. It cannot read those keys, and it cannot change your films, Home Theater or the Zidoo. |
| What only the TV can do | Turn paired computers on or off, list them, and remove one. Pairing needs the code on the TV. |
| Remove | **Setup > Computer**, choose the computer under Paired computers, **Remove**. Its key stops working at once, for good. |
| Off | Turning paired computers off stops every key at once: port 9463 stops answering. The computers stay listed, and turning it on again brings the same keys back. |
| Zinedock must be running | Port 9463 answers only while Zinedock is running. The Zidoo stops Zinedock while a disc plays. Calls that read or change the setup also need Zinedock's Home screen loaded on the TV (`tv_not_on_home`). |
| Network | Plain HTTP on your home network only. The connection is not encrypted, so anyone on your network who can see the traffic can read the key. Zinedock answers on port 9463 only at the Zidoo's home-network address. |

The [rules every request must follow](#rules-every-request-must-follow) are the same on port 9463, with `Host` being
`IP:9463`. Port 9463 has no setup pages and takes no cookie; opened in a browser it shows only a short note saying to
use Set up from a computer on the TV. The address from the TV's code screen takes no key.

A key is never in a setup file, in `GET /api/export`, or in any answer except the one that issued it.

### Refusals for a key

On port 9463, as `{"ok": false, "error": "<message>", "code": "<code>"}`:

| Status | `code` | `error` | When |
|---|---|---|---|
| 401 | `key_missing` | `Send your pairing key in the Authorization header: Bearer <key>.` | No `Authorization: Bearer` header. |
| 401 | `key_invalid` | `That pairing key is not known to this Zidoo.` | The key was never issued by this Zidoo, or is not a key at all. |
| 401 | `key_removed` | `This computer was removed on the TV, so its key no longer works. Pair it again from the setup page.` | The computer was removed on the TV. Zinedock remembers the last 100 removed keys this way; an older one answers `key_invalid`. |
| 403 | `pairing_needs_code` | `A paired key cannot pair another computer. Log in with the code on the TV and pair from the setup page.` | `POST /api/pair` with a key. |
| 429 | `too_many_attempts` | `Too many wrong keys from this address. Wait a minute and try again.` | More than ten wrong or missing keys from one address within a minute. A correct key is never refused this way. |

With paired computers turned off, or Zinedock not running, port 9463 does not answer at all (connection refused). The
other refusals in [Answers and errors](#answers-and-errors) apply on port 9463 as well, except those for the code login
(`not_logged_in`, `logged_out`, `session_ended`): a key has no session and does not expire.

## Answers and errors

Every `/api/*` answer is JSON (`Content-Type: application/json; charset=utf-8`), except `GET /api/export`, whose answer
is the setup file itself.

**HTTP 200 with `"ok": true`**: done. **HTTP 200 with `"ok": false`**: Zinedock looked at the request and refused it;
`error` says why in plain words. Most refusals are made before anything changes, but `rows` and `layout` apply their
parts in order (see each call), so **after any `"ok": false`, read `GET /api/state`** to see where things stand.

Every `"ok": false` answer also has **`code`**, a stable word for programs: decide on `code`, show `error` to people.
The words in `error` may be improved; a `code` is never renamed or reused within version 1, and new codes may be added.

Refusals from the connection itself, with a status other than 200, are always exactly
`{"ok": false, "error": "<message>", "code": "<code>"}`:

| Status | `code` | `error` | When |
|---|---|---|---|
| 400 | `bad_request` | `Invalid request.` | A `POST` with `Transfer-Encoding`. |
| 401 | `not_logged_in` | `Enter the code on the TV first.` | No `zs` cookie, or one that is not from this session. Log in again. |
| 401 | `session_ended` | `This setup session has ended. On the TV open Set up from a computer again, then use the new address and code.` | The session's time ran out: the code was not entered within 5 minutes, 10 minutes passed without a call, or 30 minutes passed in all. The cookie and the code no longer work on this address; `POST /login` answers 401 with the same words as a page. |
| 401 | `logged_out` | `Another computer or phone logged in to this Zidoo, so this one was logged out. Only one can be logged in at a time. To come back, enter the code shown on the TV.` | One login at a time: a later login with the code ends this one. |
| 403 | `forbidden` | `Request refused.` | Wrong `Host`; an `Origin` that is not this address; `Sec-Fetch-Site: cross-site`; a `POST` without `X-Zinedock: 1`. |
| 404 | `unknown_call` | `Not found.` | No such call, or a `POST` call sent as `GET` (or the other way round). |
| 405 | `method_not_allowed` | `Not allowed.` | A method other than `GET` or `POST`. |
| 413 | `too_large` | `That is too large.` | A `POST` body under 2 or over 262,144 bytes. |
| 500 | `export_failed` | `Could not make the setup file.` | `GET /api/export` only. |

**When a session ends.** Time ran out (5 minutes to enter the code, 10 minutes idle, 30 minutes in all): every call on
that address answers 401 `session_ended` until Zinedock closes or the TV opens Set up from a computer again. Close on the
TV, Zinedock closed, or a new session opened on the TV: the old address stops answering, the connection is refused and
there is no HTTP answer at all (nothing is running there to answer). A cookie from an earlier session sent to a new address
gets 401 `not_logged_in`. A paired key has its own port and its own refusals: see
[Refusals for a key](#refusals-for-a-key).

HTTP 200 `ok: false` answers carry one of these codes:

| `code` | Meaning | What to do |
|---|---|---|
| `tv_not_on_home` | `Open Zinedock's Home screen on the TV first, then try again.` Zinedock's own view is not loaded on the TV. | Try again after Home is open. |
| `tv_no_answer` | `The TV did not answer. Try again.` No answer within 45 seconds. | Try again; read `GET /api/state` first if the call changes something. |
| `tv_starting` | `Zinedock is still starting. Try again in a moment.` | Try again in a few seconds. |
| `not_understood` | `That request was not understood.` The body is not JSON. | Fix the request. |
| `refused` | The call looked at the request and turned it down: a value out of range, a rule that does not exist, a collection that no longer exists, and so on. `error` says which. | Fix the request; nothing changed unless the call says it applies parts in order. |
| `failed` | It could not be done just now (for example, the Zidoo could not save). | Try again later. |
| `pairing_off` | `POST /api/pair` only: paired computers is turned off on the Zidoo. | Turn it on on the TV: Setup > Computer > Paired computers. |

The TV shows *Saved from the computer* after any call that changed something, except saving a rule-based collection,
where the TV shows the build instead.

## Words and units used in answers

| Name | Meaning |
|---|---|
| Row id | A Home row. `continue` Continue Watching, `seasonal` Seasonal, `favourites` Favourites, `recent` Recently added, `films` Films, `tv` Television, `coll` Franchises, `discs` Discs, `collection-<n>` a Home Theater franchise, or a collection made by hand in Home Theater (n = Home Theater's own number), `local-<letters>` a collection you made in Zinedock. |
| Title id | A film or series in the Zidoo's library: a string of digits, from `search`. |
| `source` | Where a row comes from: `library` (one of Zinedock's standard rows), `collection` (a Home Theater franchise), `group` (a collection made by hand in Home Theater; from 1.2.8), `manual` (hand-picked), `smart` (rule-based), `universe` (a collection of collections), `franchise` (a franchise made in Zinedock). |
| `mediaType` | `film`, `tv` or `mixed` (films and series together). `null` for standard rows and Home Theater franchises. |
| Delay | Milliseconds. `0` means the default: 1 second on Home rows, 1.2 seconds on the Spotlight, 1.5 seconds on a film's page. |
| Home position | Counted from 1 = the top row on Home. |
| Colours | `#rrggbb`. |
| `year` | Four digits; `0` when the Zidoo has no year for it. |

## The calls

### `GET /api/version`
What this copy of Zinedock is and which calls it answers. Answers even when the TV is not on Home.

```json
{"ok":true,"app":"Zinedock","version":"1.2.8","versionCode":328,"api":1,
 "calls":[{"method":"GET","path":"/api/version"},{"method":"GET","path":"/api/state"}, ...]}
```

`version` is the app's version name `versionCode` its build number (a whole number that
only goes up), `api` the API version (the number `1`), `calls` every call this copy answers, in no meaningful order.

### `GET /api/state`
The whole current setup. Read it before changing anything and after any `ok: false`.

```json
{"ok":true,
 "rows":[{"id":"continue","title":"Continue Watching","source":"library","mediaType":null,"on":true,"featured":false},
         {"id":"local-mh2x9k-a1b2c3","title":"Rainy night","source":"manual","mediaType":"film","on":true,"featured":false},
         {"id":"collection-12","title":"A Franchise","source":"collection","mediaType":null,"on":false,"featured":false}],
 "trailers":{"spot":"sound","rows":"silent","info":"sound"},
 "spotShow":true,"spotRecess":true,"seasonSpot":true,"seasonBg":true,"pauseCard":true,
 "bg":"colour","bgColour":"#25272b","playAnim":"all",
 "delay":0,"delays":[0,2000,3000,5000],
 "poster":"standard","posters":[{"key":"small","label":"Small"},{"key":"standard","label":"Standard"},{"key":"large","label":"Large"}],
 "topMenu":[{"tag":"films","label":"Films","shown":true},{"tag":"tv","label":"Television","shown":true},
            {"tag":"coll","label":"Franchises","shown":true},{"tag":"mine","label":"Collections","shown":true}],
 "undoRestore":false,"undoRestoreAt":null}
```

| Field | Meaning |
|---|---|
| `rows` | Every row that can be on Home. The rows on Home (`on: true`) come first, **in Home order, top first**; then the hidden ones. `featured: true` on at most one row: the collection shown in the Spotlight. |
| `trailers.spot / rows / info` | The trailer in the Home Spotlight, on the highlighted film in a Home row, and on a film's page: `off`, `silent` or `sound`. `spot` reads `off` while the Spotlight is hidden. |
| `spotShow` | The Home Spotlight (the large picture at the top of Home) is shown. |
| `spotRecess` | The Spotlight is drawn set into the wall. |
| `seasonSpot`, `seasonBg` | Around holidays, the season's films take the Spotlight; the season's backdrop frames it. |
| `pauseCard` | The information card over a paused film. |
| `bg`, `bgColour` | The background behind every page: `black`, `film` (the highlighted film's picture) or `colour` (`bgColour`). |
| `playAnim` | The animation before a film starts: `discs` (discs only), `all` (films and TV), `off`. |
| `delay`, `delays` | How long before an automatic trailer starts, in milliseconds; one of `delays`. |
| `poster`, `posters` | Poster size: one of `posters[].key`. |
| `topMenu` | The entries along the top of the app. Home and Search are always there. |
| `undoRestore` | `true` when `POST /api/undo-restore` can bring back the setup from before the last import or restore. |
| `undoRestoreAt` | When that import or restore was made: milliseconds since 1 January 1970 (UTC), as JavaScript's `Date.now()`. `null` when `undoRestore` is `false`. Undo replaces everything changed since then. |

### `GET /api/export`
The setup file: everything `import` would restore. The answer is the file itself (no `ok` field), with
`Content-Disposition: attachment; filename="zinedock-setup.json"`.

```json
{"format":"zidoo-library-settings","version":1,"created":"2026-11-10T21:30:00.000Z",
 "preferences":{...},"collections":{"version":1,"order":[...],"names":{...},"collections":[...]},
 "franchiseAdds":{...},"topMenu":{"version":1,"hidden":[]}}
```

It holds Home rows and their order and names, your collections, films added to franchises, the top menu, the trailer
and layout choices. It never holds a TMDB or MDBList key, and it holds no films. Keep the file as it is; to change the
setup use the other calls, not an edited file. The poster size is not in the file.

### `POST /api/import`
Body: `{"text": "<the whole setup file, as a string>"}`. Answer: `{"ok":true}`.

**Replaces the whole setup** with the file: every choice, every Home row, every collection made in Zinedock. Collections
that are not in the file are removed. Your films and Home Theater are not touched. The setup in place before is kept
for `undo-restore`; the TV view reloads about a second later. The file is checked first; a file that is not a Zinedock
setup file, or one holding a name that could start an HTML tag (see `col-save`), is refused with
`This is not a supported Zinedock settings backup. Nothing changed.` and nothing changes.

Use the same Zidoo library: hand-picked collections refer to the library's own title numbers, so on another library
they may point at nothing.

### `POST /api/undo-restore`
Body: `{}`. Answer: `{"ok":true}`.

Brings back the setup from before the last import (from this API, the setup page or the TV's Restore backup), exactly as
it was, and the TV view reloads. One step: afterwards `undoRestore` is `false`, and a second call answers
`There is no restore to undo.` To go forward again, import the same file again. A second import replaces the kept copy
with the setup that import replaced. A copy kept by Zinedock 1.2.7 or earlier carries no date and is not offered.

### `POST /api/rows`
Body: `{"order": [row ids], "names": {row id: "name"}, "featured": row id | null}`. Answer: `{"ok":true}`.

- `order` (required): the rows on Home, **top first**. A row left out is taken off Home (it is not deleted and can be
  put back). Every id must be one of `state.rows[].id`, or nothing changes: `Unknown Home row.` A repeated id counts once.
- `names` (optional): new names, 1 to 80 characters. A blank name leaves that row's name as it is. All names
  are checked before anything changes: `Use a name between 1 and 80 characters.` A name is plain text: one with `<` directly before a letter, `/`, `!` or `?` (the start of an HTML tag) is refused: `A name cannot have < directly before a letter. Put a space after it, or leave it out.` `Films <1960`, `&` and quotes are fine.
- `featured` (optional): the row shown in the Spotlight, or `null` for the usual mix of films. A standard row cannot be
  featured: `That row cannot be featured in the Spotlight.`

Applied in that order: order, then names, then featured.

### `POST /api/layout`
Body: any of these fields. Answer: `{"ok":true}`, or `{"ok":true,"darkened":"#rrggbb"}` when a colour was too light to
read text on and was stored darker (the stored colour).

| Field | Value |
|---|---|
| `spotShow`, `seasonSpot`, `seasonBg`, `spotRecess` | `true` / `false` |
| `trailers` | `{"spot":…, "rows":…, "info":…}`, each `off`, `silent` or `sound`; any of the three, applied spot, rows, info. `spot` is skipped while the Spotlight is hidden. |
| `bg` | `black`, `film` or `colour`; with `colour`, also `bgColour` as `#rrggbb`. |
| `playAnim` | `discs`, `all` or `off` |
| `pauseCard` | `true` / `false` |
| `delay` | One of `state.delays`, in milliseconds |
| `poster` | One of `state.posters[].key` |
| `topHidden` | The `topMenu` tags to hide; every other entry is shown. `[]` shows them all. |
| `featured` | As in `rows`. |

A field left out is left as it is. The fields are applied in the order of this table (`spotShow` first, `featured`
last); a refused field stops there, and those before it are already saved.

### `POST /api/collections`
Body: `{}`. Your collections, the Home Theater franchises, and the collections made by hand in Home Theater.

```json
{"ok":true,
 "franchises":[{"id":"collection-12","title":"A Franchise"}],
 "collections":[{"id":"local-mh2x9k-a1b2c3","title":"Rainy night","kind":"manual","mediaType":"film","count":12,"on":true,"featured":false,"why":""}]}
```

`homeTheater` (from 1.2.8): `[{"id":"collection-<n>","title":"…","on":false}]`, the collections made by hand in Home
Theater (long hold OK on a film there). Zinedock shows them with your collections; they can be put on Home (`onhome`, `rows`)
and inside a nested collection, and are changed in Home Theater, not here. `on`: on Home.

`kind` as `source` above. `count`: the number of titles in it, or for a `universe` the number of collections in it;
it may be `null` for a rule-based collection. `on`: on Home. `why`: empty, or why Home leaves it out today even when
`on` is `true`: `None of its titles are on this Zidoo.`, `Fewer than 3 whole pairs are on this Zidoo.` (a list made of
pairs, such as a film and its remake), `Nothing on this Zidoo matches its rules.`, `Nothing matches yet: some details are
still to be read.` or `Nothing inside it is on this Zidoo.`

A hand-picked collection keeps the name, year and kind of each title. If the Zidoo gives a title a new id (for example
after the library is added again), Zinedock finds it again by the same name and kind, year within one, and updates the
collection.

### `POST /api/search`
Body: `{"q": "text"}`. Titles in the library whose name contains the text, any case; at most 40. Fewer than two
characters finds nothing.

```json
{"ok":true,"titles":[{"id":"4711","name":"An Example Film","year":1958,"tv":false}]}
```

`tv: true` for a series.

### `POST /api/col-get`
Body: `{"id": "local-…"}`. One of your hand-picked, nested or rule-based collections, with its contents in order.

```json
{"ok":true,"collection":{"id":"local-…","title":"Rainy night","kind":"manual","mediaType":"film",
  "items":[{"id":"4711","name":"An Example Film","year":1958,"tv":false},
           {"id":"999","name":"Another Film (no longer on this Zidoo)","year":1961,"tv":false,"missing":true}]}}
```

A nested collection (`kind: universe`) has `mediaType: mixed`, `rowAs` (`collections`: Home shows one card per
collection inside; `films`: Home shows all their films in one row) and `items` of `{"id","name","collection":true}`.
A rule-based collection (`kind: smart`) answers its rules and the titles that match them now, in the collection's own order:
`{"id","title","kind":"smart","mediaType":"film"|"tv"|"mixed","rules":{…},"complete":true,"items":[{"id","name","year","tv"}]}`
(see [Rule-based collections](#rule-based-collections)). `complete` is `false` while some titles' details are not read
yet, so `items` may grow; `POST /api/build` with `action: read` reads them.

To check a change took effect, read it back: `GET /api/state` for rows and layout, `POST /api/col-get` for a collection's
titles in order.
Other kinds are refused: `Only hand-picked, nested and rule-based collections can be edited here. Use the TV for the others.`
An unknown id: `That collection no longer exists.`

### `POST /api/col-save`
Makes a collection, or changes one. Answer: `{"ok":true,"id":"local-…","homeRow":2}`; `homeRow` is its position on
Home counted from 1 at the top, or `null` when it is not on Home.

| Field | Value |
|---|---|
| `id` | Leave out to make a new collection; give it to change that one in place. |
| `kind` | `manual` (titles you pick), `universe` (a collection of collections) or `smart` (rule-based, which takes other fields: see [Saving a rule-based collection](#saving-a-rule-based-collection)). Anything else: `Choose hand-picked, nested or rule-based.` |
| `title` | 1 to 80 characters. A name is plain text: one with `<` directly before a letter, `/`, `!` or `?` (the start of an HTML tag) is refused: `A name cannot have < directly before a letter. Put a space after it, or leave it out.` `Films <1960`, `&` and quotes are fine. |
| `ids` | At least one. `manual`: title ids from `search`, in the order to show them; films and series may be mixed. `universe`: `local-…` ids of your collections and `collection-<n>` franchises. Ids that are not of that form, repeats, and for `universe` a collection that would end up inside itself, are left out. |
| `homePosition` | New collections only: where on Home it goes, counted from 1 at the top. Default: the top, under Continue Watching. |
| `rowAs` | `universe` only: `films` or `collections` (default). |

`mediaType` is worked out from the titles: films only `film`, series only `tv`, both `mixed`. A film collection cannot
become a series-only one in an edit: `Use a mixed collection to keep existing film and TV selections.`

### `POST /api/col-delete`
Body: `{"id": "local-…"}`. Answer: `{"ok":true}`. Removes one of your collections (and takes it out of any nested
collection). Your films are not touched. Home Theater franchises cannot be deleted:
`Only app-created collections can be deleted here.`

### `POST /api/onhome`
Body: `{"id": row id, "on": true | false}`. Answer: `{"ok":true}`. Puts a row on Home (just under Continue Watching) or
takes it off. Already as asked: `{"ok":true}` and nothing changes.

### `POST /api/suggested`
Body: `{}`. Ready-made collections that match the library, not yet added.

```json
{"ok":true,"suggestions":[{"id":"twin-films","media":"film","title":"Same Idea, Same Year","blurb":"…",
  "have":3,"of":8,"unit":"pairs","names":"Title one, Title two, …"}]}
```

`have`: how many you own, of `of` in the list. **`unit`** says what is counted: `""` titles, `pairs` whole pairs (a
list of films that belong together in twos; a half pair is not counted). A suggestion needs `have` of at least 3.
`names`: up to twelve of the matching titles, comma separated. `media`: `film`, `tv` or `mixed`.

### `POST /api/suggested-add`
Body: `{"id": suggestion id, "media": "film" | "tv" | "mixed"}` (`media` from `suggested`; send it, as one id can be offered for more than one). Answer: `{"ok":true,"id":"local-…","homeRow":1}`.
Adds the suggestion as a hand-picked collection of the matching titles. One that no longer matches:
`That suggestion no longer matches this Zidoo.`

### `POST /api/tmdb-status`, `POST /api/mdb-status`
Body: `{}`. Answer: `{"ok":true,"saved":true}`. `saved` says whether your own TMDB key, or your MDBList key, is saved on
the Zidoo. The key itself is never sent.

### `POST /api/tmdb-save`, `POST /api/mdb-save`
Body: `{"key": "<your key>"}`. Answer: `{"ok":true}`. Saves your own key on the Zidoo, replacing any saved before. The
Zidoo first checks the key with TMDB or MDBList over the internet, so the answer can take a few seconds; a key that does
not pass is refused (`refused`, for example `TMDB did not accept this key. Please check it.`) and the key saved before
stays. The TMDB key is the 32-character **API Key (v3 auth)**, not the long read-access token. A saved key is kept only
on the Zidoo: it is never in a setup file and no call sends it back.

### `POST /api/tmdb-remove`, `POST /api/mdb-remove`
Body: `{}`. Answer: `{"ok":true}`. Removes your saved key. Without your own TMDB key, Zinedock uses its built-in one
(see [People](#people) for what that means for a computer).

### `POST /api/pair-status`
Body: `{}`. Whether paired computers is on, and where a paired computer connects.

```json
{"ok":true,"on":true,"ready":true,"port":9463,"address":"192.168.1.50:9463"}
```

`on`: paired computers is turned on on the TV. `ready`: on, and the port is listening (`false` with `on: true` means
other apps on the Zidoo have the port; the TV says so). `port`: `9463`, or the next free port up to `9472` that this
Zidoo chose and keeps. `address`: the `Host` a paired computer uses. It never lists the paired computers or their number.

### `POST /api/pair`
Body: `{"name": "Study PC"}`. Only after logging in with the code on the TV; a key cannot call it.

```json
{"ok":true,"key":"zdk_0123...","name":"Study PC","port":9463,"address":"192.168.1.50:9463"}
```

Pairs this computer and answers its key, **once**: no call gives it again. `name`: 1 to 40 characters, shown on the TV;
spaces at the ends are dropped. Refused: `pairing_off` when paired computers is off; `refused` for a name that is empty
or too long, a name already paired (`A computer with that name is already paired. Use another name, or remove it on the
TV first.`), or more than 20 paired computers. See [Pairing a computer once](#pairing-a-computer-once).

## Rule-based collections

A rule-based collection keeps up by itself: Zinedock picks the titles that match its rules. The rules, and what they
mean, are the same as on the TV. Genres, studios, certificates and the people in a film come from details Zinedock reads
from the Zidoo one title at a time, so read them first (`POST /api/build` with `action: read`) or let the build read
what is missing.

`media` is `film`, `tv`, or `mixed` for films and TV series together. `mixed` takes the TV rules (genre, title, year,
rating, 4K, HDR, library source, and TMDB people). TMDB names some series genres differently, so a series' `Sci-Fi & Fantasy` matches `Science Fiction`
and `Fantasy`, `Action & Adventure` matches `Action` and `Adventure`, and `War & Politics` matches `War`; for
`mixed`, `smart-choices` offers the film names. A rule a series cannot have is refused, for example
`Films and TV collections have no studio rule: the box keeps no studio for a series.`

### Rules

All are optional. Send only the ones you want.

| Rule | Value |
|---|---|
| `genre` | A list of genre names from `smart-choices` (up to 50). A title matches if it has any of them. |
| `studio` | Text a studio name must contain. Films only. |
| `people` | Up to one person for each role (`director`, `actor`, `producer`); see [People](#people). TV and mixed: TMDB people only. |
| `text` | Text the name must contain (up to 100 characters). |
| `from`, `to` | Years, `0` for no limit. `from` must not be later than `to`. |
| `rating` | Lowest TMDB rating, `0` to `10`; `0` for no limit. |
| `format` | `all`, `4k`, `hd`, `sd`, `disc` or `non4k`. TV and mixed: `all`, `4k` or `non4k`. |
| `hdr` | `all`, `sdr` (no HDR of any kind), `hdr` (any), `hdr10`, `hdr10plus`, `dv` or `hlg`. TV and mixed: `all`, `sdr` or `hdr`. A film counts as `sdr` only once its video details have been read. |
| `maxCert` | Highest certificate: `u`, `pg`, `12` or `15`. Films only. |
| `source` | A list of library sources (up to 20), as the `key` values from `smart-choices`. A film or series matches if one of its files (for a series, its episodes' files) is in any of them. |
| `language` | A list of language codes (up to 40), as the `code` values from `smart-choices`, for example `["it", "ja"]`. A film matches if the language you hear is any of them: its original language when the file has a track in it, otherwise a track the release titles "Original", otherwise its first audio track (commentaries left out). Films only. |
| `mode` | `all` (a title must pass every rule, the default) or `any` (one is enough). |
| `excludeText`, `excludeFormat`, `excludeGenre` | Leave out titles whose name contains this text, of this format, or with any of these genres. Leaving out always wins, also with `mode: any`. |
| `sort` | `title` (default), `year-desc`, `year-asc` or `rating-desc`. |
| `limit` | At most this many titles, `0` to `2000`; `0` for all. |

A rule that is not in this list, or a value out of range, is refused before anything changes, for example
`Unsupported smart rule: colour` or `Check rating (0–10).`

### People

**Standard: names the Zidoo holds.** A director or actor rule is `{"source":"zidoo","role":"director","name":"…"}`, with
a name from `people-search`. It matches the films where the Zidoo lists that person. No online service is used.
The Zidoo keeps no cast or crew for a series, so this kind is for films only.

**With your own TMDB key: TMDB person search.** If you have saved your own free TMDB key in Zinedock (Setup › Online
services on the TV, or API keys on the computer page), `people-search` searches TMDB instead, and a person rule can be a TMDB person,
`{"id":578,"name":"…","role":"producer"}`. Zinedock reads that person's credits from TMDB once and matches them to the
films you own. Producers are only available this way.

**TV and mixed collections.** A TMDB person rule also reads the person's TMDB TV credits. A series on
the Zidoo has no TMDB number, so a series matches by its name and first-air year (within a year). For TV, `director`
means Director or Creator on TMDB, so a director rule finds the shows someone created as well as those they directed
episodes of; `producer` is any job ending in Producer. A rule `col-get` gave back carries `tvShows` (the TV credits,
`[{"n": name key, "y": year}]`) beside `movieIds`; send it back unchanged. A rule without `tvShows` for a `tv` or `mixed`
collection is read again from TMDB, which needs your own key.

**Why a computer needs your own key.** Zinedock's built-in TMDB key is licensed to Zinedock. TMDB's
[terms of use](https://www.themoviedb.org/api-terms-of-use) do not let that licence be passed on to other programs, and
they do not allow TMDB data to be used in connection with an AI-based application. So searches started from a computer,
which may be a script or an AI assistant, never use the built-in key. With your own key, the requests are yours, made
under TMDB's terms as they apply to you, including the clause on AI.

Without your own key, a TMDB person rule and a producer search are refused:
`TMDB person search from a computer needs your own free TMDB key. Add it under API keys on the setup page.`

### `POST /api/smart-choices`
Body: `{"media": "film" | "tv" | "mixed"}` (default `film`). What can be chosen, from the details read so far.

```json
{"ok":true,"media":"film","read":{"ready":180,"total":201},"genres":["Comedy","Drama"],"studios":["A Studio"],
 "sources":[{"key":"smb://192.168.1.10/Films-4K/Movies","label":"Movies (Films-4K/Movies)"}],
 "ownTmdb":false,"reading":null}
```

`read`: how many titles have their details read, of how many. `genres` and `studios` (films only) are in A to Z order
and fill in as titles are read. `sources` (every `media`): the library sources set up in Home Theater; `label` is the name
given in Home Theater with its folder, as two sources can have the same name. `key` is the source's location, which
stays the same if the source is removed and added again, so a rule keeps working; use `key` in a `source` rule.
`languages` (films only; `[]` for `tv` and `mixed`): `{code, label, count}`, the languages of the films read
so far first, with how many films each, then the other common languages with `count` 0. Use `code` in a `language` rule.
`ownTmdb`: `true` when your own TMDB key is saved. `reading`: as in `build` below.

Which source a film or series is in comes from the same details read as genres, so a `source` rule counts unread titles as
`missing` until they are read. A series whose details were read by Zinedock 1.2.7 or earlier is read once more.

### `POST /api/smart-preview`
Body: `{"media": "film" | "tv" | "mixed", "rules": {…}}`. The titles that match now.

```json
{"ok":true,"total":201,"count":2,"missing":21,"missingTmdb":0,
 "items":[{"id":"4711","name":"An Example Film","year":1958,"rating":7.4,"tv":false}]}
```

`total`: how many titles match. `count`: how many of those are in this answer's collection once `limit` is applied
(the same as `total` when there is no limit). `missing`: titles whose details are not read yet, so `count` may
grow. `missingTmdb`: titles with no TMDB id, which a TMDB person rule cannot match. `items`: the first 200 matches, in
the order of `sort`; `rating` is `0` when unknown; `tv` is `true` for a series.

### `POST /api/people-search`
Body: `{"role": "director" | "actor" | "producer", "q": "text"}`, 2 to 100 characters.

Without your own TMDB key, from the names the Zidoo holds (directors and actors only):

```json
{"ok":true,"source":"zidoo","people":[{"name":"A Director","known":"An Example Film, Another Film"}]}
```

With your own key, from TMDB:

```json
{"ok":true,"source":"tmdb","people":[{"id":578,"name":"A Director","department":"Directing","known":"An Example Film"}]}
```

`known`: some titles they are known for.

### `POST /api/build`
Body: `{"action": "status" | "pause" | "resume" | "discard" | "read" | "stop-read", "media": "film" | "tv" | "mixed"}`.
Saving a rule-based collection runs in the background on the TV; this call shows and controls it.

```json
{"ok":true,"build":{"state":"done","message":"","title":"Rainy night","media":"film","ready":201,"total":201,
  "count":12,"homeRow":2},"reading":null}
```

`build` is `null` when there is none. `state`: `running`, `paused`, `error` or `done`; `ready` and `total` count the
titles read so far; `count` (titles in the collection) and `homeRow` (counted from 1, or `null`) come with `done`.
`pause` and `resume` pause and resume it; `discard` drops an unfinished build (nothing is saved) or clears the notice of
a finished one.

`read` (with `media`) reads the details of every title not read yet, without saving anything, so that `smart-choices`
fills in. `reading` then shows `{"media","state","ready","total","message"}`, `state` being `running`, `done`,
`stopped` or `error`; `stop-read` stops it. Refused while a build is running:
`A collection is building on the TV. Its details are shared when it finishes.`

### Saving a rule-based collection

`POST /api/col-save` with `kind: smart`:

| Field | Value |
|---|---|
| `id` | Leave out to make a new one; give it to change that one in place. It must be rule-based: `That collection is not rule-based.` |
| `kind` | `smart` |
| `media` | `film` (default), `tv` or `mixed`. |
| `title` | 1 to 80 characters, plain text as for other kinds. |
| `rules` | As above. |
| `homePosition` | New collections only, as for other kinds. |

The answer comes at once, before the collection is saved: `{"ok":true,"building":true,"build":{…}}`. Zinedock reads
the details still missing, then saves the collection, and puts a new one on Home. Follow it with
`POST /api/build` until `state` is `done`. The TV shows the build, not *Saved from the computer*. Only one build at a
time: `Another collection build is unfinished. Open Collection build to resume or discard it first.`

`POST /api/col-get` on a rule-based collection gives back its rules in the form saved; send them back unchanged to keep
a person rule as it is.

## What Home Theater owns, and what Zinedock adds

Zinedock reads the Zidoo's own library. **Home Theater owns, and Zinedock cannot change:** the library itself; which
files count as one film (the grouping of versions); matching and scraping (an unmatched film needs a `.nfo` file of the
same name with its ids); the posters and backdrops it downloaded; and media analysis, which the badges come from. The
Zidoo offers Zinedock no way to write to any of these.

Films inside a Home Theater franchise, and inside a group you made in Home Theater, are films to Zinedock: they can be
picked for a collection and matched by rules like any other.

A collection made by hand in Home Theater comes across as it is: Zinedock lists it with your collections, with the same
films in the same order, and it follows Home Theater when you change it there.

**Zinedock adds, on top:**

- **Version labels from the file name.** Home Theater labels versions by resolution only. Zinedock reads, in order: a
  Radarr/Plex `{edition-Name}` tag in the file name; otherwise a cut word after the year (Director's Cut, any
  "&lt;word&gt; Cut", Extended, Theatrical, Ultimate, International, Unrated, Uncut, Workprint, IMAX, Remastered,
  Special / Collector's / Anniversary Edition, Criterion); otherwise the resolution. Two versions with the same label are
  told apart by resolution, then HDR type. Up to 10 versions per film; the one chosen is remembered and Play plays it.
- **Missing posters and backdrops.** Only when a picture fails to load from the Zidoo, Zinedock uses the TMDB picture
  named in the Zidoo's own record, or looks the film up on TMDB by its number. Kept for 30 days.
- **Title logo** on a film's page, from a file in the film's folder: `<file name>-clearlogo.png`, `clearlogo.png`,
  `<file name>-logo.png` or `logo.png`. Nothing is fetched.
- **Disc art** for the Play animation, from the film's folder: `<file name>-discart.png`, `<file name>-disc.png`,
  `discart.png` or `disc.png`; otherwise fanart.tv, for the disc being played only.

So version labels and title logos can be set from your own server, through file names and files beside the film, with
no API. Posters, backdrops and which files group together stay Home Theater's.
