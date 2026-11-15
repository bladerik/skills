---
name: prelaunch-security-check
description: Runs a read-only, 16-point security check on an AI-built (vibe-coded) web app or website before it goes live, then writes a plain-language report with severity and fixes and offers to fix findings one at a time. Covers access control, leaked keys, public files, XSS and SQL injection, uploads, prompt injection, input limits, rate limiting, AI spend caps, e-mail aliases, disposable e-mails, review before deploy, dependencies, server patching, security headers and error handling. Use when the user asks for a security check or audit, asks whether their app is safe to launch, or is about to deploy an app to real users.
disable-model-invocation: true
---

# Prelaunch security check

16 checks for an app built with AI, run before real people use it. The human-readable version with
manual tests (in Slovak) is at https://codingfry.sk/checklist/bezpecnost.

The user is often not a programmer. They built the app with an AI and want to know whether it is safe
to put in front of real people. Find what is actually wrong, explain it in plain words, and help fix it
step by step.

## Rules (they override everything below)

1. **Read-only until the user says otherwise.** During the check do not edit, create, delete, install,
   commit, push, deploy or migrate anything, and do not run the project's own scripts (`npm run ...`,
   build, dev server, tests, migrations, `postinstall`). Allowed: reading files, listing files and `git` read
   commands. Do not run package managers or audit tools on your own: they can load project-supplied code
   (for example a Yarn `yarnPath` binary or plugins) and most audits send the dependency list to a server.
   If a check would need an install, a resolver run, a network audit or running code, mark it
   `Could not verify` and give the user the command; the exception in check 13 needs the user's explicit yes.
2. **Never print a secret, not even in tool output.** Not in full, not partially, not "the first 4
   characters". Run searches so that matched lines never reach your output: list only paths, line numbers
   and commits (for example `git grep --untracked -n -I -E '<pattern>' | cut -d: -f1,2`, `git grep -l`,
   `git log --all --no-ext-diff --no-textconv --diff-merges=separate --no-patch --format='%H %P' -G '<regex>'`).
   Do not use `git log -p`, `git show` or `cat` on files that may hold secrets, and never open `.env`
   files; to check whether `.env` is ignored or tracked, use `git check-ignore` and `git ls-files`. Report
   file, line, the kind of secret and (for git history) the commit. Do not copy secret values into the
   chat or into any file.
3. **Only this project.** Do not send requests to servers you were not asked to test. Do not run
   load, brute-force or injection attempts against a live site. If a check needs a live test, put it in
   the "Test it yourself" section for the user.
4. **No invented findings.** Every finding needs evidence: a file and line, a config value, or command
   output. If you cannot verify a check from the code, say "could not verify" and say what would verify it.
5. **Write the report in the user's language** (the language they write to you in). Plain words, short
   sentences, explain a term the first time you use it. Keep file names, commands and code in their
   original form.
6. **This is not a penetration test.** Say so once at the end of the report. Recommend a human review
   when the app holds sensitive customer data or handles money.

## Step 1: get oriented (ask at most 3 short questions)

Find out, from the repository first and from the user only if it is unclear:

- What the app does and what data it holds (customer names, e-mails, payments, health data, nothing).
- The stack: framework, database, auth provider, hosting, whether it calls a paid AI API.
- Whether it is already live and at which address.

Skip any check that cannot apply (no uploads means check 5 is `Not applicable`) and say so in the report
instead of padding it.

## Step 2: run the 16 checks

Read [references/checks.md](references/checks.md) and work through every check in order. Each one lists
where to look and what counts as a problem. For each check decide `OK`, `Problem`, `Could not verify` or
`Not applicable`, and keep the evidence.

| # | Check | Group |
|---|-------|-------|
| 1 | The server verifies who you are and what you may do | Access and data |
| 2 | Keys are not in the browser; a leaked key is revoked and replaced | Access and data |
| 3 | Nothing sensitive is publicly reachable | Access and data |
| 4 | User text is displayed as text, not as code (XSS, SQL injection) | Input |
| 5 | File uploads have rules | Input |
| 6 | An AI that reads foreign text has no more rights than the user | Input |
| 7 | Every input has a size limit | Input |
| 8 | Forms and login are rate limited | Abuse |
| 9 | The AI bill has a ceiling | Abuse |
| 10 | One mailbox, one reward (e-mail aliases) | Abuse |
| 11 | Disposable e-mails are rejected | Abuse |
| 12 | Nothing reaches production without a check | Deploy and operations |
| 13 | Dependencies are real and without known holes | Deploy and operations |
| 14 | The server has a watchman (patches and end-of-life versions) | Deploy and operations |
| 15 | Security headers and HTTPS | Deploy and operations |
| 16 | Errors reach the owner; users never see a stack trace | Deploy and operations |

## Step 3: the report

Write the report in the user's language, in this shape:

1. **Result in one or two sentences**: whether, within the scope of these 16 checks, you found a blocking
   problem, what stayed unverified, and what needs an expert. Never give a security guarantee or a general
   approval to launch.
2. **Summary table** of all 16 checks: number, name, status (`OK`, `Problem`, `Could not verify`,
   `Not applicable`), highest severity found.
3. **Findings**, most severe first. For each:
   - severity: `Critical` (someone can read or change other people's data, take over an account, or run
     up a bill today), `High` (exploitable with some effort, or leaks sensitive information),
     `Medium` (makes an attack easier, or abuse that costs money or reputation), `Low` (hardening);
   - where: file and line, config key, or command output (never a secret value);
   - what can happen, in one or two plain sentences with a concrete example for this app;
   - how to fix it, concretely for this stack;
   - effort: small / medium / large;
   - who does what: one sentence on what the user has to do themselves and what you can do for them.
4. **Test it yourself**: the checks you could not verify from code, as short manual tests the user can do
   in a browser (two test accounts and changing an ID in the URL; opening `/.env` and `/.git/config` on
   their domain; submitting a form 20 times; securityheaders.com; provider spending limit; branch
   protection). For every test say where to click and what a good and a bad result looks like.
   Start this section with the rule: test only your own app, ideally in a test environment, with your own
   test accounts and data, with real payments, e-mails and SMS switched off.
5. **What is fine**: one short list, so the user knows what was actually checked.
6. One closing line: this is a checklist-based review, not a penetration test or a security guarantee.

Translate the section names, statuses and severities into the user's language. Slovak example:
`V poriadku`, `Problém`, `Nedá sa overiť z kódu`, `Netýka sa`; `Kritické`, `Vysoké`, `Stredné`, `Nízke`;
"Test it yourself" is `Skús sám`.

## Step 4: offer fixes, one at a time

After the report, ask whether the user wants to start fixing, beginning with the most severe finding.
Then, for each finding in order:

1. Say what you will change and in which files. Wait for a yes.
2. Make the smallest change that closes the hole. Do not refactor unrelated code.
3. Run the project's existing checks (tests, type check, build) if there are any.
4. Show what changed and how the user can verify it themselves.
5. Move to the next finding only after the user confirms.

Hand these back to the user instead of doing them yourself: rotating keys at a provider, setting
spending limits, changing DNS or hosting settings, anything that needs their login. Give exact steps.
Never commit, push or deploy unless the user asks for it.
