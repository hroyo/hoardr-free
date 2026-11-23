# PLAN_REBUILD — Hoardr from first principles

## What Hoardr is

Hoardr is a marketplace for free stuff, monetized like an ad network.

The wedge: a personal agent that watches free-stuff feeds so you don't
have to — it filters junk and alerts you only to gems, each alert
carrying a receipt good enough that you trust it without re-checking the
feed yourself. And the demand doesn't just come from Hoardr's own
watchers: **your personal agent — the one that knows you best — can tell
Hoardr what you need.** "My human is furnishing a nursery" becomes a
want on the board without the human ever opening the app. That agent
aggregates the scarcest resource in the market: **attention from people
who take free things**, plus their declared and revealed wants — the
targeting profile no listing site has ever collected.

It's three-sided by design. Recipients get certainty ("N scanned · 3
worth your time"); givers get their stuff gone fast to a matched
recipient; buyers — brands, local businesses, other agents — pay to
reach exactly these people. The recipient and the payer are different
parties: the signature ad-network shape.

The match engine is the asset, and it runs twice: **client↔stuff** (what
does this human want?) and **agent↔agent**. Agent-to-agent is the
emphasis, not a side feature: personal agents declare their humans'
needs to Hoardr, query the judged supply, and pay by the call for
verdicts and match data (the LNP market). Hoardr is the venue where
"my person wants X" meets "someone is giving X away" — both sides can
be agents. Every match produces evidence — claimed, passed, gone — that
retunes the engine. The listings are commodity; the judge and the demand
graph it accumulates are the moat.

So: free stuff is the content, the receipt is the trust, attention is
the product, and ads/intros/agent queries are the business.

## Taglines

- **"Free stuff you didn't know you ABSOLUTELY needed."**
- **"Doomscrolling, fixed."**
- **"In the future, everything will be free."**
- "94 scanned. 3 worth your time."
- "Your agent already found it."
- "We dig the free pile so you don't have to."
- "Someone's trash, algorithmically confirmed treasure."
- "The whole internet's curb, curated."

> "If you go out long enough, assuming there's a continued improvement
> in AI and robotics, which seems likely, money will stop being
> relevant at some point in the future… I think at some point currency
> becomes irrelevant."
> — Elon Musk, U.S.-Saudi Investment Forum, Nov 2025

## The three sides

| Role | Who | Gets | Pays |
|---|---|---|---|
| Recipient | the human watcher — or their personal agent, declaring needs on their behalf | keeper alerts + receipts | attention + declared/revealed wants |
| Supplier | the giver — or a promoter running a giveaway campaign (credits, coupons, compute) | their thing gone fast, to a matched person — or targeted distribution for the campaign | nothing — supply is free |
| Buyer | brands, local businesses, other agents | access to revealed demand | money — ads, intros, query calls |
| Operator | us | the demand graph | builds and runs the judge |

## Core scope

- **Contact DM.** The transaction. A match that can't reach the other
  side isn't a marketplace. Recipients message givers — and buyers reach
  recipients — through Hoardr itself.
- **Ads.** The business. Attention that can't be sold is just a nice
  tool. Buyer access to the demand graph — placements, intros, or paid
  agent queries — is part of the first-principles definition.

## What Hoardr must do

The functional contract — the capabilities the app ships with:

- **Watch** — continuously ingest free-stuff supply from every relevant
  source; normalize, dedupe, drop what's already gone. Supply is
  physical **and digital**: a startup's promo campaign giving away
  inference credits, compute time, or coupons is a giveaway too — no
  pickup, no photos, instant handoff.
- **Run autonomous loops** — digital supply closes the loop without a
  human round-trip: a promoter's campaign agent posts the giveaway, a
  recipient's agent has declared the want, the match fires, the agent
  claims and redeems — end to end, unattended. Physical claims keep the
  human's one-tap approval; digital ones don't need it.
- **Judge** — every listing gets the battery; typed answers → code →
  verdict + receipt. Noise dies free; spend escalates only on promise.
