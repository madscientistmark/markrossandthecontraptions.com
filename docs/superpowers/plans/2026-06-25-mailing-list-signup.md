# Mailing List Signup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a styled signup form to the band site that persists Name/Email/Phone/Comment submissions to a Google Sheet via Google Apps Script.

**Architecture:** A static HTML form in `index.html` posts (form-encoded, via `fetch`) to a Google Apps Script web app, which appends a row to a Google Sheet owned by `markaldenross@gmail.com`. No backend, no build step, no third-party email service.

**Tech Stack:** Plain HTML, CSS (existing `styles.css` design tokens), vanilla JS (inline `<script>`, matching the existing footer-year pattern), Google Apps Script.

## Global Constraints

- Hosting: Render static site. No backend, no build step, no npm.
- All contact/storage address: `markaldenross@gmail.com` (verbatim).
- No third-party email/marketing service.
- Match existing design: use CSS variables (`--accent`, `--bg-alt`, `--muted`, etc.), `.btn`/`.btn--primary` classes, "Oswald" for headings/labels, `var(--maxw)` column width.
- No test framework exists — verification is manual (browser + Sheet inspection).
- The existing `#contact` section blurb already says "let us add you to our correspondence" — the form belongs in/near that section.
- Target Sheet: https://docs.google.com/spreadsheets/d/1UeOeVYjY2nbxGrOqqyDy1eYSb4f8WauXkGd5wYQlAzI/ (already created by user).

---

### Task 1: Google Apps Script web app (USER-PERFORMED, Claude-guided)

This task is performed by the user in their browser (Google requires login as
`markaldenross@gmail.com`). Claude provides the exact script and steps, then
records the resulting deployment URL for Task 3.

**Files:**
- No repo files. Output is a deployment URL string consumed by Task 3.

**Interfaces:**
- Produces: a web-app URL of the form
  `https://script.google.com/macros/s/XXXXXXXX/exec` that accepts an
  `application/x-www-form-urlencoded` POST with fields `name`, `email`,
  `phone`, `comment` and appends a row `[timestamp, name, email, phone, comment]`.

- [ ] **Step 1: Open the bound script editor**

In the Sheet (the URL above), click **Extensions → Apps Script**. A new editor
tab opens with an empty `Code.gs` containing `function myFunction() {}`.

- [ ] **Step 2: Replace the script with the handler**

Select all in `Code.gs`, delete, and paste exactly:

```javascript
function doPost(e) {
  var lock = LockService.getScriptLock();
  lock.waitLock(30000);
  try {
    var sheet = SpreadsheetApp.getActiveSpreadsheet().getSheets()[0];
    if (sheet.getLastRow() === 0) {
      sheet.appendRow(['Timestamp', 'Name', 'Email', 'Phone', 'Comment']);
    }
    var p = e.parameter;
    sheet.appendRow([new Date(), p.name || '', p.email || '', p.phone || '', p.comment || '']);
    return ContentService
      .createTextOutput(JSON.stringify({ result: 'success' }))
      .setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    return ContentService
      .createTextOutput(JSON.stringify({ result: 'error', message: String(err) }))
      .setMimeType(ContentService.MimeType.JSON);
  } finally {
    lock.releaseLock();
  }
}
```

Click the **Save** (disk) icon.

- [ ] **Step 3: Deploy as a web app**

Click **Deploy → New deployment**. Click the gear next to "Select type" and
choose **Web app**. Set:
- **Description:** `mailing list signup`
- **Execute as:** **Me (markaldenross@gmail.com)**
- **Who has access:** **Anyone**

Click **Deploy**. Approve the authorization prompt (choose the
`markaldenross@gmail.com` account; click **Advanced → Go to … (unsafe)** → **Allow**
— this is normal for your own scripts).

- [ ] **Step 4: Copy the Web app URL**

Copy the **Web app** URL (ends in `/exec`). Paste it back to Claude. Claude
records it for Task 3. Do NOT commit this URL anywhere secret-sensitive — it is
a public write endpoint by design (acceptable: worst case is spam rows, mitigated
by the honeypot in Task 2).

- [ ] **Step 5: Smoke-test the endpoint (Claude-performed once URL is provided)**

Run (replace URL):

```bash
curl -s -L -X POST "https://script.google.com/macros/s/XXXX/exec" \
  --data-urlencode "name=Test Person" \
  --data-urlencode "email=test@example.com" \
  --data-urlencode "phone=555-1234" \
  --data-urlencode "comment=hello from curl"
```

