# Yaffa Records — Release Operations

Internal release operations dashboard for [Yaffa Records](https://yaffarecords.com).

## What's here

- **`docs/index.html`** — the live dashboard, served via GitHub Pages at https://therealchucknorrisforreal.github.io/yaffa-records-ops/
- **`brand/`** — email HTML templates (VIP early access, Broadcast) for each release
- **`content/`** — content calendar, captions, release content plans
- **`outreach/`** — Ben Schwarz outreach strategy, target lists, email templates
- **`n8n/`** — n8n workflow JSONs for daily Brevo contact sync from Google Sheets

## Stack

- **Email:** Brevo (formerly Sendinblue) for mass sends, billed via PayPal. Verified senders: hello@yaffarecords.com (mass), info@yaffarecords.com (label inbox), ben@yaffarecords.com (1-to-1 outreach).
- **Contact DB sync:** n8n workflows pull from two Google Sheets (Hypeddit webhook + VIP form) daily at 03:00 / 03:05 → Brevo "Yaffa Master" list with tagging via custom attributes (SIGNUP_SOURCE, SIGNUP_DATE, GATE_NAME).
- **Dashboard state:** localStorage (per-browser) for now. Shared state via Google Sheet + Apps Script Web App is the next iteration.

## Access

The dashboard is password-protected (client-side SHA-256 gate). Get the password from Mosko.

## Roster

- **Oi** (Omer Inbar) — co-founder, lead producer, creative director
- **Mosko** (Tomer Mosko) — co-founder, operations, strategy, ops
- **Mishell** — collaborator (Mishell & Oi duo)

## Releases (active)

- **14.5.2026** — Bulletproof EDIT (Free, La Roux × Yaffa)
- **10.6.2026** — All Night (YFR010, Oi & Mosko)
- **25.6.2026** — Emta (YFR009, Oi, 2 versions)
- **15.7.2026** — Higher (YFR013, Oi)
- **29.7.2026** — Smells Like Teen Spirit EDIT (Free, Oi)

---

This is an internal tool. Not for redistribution.
