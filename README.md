# Contact Details widget (Zoho CRM)

A widget for the Contact record detail page that shows and edits First
Name, Last Name, Phone and Email, plus a mailing address block where a
postal code auto-fills city / state / country via a free geocoding API.

Built as a plain CRM widget (HTML/CSS/JS + the Zoho Widget SDK), not a
full zet extension — this keeps the deliverable to three small files,
per the brief's "any approach that produces a working widget in your
org is acceptable."

The color palette (navy `#1b5187` / gold `#8c7032`) is sampled from
qualicare.com, so the widget reads as something built for this client
rather than a generic demo.

## Project structure

```
zoho-contact-widget/
├── widget.html   markup (entry point)
├── style.css     styling
├── app.js        all widget logic (load, edit, save, postal lookup)
└── README.md
```

Everything is flat at the repo root on purpose — see "Hosting: External
via GitHub Pages" below for why.

## How to install and run it in a fresh Zoho org

**Widget hosting:** this widget is set up with **Hosting: External**,
pointing at the three files published via **GitHub Pages**, not
uploaded as a zip through "Hosting: Zoho". See *Troubleshooting notes*
below for why — short version: internal zip upload rejected every zip
I tried in this org (including one packed with Zoho's own `zet pack`
CLI) with a generic "please upload a proper file" error, so I switched
to external hosting rather than burn the time budget chasing it.

1. **Get a developer org.** Sign up for a free Zoho CRM account at
   zoho.com if you don't already have one (Trial / Developer edition
   is fine).
2. **Create 2–3 sample Contacts** (Contacts module → Create Contact).
   Give at least one of them a first/last name, phone and email so
   there's something to see on first load.
3. **Publish the widget files.**
   - Push `widget.html`, `style.css`, and `app.js` to the root of a
     public GitHub repo (this repo).
   - In the repo, go to **Settings → Pages** → Source: **Deploy from a
     branch** → Branch: `main`, folder `/ (root)` → Save.
   - After it finishes deploying, note the published URL, e.g.
     `https://<you>.github.io/<repo>/widget.html`. Open it directly
     once to confirm it's reachable (it'll spin forever in a bare tab
     since it's waiting for a CRM context — that's expected, you're
     just checking it isn't a 404).
4. **Create the widget in Zoho.**
   - Go to **Setup → Developer Space → Widgets** → **Create New
     Widget**.
   - Name it, set **Type: Related List**, **Hosting: External**, and
     paste the GitHub Pages URL from step 3 into the **Base URL**
     field.
5. **Place it on the Contact page.** This is *not* done through the
   Setup layout editor (that editor is for standard form fields, and
   has no "Widget" component to drag). Instead:
   - Open any existing **Contact record** (not the create-form).
   - Click **More (•••)** on the record detail page → **Add Related
     List**.
   - Under "Unselected Related List", pick the widget you just
     created.
6. **Test it.** The widget should load that contact's details within a
   second or two. Click **Edit**, change a field, type a postal code
   to confirm the address autofill, click **Save changes** — then
   refresh the page to confirm the CRM record (not just the widget)
   shows the new values.

If you edit `app.js` or `style.css` after the first deploy, GitHub
Pages / your browser can cache the old copy aggressively. Either open
DevTools → Network → "Disable cache" while testing, or add a cache-
busting query string to the script tag in `widget.html` (e.g.
`app.js?v=2`, bumping the number each time) while iterating.

## Address API: OpenStreetMap Nominatim

**Endpoint used:** `https://nominatim.openstreetmap.org/search?postalcode=<value>&format=jsonv2&addressdetails=1`

**Why this one:**
- Free, no API key or account signup — one less thing to configure in
  a fresh org, and nothing to leak in client-side JS.
- Global coverage. The brief doesn't scope the widget to one country,
  and Nominatim will attempt a lookup for postal codes from most
  countries without needing a separate country selector in the UI
  (unlike e.g. Zippopotam.us, which requires the country code as part
  of the request path). Verified working against both a US ZIP and a
  Brazilian CEP during testing.
- Returns `city`/`town`/`village`, `state`, and `country` in one call,
  which maps directly onto the three auto-filled fields the brief
  asks for.

**Limitations (worth knowing before relying on this in production):**
- **Usage policy, not a real SLA.** Nominatim's public instance asks
  for roughly ≤1 request/second and no bulk/business use. Fine for a
  single user typing into a widget; not something to point a
  high-traffic integration at without self-hosting Nominatim or
  switching to a paid geocoder.
