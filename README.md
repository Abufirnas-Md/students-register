# Student Portal

Static HTML/CSS/JS front end + Supabase Auth + PostgreSQL (Row Level Security). No build step, no framework.

```
student-portal/
  index.html login.html register.html reset.html dashboard.html
  css/style.css
  js/  supabase-client.js  auth.js  validators.js  register.js  login.js
       reset.js  profile.js  dashboard.js  index.js
  supabase/  schema.sql  policies.sql
  tests/     rls_test.sql  validators.test.mjs
  netlify.toml  .gitignore  README.md
```
(`reset.html`/`reset.js` implement the secure password-reset page, `validators.js` holds pure validation so it can be unit-tested, `index.js` is the redirect.)

## Test status (read this first)

| Area | Status |
|---|---|
| Table, constraints, sign-up trigger, duplicate register no./student ID/email rejection | **Tested** on a local PostgreSQL 16 with stubbed Supabase roles (`tests/rls_test.sql`) |
| RLS: student sees only own row; cannot update another's row; cannot change register no., student ID, college email, id; cannot insert/delete; anon cannot read; allow-list table hidden; column-grant + trigger both block identity changes; admin (SQL editor) can still correct | **Tested** (same harness) |
| Validators (password strength, phone, DOB, dropdown values) | **Tested** (`node tests/validators.test.mjs`) |
| JS syntax of every module; every element id the JS uses exists in the HTML | **Checked** |
| Real Supabase Auth: sign-up, email confirmation, login, remember-me, session refresh/expiry, password-reset email, logout | **NOT tested**: needs a live Supabase project. Follow section 4 below. |
| Page appearance in real browsers / on phones / keyboard-only use | **NOT tested** |
| Netlify deployment | **NOT tested** |

The harness stubs `auth.users` and `auth.uid()`; real Supabase uses the same mechanism, but please re-run the cross-student test (4.10) on your real project. Nothing here is "certified secure"; see Limitations.

## 1. Create the Supabase project
1. https://supabase.com → **New project**. Pick a region near you, set a strong database password (store it in a password manager; it is **never** used in this website).
2. Wait for provisioning.

## 2. Configure authentication
**Authentication → Providers → Email**: enable. Turn **Confirm email ON** (recommended: proves the student owns the address).
**Authentication → Sign In / Providers → Password policy** (or Settings): minimum length 10, require upper/lower/digits/symbols (matches the client check).
**Authentication → URL Configuration**: Site URL = your site (`http://localhost:8000` while testing, your Netlify URL in production). Add Redirect URLs: `http://localhost:8000/login.html`, `http://localhost:8000/reset.html`, and the same two paths on your Netlify domain.
**Authentication → Rate Limits**: keep defaults or tighten (sign-ups, sign-ins, emails per hour). This is the abuse protection; there is no custom auth endpoint.
**Authentication → Sessions**: optionally set *Time-box sessions* / *Inactivity timeout* (paid plans) and JWT expiry (default 1 hour; refresh is automatic).
**Email templates**: default confirmation and reset templates work. For real use, configure custom SMTP (the built-in sender is heavily rate-limited).
**Do not create any admin account with a hardcoded password.** Staff use the Supabase dashboard (own login, ideally with MFA).

### Authorized account creation
Only people with a college mailbox should register. Enforced in the database, not just JS:
```sql
insert into public.allowed_email_domains values ('mycollege.edu.in');  -- run in SQL Editor
```
While the table is empty, any domain is accepted. Also set `ALLOWED_EMAIL_DOMAINS` in `js/supabase-client.js` for a friendlier message (cosmetic only). Confirm-email ON then proves mailbox ownership. For a stricter roster-based approach (pre-loaded register numbers that must match), add a `roster` table and check it inside `handle_new_user()`.

## 3. Run the SQL
SQL Editor → paste and run **`supabase/schema.sql`**, then **`supabase/policies.sql`** (in that order). Both are safe to re-run.
- `student_profiles`: all fields from the brief; register no./student ID stored as `text`; `UNIQUE` + `CHECK` constraints.
- `handle_new_user()` trigger: creates the profile row from sign-up metadata, so it works even before email confirmation. Duplicate or malformed values abort the sign-up.
- `check_registration_available()`: lets the form say *which* ID is taken before sign-up (the UNIQUE constraints stay the real guard; this does let someone probe whether an ID exists).
- `student_profiles_guard()` trigger: blocks API users from changing identity fields and rejects implausible dates of birth; stamps `updated_at`.
- `allowed_email_domains`: optional allow-list (no API access).
- RLS: select/update own row only; column-level `GRANT UPDATE` excludes identity columns; no insert/delete rights.

Verify: **Table Editor → student_profiles** should show an "RLS enabled" badge.

