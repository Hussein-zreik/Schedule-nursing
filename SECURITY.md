# Securing the schedule

Nurse names and rosters are personal data. This document is the complete setup.

## What protects what

| Layer | Protects | Strength |
|---|---|---|
| **Firestore rules + a real account** | The shared schedule in the cloud | **Strong** — enforced by Google's servers. Cannot be bypassed from the browser. |
| The sign-in screen | Casual viewing of this device's copy | Convenience only |
| Your device's own lock screen | The local copy on that device | Use it — this is the right layer for local data |

The important point: **the cloud data is protected by the rules, not by the
sign-in screen.** Without an account that is a member of the unit, the schedule
cannot be read at all — no matter what anyone does in the browser.

## Why the old setup was not safe

The previous version signed in **anonymously** and the rules said
`allow read, write: if request.auth != null`. Two problems:

1. **Anonymous sign-in is free.** Anyone could get an anonymous session with one
   API call and satisfy that condition. It authenticated nobody.
2. **`allow read` also grants `list`.** The whole `schedules` collection could be
   downloaded in a single query — every unit, without guessing any unit code.

The app no longer uses anonymous sign-in, and the rules now grant `get` only.

---

## Setup — do these in order

### Step 1 — Turn on Email/Password sign-in

1. Open the [Firebase console](https://console.firebase.google.com/) and pick
   the **schedule-nursing-f14e5** project.
2. **Build → Authentication** → **Get started** (if you have never opened it).
3. **Sign-in method** tab → **Email/Password** → **Enable** → **Save**.

Leave "Email link (passwordless)" off. Do **not** enable Anonymous.

### Step 2 — Create an account for each person who edits schedules

Still in **Authentication**, open the **Users** tab → **Add user**, and enter a
work email and a password for each manager. There is deliberately **no sign-up
form in the app**, so only accounts you create here can ever sign in.

Give each person their own account — that is what makes it possible to tell who
changed what later.

### Step 3 — Publish the rules

1. **Build → Firestore Database** → **Rules** tab.
2. Replace everything in the editor with the contents of
   [`firestore.rules`](./firestore.rules).
3. **Publish**.

### Step 4 — Claim your existing unit

Your current schedule was created before membership existed, so it has no owner
yet. The first time you sign in and the app saves, your account is written into
the unit's `members` list and the unit becomes yours.

1. Hard-refresh the app (**Ctrl+Shift+R**; on a tablet, close the tab and reopen).
2. Sign in with the account from Step 2.
3. Open **Cloud** — it should show your unit and `🔒 Protected`.
4. Make any small change so it saves once.

After that the unit is claimed and no other account can open it.

### Step 5 — Check it worked

Open **Cloud**. You should see:

- `Signed in as <your email>`
- `✓ Live-syncing "<unit>" … Only the N accounts on this unit can open it.`
- `🔒 Protected: a real account that belongs to this unit is required.`

If it says **No access**, that account is not a member of that unit — see below.

---

## Adding or removing people

Membership is the `members` list on the unit's document.

**To add someone:** create their account (Step 2), then in **Firestore Database
→ Data → schedules → _your unit_**, edit the `members` array and add their
**User UID** (copy it from the Authentication → Users table).

**To remove someone:** delete their UID from `members` *and* disable or delete
their account in Authentication.

## Working offline

Firebase keeps the session on the device, so each device signs in **once**;
after that it works with no connection. If the sign-in service cannot be reached
at all, the app offers an offline, device-only mode — that mode can never reach
cloud data, because the rules require a real member account.

## Still outstanding

These were raised in the security audit and are **not** fixed by the steps above:

- **No roles.** Every member of a unit can edit everything, including *Reset
  everything*. There is no view-only access for staff.
- **No audit trail.** Change history is per-device and not attributed to a named
  account, so "who changed this shift?" cannot be answered.
- **Third-party scripts load without Subresource Integrity**, so a compromise of
  a CDN would run code inside the app.
- **Custom shift codes/colours are not validated** before being written into a
  stylesheet, which allows CSS (not script) injection via a synced setting.