Expected: `{"result":"success"}`. Then confirm with the user that a row appeared
in the Sheet. (User deletes the test row afterward.)

---

### Task 2: Signup form markup + styling + submit JS

**Files:**
- Modify: `index.html` (insert a `.signup` section inside `#contact`, after the
  "Email the band" button at line 99; add a submit script near the existing
  footer-year `<script>` at lines 113–115)
- Modify: `styles.css` (append a `/* ---------- Signup form ---------- */` block)

**Interfaces:**
- Consumes (from Task 1): the `/exec` web-app URL. Until Task 3 wires the real
  URL, use the literal placeholder constant `SIGNUP_ENDPOINT = 'PENDING_TASK_3'`
  and have the submit handler early-return with the error state if it still
  equals `'PENDING_TASK_3'`.
- Produces: a working form whose submit handler reads fields `name`, `email`,
  `phone`, `comment` and POSTs them form-encoded.

- [ ] **Step 1: Add the form markup inside the contact section**

In `index.html`, immediately after line 99 (`<a class="btn btn--primary" href="mailto:markaldenross@gmail.com">Email the band</a>`),
insert:

```html

      <form class="signup" id="signup-form" novalidate>
        <p class="signup__label">Or join our mailing list</p>
        <div class="signup__field">
          <label for="su-name">Name</label>
          <input type="text" id="su-name" name="name" autocomplete="name" required />
        </div>
        <div class="signup__field">
          <label for="su-email">Email</label>
          <input type="email" id="su-email" name="email" autocomplete="email" required />
        </div>
        <div class="signup__field">
          <label for="su-phone">Phone <span class="signup__opt">(optional)</span></label>
          <input type="tel" id="su-phone" name="phone" autocomplete="tel" />
        </div>
        <div class="signup__field">
          <label for="su-comment">Comment <span class="signup__opt">(optional)</span></label>
          <textarea id="su-comment" name="comment" rows="3"></textarea>
        </div>
        <!-- Honeypot: hidden from humans; bots fill it and get silently dropped. -->
        <div class="signup__hp" aria-hidden="true">
          <label for="su-website">Website</label>
          <input type="text" id="su-website" name="website" tabindex="-1" autocomplete="off" />
        </div>
        <button type="submit" class="btn btn--primary" id="su-submit">Sign me up</button>
        <p class="signup__status" id="su-status" role="status" aria-live="polite"></p>
      </form>
```

- [ ] **Step 2: Add the submit script**

In `index.html`, replace the existing script block (lines 113–115) with:

```html
  <script>
    document.getElementById('year').textContent = new Date().getFullYear();

    (function () {
      var SIGNUP_ENDPOINT = 'PENDING_TASK_3';
      var form = document.getElementById('signup-form');
      var statusEl = document.getElementById('su-status');
      var submitBtn = document.getElementById('su-submit');
      if (!form) return;

      var EMAIL_RE = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

      function setStatus(msg, kind) {
        statusEl.textContent = msg;
        statusEl.className = 'signup__status' + (kind ? ' signup__status--' + kind : '');
      }

      form.addEventListener('submit', function (ev) {
        ev.preventDefault();

        // Honeypot: silently pretend success if filled (bot).
        if (form.website.value) {
          form.reset();
          setStatus('Thanks for signing up!', 'ok');
          return;
        }

        var name = form.name.value.trim();
        var email = form.email.value.trim();
        if (!name) { setStatus('Please enter your name.', 'err'); form.name.focus(); return; }
        if (!EMAIL_RE.test(email)) { setStatus('Please enter a valid email.', 'err'); form.email.focus(); return; }

        if (SIGNUP_ENDPOINT === 'PENDING_TASK_3') {
          setStatus('Signup isn’t live yet — email us at markaldenross@gmail.com', 'err');
          return;
        }

        submitBtn.disabled = true;
        setStatus('Sending…', '');

        var body = new URLSearchParams({
          name: name,
          email: email,
          phone: form.phone.value.trim(),
          comment: form.comment.value.trim()
        });

        fetch(SIGNUP_ENDPOINT, { method: 'POST', body: body })
          .then(function (r) { return r.json(); })
          .then(function (data) {
            if (data && data.result === 'success') {
              form.reset();
              setStatus('Thanks for signing up!', 'ok');
            } else {
              throw new Error(data && data.message ? data.message : 'unknown');
            }
          })
          .catch(function () {
            setStatus('Something went wrong — email us at markaldenross@gmail.com', 'err');
          })
          .finally(function () { submitBtn.disabled = false; });
      });
    })();
  </script>
```

