# Xpense

A mobile-first web app (PWA) that turns business‑travel meal receipts into a
ready‑to‑send expense report. Snap a photo of a receipt, let the app read the amount,
confirm it, and at the end of the month generate the exact company expense module and a
PDF of the receipts — with the reimbursement email already drafted.

**Live app:** https://jaycee09.github.io/Xpense/

---

## What it does

- **Capture** a receipt with the phone camera.
- **OCR** reads the total automatically (Italian, on‑device) and lets you tap the correct
  amount from the detected values if needed.
- **Record** each expense (lunch / dinner / other), with client, location, number of
  people, and payment method (company card / personal card / mixed).
- **Caps** are applied automatically: €15 for lunch, €25 for dinner (configurable).
- **Generate the reimbursement**, selecting one or more months:
  - the **expense module** as a real `.xlsm` file — the original company template with
    only the input cells filled in, so formatting, formulas and macros are preserved and
    it opens identically in Excel;
  - a **PDF of the receipts** (giustificativi);
  - a **pre‑filled email** to the reimbursement address, ready to send with the two files
    attached.
- **Works offline** and **syncs across devices** (phone ⇄ PC) when signed in.

## How it works

Everything runs client‑side in a **single self‑contained HTML file**. There is no build
step and no server to maintain.

- **Local storage:** expenses and photos are saved in the browser (IndexedDB), so the app
  works fully offline on each device.
- **Multi‑device sync:** an optional Supabase backend keeps expenses and photos aligned
  across devices. Sync runs automatically on startup and when the app returns to the
  foreground, plus a manual **↻ Sync now** button.
- **Accounts:** login / registration with **admin approval** — a new user registers, and
  an administrator approves them from the Supabase dashboard before they can access data.
  The connection is embedded in the app, so users just open the URL and sign in.

## Tech stack

- Single‑file HTML / CSS / JavaScript PWA (dark "cyberpunk" theme, Orbitron display font,
  Font Awesome icons)
- [Tesseract.js](https://github.com/naptha/tesseract.js) for on‑device OCR (with Otsu
  binarization preprocessing and a dual‑PSM pass)
- [jsPDF](https://github.com/parallax/jsPDF) + autoTable for PDF generation
- [fflate](https://github.com/101arrowz/fflate) for surgical `.xlsm` (OOXML) cell editing
- [Supabase](https://supabase.com/) for authentication and multi‑device sync
  (Postgres + Row Level Security)
- Hosted on **GitHub Pages**

## Setup

1. **Host the app.** Publish `index.html` via GitHub Pages (or any static host).
2. **Configure sync (optional but recommended).** Follow
   [`SINCRONIZZAZIONE.md`](SINCRONIZZAZIONE.md) to create the Supabase project, tables,
   Row Level Security policies, and the approval flow. The connection is already embedded
   in the app; to point at a different Supabase project, update the `SUPA_URL` /
   `SUPA_KEY` constants at the top of the script in `index.html`.
3. **Keep the project awake.** Free Supabase projects pause after 7 days of inactivity.
   Add the included keep‑alive GitHub Action at `.github/workflows/keepalive.yml` to ping
   the database twice a week automatically (details in `SINCRONIZZAZIONE.md`).

## Usage

1. Open the app URL on your phone, register, and wait for admin approval.
2. Fill in your details under **Settings → User data** (name, tax code, address).
3. Tap **Scontrino** to add a receipt: take a photo, confirm the amount and details, save.
4. At month end, open **Generate reimbursement**, select the month, and produce the Excel
   module + receipts PDF. The reimbursement email opens pre‑filled — attach the two files
   and send.

## Notes

- The **anon (public) key** is meant to live in the client; data stays private because
  Row Level Security only exposes each user's own rows, and access requires admin
  approval. Never embed the `service_role` key.
- Photos are included in sync; the first sync of a full month may download a few MB —
  prefer Wi‑Fi.
- If the same expense is edited on two devices, the most recently saved version wins.

## Project structure

```
index.html            The entire application (single self‑contained file)
SINCRONIZZAZIONE.md   Supabase setup guide (sync, approval, keep‑alive)
keepalive.yml         GitHub Action to keep the free Supabase project awake
                      (place at .github/workflows/keepalive.yml)
```
