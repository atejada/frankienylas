# frankienylas

A [Frankie](https://github.com/atejada/Frankie) stitch over the
[Nylas](https://nylas.com) v3 API — Email, Calendar, and Contacts in one
connected account. Zero dependencies: built entirely on Frankie's own
`http_get`/`http_post`/`http_put`/`http_delete` + JSON, the same stdlib
every other stitch uses.

> The Frankie official / Nylas unofficial stitch

## ⚠️ v1 — API key only

This first version was built while Frankie's language itself is
[frozen](https://github.com/atejada/Frankie/blob/main/CLAUDE.md) (Blag's
writing *The Book of Frankie* against a fixed language version), which
kept the scope deliberately small: **API key + grant ID only, no OAuth**.

Full OAuth 2.0 — the hosted-auth flow so *your users* can connect their
own mailbox without you ever touching their password — is on the roadmap
for v2, once the freeze lifts and a proper `frankiec` web-redirect story
can be designed around it. If you need end-user OAuth today, this isn't
that yet.

## Setup

You need two things from Nylas:

1. **An API key** — authenticates your application.
2. **A grant ID** — identifies which connected mailbox to act on.

Get both from the [Nylas Dashboard](https://dashboard-v3.nylas.com/) or
the `nylas` CLI (`nylas init`). See [Nylas's own
quickstart](https://developer.nylas.com/docs/v3/getting-started/) if
you're starting from zero.

## Install

Stitches install straight from a raw URL — no registry, no publishing step:

```bash
frankiec stitch install https://raw.githubusercontent.com/atejada/frankienylas/main/stitches/frankienylas.fk --global
```

If you want it to be installed on a single project, then use:

```bash
frankiec stitch install https://raw.githubusercontent.com/atejada/frankienylas/main/stitches/frankienylas.fk
```

That drops `frankienylas.fk` into `./stitches/` and pins it in
`stitch.lock`. From then on:

```ruby
stitch "frankienylas"
```

## Usage

```ruby
stitch "frankienylas"

nylas = nylas_client(env("NYLAS_API_KEY"))   # region: "us" (default) or "eu"
grant = env("NYLAS_GRANT_ID")

# List
inbox = nylas_list_messages(nylas, grant, params: {limit: 10})
inbox.each do |m|
  puts m["subject"]
end

# Send
nylas_send_message(nylas, grant, {
  to: [{email: "friend@example.com"}],
  subject: "Hello from Frankie",
  body: "Sent with frankienylas."
})

# Search contacts
matches = nylas_list_contacts(nylas, grant, params: {email: "friend@example.com"})

# Create a calendar event
nylas_create_event(nylas, grant, "primary", {
  title: "Standup",
  when: {start_time: 1_735_000_000, end_time: 1_735_003_600}
})
```

Every call raises `NylasApiError` on a non-2xx response — with Nylas's
own error message when the API provides one — and lets network-level
`RuntimeError`s (timeouts, DNS failures) propagate as-is:

```ruby
begin
  nylas_get_message(nylas, grant, "bad-id")
rescue NylasApiError e
  puts "Nylas said no: #{e}"
end
```

More runnable examples in [`examples/`](examples/).

## Method reference (v1 scope)

Every function takes a `client` (from `nylas_client`) and a `grant_id`
as the first two arguments — omitted below for brevity.

| Resource | Functions |
|---|---|
| **Messages** | `nylas_list_messages(params:)`, `nylas_get_message(id)`, `nylas_update_message(id, body)`, `nylas_send_message(body)` |
| **Threads** | `nylas_list_threads(params:)`, `nylas_get_thread(id)`, `nylas_update_thread(id, body)` |
| **Drafts** | `nylas_list_drafts(params:)`, `nylas_get_draft(id)`, `nylas_create_draft(body)`, `nylas_update_draft(id, body)`, `nylas_delete_draft(id)` |
| **Folders** | `nylas_list_folders(params:)`, `nylas_get_folder(id)`, `nylas_create_folder(body)`, `nylas_update_folder(id, body)`, `nylas_delete_folder(id)` |
| **Calendars** | `nylas_list_calendars(params:)`, `nylas_get_calendar(id)`, `nylas_create_calendar(body)`, `nylas_update_calendar(id, body)`, `nylas_delete_calendar(id)` |
| **Events** | `nylas_list_events(calendar_id, params:)`, `nylas_get_event(calendar_id, id)`, `nylas_create_event(calendar_id, body)`, `nylas_update_event(calendar_id, id, body)`, `nylas_delete_event(calendar_id, id)`, `nylas_rsvp_event(calendar_id, id, status)` |
| **Contacts** | `nylas_list_contacts(params:)` (pass `{email: "..."}` to search), `nylas_get_contact(id)`, `nylas_create_contact(body)`, `nylas_update_contact(id, body)`, `nylas_delete_contact(id)` |

Not in v1: Attachments, Scheduled Send/Templates/Smart Compose, Free/Busy
& Availability, and anything under Scheduler/Notetaker/Agent Accounts
(separate Nylas products). Good candidates for v2 alongside OAuth.

## Design notes

- One private helper (`_nylas_request`-style: `_nylas_get`/`_nylas_post`/`_nylas_put`/`_nylas_delete`)
  builds every request; the public `nylas_*` functions are thin named
  wrappers over it. Response envelopes (`{request_id, data: ...}`) are
  unwrapped automatically — you get the resource straight back, not the
  envelope.
- Uses `json_parse(resp.body)` rather than `resp.json()`. Root cause,
  for whoever patches core Frankie: `compiler/codegen.py` special-cases
  `.json` as a no-parens property access (correct for `FrankieRequest.json`
  in a web route handler) *before* it ever reaches the generic zero-arg
  fallback (`_fk_attr_or_method`, which already correctly calls a method
  if it's callable and just returns a plain attribute otherwise — this is
  exactly what resolves the same ambiguity everywhere else). Because both
  branches match on the method name alone with no receiver-type info at
  compile time, the property branch always wins for `resp.json()` too, so
  the call silently never fires and you get a bound-method object back
  instead of a hash. Deleting that one early-return branch (~5 lines) and
  letting `.json` fall through to the generic dispatch should fix both
  cases at once. `json_parse(resp.body)` sidesteps it entirely here in
  the meantime and is equally correct.
- `stub()`/`unstub()` can't reach into a stitch's internals from outside
  it — `stitch "x"` (and `require`) execute the loaded file in an
  isolated globals snapshot and copy the resulting functions back up, so
  a stub applied from the caller never affects what `http_get` resolves
  to *inside* the stitch's own functions. `test.fk` here runs against a
  real local `web_app()` mock server instead — no network access needed,
  and it exercises the actual HTTP + JSON round-trip rather than a faked
  one.

## Testing

```bash
frankiec test test.fk
```

20 tests, all against a local mock server on `127.0.0.1` — no API key
or network access required.

## License

GPL-3.0, matching Frankie itself.
