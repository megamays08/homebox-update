# Fork Enhancements

## 1. Typed custom fields
The foundation — custom fields can now hold real typed data instead of everything being text.
- **Types:** Text, Number, Date, Yes/No — pick per field in the item editor; the value input adapts (number box, date picker, checkbox).
- **Why it matters:** food servings, expiration dates, battery counts, water gallons — all structured and sortable now, instead of stuffed into the description.
- Backend wired the date (`time`) value through the item save/load paths (no DB migration needed).
- *Merged — PR #1*

## 2. "Coming Due" dashboard section
- A home-dashboard card listing date fields that are **due soon or overdue**, with urgency-colored pills (e.g. "Expiration · in 5 days").
- **Smart filter:** only treats *deadline-like* fields as actionable (expiration, refill, replace, check…), so record dates like "Date filled" don't create noise.
- Backend endpoint `GET /entities/expiring`.
- *Merged — PR #1*

## 3. Custom Fields section on the item view
- Custom fields now render in their **own dedicated, collapsible card** instead of blending into the built-in Details list — so your data stands out from Serial Number / Model Number / etc.
- *Merged — PR #2*

## 4. Battery Readiness dashboard
- A home card showing, per battery type: **stock on hand vs. reorder threshold, dependent device count, and a status pill** (ok / low / reorder / no stock item).
- **Convention:** a stock item carries `Battery Type` + `Min Stock` fields (quantity = cells on hand); a device carries just `Battery Type`. Types with devices but no stock surface as "no stock item."
- Backend endpoint `GET /entities/battery-readiness`.
- Generalizes to any low-stock/reorder tracking (filters, propane, meds) — batteries are just the first use case.
- *Merged — PR #3*

## 5. Custom-field value autocomplete
- Text custom-field inputs now suggest **previously-used values for a field of the same name** (type `Battery Type` → suggests AA / AAA / CR2032), keeping values consistent instead of free-typed.
- *Merged — PR #3 (shipped alongside Battery Readiness)*

---

### Also done (not features, but worth noting)
- **Published** your fork as a Docker image to **`ghcr.io/megamays08/homebox-fork:latest`** — verified it contains **no personal data**, safe to run on the NAS.
- **`.dockerignore` hardening** to keep your real DB out of future build contexts — committed on `chore/dockerignore-local-data` (pending your push/PR).

All three feature PRs are merged into your fork's `main`, so an image built from `main` has the full set. Nice work driving this — you've turned HomeBox into a genuine readiness tracker.

Here's a single 5-minute walkthrough that lights up every feature — an emergency-kit scenario.

## Setup: 3 items

**1. Add a food item** — "Emergency Food Bucket"
- Quantity: `1`
- Custom fields (Add → pick type):
  - `Servings` (Number) = `119`
  - `Expiration` (Date) = ~2 weeks out
- → *Showcases: typed number + date fields*

**2. Add a battery stock item** — "AA Batteries"
- Quantity: `6` (cells on hand)
- Custom fields:
  - `Battery Type` (Text) = `AA`
  - `Min Stock` (Number) = `8`
- → *Showcases: stock modeling; note stock (6) is below min (8)*

**3. Add a device** — "Emergency Flashlight"
- Custom field:
  - `Battery Type` (Text) = `AA` ← **notice it autocompletes** from step 2
- → *Showcases: autocomplete + device linkage*

## Payoff: look at the two views

**4. Open the Emergency Food item → Details tab**
- Servings and Expiration appear in their **own "Custom Fields" card**, separate from Serial Number / Model Number.
- → *Feature: custom fields section*

**5. Go to Home (dashboard)**
- **Coming Due** shows: *Emergency Food Bucket · Expiration · in 14 days* with an amber pill.
- **Battery Readiness** shows: *AA · in stock 6 · min 8 · 1 device · ⚠ low*.
- → *Features: Coming Due + Battery Readiness in one glance*

## The "aha" tweak
**6. Bump the AA quantity to `12`** (item view has +/− buttons), reload Home → the AA row flips to **✔ ok**. Then add a `Battery Type = CR2032` device with no matching stock item → it appears as **✖ no stock item** (gear you can't power).

That sequence hits all five: typed fields, the custom-fields section, Coming Due, Battery Readiness, and autocomplete — and ends on the readiness dashboard telling you exactly what to restock.