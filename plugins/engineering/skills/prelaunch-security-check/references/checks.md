# The 16 checks

Reference for `prelaunch-security-check`. For each check: look in the places listed, decide `OK`,
`Problem`, `Could not verify` or `Not applicable`, and keep the evidence. The rules in `SKILL.md`
(read-only, never print a secret, only this project, no invented findings) apply to every check here.

Contents: A. Access and data (1–3) · B. Input (4–7) · C. Abuse (8–11) · D. Deploy and operations (12–16)

## A. Access and data

**1. The server verifies who you are and what you may do** (OWASP A01, broken access control)
- List every endpoint, server action, RPC, serverless function and database rule that reads or writes data.
- For each: is the caller authenticated on the server, and is the specific record checked against the
  caller (owner, role, tenant, sharing)? A check only in the browser or only on the page route does not count.
- Look for IDs taken from the URL, body or query and used directly in a query without an ownership filter (IDOR).
- Supabase: is row-level security enabled on every table exposed through the API, do the policies actually
  restrict by user, and which grants do the `anon` and `authenticated` roles have? Is the service-role key
  used only on the server?
- Firebase (Firestore, Realtime Database, Storage): do the Security Rules restrict reads and writes by user
  (no `allow read, write: if true`)? Is the Admin SDK used only on the server?
- Admin functions: is the role checked on the server on every call?

**2. Keys are not in the browser; a leaked key is revoked and replaced at the provider**
- Search the working tree and the git history for hard-coded passwords, API keys, tokens, private keys,
  connection strings (common prefixes such as `sk-`, `sk_live_`, `AKIA`, `ghp_`, `xoxb-`, `-----BEGIN`,
  `service_role`). Follow rule 2: output paths, line numbers and commits only. For the working tree use
  `git grep --untracked -n -I -E '<regex>' | cut -d: -f1,2` (tracked and untracked files). Files ignored by
  git (build output, `.env`) are not searched this way: list which ignored paths exist and mark them
  `Could not verify` unless check 3 covers them. In a project without git use
  `grep -rIlE '<regex>' --exclude-dir=node_modules .` (file names only). For the history use
  `git log --all --no-ext-diff --no-textconv --diff-merges=separate --no-patch --format='%H %P' -G '<regex>'` (it searches changed lines
  that match the pattern, merge commits included, also when one key replaced another, and prints only the
  commit and its parents), then check that commit and every parent listed in the output with
  `git grep -l -E '<regex>' <commit>` (the very first commit has no parent). A prefix search is not a complete secret scanner: say so, and mark
  what it cannot cover as `Could not verify` (a dedicated scanner such as gitleaks or trufflehog, run by the
  user, covers more).
- Check which environment variables are exposed to the client bundle (`NEXT_PUBLIC_`, `VITE_`, `PUBLIC_`,
  `REACT_APP_`, `NUXT_PUBLIC_`) and whether any of them is a secret.
- Is `.env` in `.gitignore` (`git check-ignore`)? Was it ever committed (`git log --all --format=%h -- .env`)?
- Any secret that reached a public place (git history of a public repo, client bundle, a log) must be
  **revoked at the provider and replaced with a new one**. Deleting the line or rewriting history is not enough; the old key stays valid until it is revoked. Say this explicitly.
- Publishable keys (Supabase anon key, Stripe publishable key, public analytics IDs) are fine in the
  browser if check 1 holds.

**3. Nothing sensitive is publicly reachable** (OWASP A02, security misconfiguration)
- Files served from the web root or the static/public folder: `.env`, `.git`, database dumps and backups
  (`*.sql`, `*.sqlite`, `*.bak`, `*.zip`), logs.
- Debug and test pages, `phpinfo`, admin panels and dashboards without authentication.
- Public API docs and source maps are not a problem by themselves; judge them by content (internal
  endpoints, secrets, comments that reveal something sensitive).
- Server and container config: which ports are published to the internet. Docker publishes ports past
  host firewalls such as ufw; ports should bind to `127.0.0.1` or stay on an internal network.
- Debug mode switched off in production config. Default credentials changed. Storage buckets not public by accident.

## B. Input

**4. User text is displayed as text, not as code** (XSS and SQL injection)
- XSS: find every place where user-controlled or third-party text is rendered as HTML:
  `innerHTML`, `outerHTML`, `document.write`, `dangerouslySetInnerHTML`, `v-html`, `set:html`, `{@html}`,
  `|safe`, `raw`, unescaped template output, markdown rendered without sanitising, user-controlled
  `href`/`src` (`javascript:` URLs). AI-generated code fails this very often, so look carefully.
- SQL/NoSQL injection: queries built by string concatenation or template strings with user input
  instead of parameters. ORMs help, but check raw queries and dynamic `ORDER BY`/column names.
- Also: user input passed to shell commands, file paths (path traversal), redirects (open redirect).

**5. File uploads have rules**
- An allow-list of types, not a block-list. The server checks the extension, the declared type and the
  actual content together (magic bytes alone are not enough; parse the file where it matters, for example
  re-encode images).