## 4. Configure, run and test
1. **Settings → API**: copy *Project URL* and the *publishable (anon)* key into `js/supabase-client.js`. **Never** use the `service_role` key.
2. Serve locally (ES modules need http, not `file://`): `cd student-portal && python3 -m http.server 8000` → http://localhost:8000
3. **Register**: `/register.html`, e.g. Test Student / `0012345` / `S-001` / your email / strong password. With confirmation ON, click the email link, then log in. (Leading zeros are preserved.)
4. **Log in and complete the profile**: Edit profile → fill required fields (`*`) → Save changes. Completion bar updates.
5. **Persistence**: Log out (confirm dialog) → log in again → data is loaded, not duplicated. Check Table Editor: still one row.
6. **Duplicates / weak passwords**: register again with the same register number, then same student ID, then same email; try `password`, `Password1`, mismatched confirm. Each must be refused with a clear message.
7. **Cross-student isolation**: register a second student. In the browser console on the dashboard of student A run:
   ```js
   const {supabase}=await import('./js/supabase-client.js');
   (await supabase.from('student_profiles').select('*')).data.length        // expect 1
   await supabase.from('student_profiles').update({phone:'+911111111111'}).eq('register_number','<B number>')  // expect 0 rows changed
   await supabase.from('student_profiles').update({register_number:'HACK123'}).eq('id',(await supabase.auth.getUser()).data.user.id) // expect error
   await supabase.from('student_profiles').delete().neq('id','00000000-0000-0000-0000-000000000000') // expect error
   ```
8. **Password reset**: login page → type email → *Forgot password?* → open emailed link → set new password → log in. Reuse of an old/expired link must say it is invalid.
9. **Remember me / expiry**: log in with the box ticked → close and reopen browser → still signed in. Untick → close the browser → signed out. For expiry: delete the `sb-…` entries in DevTools → Application → Storage, reload `dashboard.html` → redirected to login. (Server-side timeouts: section 2.)
10. Confirm no passwords appear in Table Editor or the browser console/network logs you control (Supabase stores only hashes in `auth.users`).

## 5. Deploy: GitHub + Netlify
1. `git init && git add . && git commit -m "Student portal"`; create an empty GitHub repo; `git remote add origin <url> && git push -u origin main`. (`.gitignore` excludes secrets; the publishable key in `supabase-client.js` is meant to be public. Double-check you never pasted a `service_role` key.)
2. Netlify → **Add new site → Import from Git** → pick the repo. Build command: *(empty)*. Publish directory: `.`. Deploy.
3. Add the Netlify URL to Supabase **Site URL** and **Redirect URLs** (section 2).
4. Repeat tests 3–9 on the live URL. `netlify.toml` sets security headers incl. a CSP that allows only this site, jsDelivr (Supabase JS, pinned to 2.45.4) and `*.supabase.co`.

## Login identifier
Standard Supabase Auth uses **email + password**, so login uses the college email; the register number is the unique identity key in the database. Logging in by register number would need a server-side lookup (e.g. a rate-limited Supabase Edge Function that maps register number → email server-side without returning the email). That is not included, to avoid an untested custom auth endpoint.

## Security notes
- Passwords are handled only by Supabase Auth; the app never reads or stores them or touches the browser's password manager (inputs carry proper `autocomplete` hints so the browser can offer to save).
- Authorization lives in PostgreSQL (RLS, column grants, triggers), not in hidden buttons. Client and DB both validate.
- All user text is written with `textContent`/`.value`, never `innerHTML`. Console logging only records error messages, never form values.
- Login failures show a generic message; reset requests give the same reply whether or not the account exists.

## Limitations
- Real Supabase flows and browser rendering untested (see table above).
- Changing the auth email (Supabase `updateUser`) is not synced to `college_email`; the UI offers no email change. Disable "Allow email change" if you do not want it, or add a sync trigger.
- `check_registration_available` reveals whether an ID is registered.
- Profiles are never deleted from the UI; deleting the Auth user in the dashboard cascades to the profile.
- No admin UI; staff use the Supabase dashboard (SQL Editor edits bypass the identity guard on purpose, for corrections).
- Data is personal data (DOB, phone, address): check your institution's privacy/legal requirements, publish a real privacy notice, and consider a retention policy.

## Troubleshooting
| Symptom | Fix |
|---|---|
| Yellow "Setup needed" banner | Fill URL/key in `js/supabase-client.js` |
| Blank page / module error | Don't open via `file://`; use a local server |
| "Registration was rejected" | Duplicate ID, bad format, or email domain not in allow-list (Supabase → Logs → Postgres shows the real message) |
| "Dashboard: No profile found" | Account was created before `schema.sql` ran; insert the row manually or recreate the user |
| Confirmation/reset link goes to wrong site or "invalid" | Fix Site URL / Redirect URLs; links are single-use and expire |
| Emails not arriving / "rate limit" | Built-in mailer is limited; configure custom SMTP, check spam |
| Save says "not allowed" | You edited an identity field, or RLS/grants were changed; re-run `policies.sql` |
| Works on localhost, breaks on Netlify | Add the Netlify domain to Redirect URLs; check CSP in browser console |