- **Alert** — keepers reach the right person at their chosen strictness.
  "N scanned · M worth your time" is always answerable.
- **Remember wants** — wants are first-class records: declared (by the
  human, or by their agent on their behalf) and revealed (what they
  claimed, passed on, kept). Queryable, decaying, honest.
- **Match** — pair judged supply to live demand, person↔item and
  agent↔agent. Every match explains itself in one sentence.
- **Contact** — recipient↔giver DMs inside Hoardr, with replies and
  pickup coordination flowing back through it.
- **Sell demand** — buyer placements and paid agent queries
  (search · judge · match · watch) against real billing.
- **Prove** — every verdict is a receipt; every receipt is auditable;
  every metric traces to the ledger.

## Build for production

This ships the real thing:

- **Live availability** — match only to items still there; stale supply
  is a bug, not a caveat. Mechanism: every listing carries `lastSeenAt`,
  `status`, and a per-source `freshUntil` TTL; match filters on
  `ACTIVE + fresh`; polls mark feed-absent items `GONE`/decay to `STALE`;
  and a Trigger.dev `revalidate` step does a single live check (HEAD or
  API lookup) on each candidate *before* any alert, DM, or claim —
  polling keeps the pool warm, point revalidation guarantees the action.
  Receipts record `verifiedAt`; digital supply validates at redemption.

  At scale: **poll indexes, not items** (one feed request covers
  hundreds of listings; diff on IDs, fetch details only for deltas);
  **adaptive per-source cadence** (poll rate learns real churn, backs
  off quiet sources); subscribe over pull where feeds/webhooks exist;
  and **revalidation budget follows live demand** — items with zero
  matched wants may sit stale at near-zero cost, hot items get checked
  aggressively. Optimistic TTL + verify-on-action is how scaled
  marketplaces work; there is no global freshness truth to maintain.
- **Real contact** — recipients DM givers through Hoardr; replies and
  pickup coordination come back through it.
- **Real monetization** — ad placements and paid agent queries with
  actual billing, not vouchers-for-show.
- **Comp-backed pricing** — resale estimates cite real sold listings.
- **Supply breadth** — multiple sources, deduped into one pile.
- **Always-on** — scheduled scans, push on keepers; nobody should have
  to press "scan."

## Stack

Agent-first: the API is the product; apps are clients.