- [ ] **Step 3: Add the styles**

Append to `styles.css`:

```css
/* ---------- Signup form ---------- */
.signup {
  max-width: 460px;
  margin: 2rem auto 0;
  text-align: left;
}

.signup__label {
  font-family: "Oswald", sans-serif;
  text-transform: uppercase;
  letter-spacing: 0.15em;
  color: var(--muted);
  font-size: 0.85rem;
  text-align: center;
  margin-bottom: 1.25rem;
}

.signup__field {
  margin-bottom: 1rem;
}

.signup__field label {
  display: block;
  font-family: "Oswald", sans-serif;
  letter-spacing: 0.04em;
  font-size: 0.9rem;
  margin-bottom: 0.35rem;
}

.signup__opt {
  color: var(--muted);
  font-size: 0.8rem;
}

.signup__field input,
.signup__field textarea {
  width: 100%;
  background: var(--bg-alt);
  border: 1px solid #2a2a33;
  border-radius: 6px;
  color: var(--text);
  font-family: inherit;
  font-size: 1rem;
  padding: 0.65rem 0.8rem;
}

.signup__field input:focus,
.signup__field textarea:focus {
  outline: none;
  border-color: var(--accent-soft);
}

.signup__field textarea { resize: vertical; }

/* Honeypot — hidden from humans, present for bots. */
.signup__hp {
  position: absolute;
  left: -9999px;
  width: 1px;
  height: 1px;
  overflow: hidden;
}

.signup #su-submit {
  display: block;
  margin: 1.25rem auto 0;
  cursor: pointer;
  border: none;
}

.signup #su-submit:disabled {
  opacity: 0.6;
  cursor: default;
  transform: none;
}

.signup__status {
  min-height: 1.4em;
  margin-top: 1rem;
  text-align: center;
  font-size: 0.95rem;
}

.signup__status--ok { color: var(--accent-soft); }
.signup__status--err { color: #ff8a7a; }
```

- [ ] **Step 4: Verify locally (manual)**

Run: `python3 -m http.server 8000` and open `http://localhost:8000#contact`.
Expected:
- Form renders, styled to match the site (dark fields, Oswald labels, red button).
- Submitting empty → "Please enter your name."
- Bad email → "Please enter a valid email."
- Valid submit (endpoint still `PENDING_TASK_3`) → "Signup isn't live yet — email us at markaldenross@gmail.com".
- The honeypot field is not visible.

- [ ] **Step 5: Commit**

```bash
git add index.html styles.css
git commit -m "feat: add mailing-list signup form (endpoint pending)"
```

---

### Task 3: Wire the live endpoint and verify end-to-end

**Files:**
- Modify: `index.html` (the `SIGNUP_ENDPOINT` constant in the signup script)

**Interfaces:**
- Consumes (from Task 1): the real `/exec` web-app URL.

- [ ] **Step 1: Replace the placeholder endpoint**

In `index.html`, change:

```javascript
var SIGNUP_ENDPOINT = 'PENDING_TASK_3';
```

to the real URL from Task 1, e.g.:

```javascript
var SIGNUP_ENDPOINT = 'https://script.google.com/macros/s/XXXXXXXX/exec';
```

- [ ] **Step 2: Verify end-to-end (manual)**

Run: `python3 -m http.server 8000`, open `http://localhost:8000#contact`,
fill in real-looking values, submit.
Expected: "Thanks for signing up!" and a new row appears in the Sheet with
`[timestamp, name, email, phone, comment]`. Confirm with the user that the row
is present. User deletes the test row.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: wire mailing-list signup to live Apps Script endpoint"
```

- [ ] **Step 4: Deploy**

Push to the branch Render builds from; confirm the live site renders the form
and a submission lands in the Sheet.

---

## Self-Review Notes

- **Spec coverage:** Form fields (Name/Email/Phone/Comment) ✓ Task 2; required
  Name+Email, optional Phone+Comment ✓ Task 2 validation; persistence to Sheet
  ✓ Tasks 1+3; CSV export is a native Sheet feature (no code); honeypot ✓ Task 2;
  error fallback to markaldenross@gmail.com ✓ Task 2; inline success ✓ Task 2;
  $0 / no third-party service ✓ architecture.
- **Placeholders:** `PENDING_TASK_3` is an intentional, resolved-in-Task-3
  sentinel, not an unfilled gap. No other placeholders.
- **Type consistency:** Field names `name`/`email`/`phone`/`comment` match across
  the HTML inputs, the JS `URLSearchParams`, and the Apps Script `e.parameter`
  reads and row order.
