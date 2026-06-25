# Mailing List Signup — Design

**Date:** 2026-06-25
**Status:** Approved (pending spec review)

## Goal

Add a styled mailing-list signup form to markrossandthecontraptions.com that
collects **Name, Email, Phone, Comment** and persists each submission as a row
in a Google Sheet owned by `markaldenross@gmail.com`. Exportable to CSV anytime.
Total cost: **$0**. No third-party email/marketing service.

## Constraints / Context

- Site is pure static HTML/CSS hosted on **Render** (static site, per
  `render.yaml`), custom domain `markrossandthecontraptions.com`. Not GitHub Pages.
- No backend, no build step. Any persistence must come from an external endpoint.
- All contact/storage uses `markaldenross@gmail.com`.
- No third-party email service (Mailchimp/Buttondown/Formspree). Use the user's
  own Google account via Sheets + Apps Script.

## Architecture

```
Visitor → [HTML form in index.html] → JS fetch POST → [Google Apps Script web app] → [Google Sheet]
```

### Piece 1 — The form (Claude builds)
HTML form in `index.html`, styled to match the existing site (`styles.css`).
Fields:
- **Name** — required
- **Email** — required, email-format validated
- **Phone** — optional
- **Comment** — optional (textarea)
- Hidden **honeypot** field for spam bots (not shown to humans).

### Piece 2 — Submit logic (Claude builds)
Small vanilla-JS handler:
- Validates required fields + email format before sending.
- POSTs as `application/x-www-form-urlencoded` (avoids CORS preflight that
  Google Apps Script endpoints can't satisfy). Sent via `fetch`.
- Shows inline success state ("Thanks for signing up!") with no page reload.
- On failure, shows fallback: "Something went wrong — email us at
  markaldenross@gmail.com" so no signup is silently lost.
- If honeypot is filled, silently drops the submission (pretend success).

### Piece 3 — Apps Script + Sheet (User creates; Claude provides exact steps)
- New Google Sheet owned by `markaldenross@gmail.com`.
- Bound Apps Script (~15 lines): on `doPost`, append a row
  `[timestamp, name, email, phone, comment]`.
- Deployed as a web app, "Execute as me", "Anyone" access.
- User pastes the deployment URL back; Claude wires it into the JS.

## Data Model

One row per signup in the Sheet:

| Timestamp | Name | Email | Phone | Comment |
|-----------|------|-------|-------|---------|

Export: File → Download → CSV.

## Error Handling

- Client-side required-field and email-format validation.
- Network/script error → friendly inline fallback pointing to
  `markaldenross@gmail.com`.
- Honeypot hidden field to cut basic bots. No CAPTCHA (keeps it free + frictionless).

## Out of Scope (YAGNI)

- No double opt-in / confirmation email.
- No CAPTCHA.
- No backend server.
- No outbound email sending.
- No third-party marketing integration.

These can be added later if the simple approach is outgrown.

## Division of Labor

| Part | Who |
|------|-----|
| Form HTML + CSS | Claude |
| Submit JS | Claude |
| Create Sheet, paste script, deploy web app | User (Claude provides click-by-click steps) |
| Wire deployment URL into JS | Claude (after user provides URL) |