| Layer | Pick | Notes |
|---|---|---|
| Agent API (primary surface) | REST + MCP server + signed webhooks | Hosted agents (ChatGPT actions) call plain HTTP; capable agents use MCP |
| Mobile | Expo (React Native) | Cross-platform; widgets via react-native-android-widget + expo-apple-targets |
| Web | Thin Next.js on Vercel | Landing, giver reply pages (no account), receipt links, promoter dashboard later |
| Database | Postgres + pgvector on **Neon** | One store: users, agents, wants, listings, judgments, matches, DMs, meters. Branching = eval corpora from prod clones; shared-storage read replicas = demand-graph analytics offload. Scale-to-zero off on prod. |
| Orchestration | **Trigger.dev Cloud** | Durable loops: cron polls, watch→judge→match→contact→redeem, `wait.forToken()`/`wait.forEvent()` for human approvals + DM replies, retries, step replay. Replaces Workflow SDK + hand-rolled CAS state. Managed — never self-hosted on the scrape box. |
| Execution plane (scrapers + computer use) | **No VPS — API-managed compute only** | HTTP/`impit` actors run as Trigger.dev tasks on managed compute; Patchright/Playwright + computer-use sessions run as **Agent37 sandboxes** (BYO Docker, per-minute, auto-sleep). Everything is disposable; if compute dies, Trigger.dev holds state and replays. **Hard rule: no SSH boxes — every layer must expose logs/ops via API/CLI so the agent can operate and debug it.** Escape hatch if sustained scraping outgrows per-minute sandboxes: Fly.io Machines (API/CLI-visible, no SSH) — not a VPS+Tailscale |
| Scraping | DIY mini-Apify, tiered | Per-source actors (`fetch → parse → Listing`); `impit`/TLS-fingerprinted HTTP first → Patchright only on JS/bot walls. Proxies tiered: **static ISP (unlimited bw) first → metered residential only on 403/429**. Apify kept as per-source fallback |
| Judge | AI SDK + Gateway + Jev | `experimental_decide` battery → typed answers → code decides → receipt. `judgeListing` is a port — model swappable. Deterministic regex/geo pre-filter kills obvious junk before the model spends anything |
| Metering & payments | **Limitr engine (Apache-2.0, in-process Wasm) + dual credentials** | ~1ms policy checks in-app, no billing-API round-trip; policy-as-data = live repricing; margin measurement tracks Jev cost vs verdict price. LNP vouchers for capable agents; metered API keys for hosted agents (Stripe-funded). Stripe for human/promoter billing. **State fix required:** engine state → Postgres persist/restore (or Limitr Cloud $49/mo for managed customers/ledger/Stripe invoices) — no in-memory voucher customers |
| Realtime | Trigger.dev events → SSE | Lanes/counters/inbox are event-driven, not table subscriptions |
| Object store | R2 (S3-compatible) | Photos, raw scrape payloads, receipt blobs — egress-free |
| Push | Expo push → FCM/APNs | Keeper alerts to the phone |
| Auth | Better Auth (humans) + API keys/wallets (agents) | Agent identity first-class; humans are a client of the same API |
| Sandboxing | **Agent37** when needed | Own actors run in containers. Third-party/untrusted actor code → Agent37 (gVisor, per-minute, ~25–52× cheaper than E2B/Modal/Vercel Sandbox; BYO Docker = portable exit) |

**Run-rate:** ~$45–115/mo at startup scale (Vercel $0–20 · Neon ~$15 ·
Trigger.dev $10–50 incl. managed compute · Agent37 ~$5–15 · proxies
$5–15 · R2 ~$0 · judge usage $5–10 · Expo free→$19). The only stateful
systems are Neon + R2 + Trigger.dev's managed state — everything else
is disposable.

**Agent-operable constraint.** The whole stack must be drivable from a
phone through the agent: cloud builds (Expo), git-push deploys (Vercel),
and API-readable logs/metrics on every layer (Vercel drains, Neon,
Trigger.dev runs, Agent37 sandbox logs). No SSH, no self-managed
daemons, no state that only exists on a box.

**Compute ↔ orchestrator contract.** Trigger.dev owns every loop.
Heavy work (Playwright scrapes, multi-minute computer-use sessions)
runs as Agent37 sandboxes via start/poll — never one long HTTP call —
writing artifacts to R2 and compact JSON back. No workflow state, no
queues, no durability on the compute layer.

**DB long-term — Neon over Supabase.** Supabase's best features (Realtime,
auto-APIs) are muted in an agent-first build: lanes run on orchestrator
events and the API is custom. What Hoardr needs — instant branchable
prod clones for eval corpora and cheap shared-storage read replicas for
demand-graph analytics — is exactly Neon's signature. Revisit if bundling
auth+storage+realtime under one vendor ever outweighs elasticity.
Graduation path stays open: SingleStore when analytics + vector outgrow
pgvector (measured, not anticipated).

**Upstream asks — Limitr.** Verified against the 0.8.9 skill docs: the
giveaway-campaign machinery mostly exists (arbitrary units, grant pools,
TTL'd reserve/settle/release, decrement reversals, governors, hard caps,
`scope`-scoped metering = donor pool drawn down by recipients). Sent to
the Limitr team as feature requests:

1. Pluggable persistence — store adapter + lazy per-customer hydration
   (serverless-correct caps; kills the cold-start state-restore latency)
2. Scoped keys — `issueKey(customer, {caps, ttl})` for Bearer-only
   hosted agents