- **Coverage is uneven.** Postal-code-only search works well in
  countries with structured postal systems (US, UK, most of Europe,
  Brazil) and is patchier in places with sparse OpenStreetMap postal
  data — the widget surfaces "No match" rather than failing silently
  in that case.
- **No official CORS guarantee.** It works from browser `fetch` today
  and is commonly used this way, but Nominatim's docs don't formally
  commit to CORS support continuing — worth a fallback plan if this
  went to production.
- **Postal code only gives you the region, never the street.** That's
  exactly why the brief has street stay manually entered — a CEP/ZIP
  identifies an area, not a specific address.
- The returned `state`/`region` naming isn't always a clean US-style
  "state" (e.g. it may return a broader or narrower administrative
  region depending on the country), so it's shown as an editable
  field rather than locked, and the save button always sends whatever
  is currently in the field — auto-filled or hand-corrected.

## Troubleshooting notes (from actually building and installing this)

Two real issues came up getting this running end-to-end, worth
recording since they cost the most time and aren't documented clearly
by Zoho:

1. **Internal ("Hosting: Zoho") zip upload rejected everything.**
   Every zip I tried — a hand-built one, a minimal one-file test zip,
   and one produced by `zet init` / `zet validate` / `zet pack` (which
   passed validation locally) — was rejected on upload with "Please
   upload a proper file." I couldn't isolate the exact cause from
   Zoho's public docs or community threads in a reasonable amount of
   time, so I switched the widget to **Hosting: External** via GitHub
   Pages instead, which sidesteps the internal upload path entirely
   and doubles as the public-repo deliverable. If I had more time, I'd
   open a ticket with Zoho support to get a definitive answer (my
   working theory is a per-org/edition restriction on internal widget
   file storage that surfaces as a generic error instead of a clear
   permission message).
2. **`data.EntityId` is a string, not an array, in `PageLoad`.** An
   early version of `app.js` did `data.EntityId[0]` based on an
   example that showed it as an array. In this org it came back as a
   plain string (`"7580941000000618571"`), so `[0]` silently grabbed
   just the first *character* — the widget then asked CRM for contact
   `"7"`, got an empty `204 No Content` response, and failed with a
   generic "couldn't load this contact" error that gave no hint of the
   real cause. Fixed by checking `Array.isArray()` and handling both
   shapes. Left as a comment in `app.js` in case this varies by CRM
   version/DC and bites someone else.

## What I'd do differently with more time

- **Debounced validation feedback** on phone/email format before save,
  not just the empty-name check that's there now.
- **Optimistic UI** for the save button instead of a blocking spinner,
  with a proper rollback path if `updateRecord` fails.
- **A country-aware fallback**: try Nominatim first, and if it returns
  nothing, offer a manual country selector so the postal-code-only
  search has a second chance (e.g. Zippopotam.us as a secondary
  lookup once a country is picked).
- **Field-level diffing** so `updateRecord` only sends changed fields
  instead of the whole payload every save — smaller requests and
  cleaner audit history in CRM.
- **Get to the bottom of the internal-hosting zip upload failure**
  instead of routing around it, since External hosting means the
  widget depends on GitHub Pages staying up rather than being fully
  self-contained inside the CRM org.
- **A short Playwright/Jest smoke test** for the save round-trip,
  even though automated tests were explicitly out of scope here —
  it's the first thing I'd add back for anything beyond a case study.

## Assumptions made

- "Phone" in the brief maps to the Contacts module's standard `Phone`
  field (not `Mobile`).
- The address fields map to the Contacts module's standard mailing
  address fields (`Mailing_Street`, `Mailing_City`, `Mailing_State`,
  `Mailing_Zip`, `Mailing_Country`), since those exist by default on
  any fresh org and the brief doesn't call for a custom field set.
- "Street address stays manually entered" means it's never
  auto-filled by the API, but it's still editable inline like the
  other fields — not a separate, differently-styled input.
- The widget is placed via **Record detail page → More → Add Related
  List**, and hosted **externally** (GitHub Pages) rather than
  uploaded as a zip, since internal hosting's zip validator rejected
  every zip tried in this org (see *Troubleshooting notes*) and the
  brief explicitly allows "any approach that produces a working widget
  in your org."
- No authentication/permission handling was built, per the "explicitly
  out of scope" list — the widget relies entirely on the permissions
  of whichever CRM user has the record open.

## Recording

Not included in this repo — see the case study brief for the 3-minute
walkthrough deliverable (Loom or similar), recorded separately after
confirming the widget works end-to-end in a live org.
