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
  the estimate (the row turns **yellow** and a multiplier field appears so you
  can charge it 2×, 0.5×, etc. — decimals OK). Click ✕ on the row to remove it.
- **Quick Estimate** — add charges by typing (autocomplete) or by multi-selecting
  in the picker. Edit quantities, remove lines, *Reset* to clear, *Copy as text*
  to dump a plain-text version.
- **Manage** — add new codes, edit existing ones, delete, mark discontinued.
  Edits are draft until you click **SAVE TO CHARGEMASTER**, which publishes
  them to GitHub for everyone. The banner shows whether you have unsaved
  changes; the header pill shows the latest save state (Saved / Unsaved /
  Saving / Save error).

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

## Saving to GitHub

Editing a code (or adding/deleting) in the **Manage** tab updates the local
draft. Click **SAVE TO CHARGEMASTER** to publish the draft to GitHub —
this commits an updated `data.js` to the repo so everyone sees the new
prices on their next page load.

The setup that makes this work (the encrypted GitHub token + the target
repo/branch/file) lives in `auth.json` and is shared across all devices.
There's no per-device setup — every browser that unlocks with the team
password is already wired to push.

If two people edit at once, the second save will hit a 409 conflict; the
banner says to refresh and re-apply your changes.

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