3. Single-shot header voucher — pay-and-call in one request (kills the
   LNP negotiation round-trip)
4. Voucher-bound identity — one signed credential proving payment *and*
   which customer the agent acts for
5. Gift/claim codes — a pool any unknown claimant redeems once
6. Multi-party settlement — one action splitting donor → platform fee →
   agent share
7. Cross-provider vouchers — donor voucher spendable at Hoardr *and*
   other network providers (confirm)
8. Campaign reference policy — `scope`-based donor-pool metering
   published officially; Hoardr = reference deployment

**Repo skills & installs.**

- **Egbe** (required per `EGBE.md`): the `egbe-os-*` skill suite under
  `.agents/skills/` (currently ~40 skills: db, deploy, domains, errors,
  git, keys, mail, notifications, payments, product-*, research,
  telemetry, ads suites, etc.), the `egbe-docs/` directory, and
  `skills-lock.json` to pin versions. Invoke `egbe-os-product-build`
  for implementation work; `egbe-os-playbook` before method work.
- **Limitr**: `npm i @formata/limitr` + `@formata/stof`. Skills ship
  inside the packages (`node_modules/@formata/limitr/skill/`):
  `limitr` (policy/engine guide), `limitr-network` (LNP vouchers),
  `limitr-ts` (TS SDK), `stof` (the policy language). Copy them into
  `.devin/skills/` in the new repo so agents load them on demand;
  `limitr-py` unneeded (TS stack).
- **Core deps**: `ai` (AI SDK), `zod`, `better-auth`, `pg`/`drizzle`
  (or preferred Postgres client), `@trigger.dev/sdk`,
  `@aws-sdk/client-s3` (R2), `impit` or fetch shim, `playwright`
  (Patchright for protected sources).
- **Env keys needed**: `AI_GATEWAY_API_KEY`, `DATABASE_URL` (Neon),
  `TRIGGER_SECRET_KEY`, `R2_*`, `LNP_PROVIDER_KEY` (real key — the
  ephemeral fallback dies on restart), `BETTER_AUTH_SECRET`,
  Stripe keys, proxy provider creds, Trash Nothing API key.

