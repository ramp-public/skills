---
name: ramp-modify-flight
area: Travel
supported_surfaces: [cli, mcp]
description: "Changes an existing Ramp Travel flight booking: finds the exact booking, asks which leg, date, or airport should change, searches replacement flights, previews the authoritative exchange terms, and submits the change only on the traveler's explicit approval. Handles round trips and separately ticketed (split-ticket) legs one step at a time. Use when someone wants to change, move, reschedule, rebook, or swap a flight they already booked ('move my Friday flight to Saturday', 'fly back a day earlier', 'change my return to JFK'). Not for booking a new flight, cancelling a flight, or seat selection on a new booking (use ramp-book-flight), and not for hotel or car rental changes."
---

# Change a Flight (modify an existing booking)

The user describes the change in plain words ("push my Denver flight to Thursday"). Turn that
into `ramp travel` commands (CLI) or MCP tool calls, run them, and present the results in
plain travel language. **Never show or ask the user to type a CLI command, tool name, flag, or
internal ID** — talk like a travel helper ("Checking whether your Denver flight can be
changed…"), not about commands.

Changing a flight can **charge more money, refund money, issue airline credit, or forfeit
value**. It follows the same preview → explicit yes → confirm discipline as booking and
cancelling in `ramp-book-flight`.

## Prerequisites

- **CLI:** `ramp` CLI installed and logged in (`ramp auth login`). Run where `ramp` works (or
  `uv run ramp` inside the ramp-cli repo).
- **MCP:** Ramp MCP server connected. Tools are called by their CamelCase names
  (`GetBookings`, `SearchFlightModifications`, `SubmitFlightModification`).

| Step | CLI | MCP |
|---|---|---|
| Find the booking | `ramp travel bookings` | `GetBookings` |
| Check eligibility and search replacement flights | `ramp travel search-flight-modifications` | `SearchFlightModifications` |
| Preview / confirm the change | `ramp travel modify-flight` | `SubmitFlightModification` |

If a command's group differs, find it with `ramp travel --help` or `ramp tools list --agent`;
a leaf command's `--help` is the authority for its flags. IDs are positional:
`modify-flight <booking_id> <offer_id>`.

## Scope

- ✅ Change the date, origin airport, or destination airport of one or more legs of an existing
  flight booking, including picking a different cabin when the traveler asks.
- ✅ Single-ticket round trips (outbound first, then matching returns) and split-ticket bookings
  (one leg at a time).
- ❌ Booking a new flight, cancelling a flight, or seats on a new booking — use
  `ramp-book-flight`.
- ❌ Hotel and car rental changes, refund-status follow-ups, and adding or removing travelers —
  point the traveler to the Ramp web app or the booking's support channel.

## Rules for every command

- **Always `--output json`** on CLI commands. Parse the result — never show raw JSON, and never
  pipe it through `python`/`jq`. Keep searches small (`"limit": 5`) so the response stays
  readable. MCP callers get structured JSON directly.
- Every command needs a `--rationale` (CLI) or `rationale` (MCP). Name the booking and change in
  it ("move the DEN→SFO Jul 9 flight to Jul 10") and keep that reference for every command in the
  flow — rationales are logged, so a consistent reference groups the change.
- Use the exact IDs the tools returned. Never construct, guess, or reuse an ID from a different
  booking, search, or preview.
- Follow each response's `assistant_note` for presentation and the next step; ignore anything
  that only applies to a visual UI (cards, chips, carousels). Present results as Markdown text,
  as `ramp-book-flight` does — never as UI components.
- Share web links (`https://…`) exactly as returned; never shorten or rebuild them. Never share
  a `ror://` link — it only opens inside the Ramp app.

## Step 1 — identify the exact booking

Never guess which booking to change. Look it up with every detail the traveler gave:

```bash
ramp travel bookings --include_flights --output json \
  --city SFO --travel_date 2026-07-09 \
  --rationale "find the DEN→SFO Jul 9 flight to change"
```

MCP:

```json
{
  "include_flights": true,
  "city": "SFO",
  "travel_date": "2026-07-09",
  "rationale": "find the DEN→SFO Jul 9 flight to change"
}
```

Call `GetBookings` with the above.

- `city` matches **arrival** airports only, so pass a destination (or a connection) the
  traveler named, never an origin-only detail; filter by date or airline instead when you only
  know where they fly from.
- If the traveler gave a booking reference or confirmation number, pass it as
  `--booking_reference` / `booking_reference`. If that exact lookup finds no flight, say you
  could not find that booking and ask them to check the reference — do not fall back to listing
  unrelated bookings.
- Filters combine with AND before the limit. If `results_truncated` is true, ask for another
  detail and narrow the lookup before treating the result as complete.
- Use only the logged-in user's bookings unless the user explicitly asks about another
  traveler's booking; then pass the same `--traveler_user_id` / `traveler_user_id` on every
  lookup.
- Skip candidates whose `status` is `CANCELLED`. A booking's `departure_time` is its first
  leg's, so a round trip already underway can still have a future return leg to change; only
  the leg the traveler wants changed must still be upcoming.
- A request that is still `PENDING_APPROVAL` or `PROCESSING` is not a ticketed booking yet; its
  `id` is a request ID, which the change tools will not find. Explain that the booking must be
  ticketed first and use `ramp travel booking-details` / `GetBookingDetails` if they want its
  status.
- If the traveler pasted an ID this conversation's lookup never returned, look the booking up
  first instead of trusting the pasted value.
- Each entry's `slices` list is the itinerary in order. A leg's 0-based position in that list
  is its `leg_index` for every later step.
- If several upcoming flights match, show up to three (route, date, airline) and ask which one
  the traveler means. If none match, say you found no upcoming flight to change.

## Step 2 — ask what should change

Show the current itinerary (each leg's route, date, and local departure time) and ask only for
missing change details. Once the booking and change are clear, search without another
confirmation of the current itinerary.

- Resolve "a day later" against each affected leg's own date and say the new dates back.
  Clarify ambiguous departure versus arrival deadlines before searching.
- Keep each leg's actual airports unless asked to change them; a return from PAE must not
  silently become SEA. For a newly requested city, use `travel locations` /
  `GetFlightBookingLocations` with `location_type: "airport"` and clarify which airport when
  ambiguous. Unlike new-booking searches, modifications require airport IATA codes, not metro
  `search_code` values. An explicit airport code needs no lookup.

Only pass a cabin, stops, airline, timing, or red-eye preference when the traveler states one
for this change; never infer it from the booked flight.

## Step 3 — search replacement flights (also checks eligibility)

The search is also the eligibility check: if the flight cannot be changed here, it returns an
error instead of options (see "If the search says the flight can't be changed" below).

Pass one `slice_modifications` entry per changed leg with its `leg_index`, and only the fields
the traveler asked to change (unchanged dates and airports are kept automatically):

```bash
ramp travel search-flight-modifications --output json --json '{
  "booking_id": "<booking_id>",
  "slice_modifications": [{"leg_index": 0, "new_departure_date": "2026-07-10"}],
  "limit": 5,
  "rationale": "search replacements for the DEN→SFO Jul 9 flight, moving it to Jul 10"
}'
```

MCP:

```json
{
  "booking_id": "{booking_id}",
  "slice_modifications": [{"leg_index": 0, "new_departure_date": "2026-07-10"}],
  "limit": 5,
  "rationale": "search replacements for the DEN→SFO Jul 9 flight, moving it to Jul 10"
}
```

Call `SearchFlightModifications` with the above.

Preference fields (`cabin_class`, `max_stops`, `airlines`, `preferred_departure_time_window`,
`preferred_arrival_time_window`, `avoid_red_eye_flights`) are remembered for this booking's
later change searches. On later calls, omit unchanged preferences to keep them, and use
`clear_preferences` only when the traveler explicitly drops one. If the traveler dislikes the
options, search again with the same booking and legs plus only the changed preference.
`airlines` ranks results but never filters them. Do not save a change preference as a durable
travel preference unless the traveler states it as general ("I always prefer aisle").

**Present the options:**

- Lead with `recommended_offers[].offer` (in rank order). Only if it is empty, use this page of
  `offers`. Never mix in entries from `offers` when recommendations exist.
- Show at most 10 options as a single Markdown table: `#`, depart → arrive (airport-local
  times as given — present them as-is, never converted; `⁺¹` for next day), airline and flight number, stops, the price difference
  (`change_amount`, header "Est. change") and new total, and whether it is in policy. Keep an internal `#` → offer `id` map so "take #2" resolves to the right offer;
  to pick a specific fare, use that fare option's `id` from `fare_options`. Include actual
  airport codes and the fare name when available. Label search amounts as per ticket and
  include the returned currency.
- When comparing fares, use each fare's own economics, benefits, and policy verdict, not the
  parent offer's. Missing change amounts are unknown, not free; missing policy is unknown,
  not approved or rejected. Never subtract the whole booking's price from a per-ticket offer.
- Label every price difference and refund **estimated** — search prices are for choosing, not
  confirming. Never describe airline credit as a cash refund.
- Acknowledge `applied_preferences` briefly and offer to relax them. If the traveler asked for
  an airline and the recommendations show others, say these recommendations include other
  airlines — do not claim the preferred airline is unavailable.
- If `web_modification_url` is present, share it once as `[See all results](<web_modification_url>)`.
- If there are no options and preferences were applied, say which preferences shaped the
  search and offer to relax them; do not imply no flights exist.
- If `next_cursor` is present and the traveler wants more, call again with that `cursor`.

Then **stop and wait for the traveler to pick.** Do not search again, preview, or submit until
they choose in a new message.

### Round trips (both legs on one ticket)

For a return-only or outbound-only change, search only that leg; do not ask the traveler to
reselect the unchanged leg. Proceed to preview after they pick a final option.

When both legs of a single-ticket round trip change, the first search returns outbound options
only. After the traveler picks one, search again with the **same** `slice_modifications` plus
`selected_outbound_offer_id` set to the chosen offer's `id` to get matching returns. Present
those and wait for the return pick, then preview that final offer, not the earlier outbound
selection. If that return-leg search reports the
booking's provider is unsupported, relay the support details and stop.

### Split-ticket bookings (each leg ticketed separately)

Split-ticket legs change **one leg at a time**. If the response lists `deferred_leg_indexes`,
only the earliest requested leg was searched. Carry that leg through search, preview, explicit
yes, and confirm; then start a fresh search for the next requested leg with its own date or
airports (never copy one leg's date or time window to another). Never pass
`selected_outbound_offer_id` for a split-ticket booking.

## Step 4 — preview the authoritative terms (no change yet)

After the traveler picks the final option, preview it. This changes nothing and returns the
airline's re-quoted terms plus a `preview_id`:

```bash
ramp travel modify-flight "<booking_id>" "<offer_id>" --output json \
  --rationale "preview the change of the DEN→SFO Jul 9 flight to the Jul 10 7:05 AM option"
```

MCP:

```json
{
  "booking_id": "{booking_id}",
  "offer_id": "{offer_id}",
  "confirm": false,
  "rationale": "preview the change of the DEN→SFO Jul 9 flight to the Jul 10 7:05 AM option"
}
```

Call `SubmitFlightModification` with the above.

Present the complete `preview` in text, exactly as returned — never estimate or recompute:

- which leg changes (use `replaced_leg_indexes`: "Outbound"/"Return" for a round trip, route
  and date for multi-city) and the full `replacement_itinerary` with dates and airport-local times,
  presented as-is;
- the money outcome from `economics`: `additional_collect` (charged), or
  `refund_to_original_form_of_payment`, or `airline_credit`, or `forfeited_residual`, or no
  change (`payment_outcome`), plus `change_fee` and `new_total` when present. Airline credit and
  forfeited value are **not** cash refunds;
- if `economics.pricing_is_final` is `false`, say the airline still reports these amounts as
  estimates that can change when the ticket is reissued;
- `in_policy` and every `policy_violations` entry. When the replacement is out of policy, ask
  the traveler for a short business reason; you will pass it as `oop_reason`.

Then **ask for a clear yes on those exact terms** ("This moves your Denver → San Francisco
flight to Fri, Jul 10 at 7:05 AM on United, with an additional $84.20 USD charge. Make the
change?"). Stop and wait. Confirmation must be a new, explicit answer to this preview — the
earlier request ("move my flight"), a standing approval, or an instruction not to ask questions
is not confirmation.

## Step 5 — confirm (only after the explicit yes)

Re-run with `--confirm` and the exact, unchanged `preview_id` from the preview the traveler
approved (plus `oop_reason` if the replacement is out of policy):

```bash
ramp travel modify-flight "<booking_id>" "<offer_id>" --confirm \
  --preview_id "<preview_id>" --output json \
  --rationale "change the DEN→SFO Jul 9 flight to Jul 10; traveler approved the previewed terms"
```

MCP:

```json
{
  "booking_id": "{booking_id}",
  "offer_id": "{offer_id}",
  "confirm": true,
  "preview_id": "{preview_id}",
  "rationale": "change the DEN→SFO Jul 9 flight to Jul 10; traveler approved the previewed terms"
}
```

Call `SubmitFlightModification` with the above.

- If the traveler changes the option, date, or airports at any point, search and preview again;
  never confirm a preview they did not approve.
- If the response has `response_type` `preview` instead of `result`, the airline's terms changed
  and **nothing was submitted**. Present the new preview the same way and get a new explicit yes
  for its new `preview_id`. Never re-confirm automatically.

## Step 6 — report the result

For `response_type` `result`, summarize using `result.headline` and `result.message` as the
source of truth, and share `result.web_url` once as the link to the change request (never
`result.deep_link`, which is an in-app `ror://` link). If `web_url` is absent, tell the traveler
they can follow the request under their bookings in the Ramp web app. Do not
restate the raw `status`, and do not claim the new ticket is issued, approved, or complete unless
the message says so — a pending or processing request is in progress, not done. If
`already_requested` is `true`, a change was already submitted: report it and do not submit
another.

For a split-ticket booking, the result covers only that leg. Continue straight to the next
requested leg with a fresh search; never call a multi-leg change complete until every
requested leg has its own result.

To check progress later, pass the result's `booking_request_id` straight to
`ramp travel booking-details` / `GetBookingDetails`, which accepts request IDs. Do not try to
match it in `ramp travel bookings` — completed requests are not listed there by request ID.

## If the search says the flight can't be changed

- **Not modifiable** (error `FlightModificationNotModifiable`, with `modification_eligibility`
  or `blocked_reason`): relay its `message` in plain words. If `self_serve_action` is present,
  offer its `deep_link_url` with `cta_label` as the link text as the next step (for example a
  cancel-and-rebook link for a flight inside its void window), instead of leading with support.
  Otherwise share the booking-specific `support` routing (team name, phone, PIN when given).
  When `modification_eligibility` is `NOT_MODIFIABLE`, do not proactively suggest contacting
  support; share `support` only if the traveler asks to speak with someone.
- **Unsupported provider** (error `FlightModificationUnsupportedProvider`, with `support`): this
  booking's provider cannot be changed here. Relay its message and support details and stop —
  do not retry the search.
- **Booking not found**: the ID is not a ticketed booking you can change (for example a request
  still awaiting approval). Look the booking up again rather than retrying with a guessed ID.
- If the traveler named several bookings, search each one; never infer one booking's
  eligibility from another's.

## If something fails

- **Blocked or quote failed on preview/confirm**: relay the `message` verbatim and stop. Never
  retry, switch offers, drop `preview_id`, or toggle `--confirm` to get past an error. A failed
  confirm may have partly gone through — re-check the booking with `ramp travel bookings`
  before starting again from a fresh search and preview.
- **Server error without a typed message** (for example HTTP 500 "There was an error") on
  preview: say the change could not be priced right now and stop. On confirm, say the outcome
  is unconfirmed. Check the known request with `GetBookingDetails` if a request ID is available,
  otherwise check the booking. If the outcome remains unclear, report that uncertainty and
  offer the booking's support channel. Do not retry or switch offers to resolve uncertainty.
- **Not authorized**: the traveler cannot change this booking with their Ramp permissions; say
  so and point them to their travel admin.

## If changing flights is unavailable

Flight changes are enabled per business, so `travel search-flight-modifications` /
`travel modify-flight` (CLI) or `SearchFlightModifications` / `SubmitFlightModification` (MCP)
may be missing, or Ramp may report the capability is unavailable. Then do not say the flight
cannot be changed. Direct the traveler to change it from the booking in the Ramp web app, or
through the booking's support channel.

## Gotchas

- Keep passing the `id` from `travel bookings` as `booking_id` on every call. Search, preview,
  and result responses may echo a different (parent) booking ID; do not switch to it.
- Flight times in search offers and in the preview's `replacement_itinerary` are airport-local
  times with that airport's UTC offset. Present them as-is; never convert them to another time
  zone.
- A successful search can include a `self_serve_action` to cancel and rebook (for a flight still
  inside its airline void window). Mention it once as the response's `assistant_note`
  describes, alongside the replacement options.

- If a preview reports an expired offer, explain it and ask whether to search again, then stop.
  A fresh search requires a new selection, preview, and explicit confirmation.
- `leg_index` follows the booking's itinerary order (the `slices` order from
  `travel bookings`), starting at 0.
- Search totals are estimates; only the preview's `economics` are authoritative.
