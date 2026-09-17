# Tax Perks — Spec (React Native / Expo)

Cursor Hackathon Berlin @ Taxfix · 2026-09-17 · Challenge "Tax 365"
Video upload 21:00. Everything is scoped to one 90-second demo on a phone.

## 1. Vision

A tax-perk assistant inside the Taxfix app that turns the perks a user is already entitled to into **goals they can see, track and document all year**. Snap or upload an invoice, the app reads it, tells you what it's worth to you, and files it against the goal — so nothing is lost by July.

Users feel *in control, financially savvy, a little bit clever*. Never dutiful, anxious or guilty.

## 2. Non-goals

- No streaks, points, badges, countdowns, push notifications. Progress is always **euros toward a real statutory cap**.
- No real filing, no ELSTER, no auth. One seeded user.
- No perk catalogue browsing — the user only sees perks recommended for them.
- All amounts are labelled *estimate*.

## 3. Stack

- **Expo SDK 52+, React Native, TypeScript, Expo Router.** Runs in **Expo Go** — no custom native modules, so anyone on the team can run it on their phone in a minute.
- `expo-image-picker` (camera + photo library) and `expo-document-picker` (PDF / image files).
- **OCR + extraction = one vision-model call.** Image/PDF → base64 → `/api/extract` → LLM with vision → structured JSON. No on-device OCR library (ML Kit needs a dev build; not worth it tonight). PDFs: send first page as image if the model rejects PDF.
- Backend: Expo API routes (`app/api/*.ts`) or a 40-line Express server — whichever the backend person has warm. Holds the LLM key. Client never sees it.
- State: Zustand or React context + `AsyncStorage`. `perks.json` bundled in the app **and** served by the API so both read the same rules.

## 4. Taxfix design system (tokens pulled from taxfix.de CSS)

```ts
export const colors = {
  forest:   '#154618', // primary — headers, primary buttons, progress fill
  lime:     '#adee68', // accent — highlights, secured € , success
  limePale: '#cef5a4', // progress track on dark, chips
  limeMist: '#ecffc7', // soft success background
  ink:      '#0c0b0a', // body text
  stone:    '#9a9288', // secondary text, captions
  sand:     '#eae0d7', // dividers, card borders, disabled
  cream:    '#fdf8f2', // app background
  white:    '#ffffff', // cards
  green:    '#36893b', // links, positive deltas
  amber:    '#f8c677', // "needs review"
  red:      '#c8102e', // "not counted"
};
export const radius = { card: 24, pill: 999, input: 12 };
export const space  = { xs: 4, s: 8, m: 16, l: 24, xl: 32 };
export const type = {
  display: { fontSize: 32, fontWeight: '700', lineHeight: 38 },  // the refund number
  h1:      { fontSize: 24, fontWeight: '700' },
  h2:      { fontSize: 18, fontWeight: '600' },
  body:    { fontSize: 16, lineHeight: 24 },
  caption: { fontSize: 13, color: colors.stone },
};
```

- Font: Taxfix uses ABC ROM (proprietary). Use **Inter** (`@expo-google-fonts/inter`) — same geometric feel, bold headings.
- Shapes: big rounded cards (24), pill buttons (full radius), pill category chips (lime on forest, or forest on lime). Cream page background, white cards, forest header block with lime text for the hero number — exactly the deck's look.
- Buttons: primary = forest bg / lime text; secondary = white bg / forest text / sand border.
- Logo: don't fake the wordmark; a lime "tax perks" text in the header is enough.

Components to build (in `components/`): `Screen`, `Card`, `PillButton`, `Chip`, `ProgressBar` (forest track on cream, lime fill, € labels at both ends), `HeroNumber`, `DocumentRow` (status colour: green counted / red not counted / amber needs review).

## 5. Seeded user (the only user)

```json
{ "name": "Lena", "city": "Berlin", "taxYear": 2026, "employment": "employee",
  "grossIncome": 58000, "marginalRate": 0.35, "commuteKm": 18, "workDaysPerYear": 220,
  "homeOfficeDaysPerYear": 90, "housing": "renter", "children": [{ "age": 6 }],
  "married": false, "hasCleaner": true }
```

This is "what Taxfix already knows from last year's filing".

## 6. Screens (Expo Router)

| Route | Screen | Contents |
|---|---|---|
| `/` | **Home** | Forest hero block: "Your 2026 refund so far" + `HeroNumber` (lime) + "estimate". Below on cream: active goals (`ProgressBar` cards), then "Recommended for you" (≤3 perk cards, each with why-you line, est. € and `Set as goal`). |
| `/goal/[id]` | **Goal** | Perk name, one-sentence rule, `ProgressBar` (secured / cap), document list (`DocumentRow`), sticky `Add document` pill. |
| `/goal/[id]/add` | **Add document** | Two big buttons: **Take photo** / **Upload file**. After pick: preview thumbnail → "Reading…" → result card (vendor, date, amount, paid by transfer?) → status banner **Counted +€X** (limeMist) or **Not counted — paid in cash** (red) or **Needs review** (amber) → `Done`. |

