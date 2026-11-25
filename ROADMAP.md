# Fichtner Hof — migration assessment and roadmap

_Assessed 2026-09-19. Nothing has been built, downloaded, registered, or changed. This file
is the plan; the decisions in §4 gate the first step._

---

## 1. What exists today — fichtnerhof.wordpress.com

A free WordPress.com site (theme Radcliffe 2, no custom domain, no Jetpack, no plan) for a
5-bedroom / 4-bath, 3,700 sf vacation house at 268 Ridgecrest Drive, Mammoth Lakes, near
Canyon Lodge. Created around November 2014; **last edited 2018-12-13**. Everything on it
was pulled through the public WP REST API — no login was needed, and none will be.

| Piece | What it is | State |
| --- | --- | --- |
| Front page | The blog feed, showing the one post | The post _is_ the site |
| Post: "Tour the Fichtner Hof!" | ~60-word intro + a tiled gallery of **56 photos**, ~20 with captions (Tioga bedroom, bunk room, Louise bedroom, master bath, mancave, jacuzzi, Sherwins view…) | Photos span 2014–2018 |
| Page: Amenities (id 107) | ~130 words, floor-by-floor description | Current-ish, best copy on the site |
| Page: Amenities (id 333) | ~50-word shorter duplicate | Redundant — merge |
| Page: About | Location + **Nancy's personal Gmail and cell number** | Contact, nothing else |
| Page: FichtnerHof (id 311) | Empty | Delete |
| Footer | Google Maps link to the street address | Publishes the exact address |
| Comments / subscribers / posts feed | Off, 0, 1 | Nothing to preserve |

**Total prose: about 1,300 words.** No booking, no availability, no rates, no reviews, no
forms, no house rules, no arrival/departure info. The copy still says "new heating system"
and "winter 2018-19" — it needs a refresh pass, not just a port.

**Photos:** the originals are directly downloadable (`/wp-content/uploads/...`), 5–7 MB
each, so **~300–350 MB total**. They cannot ship as-is and should not live in git.

**Domain:** none. `fichtnerhof.com`, `.net`, `.org` and `thefichtnerhof.com` are all
unregistered (checked against the registries via RDAP; GoDaddy's availability endpoint is
gated for this account, so the CLI could not confirm). The GoDaddy account currently holds
`snackbyte.io`, `snackbyte.dev`, `bishopnewyearseve.com`, `bishopnye.com`.

## 2. What the tooling gives us

**`tools/project-setup`** — the ordered, one-step-at-a-time setup checklist (Spec Kit →
spec-render → template spin-up → `specify init` → speckit engine → `git init` →
constitution → GitHub remote → release-flow Action → GCP hosting) plus `GCP-SETUP.md`, the
Cloud Run + shared-load-balancer runbook. Every step applies here: this is a deployed
consumer app. The GCP step is _per project_, and the project already exists, so what is
left is the runbook's **fleet pattern** (one NEG, one backend, one host rule, one DNS
record) **plus the new-TLD cert work** — a fresh domain needs its own DNS authorization,
managed cert and cert-map entries; the existing `snackbyte.*` wildcards do not cover it.

**`code/snackbyte-base` v1.5.8** — Vite + React + TypeScript + Express on Node 24; Vitest,
ESLint, Prettier; `Dockerfile` + `cloudbuild.yaml` for Cloud Run; `environments.json`
(main → production, dev → staging with `noindex` and a version chip); `/api/health`,
`/api/version`. The resolver bakes two identity choices — `--mode static|server` and
`--render prerender|dynamic` — and refuses to default them. Prerender supports several
pages via `entries[]`, but there is **no router and no multi-page Vite config** shipped:
the skeleton is one page.

**Precedent — `clients/playhouse395/rehearsal-board`.** Spun up from this template on
2026-09-02: `prerender`, deployed into the shared **`snackbyte-apps`** project
(`us-central1`), client hostnames CNAME'd to `lb.snackbyte.io`, staging at
`dev.<host>`, content kept out of git in a GCS bucket, a `ROADMAP.md` as the post-launch
idea bucket. That is the shape to copy.

## 3. The gaps — what has to be built that nothing ships

1. **Multi-page site structure.** Home, gallery, amenities, location, guest guide,
   contact. Needs either Vite multi-entry (true MPA, each page prerendered) or a client
   router with one prerender entry per route. Spec-time choice; MPA is the simpler fit for
   a mostly-static site.
