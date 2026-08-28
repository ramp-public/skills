---
name: ramp-book-flight
area: Travel
supported_surfaces: [cli, mcp]
description: "Books flights conversationally: resolves cities to airports, searches one-way and round-trip flights, presents and compares offers, previews the fare, and tickets the booking on the traveler's explicit approval. Also cancels an existing flight booking with a preview-then-confirm flow when the cancellation capability is enabled. The user describes a trip in plain language ('book a flight from Toronto to SFO') and never needs to know a CLI command or tool name. Use when someone wants to book, find, search, or compare flights, says 'fly from X to Y', or wants to cancel a flight they booked. Not for changes, refund-status follow-ups, seat selection, hotels, cars, or multi-city trips."
---

# Book a Flight (conversational flight search)

The user describes a trip in plain words. Turn that into `ramp travel` commands (CLI) or
MCP tool calls, run them, and show clean results. **Never show or ask the user to type a CLI
command or tool name** — talk like a travel helper ("Searching Toronto → San Francisco,
Jul 1…"), not about flags or tool names.

The Steps, Phases, checklist, and flag names in this guide are **your** internal scaffolding —
never surface them to the user. Don't say "Phase 1," "Step 3," or name flags; just narrate in
plain travel language ("Let me pull up the fare and check it before booking…").

## Prerequisites

- **CLI:** `ramp` CLI installed and logged in (`ramp auth login`). Run where `ramp` works (or
  `uv run ramp` inside the ramp-cli repo).
- **MCP:** Ramp MCP server connected. Tools are called by their CamelCase names
  (`SearchFlights`, `SubmitFlightBooking`, `SubmitFlightCancellation`, etc.).

## Scope

- ✅ Resolve cities to airports; search one-way and round-trip; show offers.
- ✅ Round-trip: fetch the matching **return** flights for the outbound the user picks (Step 5).
- ✅ **Book the ticket** — preview, confirm only on an explicit yes, then verify (Step 6).
  Booking spends **real money**; it defaults to the logged-in user unless the user explicitly
  asks to book for another traveler.
- ✅ Compare cabins/fares — **only when asked** (see "Cabin and fare options").
- ✅ Read or update the traveler's profile, and read trips and bookings (see "Supporting tools").
- ✅ Book for another traveler when explicitly asked and authorized (see "Delegated booking").
- ✅ **Cancel an existing flight booking** — preview the exact terms, confirm only on an explicit
  yes (see "Cancelling a flight booking"). Availability is per-business; degrade gracefully when
  the command is missing.
- ❌ Changes/modifications, refund-status follow-ups after a cancellation, seat selection,
  hotels, cars, and multi-city are outside this flow — point the traveler to the Ramp web app
  or the booking's support channel instead.

## Rules for every command

- **Always `--output json`** on every `travel search-flight` call (including the Step 5 return
  search). Parse it into a friendly table — never show raw JSON, and never pipe through
  `python`/`jq`. MCP callers get structured JSON directly; the same table rule applies.
- Every command needs a `--rationale` (CLI) or `rationale` (MCP). Once the trip is known, name it
  in every related command's rationale (route + dates, e.g. "Toronto→SFO, Jul 1-8") and keep that
  same reference through search, returns, preview, book, and verify — rationales are logged, so a
  consistent trip reference groups a trip's commands.
- `--departure`/`--arrival` (CLI) or `departure`/`arrival` (MCP) each take **one** value: an
  airport code (`SFO`) or a Ramp city id (`search_code` from Step 2). No lists.
- **Both surfaces present results as Markdown tables** — never as UI components, cards,
  interactive widgets, bullet lists, numbered lists, or prose. A Markdown table is the only
  acceptable presentation format for flight offers, fare comparisons, and fund displays.
- `cabin_class` — offer **Economy**, **Premium Economy**, **Business**, **First** (API values
  `ECONOMY`, `PREMIUM_ECONOMY`, `BUSINESS`, `FIRST`; never offer Basic Economy, though it can
  still appear as a returned fare). Pass one value, or a list for alternatives like "Business or
  First." **CLI:** required before the first search — ask it as part of Step 1's grouped
  question. **MCP:** omit on the first search unless the traveler already gave a cabin; ask for
  it after kickoff and send it on the next `job_id` read (Step 3).
- `include_fare_options` — always on, including pagination and the return-leg search.
- `wait_for_results` — **CLI:** always `true` (blocks until results are ready). **MCP:** always
  `false` (returns immediately with a `job_id` and `search_complete: false`; gather preferences
  and poll as described in Step 3) — **except** the Step 5 return-offer read, which is always
  synchronous and rejects `false` on both surfaces.

These stay optional — add only when the traveler asks:

- `--limit` — off returns a default page. **Show at most 10 flights** in the results table;
  if the response contains more, present the first 10 and tell the traveler they can ask for
  more. Paginate with `next_cursor` (Step 3) when they do.
- `--sort_key` — off uses `WEIGHTED_SCORE`. Other keys: `LOWEST_TOTAL_AMOUNT`,
  `SHORTEST_DURATION`, `LEAST_NUMBER_OF_STOPS`, `EARLIEST_DEPARTURE_TIME`,
  `LATEST_DEPARTURE_TIME`, `EARLIEST_ARRIVAL_TIME`, `LATEST_ARRIVAL_TIME`. Applies to a new
  search only — re-sort by starting fresh, not on a `job_id` page.
- Use `ramp travel search-flight` (CLI) or `SearchFlights` (MCP) for flight searches; check
  `ramp travel --help` if the CLI alias is unavailable.

## Delegated booking

Only set `traveler_user_id` when the user explicitly asks to book for another person. Otherwise
omit it everywhere and book for the logged-in user.

When the user asks to book for someone else:

1. Resolve that traveler before profile preflight or search:

```bash
ramp users list --name_search "Taylor Smith" --page_size 5 \
  --rationale "resolve the traveler for the Toronto→SFO trip" --output json
```

MCP:

```json
{
  "name_search": "Taylor Smith",
  "page_size": 5,
  "rationale": "resolve the traveler for the Toronto→SFO trip"
}
```

Call `GetAllReducedUsers` with the above.

Use the returned traveler user UUID as `traveler_user_id`. If more than one user could match,
ask the requester to pick the exact traveler before continuing.

2. Preserve the same `traveler_user_id` across the whole delegated flow: profile preflight,
   `profile-update` if needed, every `search-flight` / `SearchFlights` call (initial search,
   resume pages, and round-trip return search), both `travel book` / `SubmitFlightBooking`
   preview and confirm calls, and booking verification/retries. Do not switch traveler ids
   mid-flow.

3. If traveler lookup fails or a target-aware call returns an authorization/empty-access result,
   respond in plain travel-helper language. Do not mention `traveler_user_id`, flags, command
   names, command counts, or CLI mechanics. Say you couldn't find/access that traveler and ask
   whether to continue for the requester or search again with a Ramp email/exact profile name.
   Example: *"I couldn't find a traveler named Emmy Song in this Ramp directory. If you want to
   book for yourself, I can continue using your own traveler profile. Otherwise, send me the
   traveler's Ramp email or exact profile name and I'll search again."* Do not silently fall back
   to self-booking.

## Step 1 — gather trip details

Use what the user gave you; infer the rest.

- **CLI:** gather every required detail plus cabin and other preferences together in one grouped
  question before searching. Never search first and ask about preferences later.
- **MCP:** gather only the required details before searching — destination, origin, dates, trip
  type. Cabin and other preferences wait until after the search starts (Step 3).

Infer silently, then say back (don't ask):

| Slot | Assume |
|---|---|
| **Trip type** | round-trip if there's a return date, "back on…", or a stay length; else one-way. Ask only if truly unclear. |
| **Relative dates** | resolve to `YYYY-MM-DD` and always say the date back so mistakes surface before money moves. |
| **Airport given** | `SFO`, `JFK`, etc. → use directly, skip Step 2. |

Say assumptions in one line as you go — *"Searching JFK → SFO, Mon Jul 6, round-trip…"*.

If a required detail is still missing — plus, on CLI, cabin and other preferences — ask for all
of it in one grouped question (selectable options, one question per item). Required:
**destination**, **origin** (if no home airport to guess), **departure date**, **one-way vs
round-trip** (if unclear), and **return date** (round-trip). Keep dates in the future — 14+ days
out is safest (a common policy cutoff). Never re-ask what they told you.

## Step 2 — resolve a place to a `--departure`/`--arrival` value

- **Airport code** (`SFO`, `JFK`) → use directly, skip this step.
- **City/vague place** ("New York", "the Bay Area") → look it up first:

```bash
ramp travel locations --query "New York" --location_type city --limit 5 \
  --rationale "resolve New York to a metro id for the user's trip" --output json
```

MCP:

```json
{
  "query": "New York",
  "location_type": "city",
  "limit": 5,
  "rationale": "resolve New York to a metro id for the user's trip"
}
```

Call `GetFlightBookingLocations` with the above.

For a **city**, the metro id is its **`search_code`** (a UUID; `iata_code` is empty). Pass
that one `search_code` to search the whole metro (New York covers JFK/LGA/EWR; Toronto covers
YYZ/YTZ). For one specific airport, search `--location_type airport` and pass its `iata_code`.

## Step 3 — search flights

Add `--return_date` only for round-trips and `--traveler_user_id` only for delegated bookings.
Pass `cabin_class`, `include_fare_options`, and `wait_for_results` per "Rules for every command."

**When the traveler named a departure weekday** ("leave Sunday", "out next Friday"), also pass
`--requested_weekday` with the lowercase day (`sunday`), if the flight-search command lists it
(skip it on an older CLI). The server rejects the search when the departure date doesn't fall
on that weekday, and the error names the correct nearby dates — retry with the corrected date
from the error; never clear the error by changing the weekday. It's departure-only: for an
**arrival** day ("be home by Sunday"), leave it off — the right flight may depart the day
before (a red-eye) — and let the Step 4 date strings show both days. If the rejection, or phrasing like "Sunday night", leaves it ambiguous whether the
traveler means late Sunday or a just-after-midnight Monday departure, ask them which they mean
instead of silently moving the date.

```bash
ramp travel search-flight --output json \
  --departure YYZ --arrival SFO \
  --departure_date 2026-07-01 --return_date 2026-07-08 \
  --cabin_class ECONOMY --include_fare_options --wait_for_results=true \
  --rationale "search flights for the Toronto→SFO trip, Jul 1-8"
```

MCP:

```json
{
  "departure": "YYZ",
  "arrival": "SFO",
  "departure_date": "2026-07-01",
  "return_date": "2026-07-08",
  "include_fare_options": true,
  "wait_for_results": false,
  "rationale": "search flights for the Toronto→SFO trip, Jul 1-8"
}
```

Call `SearchFlights` with the above; include `cabin_class` only if the traveler already gave one.

### MCP: kick off search first, then gather preferences

1. The initial call above returns immediately with `search_complete: false` and a canonical
   `job_id` — do not present empty offers as final; the search is still running. (It may already
   return `search_complete: true` with offers; if so, skip straight to Step 4 and present them.)
2. In that same turn, ask the 1-4 most important unresolved preferences in one grouped
   question: cabin class (always ask when not already known), then timing, airline, nonstop,
   fare-tier, and price-versus-schedule preferences when relevant — the same set described in
   "Rules for every command." Never mention that a search was started, is running, or is likely
   to finish soon; lead directly into the questions. Stop after asking and do not call
   `SearchFlights` again this turn.
3. On the traveler's next turn, process every answer, then call `SearchFlights` again with the
   same `job_id` and `wait_for_results=false`. Resend `include_fare_options` and every currently
   known preference — including `cabin_class` — since neither persists on the job on its own;
   the traveler's newly given answers just add to that resent set. Omit `departure`, `arrival`,
   `departure_date`, and `return_date` — the job retains the route and dates from the first call.
   If the traveler **withdraws** a previously stated preference (e.g. "actually, airline doesn't
   matter"), pass that field name in `clear_preferences` so the job drops it; never silently omit
   a removed preference and never send a placeholder value.
4. If that response is still incomplete, acknowledge any applied preferences without narrating
   search status or timing, ask one more concise question about any remaining materially useful
   preference, then stop again. Limit preference gathering to **at most two turns** of questions
   after the kickoff; if the search is still incomplete after that, simply wait for the next user
   turn to poll without asking more questions. Once `search_complete` is `true`, present the full
   results as in Step 4.

If a `SEARCH_STILL_RUNNING` error includes a `job_id`, apply the same rule and resume that job
only on a later user turn. Never sleep, back off, or present incomplete offers as final.

```json
{
  "job_id": "{job_id_from_initial_response}",
  "cabin_class": "ECONOMY",
  "include_fare_options": true,
  "wait_for_results": false,
  "rationale": "apply the traveler's cabin preference to the Toronto→SFO flight search"
}
```

### CLI synchronous search

The CLI response (with `--wait_for_results=true`) normally contains the complete
ranked set. Check `search_complete` before presenting it. If it is `false`, do not
call search again in the same turn: explain that the search is still running and
wait for the traveler to ask to continue. On that later turn, re-call with the
response's canonical `job_id` and the same `cabin_class`/`include_fare_options`
settings. If a `SEARCH_STILL_RUNNING` error includes a `job_id`, apply the same
rule and resume that job only on a later user turn. Never sleep, back off, poll,
or present incomplete offers as final.

Once complete, empty `offers` = no match — tell the user and offer to change dates/airports.
(Output is also token-capped, so a long result may page: pass the canonical `job_id` and
`next_cursor` as `--cursor` to fetch more, only if the user wants beyond the first page.)

**Show at most 10 flights** in the results table. If the response returns more than 10
offers, present only the first 10 (in returned order) and tell the traveler more are
available on request. Never present more than 10 rows at once.

**On both surfaces, for round-trips, save this search's `job_id`** for the return search in
Step 5.

## Step 4 — show the offers

Turn the JSON into **one Markdown table** — never as prose, a bullet list, a numbered list,
or a plain sentence list. The table is the only acceptable presentation format for flight
results on both CLI and MCP. Keep each offer's `id` out of the table (you need it for
returns/booking). Each `offers[]` item has exactly these keys (don't invent others): `id`,
`airline_name`, `flight_number`, `departure_airport`/`arrival_airport`,
`departure_time`/`arrival_time`, `departure_date`/`arrival_date` (for the `⁺¹` next-day mark),
`duration`, `stops`, `price`, `in_policy`/`policy_reason`, `fare_name` (free-text fare label),
`ancillaries`, and `fare_options` (the per-fare grid — **present only when you passed
`--include_fare_options`**).

| # | Airline | Flight | Fare | Depart → Arrive | Duration / stops | Price (round-trip total) | Policy |
|---|---------|--------|------|-----------------|------------------|--------------------------|--------|
| 1 | JetBlue | B6 0115 | Blue Basic | 6:00 AM → 9:15 AM | 6h 15m / Nonstop | **$289** | ✓ |

- **#** — the row's on-screen position (top = `1`, no gaps), not the JSON index. If you
  reorder (e.g. cheapest in-policy first), renumber top to bottom. Keep a private `#`→`id`
  map so "book #3" resolves correctly.
- **Depart → Arrive** — local times; add `⁺¹` when arrival is next-day.
- **Fare** — show the selected fare's `fare_name` when present; otherwise `—`. For a
  fare-options comparison, use the selected row's fare name, category, and price rather than
  the parent offer's cheapest-fare values.
- **Duration / stops** — combine the returned `duration` and `stops` (for example,
  `6h 15m / Nonstop`).
- **Ancillaries** — when fare benefits matter to the comparison, summarize each returned
  `ancillaries` entry using its `display_name`, `offer_type`, and `price` when present. Show
  `INCLUDED`, `CHARGEABLE`, or `NOT_INCLUDED` as returned; a missing category is unknown and
  must not be presented as excluded. Apply the same rule to nested fare-option ancillaries.
- **Price** — always show. Round-trip header says **"round-trip total"** (covers both legs);
  one-way says **"Price"**. Say which in words.
- **Policy** — when `fare_options` is present, use the applicable nested fare's
  `in_policy`/`policy_reason`, never the parent offer's usually-null verdict. For the initial
  offer row, use a nested fare only when the top-level offer `id` exactly matches that fare's
  `id`; otherwise treat policy as unknown until the traveler selects an exact fare row. Never
  match fares by price, amount, currency, or array position. After the traveler chooses a fare,
  use that exact fare-option row. Render `true` as **✓**,
  `false` as **✗** plus the returned reason, and `null` as **—** (unknown, not out of policy).
  When `fare_options` is absent, use the offer-level verdict.

Above the table, lead with the route and travel date, taking the weekday from the offers'
`departure_date` strings (weekday included, e.g. "Mon, Jul 13, 2026" → **"SFO → EWR — Mon,
Jul 13"**). The day you show must come from the API's date strings — never pair the traveler's
words ("Sunday") with a date you computed. Judge a mismatch against the day the traveler
actually named: a departure day against `departure_date`, an arrival day against
`arrival_date` — a Saturday red-eye arriving Sunday **matches** "be home by Sunday". On a real
mismatch, re-search with the corrected date instead of presenting these offers. For an
arrival-day request, show each offer's `departure_date` **and** `arrival_date` as returned so
the traveler sees both days. If `search_policy_summary` has text, show it once as a short banner.
Do not invent recommendation reasons or infer policy from price, cabin, or approval data.

When the response includes `web_search_url`, end the results message with one final markdown link
labeled `See all results` pointing at the exact returned URL. Never rewrite, re-encode, shorten,
or substitute any part of it, and never use another label.

## Step 5 — round-trip: confirm the outbound, then fetch returns

Round-trips only. **Wait for the user to name the outbound.** Don't guess or default to
cheapest/first. Ask *"Which outbound do you want? I'll pull the matching returns once you
pick."* If vague ("the morning one") and more than one fits, confirm the exact flight.

The return step is a **second flight-search call** — same command, with the chosen outbound's
`id` as `--outbound_offer_id` plus the Step 3 `job_id` (carries outbound
context for return-policy). For delegated bookings, also pass the same `--traveler_user_id`.
Don't pass `--departure`/`--arrival`/dates again — mixing them with `--outbound_offer_id` is
rejected.

```bash
ramp travel search-flight --output json \
  --outbound_offer_id "<chosen_outbound_offer_id>" \
  --job_id "<job_id_from_step_3>" \
  --include_fare_options --wait_for_results=true \
  --rationale "return offers for the chosen outbound, Toronto→SFO trip Jul 1-8"
```

MCP:

```json
{
  "outbound_offer_id": "{chosen_outbound_offer_id}",
  "job_id": "{job_id_from_step_3}",
  "cabin_class": "ECONOMY",
  "include_fare_options": true,
  "wait_for_results": true,
  "rationale": "return offers for the chosen outbound, Toronto→SFO trip Jul 1-8"
}
```

Call `SearchFlights` with the above.

**Return-offer reads are always synchronous, on both CLI and MCP** — unlike the outbound search,
this call rejects `wait_for_results=false`; always pass `true` and use the response directly.

The response is `is_round_trip: true` with `offers` being the return legs. (Return mode is
synchronous, so its `job_id` is null; page more returns by reusing `--outbound_offer_id` +
`--cursor`.) Show them like Step 4, including the **max 10 rows** cap. **Each return offer's
price is the full round-trip total** — say so (*"the nonstop keeps your trip at $289; the
1-stop return makes it $396 total"*). For policy, use the applicable nested fare verdict as
in Step 4; only omit the Policy column when that applicable verdict is unavailable. The id
you carry to booking is the chosen **return** offer's `id`.

## Step 6 — book (preview → confirm → verify)

Always three steps; never book in one shot, never assume a yes, never book a flight the
traveler didn't name. Booking spends **real money**.

Before previewing or confirming a booking, check whether the traveler already has a Ramp
travel profile:

```bash
ramp travel profile --output json \
  --rationale "check whether the Toronto→SFO Jul 1-8 trip traveler profile is ready for booking"
```

MCP:

```json
{
  "rationale": "check whether the Toronto→SFO Jul 1-8 trip traveler profile is ready for booking"
}
```

Call `GetTravelerProfile` with the above.

If `has_profile` is `false`, collect the required traveler details in one message, then update
the profile before continuing. Use the tool to save the details the traveler gives you; do not
send them to the Ramp web app for this.

For delegated bookings, pass the same `traveler_user_id` from the user lookup flow to
`travel profile` / `GetTravelerProfile` and, if needed, `travel profile-update` /
`UpdateTravelerProfile`. For self-booking, omit `traveler_user_id`.

```bash
ramp travel profile-update --output json \
  --first_name "Taylor" \
  --last_name "Smith" \
  --date_of_birth "1990-01-15" \
  --email "taylor@example.com" \
  --phone_number "+14155550123" \
  --id_gender MALE \
  --rationale "create the traveler profile needed to book the Toronto→SFO Jul 1-8 trip"
```

MCP:

```json
{
  "first_name": "Taylor",
  "last_name": "Smith",
  "date_of_birth": "1990-01-15",
  "email": "taylor@example.com",
  "phone_number": "+14155550123",
  "id_gender": "MALE",
  "rationale": "create the traveler profile needed to book the Toronto→SFO Jul 1-8 trip"
}
```

Call `UpdateTravelerProfile` with the above. `id_gender` is required (`MALE` or `FEMALE`); the
profile update fails without it.

Only continue when the update succeeds. Confirm the profile-update result reports success, or
re-run `travel profile` to verify the traveler now has a profile before moving on. For delegated
bookings, re-run it with the same `traveler_user_id`. If the update fails, correct the missing
details and retry instead of continuing to booking.

Once the profile exists, continue to the normal preview, confirmation, and verification flow.

**Which id:** one-way → the chosen offer's `id` from `search-flight` / `SearchFlights`;
round-trip → the chosen **return** offer's `id` from Step 5 (it represents the whole
round-trip and its both-legs total — **not** the outbound id). Pass it as the first arg
(`ramp travel book "<id>"` on CLI). Behind it is **`flight_offer_uuid`**, so a `--json` body
or MCP `SubmitFlightBooking` call uses key `flight_offer_uuid`, not `offer_id`.

### Phase 1 — preview (always first)

Run `book` **without `--confirm`** — that returns the preview and books nothing.
For delegated bookings, pass the same `--traveler_user_id` used for profile preflight/search.

```bash
ramp travel book "<flight_offer_uuid>" --output json \
  --rationale "preview fare for the Toronto→SFO Jul 1 trip before the traveler confirms"
```

MCP:

```json
{
  "flight_offer_uuid": "{flight_offer_uuid}",
  "confirm": false,
  "rationale": "preview fare for the Toronto→SFO Jul 1 trip before the traveler confirms"
}
```

Call `SubmitFlightBooking` with the above.

Show plainly: traveler (`traveler_name_display` when present), route/dates, airline/flight,
cabin/fare (`itinerary.fare_name` when present), payment timing (`payment_display` when
present), **total**, policy result, fare details, and the paying fund. If `loyalty_programs`
is present, show each matching program's `display_name` and only the last four characters of
its `loyalty_number`; do not expose its logo URL or full loyalty number. A preview without
`spend_allocation_id` auto-uses `recommended_fund_uuid` when an eligible fund is available.
Label a fund `<fund name> (recommended)` only when the tool auto-populated that
recommendation; a user-selected fund never gets that label, even when its UUID matches.

**Fare details:** when the preview returns `ancillaries_by_slice` (or equivalent ancillary
data), present a compact breakdown using each ancillary's `display_name`, `offer_type`, and
`price` when present. Group them as:
- **Included:** ancillaries with `offer_type=INCLUDED` — show `display_name` only.
- **Costs extra:** ancillaries with `offer_type=CHARGEABLE` — show `display_name` and `price`.
- **Not included:** ancillaries with `offer_type=NOT_INCLUDED` — show `display_name` only.
Omit each group when it has no entries. Do not infer ancillary terms that the preview did not
return.

The preview returns `eligible_funds`, `fund_eligibility_status`, and `selected_fund_uuid` when a
fund was explicitly passed; a non-null `selected_fund_uuid` is the booking fund. If
`fund_eligibility_status=lookup_failed`, do not confirm; repeat
the preview to resolve funding. If it is `none_eligible`, the valid path is to request new funds.
If the traveler chooses or changes to an eligible fund, call preview again with `confirm=false`
and that fund's `fund_uuid` as `spend_allocation_id`; present the refreshed preview and wait for
a separate explicit confirmation turn.

Use `approval_display_status` verbatim for approval messaging. Do not infer the wording from
`requires_approval` or `approval_steps` alone.

If `loyalty_program_names_to_offer` is non-empty and this is a self-booking, ask whether to save
one of the returned programs and stop for the answer before asking for booking confirmation. Save
only the exact returned program name with the membership number the traveler provides. For a
delegated booking, do not offer or attempt to save loyalty; the save action targets the requester,
not the selected traveler. If the traveler saves a program, run a fresh preview and require fresh
confirmation.

The preview returns the itinerary dates as weekday-qualified strings — **`outbound_date`**
(e.g. "Mon, Jul 13, 2026") and, for round-trips, **`return_date`**. The read-back must quote
them **exactly as returned, weekday included** — never re-derive the weekday or repeat one
from earlier conversation. These are departure dates — check them against a departure day the
traveler named; for an arrival day ("be home by Sunday"), check the chosen offer's
`arrival_date` instead and read that day back too (a Saturday-departing red-eye arriving
Sunday is correct). On a real mismatch, **stop and re-search (Step 3) with the corrected
date** — don't ask for confirmation. If the preview doesn't include these fields, verify each
ISO travel date with Python's calendar instead (use `python` if only that executable is
available):

```bash
python3 -c 'from datetime import date; import sys; dates = map(date.fromisoformat, sys.argv[1:]); print("\n".join("{}: {} {}, {}".format(d.isoformat(), d.strftime("%A, %B"), d.day, d.year) for d in dates))' 2026-07-06 2026-07-10
```

Then state the total in plain words and **ask for a clear yes**. The confirmation prompt must
include the preview's date string for every leg, along with the local departure time: *"This
books LHR → JFK on Delta, departing Mon, Jul 6, 2026 at 10:00 AM, for **$412 total**, paid
from the Travel fund. Book it?"* For a round-trip, include both outbound and return dates and
times. Stop and wait.

### Phase 2 — confirm (only after a clear "yes")

Add `--confirm` and pass the preview's numeric `expected_total_amount` verbatim as
`--expected_total_amount` (with no currency symbol). This rejects the booking if the fare moved
instead of quietly charging more. Do not copy the display-formatted `total_amount`.
For delegated bookings, pass the same `--traveler_user_id` used in the preview.

```bash
ramp travel book "<flight_offer_uuid>" --confirm \
  --expected_total_amount <preview_expected_total_amount> --output json \
  --rationale "book the Toronto→SFO Jul 1 trip; traveler approved the previewed fare"
```

MCP:

```json
{
  "flight_offer_uuid": "{flight_offer_uuid}",
  "confirm": true,
  "expected_total_amount": "{preview_expected_total_amount}",
  "spend_allocation_id": "{fund_uuid_from_latest_preview}",
  "rationale": "book the Toronto→SFO Jul 1 trip; traveler approved the previewed fare"
}
```

Call `SubmitFlightBooking` with the above. Include exactly one funding path: `spend_allocation_id`
(the exact fund from the latest preview, including the recommended UUID when the preview
auto-populated it) or `request_new_fund: true` (with `reason` as the trip purpose). Never omit
both and never pass both.

Extra flags, only when they apply:

- **`--spend_allocation_id <fund_uuid>`** — use the exact fund from the latest preview, including
  the recommended UUID when that preview auto-populated it.
- **`--request_new_fund=true`** plus **`--reason "<trip purpose>"`** — use only when the latest
  preview showed the new-fund path. `reason` is the trip purpose shown to approvers, not a
  generic booking note.
- **`--oop_reason "<justification>"`** — required for an out-of-policy quote.
- **`--trip_id <uuid>`** — attach to an existing trip; off to auto-pick/create.

Confirmation requires exactly one funding path: `spend_allocation_id` or
`request_new_fund=true`. Never omit both and never pass both. Preserve the exact funding path
from the latest preview.

If `confirm=true` fails for **any** reason, stop. Relay the error `message`, follow
`agent_guidance`, and never retry, tweak parameters, switch offer/fare, or confirm again without
a fresh preview and a fresh explicit confirmation. A price-change error therefore requires a
new preview and new approval; it is not permission to retry the confirmation.

### Phase 3 — verify it went through

**The confirm response is optimistic, not final** — it can say `approved`/`pending_approval`
and still fail in fulfillment. Don't say "you're booked" off the confirm alone:

On a successful confirmation, retain the exact `booking.booking_request_id` from the response.
Use it to select the matching entry from `travel bookings`; never select an older entry by route,
flight number, or timestamp.

```bash
ramp travel bookings --include_flights --output json \
  --city SFO --travel_date 2026-07-01 \
  --rationale "verify the Toronto→SFO Jul 1 booking reached a terminal status"
```

MCP:

```json
{
  "include_flights": true,
  "city": "SFO",
  "travel_date": "2026-07-01",
  "rationale": "verify the Toronto→SFO Jul 1 booking reached a terminal status"
}
```

Call `GetBookings` with the above.

Use the known city, airline, flight number, and travel date as lookup filters. They combine
with AND and are applied before the result limit. Flight-number matching ignores spaces,
punctuation, case, and leading zeroes in the numeric portion. If `results_truncated` is true,
add another known filter and retry rather than treating the returned entries as exhaustive.

For delegated bookings, pass the same `--traveler_user_id` when verifying and on every retry;
otherwise `travel bookings` checks the requester's bookings.

Each `travel bookings` entry has a generic `id` and a `booking_request_id`. Match the retained
confirmation `booking_request_id` exactly, then use that matching entry's generic `id` with
`travel booking-details` for detailed status questions. Do not fuzzy-match by route, flight number,
or `booked_at`. If the matching request is missing from the default result, retry once with
`--include_failed`; if it is still missing, report that verification could not locate the submitted
request rather than using another entry. Report the matching entry's `status`. Cancelled, rejected,
and failed requests do not block rebooking. Most read
for themselves (`CONFIRMED`, `PENDING_APPROVAL`, `CANCELLED`). Two need care:

- **`PROCESSING`** is **not final** — report that fulfillment is still processing; do not report
  it as booked yet.
- **`FAILED`** — show `error_message` exactly. If it points to missing traveler details, use
  `travel profile` / `GetTravelerProfile` and `travel profile-update` /
  `UpdateTravelerProfile` with the same traveler target to complete the profile before a new
  booking attempt.

`travel booking-details` (CLI) / `GetBookingDetails` (MCP) is flag-gated by
`OMNI_TRAVEL_BOOKING_SUPPORT_SKILL_ENABLED`. If it is not available, degrade gracefully with
the information from `travel bookings` / `GetBookings` rather than claiming the detailed
lookup succeeded. When available, relay `request_status`, `current_total_amount`,
`error_message`, and `approval.pending_approval_summary` verbatim when approval is pending.

## Cabin and fare options

Every search response already returns the fare grid because `include_fare_options=true` is
always on. The cabin answer — collected before searching on CLI, or applied to the running job
on MCP (see Step 3) — determines which cabins are policy-evaluated. If the traveler asks to
broaden or change cabins, re-read the existing `job_id` with the new `cabin_class` and
`include_fare_options=true`; do not start a new route search unless the route, dates, trip, or
traveler changed.

`include_fare_options: true` adds each flight's bookable fare classes to `fare_options` (each
with `fare_name`, `fare_category` — `Basic Economy`/`Economy`/`Economy Plus`/`Premium`/
`Business`/`First` — `price`, `in_policy`/`policy_reason`, and a bookable fare `id`).

The offer's top-level `price` is the **cheapest** fare; `fare_options` lists the upgrades. To
book a specific fare, pass that fare option's `id`, **not** the offer's top-level `id`.

Send the always-on search fields via the command flags when available, otherwise via the `--json`
body:

```bash
ramp travel search-flight --output json --json '{
  "departure": "JFK", "arrival": "SFO",
  "departure_date": "2026-07-06", "return_date": "2026-07-10",
  "cabin_class": "ECONOMY", "wait_for_results": true,
  "include_fare_options": true,
  "rationale": "compare cabin/fare classes for the JFK→SFO trip, Jul 6-10"
}'
```

Here cabin is already known because the traveler explicitly asked for a cabin/fare comparison,
so MCP callers can include `cabin_class` on this fresh search too; still set
`wait_for_results=false` and poll as in Step 3:

```json
{
  "departure": "JFK",
  "arrival": "SFO",
  "departure_date": "2026-07-06",
  "return_date": "2026-07-10",
  "cabin_class": "ECONOMY",
  "include_fare_options": true,
  "wait_for_results": false,
  "rationale": "compare cabin/fare classes for the JFK→SFO trip, Jul 6-10"
}
```

Call `SearchFlights` with the above.

Re-send `include_fare_options` and the active `cabin_class` on every follow-up call (pagination,
the cabin refinement, and the Step 5 return search); these settings do not persist on their own.
MCP callers keep using `wait_for_results=false` and poll on outbound follow-up calls, but the
Step 5 return search is always synchronous — pass `wait_for_results=true` there instead.

Present a **single cabin-grid Markdown table** — one row per flight, one column per cabin
category — ordered by departure time (a comparison, not a ranked list). Never present this
as a bullet list or prose:

| # | Depart → Arrive | Airline / Flight | Basic Econ | Economy | Econ+ | Premium | Business |
|---|-----------------|------------------|-----------|---------|-------|---------|----------|
| 1 | 7:00 AM → 9:49 AM | DL 0742 | **$154** | **$209** | $354 | $759 | $3,849 |

- **#** — display position (top = `1`); keep an internal `#`→per-cabin-fare-`id` map so
  "book #2 in economy" resolves to the right fare option `id`.
- **Depart → Arrive** — local times, `⁺¹` for next-day.
- One column per `fare_category`; cell = that fare's `price`, `—` if the flight doesn't sell it.
- **Bold = in policy** (`in_policy: true`); plain otherwise; render `null` plain (don't claim
  out-of-policy). Add a one-line legend.

Above the table, give the lead line (cheapest fare anywhere + the highest in-policy cabin) and
make the upgrade math explicit (*"Basic Economy $289, in policy; Economy $399; Business
$1,101, out of policy"*).

## Supporting tools (profile, trips, bookings)

Five supporting tools; use when relevant, not on every booking.

- **`travel profile`** (CLI) / `GetTravelerProfile` (MCP) — the traveler's saved profile (name,
  email, phone, DOB, gender, KTN/TSA, redress, loyalty). Use for "what's my known traveler
  number?" and before booking to check whether `has_profile` is true.
- **`travel profile-update`** (CLI) / `UpdateTravelerProfile` (MCP) — saves missing traveler
  details before booking when `travel profile` / `GetTravelerProfile` returns `has_profile:
  false`, or when a failed booking points to missing traveler details.
- **`travel list`** (CLI) / `GetUserTrips` (MCP) — the traveler's trips
  (`--status completed|ongoing|upcoming`, `--cursor` / `status`, `cursor`). Each has `id`,
  `trip_name`, dates, locations. Use to find a trip `id` for `--trip_id` / `trip_id` on `book` /
  `SubmitFlightBooking`. Note: delegated trip lookup is not exposed on MCP; `GetUserTrips`
  returns the caller's own trips only.
- **`travel bookings`** (CLI) / `GetBookings` (MCP) — existing flight/hotel bookings
  (`--include_flights`/`--include_hotels`, `--city`, `--hotel_name`, `--flight_airline`,
  `--flight_number`, `--travel_date`, `--limit` / their corresponding MCP fields). Supplied
  filters for the same booking type combine with AND before the limit; mixed hotel- and
  flight-specific filters match each type independently. Each entry has a `status`
  (`CONFIRMED`/`PENDING_APPROVAL`/`FAILED` + `error_message`), route/times, and
  `trip_name`/`trip_id`. If `results_truncated` is true, narrow the filters before treating the
  result as exhaustive. **Source of truth for whether a booking succeeded** (Phase 3) and for
  "what flights do I have booked?".
- **`travel booking-details`** (CLI) / `GetBookingDetails` (MCP) — detailed status for one
  exact `travel bookings` / `GetBookings` entry ID when the booking support capability is
  available. Use it for `request_status`, `current_total_amount`, `error_message`, and
  approval details; relay a pending `pending_approval_summary` verbatim. If unavailable,
  degrade gracefully to `travel bookings` / `GetBookings`.

## Cancelling a flight booking

Cancelling forfeits or spends real money, so it follows the same preview → explicit yes →
confirm discipline as booking. The command is `ramp travel cancel-flight` (CLI) /
`SubmitFlightCancellation` (MCP); it is enabled per business, so it may be absent for some
accounts (see "If cancellation is unavailable").

### Identify the exact booking first

Never guess which booking to cancel. Resolve it from `ramp travel bookings --include_flights
--output json` (same `--traveler_user_id` for delegated travelers), passing every known `city`,
`flight_airline`, `flight_number`, and `travel_date` filter, and use the exact entry `id` — the
same exact ID the skill already uses with `travel booking-details`. If more than one booking
could match ("cancel my SFO flight"), show the likely matches and ask which one; never pick by
route, date, or recency on your own. If `results_truncated` is true, ask for another identifying
detail and run a narrower lookup before selecting a booking. If the traveler pasted an ID that
this conversation's `travel bookings` never returned, look the booking up first instead of
trusting the pasted value.

Cancellation applies to a fulfilled booking. For an unfulfilled request (e.g.
`PENDING_APPROVAL`), the entry `id` is the request UUID, which this command will not find —
direct the traveler to the request in the Ramp web app instead.

### Preview the terms (read-only)

Always call without `--confirm` first. This books/cancels nothing and returns the authoritative
terms plus a `preview_id`:

```bash
ramp travel cancel-flight --booking_id "<booking_id>" --output json \
  --rationale "preview cancellation terms for the Toronto→SFO Jul 1 booking"
```

MCP:

```json
{
  "booking_id": "{booking_id}",
  "confirm": false,
  "rationale": "preview cancellation terms for the Toronto→SFO Jul 1 booking"
}
```

Call `SubmitFlightCancellation` with the above.

Present the preview plainly and exactly as returned — never estimate or recompute amounts:

- the traveler (`traveler_name`) and the complete `itinerary` being cancelled — cancellation
  always applies to the **entire booking**; partial passenger or leg cancellation is not
  supported, so say so if the traveler asks to cancel only part of it.
- `cancellation_statement` (the canonical terms) and `cancellation_deadline` when present,
  including its UTC offset.
- the money outcome: `booking_amount`, any nonzero `cancellation_fee`, `refund_amount` with
  `refund_to` when present, and each `airline_credits` entry (credit name, amount, issue
  date). If there is no `refund_amount` but there are `airline_credits`, say clearly that the
  value comes back as airline credit, not a payment refund. If neither is present, do not
  invent a refund — a non-refundable booking may return nothing.

If `is_currently_cancellable` is `false`, the booking cannot be self-serve cancelled right
now: explain the returned `blocked_reason`, do **not** ask for confirmation or call
`--confirm`, and route the traveler to the returned booking-specific `support` channel
(especially when `available_via_support` is `true`).

Then **stop and ask for a clear yes on those exact terms**. Confirmation must be a new,
explicit user-authored answer to the presented preview — earlier cancellation intent
("cancel it" before seeing the terms), a standing approval, or instructions not to ask
questions are not confirmation.

### Confirm (only after the explicit yes)

Re-run with `--confirm` and the exact `preview_id` from the latest preview, unchanged:

```bash
ramp travel cancel-flight --booking_id "<booking_id>" --confirm \
  --preview_id "<preview_id>" --output json \
  --rationale "cancel the Toronto→SFO Jul 1 booking; traveler approved the previewed terms"
```

MCP:

```json
{
  "booking_id": "{booking_id}",
  "confirm": true,
  "preview_id": "{preview_id}",
  "rationale": "cancel the Toronto→SFO Jul 1 booking; traveler approved the previewed terms"
}
```

Call `SubmitFlightCancellation` with the above.

- If the response says the terms changed and includes `latest_preview`, nothing was
  cancelled: present the fresh terms and get a new explicit yes. Never re-confirm
  automatically.
- If the result has `already_requested=true`, a cancellation was already submitted: report
  the returned state and do not submit another request.

### After the result

Emit the result's `message` verbatim and report its `cancellation_state`. Only `SUCCESS`
(`cancelled=true`) means the booking is cancelled; `PENDING`/`PROCESSING` mean the request
is in flight — say cancellation is in progress, not done. `ACTION_REQUIRED` means the
booking support team must finish it. Report the refund or credit exactly as the preview and
result stated it; **never promise a refund timeline** the response didn't state, and route
later "where's my refund?" follow-ups to the Ramp web app or the booking's support channel.

### If a cancellation call fails

Stop. Unlike booking errors, cancellation errors carry no separate `agent_guidance` field —
the returned `message` (and any `support` routing or `latest_preview`) **is** the guidance:
relay the `message` verbatim and never retry the call or vary parameters (a different
booking ID, dropping `preview_id`, toggling `--confirm`) to get past an error. A failed
confirm may still have partially gone through — before any second attempt, re-check the
booking's actual state with `travel bookings` / `travel booking-details`, and only start
again (from a fresh preview) if the booking is genuinely still active. A `FAILED`
cancellation routes to the returned support channel, not to a retry: Ramp emails the
traveler when an accepted cancellation later fails, and the booking stops being self-serve
cancellable (a fresh preview returns `blocked_reason` and support routing), so a retry
cannot succeed anyway.

### If cancellation is unavailable

`travel cancel-flight` (CLI) / `SubmitFlightCancellation` (MCP) is enabled per business. If the
command/tool is missing or Ramp reports the capability is unavailable, do not say the booking
can't be cancelled — say self-serve cancellation isn't enabled here and direct the traveler to
the booking in the Ramp web app or the booking's support channel. Cancellation also only covers bookings this flow could have
made: an unsupported provider (e.g. a Priceline-fulfilled flight) or a guest booking returns
an error with support routing — relay it and point the traveler there.

Changes, rebooking, and seat or date modifications are **not** cancellations and stay outside
this skill — send the traveler to the Ramp web app or the booking's support channel for those.

## Gotchas

- Offers expire from the cache. If a return or book call fails with a cache/offer error,
  re-run the search (Step 3) for fresh offers and continue.
