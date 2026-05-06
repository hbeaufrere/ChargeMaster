# CEAMS Chargemaster &amp; Estimate Builder

A single-page web app for browsing the CEAMS / CAPE chargemaster by section,
building quick patient estimates, and editing charge codes.

## Files

| File | Purpose |
| --- | --- |
| `index.html` | The whole app (UI + logic). |
| `data.js` | The chargemaster data (codes, prices, descriptions). |
| `Charge master 2026.pdf` | Source document. |

## Run / host

- **Local:** open `index.html` in a browser.
- **Online:** push to GitHub and enable **Settings → Pages → Source: main / root**.
  App lives at `https://<user>.github.io/<repo>/`.

## Tabs

- **Browse** — every charge grouped by section, free-text search, optional toggle
  for the 14 proposed-discontinued codes. Click `+ Estimate` on any row to add it;
  rows already in the estimate are **highlighted yellow** until you remove them
  or click *Reset*.
- **Quick Estimate** — add charges by typing (autocomplete) or by multi-selecting
  in the picker. Edit quantities, remove lines, *Reset* to clear, *Copy as text*
  to dump a plain-text version.
- **Manage** — add new codes, edit existing ones, delete, mark discontinued.
  Export/import JSON for offline backups, or set up GitHub auto-sync (below).

## Password protection

On first launch the app asks you to set a password. After that, every open
requires entering it.

What the password actually does:

- ✅ Locks the app's UI behind a password screen.
- ✅ Encrypts your GitHub access token (AES-GCM, key derived from your password
  via PBKDF2). Without the password the stored token can't be decrypted.
- ❌ Does **not** hide the published `data.js` file. Anyone with the GitHub
  Pages URL can fetch it directly. If you need real protection of the data,
  use a private repo + a host that supports auth (Cloudflare Access,
  Netlify Identity, etc.).

The password is stored only on this device — re-setting up on a second device
means choosing a new password there. Click *Forget password* on the unlock
screen to wipe local auth data and start over.

## GitHub auto-sync

Edits made in the **Manage** tab (add / edit / delete / discontinue) can
auto-commit to GitHub so everyone sees the same chargemaster.

### One-time setup

1. Open the **Manage** tab → **Set up GitHub auto-sync**.
2. Create a fine-grained personal access token:
   - github.com → Settings → Developer settings → Personal access tokens →
     Fine-grained tokens → **Generate new token**.
   - Resource owner: your account.
   - Repository access: *Only select repositories* → pick this repo.
   - Repository permissions → **Contents: Read and write**.
   - Generate. Copy the token (starts with `github_pat_…`).
3. Paste owner / repo / branch / file path / token into the dialog and click
   **Test &amp; save**. The token is encrypted with your password before being
   stored locally.

### What happens after setup

- Every add / edit / delete in the Manage tab schedules a debounced commit
  (1.5s after the last change) that overwrites `data.js` on the configured
  branch with a freshly serialized version of the chargemaster.
- A small status pill in the header shows sync state: *Pending changes…*,
  *Saving…*, *Synced HH:MM:SS*, or *Sync error*.
- GitHub Pages rebuilds the site within ~1 minute, so other users see the new
  prices on their next page load (after the CDN cache flushes).

### Disconnecting

In Manage → **Disconnect** removes the token + config from this device.
The published chargemaster on GitHub is unchanged.

## Data shape (`data.js`)

```js
window.CHARGEMASTER_SEED = {
  version: "2026.1",
  sections: ["Examinations", "Field Service", ...],
  items: [
    {
      code: "1512",
      section: "Examinations",
      service: "Emergency Exam",
      unit: "per appt",
      price: 360,
      description: "Walk in or same day scheduled ER"
    },
    // discontinued codes have `discontinued: true`
  ]
};
```

The auto-sync writes this format with one item per line for clean diffs.
Bump `version` when you ship a notable change so the header meta reflects it.
