# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**Souped Up 2026** — a dark, racing-themed, bilingual (Thai default / English) car-racing competition registration web app. It is a **single self-contained `index.html`**: no build step, no framework, no dependencies to install. All media (logo, backgrounds, ID/racer photos, favicon fallback) are embedded as base64/SVG data URIs so the one file runs anywhere. Palette: mint `#6FFFE8`, lime `#D1FF03`, navy `#0C142A`.

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

## Architecture (all inside `index.html`, one `<script>`)

A vanilla-JS screen router drives a 7-step racer-registration wizard. Account signup and mandatory personal-information onboarding happen before the wizard and are hidden from its stepper. Read these to understand the whole app:

- **`state`** — single global object holding all app data. `STEPS = ["terms","racer","parking","bank","summary","pay","done"]`.
- **`SCREENS[step]()`** returns the screen's HTML string; **`SCREEN_INIT[step]()`** (optional) wires its events after it's injected. `render()` sets `el('screen').innerHTML = SCREENS[state.step]()` then calls the matching `SCREEN_INIT`. `go(step)` sets `state.step` and re-renders.
- Because `render()` blows away and rebuilds the DOM, in-progress form values are captured back into `state` before any re-render via `persistCurrentForm()` (used by language switch) and per-screen `collect*`/`save*` helpers. When adding a screen, follow this collect-then-render pattern or typed data is lost.
- **i18n**: `I18N.th` / `I18N.en` key maps; `t(key)` looks up the current `state.lang`. Every user-facing string goes through a key in **both** language blocks.
- **Shared helpers**: `el(id)`, `esc(str)` (HTML-escape — use for all interpolated user/data text), `money()`, `toast()`, `collect(form,obj)`, `validateScreen(scope)` (validates `[data-req]` inputs, files via `state.files`), and field builders `fieldText`/`fieldSelect`/`fieldUpload`.

### Key data shapes

- **Racers**: `state.racers` is an array whose registration UI uses only its first racer. Each racer has `{id, applicantType, r_rname, r_rsurname, r_idaddr, r_team, entries[], collapsed}`, while the current form exposes one entry `{cls, engine, carno}` and has no card header, border, collapse control, add-racer button, or add-class button. `state.racer` is a flattened mirror of that racer for downstream screens.
- **Parking**: `pkSlots()` derives one booking slot per race-class entry across all racers; `state.parking.assign` maps `slotKey → pit`. A selected `privatePit` is only an additional flag: it never disables the checkbox, clears assignments, or bypasses the requirement to select every pit. Cost is per car: `PRICES.entry + PRICES.deposit` × number of slots.
- **Account/Profile**: `state.account = {username, email, phone, password}`. Demo login uses the two records in `DEMO_USERS`: the new-user record starts mandatory personal-info onboarding with an empty profile, while the existing-user record loads a populated profile with racers, history, and VIP. Registration and post-payment auto-login also populate `state.account`. `state.profile` (built by `buildProfile` for the existing demo user, or `buildProfileFromReg` after payment) feeds the "My Profile" screen.
- **Bank accounts**: `state.bank` stores both account groups. The applicant deposit-refund fields (`rf_*`) are rendered and saved with the personal-information form; the bank wizard screen contains only the winner prize-payment fields (`pz_*`). Prize account name and surname are prefilled from the first racer and rendered read-only.
- **Applicant types**: `state.applicant.a_type` is `person` or `company`. The personal-information form toggles required fields for each type. Company applicants store registered name, tax ID, head-office/branch details, contact person, registered address, and company certificate while mapping the contact name into `a_name` / `a_surname` for downstream compatibility.
- **Existing demo account**: Login tags the account with `demoKind: "existing"`, loads a populated company applicant plus `buildExistingRefundBank()`, and shows a required applicant-type dropdown inside every racer card. Each racer stores its own `applicantType`; new/demo-registered accounts do not see these selectors.
- **Post-onboarding choice**: `openChosenAction(target)` starts the racer option at a fresh terms screen before racer information and redirects the VIP option to `VIP_SITE_URL` (`https://nidss.github.io/suvip/`).

### Hidden entry points

- **Admin dashboard**: `Ctrl+Shift+Z` → admin login → KPI cards + 2-tab paginated table (`DASH` object, `renderDash`/`paintDashTable`), CSV export, per-row image/edit modals, and a printable racer-application PDF.
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

## Deploy loop

Development happens on branch `claude/souped-up-registration-system-bnxppt` and is mirrored to `main`.

1. Edit `index.html`; regenerate `souped-up.html` (command above).
2. Verify with Playwright.
3. Commit `index.html` **and** `souped-up.html` together.
4. `git push origin HEAD:main` and `git push -u origin HEAD:claude/souped-up-registration-system-bnxppt`.
   - The user often uploads new `asset/` images directly to GitHub, so `main` may be ahead: `git fetch origin main` then `git merge origin/main` before pushing, and re-embed any new asset as base64.
5. Republish the Artifact from `souped-up.html` to the URL above.
