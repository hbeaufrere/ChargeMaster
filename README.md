# CEAMS Chargemaster &amp; Estimate Builder

A single-page web app for browsing the CEAMS / CAPE chargemaster by section, building
quick patient estimates, and editing charge codes.

## What's here

| File | Purpose |
| --- | --- |
| `index.html` | The whole app (UI + logic). Open it in any modern browser. |
| `data.js` | The seed chargemaster data (codes, prices, descriptions). |
| `Charge master 2026.pdf` | Source document the seed was built from. |

## How to use it

### 1. Run locally
Open `index.html` directly in a browser, or serve the folder:

```
python3 -m http.server 8080
# then visit http://localhost:8080
```

### 2. Host online (free)
Push the repo to GitHub and enable **Settings → Pages → Source: main / root**.
Your app will be at `https://<user>.github.io/<repo>/`.

## Tabs

- **Browse** — every charge grouped by section, filterable by section and free-text
  search. Click `+ Estimate` on any row to add it to the current estimate.
- **Quick Estimate** — two ways to add charges:
  1. **Type** a code or service name and pick from the autocomplete list.
  2. **Pick from list** — open the picker, multi-select rows, then `Add selected to
     estimate`.

  Edit quantities inline, click `✕` to remove a line, `Reset` to clear everything,
  or `Copy as text` to grab a plain-text version.
- **Manage** — add new codes, edit existing ones (price, name, unit, description,
  section), mark as discontinued, or delete. **Export JSON** downloads your edits;
  **Import JSON** restores from a file. **Reset to defaults** restores the
  original 2026 data.

## How edits are saved

Edits live in `localStorage` in the browser you're using. To share new prices
with everyone:

1. Make your edits in the Manage tab.
2. Click **Export JSON** — you'll get a `chargemaster-<version>.json` file.
3. Either:
   - send the JSON to colleagues to **Import JSON** in their browser, or
   - replace `data.js`'s seed with the new values and commit it to the repo so
     the new defaults ship to all users.

## Updating the seed

The seed lives in `data.js` as `window.CHARGEMASTER_SEED`. Each item is:

```js
{
  code: "1512",
  section: "Examinations",
  service: "Emergency Exam",
  unit: "per appt",
  price: 360,
  description: "Walk in or same day scheduled ER",
  discontinued: false  // optional; omit or set false for active codes
}
```

Bump `version` whenever you edit the seed so users know to reset/import.