**Not carried forward:** Workflow SDK (replaced by Trigger.dev), SQLite×3 +
Blob×4 + SingleStore store soup (replaced by one Postgres), Apify as
platform (replaced by own actors; kept as fallback), Sandbox SDK (no
untrusted code yet), BullMQ/self-hosted queues (no hand-rolled durable
state), L402/Lightning (a fourth rail nobody can pay through yet),
Airflow (batch DAG scheduler — wrong paradigm for event-driven
per-listing loops and durable waits; Python, heavy ops, and its
strength — scheduled ETL — isn't the workload).

## The platform — the parts we build

1. **Supply ingestion** — source adapters (free-stuff feeds, promoter
   campaign intake), one normalized Listing shape, dedupe, and
   availability/expiry tracking. Two supply kinds: `physical` and
   `digital` — and two supply *tiers*: **first-party** (Expo-posted:
   structured intake, photo gate at upload, giver identity, push-driven
   "still available?" freshness, native DMs, lighter judge path) and
   **scraped** (full adapter → dedupe → battery → revalidation
   gauntlet). Scraped supply bootstraps day-one inventory; first-party
   is the destination — freshest, cheapest, most trusted. The
   photo→draft workflow becomes the giver post composer.
2. **Judge** — the frozen battery (typed answers) → deterministic
   decision layer → verdict + receipt. Photo gate + vision stage for
   physical items; a lighter evidence path for digital ones.
3. **Receipts & ledger** — every judgment, match, and action recorded
   with its inputs; the audit trail and the calibration corpus.
4. **Demand graph** — the wants registry: declared wants (human or
   agent-authored), revealed wants (claims/passes/keeps), taste
   profiles, decay, per-recipient strictness. Queryable — this is the
   sellable asset.
5. **Match engine** — judged supply ↔ live demand. Geo + fit + freshness
   for physical; instant eligibility for digital. One-sentence
   explanations; outcomes feed back (claimed/passed/gone).
6. **Agent API** — the public surface for other agents: declare a want,
   query judged supply, buy verdicts and match data. Auth, metering,
   billing. (LNP today; whatever the protocol becomes.)
7. **Contact** — recipient↔giver DMs and agent↔agent messages, with
   replies and pickup coordination inside Hoardr.
8. **Autonomous loops** — the always-on orchestrator: watch → judge →
   match → contact → redeem. Physical claims stop at human approval;
   digital claims run end-to-end unattended.
9. **Redemption** — digital fulfillment: issue, deliver, and verify
   credits/coupons/codes. Physical: claim ticket + DM thread.
10. **Ads & campaigns** — promoter placements to recipients and
    giveaway-campaign tooling (budget, targeting by want, redemption
    report). The demand graph is the inventory. Canonical pattern:
    **the giveaway IS the ad** — e.g. Coinbase funds 1,000 wallets at
    $5, Hoardr matches them to high-intent recipients, the *recipient's
    agent* receives and can act on the wallet, and activation/trade
    webhooks produce the verified redemption report the promoter pays
    for. No impressions — funded, intent-matched, provably redeemed
    distribution.
11. **Recipient app** — the human surface: scan counter, keeper cards
    with receipts, want management, approvals, DM inbox.
12. **Trust & safety** — scam/hazard vetoes, evidence rules, rate
    limits, reporting, identity for givers, recipients, promoters, and
    agents.

## Core principles

- **The model answers questions; code makes decisions.** A frozen
  question battery produces typed answers; deterministic code applies
  vetoes and thresholds → verdict + receipt. One `judgeListing`
  implementation behind every surface.
- **Insufficient evidence never confirms.** No photo, failed gate, or
  model uncertainty can manufacture a keeper.
- **Spend escalates on promise.** Cheap rules kill obvious noise free;
  the expensive vision pass runs only where it can change a verdict.
- **Evals are infrastructure.** A labeled corpus and probe harness gate
  every battery/threshold change — precision is measured, not felt.
- **Wants are first-class records.** Declared and revealed demand
  profiles feed the same match engine, whether the want came from a
  human or their agent.

## What Hoardr becomes

- **The demand layer for secondhand.** Free stuff is the wedge because
  supply is open and decisions are fast. The same engine matches wants
  to anything local and cheap — agents register standing needs, Hoardr
  fulfills them across sources.
- **The venue where agents trade needs.** Any personal agent can publish
  "my human needs X" or "my human is giving X away." Hoardr is where
  those statements meet — a protocol, not just an app.
- **An outbound agent.** From "is it still available?" through replies,
  negotiation, and pickup scheduling — the whole errand, behind one
  approval.
- **Bulk pairing — wants are projects, not items.** "Furnishing a
  nursery" isn't one listing, it's a crib + changing table + rocking
  chair across three givers. The agent assembles the bundle: watches
  for the set, claims each piece as it lands, coordinates one pickup
  run, and reports "your nursery is 4/5 sourced" — you get a whole
  project furnished free, not one thing at a time. The canonical want
  is a life event: **"moving to a new city? we'll furnish your studio
  for free"** — bed, desk, lamp, kitchen kit, sourced piece by piece
  and claimed while you pack.
- **A demand-signal business.** Aggregate, anonymized wants become the
  product for buyers: "34 people in Oakland are looking for desk lamps"
  is data nobody else has.
- **Paid priority windows.** Buyers pay for first-look on a category
  before general alerts fire — a mid-century furniture dealer gets MCM
  listings 30 minutes early, a refurbisher gets first claim on dead
  ThinkPads. Sell speed, not exclusivity: the alert still reaches the
  public, the payer just sees it first. Turns resellers — the people
  who already refresh these feeds all day — into a revenue line.
- **Complete the set — free anchors the paid sale.** Every free item
  implies its missing complement: the free crib needs a mattress nobody
  gives away; the free espresso machine wants beans on subscription;
  the free desk lacks a chair. The claim moment is the highest-intent
  shopping signal that exists — the agent suggests the paid pairing
  (affiliate/sponsored) as part of "completing the project," which is
  helpful, not an ad interrupt. Bulk pairing makes it natural: "your
  nursery is 4/5 sourced — nobody's giving away the missing piece;
  here's the $60 version."
- **Dynamic pricing on everything sellable.** The demand graph prices
  its own inventory: priority windows cost more in hot categories (MCM
  season, moving-month furniture gluts), paid pairings bid up when a
  complement is scarce, agent query rates flex with verdict demand, and
  giveaway-campaign slots auction when promoters compete for the same
  want-cluster. Real demand signals → real-time prices — an actual ad
  network, not rate cards.
- **Your hoard.** A permanent memory of everything you've claimed for
  free: items, photos, receipts, estimated value, where it came from.
  Part inventory ("what do I own?"), part trophy case ("you've claimed
  $3,400 of free stuff"), part preference data — what you kept, used,
  and re-gifted is the richest revealed-want signal there is.
- **From free to paid — auctions too.** The same match and judgment
  extend to priced goods: fixed-price listings first, then eBay-style
  auctions where the agent bids on the human's behalf within a ceiling —
  "worth it up to $X" is just another typed answer the code enforces.
- **The judge generalizes.** "Is this worth it?" works on any listing —
  the battery is free-stuff-shaped today but the receipt pattern (typed
  answers, code decides, evidence or silence) applies to any
  judged-purchase problem.

## The far future — sci-fi features

Where the same primitives extend if the "everything is free" thesis lands:

1. **Reverse supply.** The demand graph generates listings: "3 agents
   want what you're about to throw out" — a notification to the
   would-be trasher turns garbage-day into a supply event.
2. **Curb mode.** Point your camera at a curb pile; the agent judges it
   live — "worth your trunk space" or walk away. Every sidewalk becomes
   a browsable aisle.
3. **Predictive wants.** Your agent sees the want before you do —
   "based on your hoard and last summer, you'll need a window AC by
   June" — and claims it in May, when supply is plentiful.
4. **Digital-twin fitting.** Giver films a video → we generate a 3D
   asset (splat/GLB) for spin-and-inspect in-app; recipient snaps a
   photo of their room → an image model composites the item into it at
   plausible scale. No ARKit needed — video→3D + AI compositing covers
   the value through a GPU lane, not native AR.
5. **Zero-dollar auctions.** Five claimants, one listing: agents bid
   with non-money — pickup speed, need receipts, hoard reputation.
   Priority computed, not paid.
6. **The re-gift relay.** Done with something? One tap and your agent
   re-lists it to a queued next owner. Objects circulate; the ledger
   tracks every item's free-lifetime chain ("this lamp has been free
   four times").
7. **Autonomous pickup.** Match fires → a van/robotaxi slot is booked →
   the thing appears on your porch. "Free stuff, delivered" removes the
   last reason anyone pays retail.
8. **Karma reputation.** Your score = lifetime claimed value plus
   re-gift generosity. Generous givers earn matching priority — a
   reputation economy replacing the cash one.
9. **Dream registry.** Your agent registers "I'd want X if it existed"
   — matched to makers donating 3D-print time or surplus runs. Demand
   that summons supply into existence.
10. **The post-money allocation layer.** The endgame: as surplus
    outpaces scarcity, Hoardr becomes the general clearinghouse for
    free goods — agents allocate by need and fit, currency optional.
    The Musk quote, as a product.

## The consequential unknowns

- Is keeper precision high enough that recipients act without
  re-checking? (measurable from the ledger)
- Do givers answer DMs through Hoardr — does contact convert?
- Would agent buyers pay for verdicts when they could call the same
  model directly? (the defensibility question — the answer has to be
  the demand graph, not the battery)
