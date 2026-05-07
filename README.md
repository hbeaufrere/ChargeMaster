# CEAMS Chargemaster &amp; Estimate Builder

A single-page web app for browsing the CEAMS / CAPE chargemaster by section,
building quick patient estimates, and editing charge codes.

## Files

| File | Purpose |
| --- | --- |
| `index.html` | The whole app (UI + logic). |
| `data.js` | The chargemaster data (codes, prices, descriptions). |
| `auth.json` | Shared-password salt + encrypted verifier (see below). |
| `emu.jpg` | Logo (header + auth screen + favicon). |
| `Charge master 2026.pdf` | Source document. |

## Run / host

- **Local:** open `index.html` in a browser.
- **Online:** push to GitHub and enable **Settings → Pages → Source: main / root**.
  App lives at `https://<user>.github.io/<repo>/`.

## Tabs

- **Browse** — every charge grouped by section, free-text search, optional toggle
  for the 14 proposed-discontinued codes. Click anywhere on a row to add it to
  the estimate (or remove it if it's already there). Rows already in the
  estimate are **highlighted yellow** until you remove them or click *Reset*.
- **Quick Estimate** — add charges by typing (autocomplete) or by multi-selecting
  in the picker. Edit quantities, remove lines, *Reset* to clear, *Copy as text*
  to dump a plain-text version.
- **Manage** — add new codes, edit existing ones, delete, mark discontinued.
  Export/import JSON for offline backups, or set up GitHub auto-sync (below).

## Password protection (shared)

The app asks for a single team password to unlock. Everyone on the team uses
the same password, regardless of device.

### How it works

- `auth.json` in the repo holds a random salt and an encrypted verifier
  string ("OK-CHARGEMASTER"). On unlock the browser derives a key from the
  password (PBKDF2 200k SHA-256) and tries to decrypt the verifier. If
  decryption produces the expected string, the password is correct.
- The same key is used (per-device) to encrypt the GitHub PAT in localStorage.

### What the password protects

- ✅ Gates the app's UI for casual visitors.
- ✅ Encrypts the GitHub token stored on each device.
- ❌ Does **not** hide the published `data.js` file. Anyone with the GitHub
  Pages URL can fetch it directly.

### Rotating the password

Generate a new salt + verifier with the new password and overwrite `auth.json`,
then redeploy. Existing browsers will fail to unlock with the old password and
their stored PATs will become unreadable (the app silently clears them on
next unlock attempt — users just re-enter the token once).

To regenerate `auth.json` for a new password, run this Node snippet
(requires Node 18+):

```sh
node -e '
const { webcrypto } = require("crypto");
(async () => {
  const password = "NEW-PASSWORD-HERE";
  const enc = new TextEncoder();
  const salt = webcrypto.getRandomValues(new Uint8Array(16));
  const baseKey = await webcrypto.subtle.importKey("raw", enc.encode(password), {name:"PBKDF2"}, false, ["deriveKey"]);
  const key = await webcrypto.subtle.deriveKey(
    {name:"PBKDF2", salt, iterations:200000, hash:"SHA-256"},
    baseKey, {name:"AES-GCM", length:256}, false, ["encrypt"]);
  const iv = webcrypto.getRandomValues(new Uint8Array(12));
  const ct = await webcrypto.subtle.encrypt({name:"AES-GCM", iv}, key, enc.encode("OK-CHARGEMASTER"));
  const hex = b => [...new Uint8Array(b)].map(x => x.toString(16).padStart(2, "0")).join("");
  console.log(JSON.stringify({salt: hex(salt), verifier: hex(iv)+":"+hex(ct), iterations:200000, hash:"SHA-256"}, null, 2));
})();
'
```

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