2. **Photo pipeline.** Archive the 56 originals once (GCS, cold storage — the source of
   truth). Generate responsive derivatives (e.g. 480 / 960 / 1600 w, WebP + JPEG fallback,
   ~25 MB total). Serve derivatives either from the repo's `public/` or from a bucket
   behind the load balancer. Gallery UI with captions and a lightbox.
3. **Content model.** ~1,300 words that change every few years. Markdown or typed TS
   data files in the repo, edited via a PR, is enough — a CMS is only warranted if
   someone other than Jeff must edit without help (see §4, decision 4).
4. **Checkout checklist** — a new capability. Ranges from a static page with
   localStorage ticks (no backend) to per-stay instances whose completion notifies the
   owner (server, email, maybe photo upload). Sibling content the old site lacks entirely
   and guests always need: arrival info, Wi-Fi, thermostat/humidifier, jacuzzi cover,
   bear-proof trash rules, snow/ice, quiet hours, emergency numbers.
5. **Calendar / availability** — a new capability. Read-only availability rendered from
   an iCal feed (a Google Calendar's private iCal URL, or Airbnb/VRBO export feeds) fetched
   and cached server-side is the smallest useful version and needs no OAuth. Booking
   _requests_ (a form that writes an event or emails Nancy) are a separate, larger step.
   If a service account is ever used to read a shared calendar, its email is user-facing
   (Nancy pastes it into a share dialog) and must carry only the brand — `fichtnerhof@…`,
   never `fichtnerhof-prod-runtime@…`.
6. **Access control**, if any of the above is not public. There is no auth in the
   template. A guest-only area (codes, Wi-Fi, checklist) is most cheaply an unguessable
   link or a shared passcode, not accounts.
7. **Contact.** Today: a personal Gmail and cell number in plain text. A contact form
   needs a server route plus an email-sending service (a new external dependency and
   secret). `mailto:`/`tel:` links are enough for v1.
8. **Domain, TLS, DNS, cutover.** Register the domain; new-TLD cert-map entries; the
   `_acme-challenge` CNAME and apex/`www`/`dev` records via the `godaddy` CLI (writes
   need a yes each time); WordPress.com pointer or redirect.

## 4. Decisions that gate step one

Each has a recommendation; none is made. Numbered in the order they should be answered,
because the earlier ones change the later ones.

1. **Who is the site for?** (a) marketing to strangers — the house is or will be listed
   commercially; (b) a reference for friends/family who already have the dates; (c) both,
   with a public front and a guests-only back. Decides whether availability is public,
   whether the street address stays on the page, how formal the checklist is, and whether
   a contact form matters. _The rest of this list is written assuming (c); if it is (b),
   the site shrinks and most of the public-facing polish drops in priority._
2. **Domain.** Recommend registering `fichtnerhof.com` (≈$12–20/yr) now — every later
   step (cert, DNS, staging host, WP pointer) keys off it. Staging becomes
   `dev.fichtnerhof.com` under the same wildcard cert.
3. **Deploy mode and render** — the resolver's two required flags. Recommend
   `server` + `prerender`: server because the two features that motivate the migration
   (calendar fetch, any checklist submission) are server concerns; prerender because the
   marketing pages are known at build time. `static` + `prerender` is right only if v1 is
   parity-only and the flip to server is accepted as a later, documented edit (SPIN-UP.md
   §"Switching mode later").
4. **Content ownership after launch.** Who edits copy and swaps photos — Jeff via PR
   (recommended; keeps everything in the repo), or Nancy directly (a CMS requirement that
   reshapes the build)?
5. **Public vs gated.** An availability calendar next to a published street address
   advertises when the house is empty. Recommend: either drop the exact address from the
   public site ("near Canyon Lodge, walk to the Village") or gate the calendar and the
   guest guide behind a guest link. Lock codes and Wi-Fi never go on a public page.
6. **Where it runs and what it is called.** Recommend the shared `snackbyte-apps` project
   (precedent; ≈$0 marginal since the LB is already paid for) and a **private** GitHub
   repo (client work). Repo/service slug is a developer-only identifier — functional, not
   brand: e.g. `fichtner-hof-site` (client name is descriptive here, as `rehearsal-board`
   under playhouse395 is). "The Fichtner Hof" is the rendered name only.
7. **The old site's fate.** WordPress.com free plans cannot redirect (the "Site Redirect"
   upgrade is ≈$13/yr). The old site has one post and no SEO worth carrying, so recommend:
   replace its content with a one-line pointer to the new domain, then set it private a
   few months later. No paid redirect.

## 5. Phased roadmap

Dependencies run downward. Sizes are estimates in working sessions, not commitments.

### Phase 0 — Decide (this conversation)
Answer §4. Exit: the seven decisions recorded at the top of this file.