- Size and count limits enforced on the server and on the proxy.
- No executable files. SVG and HTML rejected, or allowed only with deliberate sanitising and safe
  delivery. SVG can carry scripts: they do not run when shown through `<img>`, but they do when the file is
  opened directly on the app's own origin. An upload that merely opens is not a finding by itself; one that
  renders as an active page on the app's domain is.
- Files renamed to random names, stored outside the web root, ideally in object storage on a different
  domain, served with a correct `Content-Type`, `X-Content-Type-Options: nosniff` and
  `Content-Disposition: attachment` where they should not render.
- Upload endpoints require authentication and are rate limited.

**6. Prompt injection: an AI that reads foreign text has no more rights than the user**
- Find every place where a language model reads text the developer did not write: e-mails, web pages,
  uploaded documents, reviews, user messages, tool results, database content written by users.
- For each, list which model processes it, the tools, credentials and data it can reach at that moment,
  and whether it runs unattended.
- Flag any path where an instruction hidden in that text could trigger an action with consequences
  (send, delete, pay, read other users' data, call an internal API) without a human confirming it.
- Check exfiltration paths too: even a model without write tools can leak data through its answer, a
  search query, a generated link or image URL, or a tool call that reaches the internet.
- Rate it by exposure: what data the model can reach, what tools it holds, whether it can send data out,
  whether it runs unattended. Resistance to injection differs between models, but no model is a security
  boundary, so do not lower the severity because of the model alone.
  Do not quote resistance percentages.
- Controls to look for: least privilege (the model acts with the current user's permissions only),
  confirmation for irreversible actions, no secrets in the model's context, and model output never
  treated as authorization (code and the user's permissions decide what may run; see also check 4).
  A sentence in the system prompt helps but should not be the only control.

**7. Every input has a size limit**
- The core of this check: no field, no request body and no list is accepted without a maximum size,
  validated on the server **before** anything expensive happens.
- Request body size limit on the framework and on the reverse proxy.
- Passwords: cap the length (for example 128 characters) before hashing. Be precise about the algorithm:
  bcrypt processes at most 72 bytes; depending on the library a longer input is silently truncated or
  rejected, so check which one applies (and whether a pre-hash step exists). For Argon2 and scrypt the
  main cost comes from their parameters, but receiving and processing a huge input still costs time and
  memory. Some PBKDF2 and sha256crypt/sha512crypt implementations processed the password repeatedly, so
  work grew with its length (Django CVE-2013-1443). Do not overstate the risk.
- Impact depends on hosting and is not guaranteed either way: on a single server huge or repeated inputs
  can slow it down or take it down for everyone; auto-scaling/serverless hosting has its own limits (for
  example Vercel Functions reject request bodies over 4.5 MB, as of November 2026) and may reject such requests, while what
  gets through can raise the bill or exhaust the database and other services. Check the hosting limits
  together with the app's own limits.
- Limits on array length, JSON depth, number of items per request, pagination on list endpoints.
- Text sent to an AI model capped in length (cost and abuse).
- Regular expressions run on user input checked for catastrophic backtracking.

## C. Abuse

**8. Forms and login are rate limited**
- Public endpoints: login, registration, password reset, contact and lead forms, e-mail or SMS sending,
  AI calls, search, anything that writes.
- For each: is there a limit per IP and per account/e-mail, what happens when it is exceeded, can it be
  bypassed (limit only in the browser, a header the client controls, a new session)?
- Bot protection (honeypot, CAPTCHA/Turnstile) verified on the server on every protected endpoint, not
  only shown in the form. If the server does not verify the token, a bot skips the form and calls the
  endpoint directly.
- Bot protection is a complement, never a replacement for a server-side rate limit: even when wired
  correctly it lowers the risk but does not remove it (paid solving services exist).
- In the Xint.io 2026 analysis of AI-written code, resource exhaustion / DoS findings (missing rate limits
  among them) were the largest group. Do not mark this check OK without seeing the code that enforces it.

**9. The AI bill has a ceiling**
- Every call to a paid AI API goes through the server. For features meant for signed-in users, check
  authentication and permissions; for features that are public on purpose (for example a FAQ chatbot),
  check limits on anonymous use (per IP, per session, per day). In both cases require limits on input,
  output, retries and total spend.
- `max_tokens` (or equivalent) set; input length capped; agent loops have a step limit; retries are bounded.
- A global daily budget or kill switch in the app, enforced by the app's own code. Do not rely on the
  provider alone: some provider limits only send an alert, and even an enforced cap may not stop spending
  instantly.
- Remind the user to set an alert and, where the provider offers it, an enforced spending cap, and to check
  which of the two they actually have (you cannot do that part; put it under "Test it yourself").

**10. One mailbox, one reward: e-mail aliases**
- How is e-mail uniqueness checked at sign-up, trials, coupons, webinar registrations, referral rewards?
- At Gmail, `name@gmail.com`, `name+test@gmail.com` and `na.me@gmail.com` are one mailbox, but the app
  sees three accounts. Recommend normalising before the eligibility check:
  - always lowercase the domain;
  - `gmail.com` / `googlemail.com`: lowercase the local part, strip dots in it, strip the `+tag` part,
    treat both domains as one (this does not apply to Google Workspace custom domains);
  - for other domains change the local part only by rules verified for that provider (plus addressing is
    supported for example by outlook.com, hotmail.com, live.com, proton.me, protonmail.com, fastmail.com),
    kept as a configurable per-domain list;
  - leave every other domain untouched: there a plus, a dot or letter case can be part of a real,
    distinct address.
- Store the original address for sending. Use the normalised one only to check eligibility for a trial,
  coupon or registration, never to merge existing accounts automatically.
- Normalising limits repeated claims from one mailbox; it does not prove who a person is.
- Only relevant when something of value is given per account. Otherwise mark it low severity.

**11. Disposable e-mails are rejected**
- Is the address validated on the server at sign-up and in lead forms: syntax, a domain that can receive
  mail (follow SMTP rules: a missing MX record alone is not a reason to reject, mail can fall back to the
  address record), not a disposable-mailbox domain (a maintained disposable-domain list or a verification
  service such as ZeroBounce)? Treat this as an extra layer that sometimes gets it wrong.
- Is the address confirmed by a click in a confirmation e-mail before a reward is unlocked or further
  e-mails are sent?
- Why it matters: bounces damage the sender reputation and deliverability to real customers.

## D. Deploy and operations

**12. Nothing reaches production without a check**
- Is the main branch protected, are changes merged through pull requests, do checks (tests, lint, type
  check, build) run before deploy?
- Is there a review step by someone other than the author, for example a second AI agent with a clean
  context or a review tool on the pull request?
- Is there a way to roll back a bad deploy?
- From the repository you can see CI config and branch conventions; branch protection itself usually
  needs the user to look in the hosting settings ("Test it yourself").

**13. Dependencies are real and without known holes** (OWASP A03:2025, software supply chain)
- Every dependency exists in the registry, is not a typo of a well-known package, was not created
  very recently with almost no downloads (AI models hallucinate package names and attackers register
  them: slopsquatting), and is actually used.
- Inspect dependencies statically first: read the manifest and the lockfile, look for unpinned versions,
  git or URL sources, and packages you do not recognise.
- Known vulnerabilities need an audit tool, and most of them contact a server: `npm audit` and `pnpm audit`
  send dependency names and versions to the registry. Before running one, tell the user which server it
  contacts and what it sends, and run it only after an explicit yes. Never run `yarn npm audit` yourself:
  Yarn can execute a project-supplied binary (`yarnPath`) and plugins first. Do not run `pip-audit` on
  requirements without pinned hashes (it can resolve and build packages). In every case you do not run,
  give the user the command and mark it `Could not verify` (rule 1). Report findings by severity.
- A package that exists, is old or passes the audit is not proven safe; say so when it matters.
- Lockfile committed; no `curl | sh` installs or unpinned scripts from third-party CDNs in the pages.

**14. The server has a watchman**
- What runs and in which version: OS packages, PHP/Node/Python runtime, database, web server,
  CMS plugins and themes. Flag end-of-life versions (endoflife.date).
- Is there any automated watch: a scheduled job that compares installed versions against vulnerability
  sources (OSV.dev, NVD, WPScan for WordPress, endoflife.date) and notifies the owner, or at least
  automatic security updates and Dependabot/Renovate for the repo?
- If there is none, propose one (a daily cron script that only notifies, never updates on its own).

**15. Headers and HTTPS**
- HTTP redirects to HTTPS; `Strict-Transport-Security` set (without preload it does not protect the very
  first visit over plain HTTP).
- `Content-Security-Policy` present. A new policy may start in `Content-Security-Policy-Report-Only`, but
  report-only protects nothing: recommend moving to the enforced header once the reports are reviewed.
- Protection against framing (`frame-ancestors` in an enforced CSP, or `X-Frame-Options`) to prevent
  clickjacking; it does not work from a report-only policy. `X-Content-Type-Options: nosniff`;
  `Referrer-Policy`.
- Cookies: `Secure` and `HttpOnly` on session and auth cookies; `SameSite` chosen to fit the login and
  integration flows. CORS not `*` together with credentials.
- A good grade on securityheaders.com or Mozilla Observatory is a useful signal, not proof of security.
- Where headers are set depends on the stack (framework config, `_headers` file, nginx, hosting panel).
  A live check with securityheaders.com or Mozilla Observatory goes under "Test it yourself".

**16. Errors and crashes: the owner knows first, the user never sees a stack trace** (OWASP A10:2025)
- Production error pages and API error responses contain no stack traces, file paths, SQL, versions.
- Errors are caught and logged; nothing fails open (an exception in an auth check must deny, not allow).
- Error tracking exists (for example Sentry) on server and client, scrubbed of passwords, tokens and
  personal data, with a notification to the owner (e-mail, Telegram, Slack).
- Logs do not contain secrets or full personal data.
