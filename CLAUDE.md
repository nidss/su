# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**Souped Up 2026** — a dark, racing-themed, bilingual (Thai default / English) car-racing competition registration web app. The main app is a **self-contained `index.html`**: no build step, no framework, no dependencies to install. All media (logo, backgrounds, ID/racer photos, favicon fallback) are embedded as base64/SVG data URIs so the one file runs anywhere. Palette: mint `#6FFFE8`, lime `#D1FF03`, navy `#0C142A`.

Two companion pages (`vip.html`, `vip-dashboard.html`) are **separate standalone HTML apps** that `index.html` embeds in same-origin iframes (see "Companion iframe apps" below) — they are not part of `index.html` and have their own `<script>`, `state`, and i18n.

A fourth page, **`checkin.html`**, is a standalone staff check-in page (reference-code / QR entry that looks up racers and VIP holders). It is **not** embedded as an iframe — `index.html` links to it by a plain `href="checkin.html"` in the footer (no `?v=` cache-buster), so editing it needs no `?v=` bump and no `souped-up.html` regeneration. It has its own `state`, i18n, and helpers.

## The two output files — keep them in sync

- `index.html` — the app, served by **GitHub Pages** at https://nidss.github.io/su/ (Settings → Pages → branch `main`, root). Full HTML document.
- `souped-up.html` — a **derived** copy for the Claude **Artifact** (https://claude.ai/code/artifact/f2b8159f-ea2f-46ca-926d-cfa8140f2ae0). It is `index.html` with the document skeleton stripped, because the Artifact host wraps the file in its own `<head>/<body>`.

**Never hand-edit `souped-up.html`.** After every `index.html` change, regenerate it:

```bash
node -e '
const fs=require("fs");
let s=fs.readFileSync("index.html","utf8");
const start=s.indexOf("<link rel=\"preconnect\"");
s=s.slice(start).replace(/\n<\/head>\n<body>\n/,"\n").replace(/\n<\/body>\n<\/html>\s*$/,"\n");
fs.writeFileSync("souped-up.html",s);'
```

This drops lines 1–9 (`<!DOCTYPE>` … `<link rel="icon">`), the standalone `</head>` and `<body>`, and the trailing `</body></html>`, keeping everything from the first `<link rel="preconnect">` through the closing `</script>`.

## Companion iframe apps

`index.html` embeds two independent same-origin apps by iframe `src` with a cache-busting query — **bump that `?v=` string whenever you edit the embedded file**, or the deployed iframe serves a stale copy:

- **`vip.html`** (`SCREENS.vip`, `vip.html?v=YYYYMMDDx`) — the buyer-facing VIP Village purchase flow. `index.html` prefills it and reads results back via `postMessage`; purchases land in `state.profile.vips`.
- **`vip-dashboard.html`** (admin VIP tab, `vip-dashboard.html?v=YYYYMMDDx`) — a **complete second app** (its own `state`, i18n `L.th`/`L.en`, `el`/`esc`/`t`/`money`, and an admin dashboard). The VIP buyer table and its filter bar live here, **not** in `index.html`'s `DASH`. Registrations are read from the shared `suvip_regs` localStorage key. `renderDashboard()` builds the table from `filteredRegs()`; `seedMockIfEmpty()` seeds demo rows on first load.

Each embedded file carries its own `?v=` suffix (they differ), so bump only the one you changed, in **both** `index.html` and `souped-up.html`.

## Architecture (all inside `index.html`, one `<script>`)

A vanilla-JS screen router drives a 6-step racer-registration wizard. Account signup and mandatory personal-information onboarding happen before the wizard and are hidden from its stepper. Read these to understand the whole app:

- **`state`** — single global object holding all app data. `STEPS = ["terms","racer","parking","summary","pay","done"]`.
- **`SCREENS[step]()`** returns the screen's HTML string; **`SCREEN_INIT[step]()`** (optional) wires its events after it's injected. `render()` sets `el('screen').innerHTML = SCREENS[state.step]()` then calls the matching `SCREEN_INIT`. `go(step)` sets `state.step` and re-renders.
- Because `render()` blows away and rebuilds the DOM, in-progress form values are captured back into `state` before any re-render via `persistCurrentForm()` (used by language switch) and per-screen `collect*`/`save*` helpers. When adding a screen, follow this collect-then-render pattern or typed data is lost.
- **i18n**: `I18N.th` / `I18N.en` key maps; `t(key)` looks up the current `state.lang`. Every user-facing string goes through a key in **both** language blocks.
- **Shared helpers**: `el(id)`, `esc(str)` (HTML-escape — use for all interpolated user/data text), `money()`, `toast()`, `collect(form,obj)`, `validateScreen(scope)` (validates `[data-req]` inputs, files via `state.files`), and field builders `fieldText`/`fieldSelect`/`fieldUpload`.

### Key data shapes

- **Racers**: `state.racers` is an array whose registration UI uses only its first racer. Each racer has `{id, applicantType, r_rname, r_rsurname, r_idaddr, r_team, entries[], collapsed}`, while the current form exposes one entry `{cls, engine, carno}` and has no card header, border, collapse control, add-racer button, or add-class button. `state.racer` is a flattened mirror of that racer for downstream screens. The prize-payment account is rendered after the two racer photo uploads.
- **Parking**: `pkSlots()` derives one booking slot per race-class entry across all racers; `state.parking.assign` maps `slotKey → pit`. A selected `privatePit` is only an additional flag: it never disables the checkbox, clears assignments, or bypasses the requirement to select every pit. Cost is per car: `PRICES.entry + PRICES.deposit` × number of slots.
- **Account/Profile**: `state.account = {username, email, phone, password}`. Demo login uses the two records in `DEMO_USERS`: the new/incomplete-user record starts onboarding with an empty profile (via `enterPostAuth()`), while the existing-user record loads a populated profile with racers, history, and VIP. Registration and post-payment auto-login also populate `state.account`. `state.profile` (built by `buildProfile` for the existing demo user, or `buildProfileFromReg` after payment) feeds the "My Profile" screen.
- **Bank accounts**: `state.bank` stores both account groups. The applicant deposit-refund fields (`rf_*`) are rendered and saved with the personal-information form; `rf_type` first selects Thai-bank or international-bank mode and toggles the corresponding required fields. Winner prize-payment fields (`pz_*`) are part of the racer form and use the same type-first pattern. Both Thai-account name/surname pairs are read-only: prize values mirror the first racer, while refund values mirror the applicant live. Company applicants hide the refund surname.
- **Applicant types**: `state.applicant.a_type` is `person` or `company` and is selected once from a required dropdown in the personal-information form. The chosen value applies to every racer under that applicant. Company applicants store registered name, tax ID, head-office/branch details, contact person, registered address, and company certificate while mapping the contact name into `a_name` / `a_surname` for downstream compatibility.
- **Payment window**: `state.paymentDeadline` starts when the summary screen is first opened. `paintPaymentCountdown()` displays the remaining two-hour window, updates once per second, and disables payment after expiry. Racer success and racer receipt views omit check-in QR codes; VIP receipts retain their own QR.
- **Choice-before-onboarding flow**: The `choose` action-picker (racer vs VIP) is enabled with `const CHOOSE_ENABLED=true`. After register/login-new, `enterPostAuth()` opens the picker. A member choosing racer without personal info stores the choice in `state.pendingChoice` and opens `onboard`, which collects the applicant information and required deposit-refund account before registration. The VIP option opens the original `suvip` UI directly in the isolated same-origin `vip.html` frame and prefills it from the signed-in member account via `postMessage`; the frame resizes itself, records completed purchases in `state.profile.vips`, and returns the member button to the shared profile. If a VIP-only member later adds a racer, the profile routes them through the prefilled onboarding form to collect the applicant and refund-account information first. Duplicate auth/profile/admin markup was removed from the embedded page.

### Hidden entry points

- **Admin dashboard**: `Ctrl+Shift+Z` → admin login → shared Dashboard header with a real logout action, followed by a full-width registration/VIP switch. Registration retains the KPI cards + 2-tab paginated table (`DASH` object, `renderDash`/`paintDashTable`), while VIP embeds only the KPI/table portion of the original `suvip` dashboard from `vip-dashboard.html`, backed by the shared `suvip_regs` localStorage data.
- **My Profile**: reached by logging in or finishing a registration; tabs for account info (+ change password), racers, and history.

### Images & PDFs

- Real photos are embedded as base64 in the `IMG_RACER` / `IMG_IDCARD` consts and the receipt `logo-main-b.png` `<img>`. **Changing an `asset/*.png` requires re-embedding its base64 into `index.html`** — the `asset/` files themselves are the originals, not what the running app loads. A global `error` listener swaps any still-path-referenced `img_racer.png`/`idcard.png`/`logo-main-b.png` to a generated placeholder so nothing looks broken.
- The A4 "paper" documents (racer application, receipt) render as `.appdoc` HTML inside the `#pdfBack` modal; `printRacerPdf()` copies the doc into `#pdfPrintRoot` and calls `window.print()` under a print-only stylesheet. Downloads/print are inert inside the Artifact sandbox but work on GitHub Pages.

## Verifying changes (Playwright)

Chromium is preinstalled. Drive the app by navigating screens with `go(...)` / injecting `state` from `page.evaluate`, since there is no server:

```bash
node -e '(async()=>{
  const {chromium}=require("/opt/node22/lib/node_modules/playwright");
  const b=await chromium.launch({executablePath:"/opt/pw-browsers/chromium"});
  const p=await b.newPage({viewport:{width:1180,height:1000}});
  const errs=[]; p.on("pageerror",e=>errs.push(String(e)));
  await p.goto("file:///home/user/su/index.html");
  await p.evaluate(()=>go("racer"));   // jump to any step; inject state as needed
  // ... assert via page.evaluate, screenshot via p.locator(...).screenshot(...)
  console.log("ERRORS:",errs); await b.close();
})()'
```

Always assert `pageerror` is empty and screenshot the affected screen.

To test a companion app in isolation, `goto` its file directly and drive its own globals — e.g. `vip-dashboard.html` opens the admin table with `page.evaluate(()=>{ seedMockIfEmpty(); openDashboard(); })`. Playwright's Node module lives at `/opt/node22/lib/node_modules/playwright` and Chromium at `/opt/pw-browsers/chromium`.

## Deploy loop

Work happens on a session-specific `claude/…` feature branch and is mirrored to `main` (which GitHub Pages serves).

1. Edit `index.html` (and/or `vip.html` / `vip-dashboard.html`). When you touch a companion file, bump its `?v=` in `index.html` (see "Companion iframe apps").
2. If `index.html` changed, regenerate `souped-up.html` (command above). Editing only a companion file does not require regenerating `souped-up.html`, but the `?v=` bump must be applied there too.
3. Verify with Playwright; assert no `pageerror`.
4. Commit the edited files together (include `souped-up.html` whenever `index.html` changed).
5. Push to both the feature branch and `main` (`git push origin HEAD:main`).
   - The user often uploads new `asset/` images directly to GitHub, so `main` may be ahead: `git fetch origin main` then `git merge origin/main` before pushing, and re-embed any new asset as base64.
6. Optionally republish the Artifact from `souped-up.html`.