### Phase 1 — Salvage the content (½ session, no infrastructure)
The only irreplaceable part of the old site is the photo set; do this before anything
else and independently of every other decision.
- Pull all pages/posts via the REST API into `content/` as Markdown, keeping the gallery
  order and the ~20 captions.
- Download the 56 originals; verify counts and sizes; upload once to an archive bucket
  (Standard→Archive lifecycle) under the chosen GCP project. Originals stay out of git.
- Screenshot the live site for reference.
Exit: `content/` committed later with the skeleton; originals in the bucket; a manifest
(filename, caption, room, year) checked in.

### Phase 2 — Project setup via the checklist (1 session, step-at-a-time)
Run `SETUP-CHECKLIST.md` as written — the agent presents each step, Jeff says go or skip:
Spec Kit + spec-render present → tree-copy `snackbyte-base` → `npm run init` with the
decision-3 flags → `specify init --here` → engine (**local plug**; no ClickUp) →
`git init -b main` + `.gitignore` + first commit → constitution → private remote →
release-flow Action `@v1` (build-id strategy; Actions write grant needs an explicit yes)
→ GCP fleet pattern + **new-TLD cert** (§8 of the gaps) → DNS via `godaddy` CLI.
Exit: `check:all` green, CI green, staging reachable at `dev.fichtnerhof.com` serving the
skeleton page. Domain registered at the start of this phase.

### Phase 3 — Parity, refreshed (2–3 sessions) — spec 001
The old site's content on the new stack, better organized, no new capabilities.
- Pages: home (hero, one-paragraph pitch, quick facts), gallery (responsive images,
  captions, lightbox, grouped by floor/room), amenities (the two pages merged and
  brought current), location (Village, gondola, shuttle, Canyon Lodge — address per
  decision 5), contact (per decision 1).
- Photo pipeline as a build step or a one-time script producing committed derivatives.
- Copy refresh with Nancy: what changed since 2018.
- Launch on the domain; point the WordPress site at it (decision 7).
Exit: live on `fichtnerhof.com`, Lighthouse-clean, old site pointing here.

### Phase 4 — Guest guide and checkout checklist (1–2 sessions) — spec 002
- Arrival/departure guide pages (content from Nancy — this is the real work).
- Checkout checklist v1: ordered, tickable, state kept in the browser; printable.
- Gating per decision 5.
- Deferred unless asked for: per-stay instances, completion notifications, photo proof.
Exit: a guest can open one link the week of their stay and have everything they need.

### Phase 5 — Availability calendar (1–2 sessions) — spec 003
- Read-only month view from an iCal feed, fetched and cached server-side, no OAuth.
- Source per decision 1: a family Google Calendar, or listing-platform export feeds.
- Deferred: booking-request form (needs email sending), two-way sync, rates.
Exit: availability on the site matches the source calendar within the cache window.

### Phase 6 — Idea bucket (unscheduled)
Contact form with email delivery · booking requests · seasonal content (summer vs
winter) · reviews/guestbook · rates page · analytics · a second property.

## 6. Risks and watch items
- **Vacancy exposure** — the combination of public address + public availability
  (decision 5). Decide before Phase 5 ships.
- **Photo originals live on a free WP.com account nobody has touched since 2018.** Phase 1
  first; do not let it wait on the domain.
- **Content is the long pole, not code.** Phases 3–5 all need copy and house knowledge
  only Nancy has (rules, Wi-Fi, quirks). Start collecting during Phase 2.
- **Template is single-page.** The multi-page decision (MPA vs router) is a spec-001
  architecture choice; making it casually costs a rework.
- **New TLD on the shared LB** — the managed cert goes ACTIVE only after the
  `_acme-challenge` CNAME resolves (15–60 min); plan the cutover with TTL 600.

## 7. Not verified — assumptions to confirm before relying on them
- The `snackbyte-apps` LB, WIF pool, deploy SA and cert-map are in the state
  `GCP-SETUP.md` and `rehearsal-board/DEPLOY.md` describe (not inspected this session).
- Domain availability was checked at the registries (RDAP 404), not at a registrar
  checkout; price and final availability confirm at purchase.
- Whether any wordpress.com media beyond the 56 gallery images exists (the media
  endpoint returns 401; only what the post references was enumerated).
- Whether there is an existing listing (Airbnb/VRBO) whose calendar would be the source.

## 8. Running cost
Domain ≈$12–20/yr. Hosting ≈$0 marginal: Cloud Run scales to zero, the shared LB's
~$18/mo is already paid, storage for ~350 MB of originals + derivatives is cents.
