# topbox-pinger

Scheduled pings for TopBox background jobs — `/api/cron/dispatch` and
`/api/cron/care`, both every 5 minutes. Lives in its own public repo because
public-repo Actions minutes are free; the target URL and auth secret are
repository secrets — nothing sensitive is in this code.

`/api/cron/backup` is *not* pinged from here — it runs on Vercel Cron
(`vercel.json`, daily at 08:00 UTC), which fits inside the Hobby plan's
one-cron-per-day limit. That limit is the reason the 5-minute jobs live here.

## Why one trigger an hour instead of twelve

`schedule:` is best-effort. GitHub sheds scheduled runs when its queue is busy and
sheds high-frequency ones hardest, so the original `*/5` dispatch cron was not
running every 5 minutes — measured over its first week (2026-07-22 to 07-29):

| | intended | actual |
|---|---|---|
| dispatch `*/5` | every 5 min | 61 min median gap, worst 2h21m (~8% of runs delivered) |
| care `7 * * * *` | hourly | ~6 of 12 fired, worst gap 3h06m |

Endpoints were healthy the whole time — every run that landed was green in 7–17s.
The runs simply weren't triggered.

So the workflow now asks for **one** trigger an hour and covers the hour itself with
a `sleep 300` loop. It needs a single trigger to land rather than twelve, and one has
landed every hour in practice. Dispatch granularity goes from ~61 min back to ~5 min.

## Why care rides the same 5-minute tick

Both routes were read before this change (`src/app/api/cron/*/route.ts` in the
`topbox` repo) to confirm repeated calls are safe. Every side effect sits behind a
persisted once-only marker:

- **morning report** — `org.digestLastSentAt`, compared by dealer-local date, so once
  per org per day. The five aggregate counts are *inside* that guard, so repeat calls
  are cheap, not just harmless.
- **stale-alert escalation** — selects `remindedAt: null`, then stamps `remindedAt`.
  One reminder per alert.
- **48h nudge** — selects `nudgedAt: null`, then stamps `nudgedAt`. Opt-outs get
  stamped too, so they aren't rechecked.
- **dispatch** — selects `status: "scheduled"` and marks `"sending"` before the Twilio
  call.

Since care is idempotent *and* cheap when idle, running it every 5 minutes instead of
hourly is a straight win: the morning report lands within 5 minutes of 9:00
dealer-local rather than at :07 of whichever hour survived shedding, escalations and
nudges fire within 5 minutes of their 4h/48h thresholds, and a transient Twilio error
self-heals on the next tick instead of an hour later. Out-of-window orgs short-circuit
before any per-org query, so the idle cost is small.

## The one real hazard: overlapping loops

dispatch's `findMany` → `update({status:"sending"})` is not a compare-and-swap. Two
*simultaneous* invocations can both select the same scheduled send and both text it.
The handler's comment covers a different case — a function dying mid-batch, which the
mark-before-send order does handle.

Hence `cancel-in-progress: false`. Cancelling would kill the curl client but *not* the
serverless function already running on Vercel, so a cancelled run could overlap its
replacement. Queueing avoids that: GitHub holds at most one pending run per
concurrency group, and a newer pending run replaces the older one, so only one loop is
ever in flight. The cost is a coverage seam — if the next trigger is shed, dispatch
pauses until one lands — which is strictly better than double-texting a customer.

## Other notes

- A blip doesn't cost the rest of the hour: the loop keeps ticking and fails the step
  at the end, so a genuinely broken route still turns the run red. `--retry 2` absorbs
  transient 5xx and timeouts first.
- Response bodies are echoed into the run log, so each tick shows real counters
  (`digests`/`escalations`/`nudges`, `sent`/`failed`/`suppressed`/`deferred`).
- `care` has no per-org `try`/`catch` — one throwing `sendSms` 500s the whole handler
  and skips the orgs after it. The 5-minute cadence softens that from an hour-long
  outage to one tick, but it's worth fixing in the `topbox` repo.
- If Actions still isn't tight enough, the durable fix is to move scheduling off it —
  Cloudflare Workers cron or cron-job.org are purpose-built, free at this volume, and
  support auth headers. That trades this repo's simplicity for a second place to hold
  `CRON_SECRET`.