## 7. User stories + acceptance criteria

**S1 — Relevant perks only.** Home shows ≤3 perks from the recommender, ranked by estimated € for the seeded profile; non-matching perks never appear; each card has a why-you line.

**S2 — Pick a goal.** `Set as goal` creates `{perkId, targetEur, securedEur: 0, documents: []}`; goal screen shows the bar in euros with the cap named ("€4,000 max credit") and the rule sentence; multiple goals allowed; goals persist across app restarts (AsyncStorage).

**S3 — Capture with OCR.** From a goal: take a photo or upload a file → the extraction returns `{vendor, date, amountEur, labourEur, paidByTransfer, category, confidence}` → the app computes the benefit (§9) → bar animates and Home hero updates. For §35a perks, `paidByTransfer: false` → **not counted** with the line "Pay by bank transfer next time — then this counts." `unknown` → needs review. Document appears in the goal's list with status.

**S4 — Before Dec 31 (stretch, only if S1–S3 demo cleanly by 20:15).** Card on Home with ≤3 moves and € values from `perks.json → moves`, phrased by the planner.

## 8. API

```
POST /api/extract      { imageBase64 | pdfBase64, mimeType, perkId }
  → { vendor, date, amountEur, labourEur, paidByTransfer: true|false|"unknown", category, confidence, rawText }

POST /api/recommend    { profile }
  → [{ perkId, whyYou, estimatedMaxEur }]   // ≤3; eligibility is evaluated in code, the model only writes whyYou

POST /api/plan         { profile, goals, today }        // stretch
  → [{ title, whyNow, estimatedEur, deadline }]           // ≤3, chosen only from perks.json.moves
```

Extraction prompt returns strict JSON; `paidByTransfer` inferred from "Überweisung", IBAN, "bar/cash", "Barzahlung". The model never sees or edits perk rules.

## 9. Perks data — `perks.json` (fixed; the model never edits it)

| id | name | type | rate | cap € | eligibility |
|---|---|---|---|---|---|
| household_services | Household services (cleaning, gardening, care) | credit | 0.20 | 4000 | always |
| tradespeople | Tradespeople labour (repairs, renovation) | credit | 0.20 | 1200 | always |
| minijob | Registered household mini-job | credit | 0.20 | 510 | always |
| childcare | Childcare | deduction | 0.80 | 4800 / child | child age < 14 |
| private_school | Private-school tuition | deduction | 0.30 | 5000 / child | hasPrivateSchool |
| first_degree | First degree / initial training | deduction | 1.00 | 6000 | employment == student |
| party_donation_credit | Party donations (first portion) | credit | 0.50 | 825 / person | always |
| party_donation_deduction | Party donations (additional) | deduction | 1.00 | 1650 / person | always |
| energy_renovation | Energy renovation, own home | credit | 0.20 | 40000 / property, 3 yrs | housing == owner |
| home_office | Home-office days | deduction | 6 €/day | 1260 | homeOfficeDays > 0 |
| commute | Commute (Entfernungspauschale) | deduction | 0.38 €/km/day | 4500 | commuteKm > 0 |

Benefit math, in code:
- `credit`: `min(rate × labourEur, cap)` — off the tax bill 1:1. §35a credits (first three) require `paidByTransfer === true`, else 0.
- `deduction`: `min(rate × amountEur, cap) × profile.marginalRate`.

`moves`: pay-tradesperson-by-transfer-this-year · file-2022-return-by-dec-31 · bunch-medical-bills-this-year · donate-before-dec-31 · apply-lohnsteuer-freibetrag-by-nov-30.

## 10. Fixtures (in `fixtures/`)

- `cleaner_transfer.jpg` — cleaning invoice, €720, "Zahlung per Überweisung", IBAN visible → counted, +€144.
- `plumber_cash.jpg` — plumber invoice, €900 labour, "Barzahlung" → not counted.
- Make them tonight: type the text into a Google Doc / Notes, screenshot on the phone. Real-looking enough for OCR.

## 11. Demo script (90 s, recorded on a phone screen)

1. Home: "Lena — refund so far €0", three recommended perks with € values. (S1)
2. Tap `Set as goal` on Household services → bar €0 / €4,000, rule sentence. (S2)
3. `Add document` → **Take photo** of the cleaner invoice → "Reading…" → result card → **Counted +€144** → bar animates, Home hero shows €144. (S3)
4. `Upload file` → plumber PDF → **Not counted — paid in cash. Pay by transfer next time.** (S3, the honest beat)
5. (Stretch) "Before Dec 31" card. (S4)
6. Close on the hero number: "That's why she opens Taxfix in November."

## 12. Definition of done

- S1–S3 pass on the seeded user with the two fixtures, in Expo Go on a real phone.
- Video ≤2 min named `TeamName_TaxPerks.mp4`, uploaded before 21:00.
- README: team, members, one-liner, repo link, and "how we used Cursor" (agents per spec section, MCP serving perks.json, parallel agents on screens/API/design system).
