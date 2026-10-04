# SpendLogs

A single-file web app for logging your daily **spending** and **credit** (money coming in), browsing it on a calendar, and seeing where the money goes with daily and monthly summaries and pie charts.

It is the spending twin of GymLogs: same design, same login system (Supabase Auth), same one-HTML-file approach, hosted as static files (for example GitHub Pages).

> Working name: **SpendLogs**. If you rename it (for example PaisaLogs), change the `<title>` and the `<h1>` in `index.html`, and the title and hint text in `reset-password.html`.

---

## Contents

- [Features](#features)
- [Project files](#project-files)
- [Setup](#setup)
- [How it works](#how-it-works)
- [Data model](#data-model)
- [Authentication flow](#authentication-flow)
- [Summaries and pie chart](#summaries-and-pie-chart)
- [Important quirks](#important-quirks)
- [Security notes](#security-notes)
- [Troubleshooting](#troubleshooting)
- [Ideas for future additions](#ideas-for-future-additions)
- [Design notes](#design-notes)

---

## Features

- **Accounts**: register, log in, log out, and reset a forgotten password by email. Each user only ever sees their own data.
- **Categories**: build your own list of spending categories (Food, Travel, Rent and so on). Add and remove them any time.
- **Log an entry**: choose **Spent** or **Credit**, enter an amount and an optional note, and it is saved against the day picked on the calendar (today by default).
  - **Spent** needs a category.
  - **Credit** has no category. Use the note as the source (salary, refund, gift).
- **Calendar**: browse any month. Days with entries show a dot. Tapping a day selects it and scrolls to its summary.
- **Daily Summary**: for the selected day, shows Spent, Credit and Net, a pie chart of that day's spending by category, and the list of entries with delete.
- **Monthly Summary**: for the month shown on the calendar, shows Spent, Credit and Net, average spend per day, biggest spending day, top category, number of entries, and a pie chart by category.
- **Safe deletes**: destructive actions use tap-twice-to-confirm (no browser dialogs).
- **Mobile-first**: single column, max width 540px, dark theme with an automatic light theme.

---

## Project files

| File | Purpose |
|---|---|
| `index.html` | The whole app: UI, styles, auth, data layer, summaries. |
| `reset-password.html` | Page users land on from the password-reset email, where they set a new password. |
| `schema.sql` | Database tables, indexes and row-level security policies to run once in Supabase. |
| `README.md` | This file. |

`index.html` and `reset-password.html` must be deployed in the **same folder**, because they link to each other.

---

## Setup

### 1. Create the tables

In your Supabase project open **SQL Editor**, paste the contents of `schema.sql` and run it. This creates:

- `sl_categories`
- `sl_entries`
- indexes and row-level security (RLS) policies so users can only read and write their own rows

The tables are prefixed with `sl_` so they will not clash with GymLogs tables (`exercises`, `logs`) if you use the same Supabase project. Both apps can share the same project and the same user accounts.

### 2. Add your credentials

Open `index.html` and fill in the three constants near the top of the `<script>`:

```js
const SUPABASE_URL = "YOUR_SUPABASE_URL";                    // https://xxxx.supabase.co
const SUPABASE_PUBLISHABLE_KEY = "YOUR_SUPABASE_PUBLISHABLE_KEY";
const RESET_REDIRECT_URL = "https://YOUR_USERNAME.github.io/YOUR_REPOSITORY/reset-password.html";
```

Open `reset-password.html` and fill in the same `SUPABASE_URL` and `SUPABASE_PUBLISHABLE_KEY`.

You can find the URL and publishable (anon) key in Supabase under **Project Settings, API**. If you already run GymLogs on the same project, reuse its values.

### 3. Configure Supabase Auth URLs

In Supabase go to **Authentication, URL Configuration**:

- Set **Site URL** to the URL of your deployed `index.html`.
- Add the exact `RESET_REDIRECT_URL` (the deployed `reset-password.html`) under **Redirect URLs**.

If the redirect URL is not on this list, reset emails will not send users to the right page.

### 4. Choose your email confirmation behaviour

Under **Authentication, Providers, Email** you can require email confirmation or not. With confirmation on, new users see "Account created. Check your email to confirm, then log in." With it off, they are logged in immediately.

### 5. Deploy

Upload `index.html` and `reset-password.html` together to any static host. For GitHub Pages:

1. Put both files in a repository (for example in the root).
2. Enable **Settings, Pages** for that branch.
3. Use the resulting URL for the Site URL and `RESET_REDIRECT_URL` above.

Opening the file by double-clicking works for layout checks, but login and password reset need a real hosted URL.

---

## How it works

Everything lives in one `state` object at the top of the script:

```js
state.categories = [{ id, name }]
state.logs       = [{ id, date, type, categoryId, amount, note }]
state.currentUser, state.picked (selected date), state.y / state.m (calendar month)
```

- On login, `loadAllData()` fetches categories and entries in parallel from Supabase (RLS scopes them to the user) and renders everything.
- Every add or delete is sent to Supabase first. The screen updates only after the database confirms the change, and errors are shown inline.
- Rendering is split into small functions: `renderCats`, `renderCalendar`, `renderSummary`, `renderDay`, called together by `renderAll`.
- Summaries are computed in the browser from `state.logs`, so no extra database queries are needed when you change day or month.

---

## Data model

### `sl_categories`

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key, generated by the database. |
| `user_id` | uuid | Owner. References `auth.users`, defaults to `auth.uid()`. |
| `name` | text | Category name. |
| `created_at` | timestamptz | Used to keep categories in creation order. |

### `sl_entries`

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key. |
| `user_id` | uuid | Owner, same as above. |
| `date` | date | The day the entry belongs to (`YYYY-MM-DD`). |
| `type` | text | Either `spent` or `credit` (enforced by a check constraint). |
| `category_id` | uuid | References `sl_categories`. `null` for credits. `on delete set null`. |
| `amount` | numeric(12,2) | Must be greater than 0. |
| `note` | text | Optional. For credits, use it as the source. |
| `created_at` | timestamptz | Used for ordering within a day. |

Notes:

- Deleting a category does **not** delete its entries. They keep their amounts but lose the category, so they show as **Unknown** in the pie chart.
- `numeric` values can arrive as strings, so the app converts `amount` with `Number()` when loading.
- Entries are always stored with positive amounts. The `type` field decides whether it counts as spending or credit.

---

## Authentication flow

- **Register**: `supabase.auth.signUp`. Depending on your email confirmation setting the user is logged in immediately or asked to confirm by email.
- **Log in / Log out**: `signInWithPassword` and `signOut`. The app shows or hides the private content based on the session.
- **Session handling**: `onAuthStateChange` plus an initial `getSession()` check. A guard (`state.loadedFor`) makes sure data is loaded once per user even when Supabase fires several events (token refresh, repeated sign-in events).
- **Forgot password**:
  1. The user opens **Forgot Password?** on the login form and enters their email.
  2. `resetPasswordForEmail` sends a link pointing to `RESET_REDIRECT_URL`.
  3. The link opens `reset-password.html`, where the user sets a new password.
  4. A 30 second cooldown stops repeated clicks and protects Supabase's email rate limit.
  5. The success message is identical whether or not the email is registered, so it cannot be used to discover accounts.
- **Recovery link safety net**: if a recovery link ever lands on `index.html` (for example when Supabase falls back to the Site URL), a small script at the top forwards it, tokens intact, to `reset-password.html`.
- **Expired links**: `reset-password.html` offers "Request a New Reset Email", which opens `index.html?forgot=1` and shows the forgot-password form directly.

---

## Summaries and pie chart

All amounts are shown in Indian rupees (`₹`) with Indian digit grouping, for example `₹1,25,000`.

**Stat cards** (daily and monthly)

- **Spent**: sum of all `spent` entries.
- **Credit**: sum of all `credit` entries.
- **Net**: Credit minus Spent. Shown with a minus sign when negative.

**Monthly extras**

- **Average per day**: total spent divided by the number of days. For the current month it divides by the days elapsed so far, not the full month. For past months it divides by the month's length.
- **Biggest day**: the day with the highest total spending, and that total.
- **Top category**: the category with the highest spending, and that total.
- **Entries**: count of all entries in the month (spent and credit).

**Pie chart**

- Drawn as an inline SVG donut, with no chart library.
- Only **spending** is charted. Credits are not part of the pie.
- One slice per category, sorted from largest to smallest, with a legend showing amount and percentage.
- The centre shows the total spent for the period.
- Colours come from a 10-colour palette assigned by the category's position in your list. With more than 10 categories colours repeat.
- If there is no spending for the day or month, an empty-state message is shown instead.

---

## Important quirks

### No `alert()` / `confirm()`

Like GymLogs, this app avoids native `alert()`, `confirm()` and `prompt()`, because they are blocked in sandboxed previews. Instead it uses:

- **Tap-twice-to-confirm** for deleting entries and categories. The first tap turns the control red and shows "Sure?" or "sure?". A second tap within 3 seconds deletes. After 3 seconds it resets. See `armedLog` and `armedCat` in the state and in `deleteLog` / `deleteCat`.
- **Inline hint text** for validation and errors (`logHint` under the Add Entry button, `catHint` under the category input, `auth-message` in the Account section).

Follow the same pattern for any new destructive action.

### Default row limit

Supabase returns at most 1000 rows per query by default. `loadAllData()` does not paginate, so once an account has more than 1000 entries the oldest or newest ones may be missing. See the ideas list for a fix.

### Dates and time zones

The selected day and "today" use the browser's local date. Entries are stored as plain `date` values with no time, so they will not shift when the viewer's time zone changes.

### Currency

The `₹` symbol and `en-IN` formatting are set in the `money()` helper. Change it there to switch currency.

---

## Security notes

- The **publishable (anon) key** is designed to be public and is safe in client-side code. **Never** put the `service_role` key in these files.
- Data protection comes from **row-level security**. The policies in `schema.sql` allow each user to read and write only rows where `user_id = auth.uid()`. Do not disable RLS on these tables.
- `user_id` defaults to `auth.uid()` and the app also sends it explicitly. The policies' `with check` clause prevents inserting rows for someone else.
- Output going into the page is escaped (`esc()`) so category names and notes cannot inject HTML.

---

## Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| "Failed to fetch" or nothing loads | `SUPABASE_URL` or key not filled in, or a typo. Check the browser console. |
| Login works but data never loads, or errors mention permissions | The SQL from `schema.sql` was not run, or RLS policies are missing. |
| Reset email does not arrive | Check spam. Supabase's default mail sender is rate limited. Confirm the email is registered. |
| Reset link goes to the wrong page | `RESET_REDIRECT_URL` is wrong, or it is not listed in Supabase **Redirect URLs**. |
| "Password reset is not set up yet" message | `RESET_REDIRECT_URL` still contains the `YOUR_USERNAME` / `YOUR_REPOSITORY` placeholders. |
| "Account created. Check your email" but no email | Email confirmation is on and mail is delayed or in spam. You can turn confirmation off in Supabase while testing. |
| Entry saves fail with a category error | The category was deleted in another tab. Reload the page. |
| Old entries show "Unknown" in the pie | Their category was deleted. Re-add the category and edit the entries, or leave as is. |
| Missing entries on a big account | The 1000-row default limit, see [Important quirks](#important-quirks). |

---

## Ideas for future additions

- Pagination or month-based loading in `loadAllData()` to remove the 1000-row limit.
- Editing an entry instead of delete and re-add.
- A reassign-category tool so deleted categories do not leave "Unknown" entries.
- Monthly budget per category with a progress bar.
- A "credit owed / lent" type that tracks money to receive or pay back, if you want credit to also mean that.
- Per-category history view across all months.
- Recurring entries (rent, subscriptions).
- Export and import (CSV).
- Year view with a month-by-month bar chart.
- Light and dark theme toggle (the light `prefers-color-scheme` block exists but is not user-toggleable).
- Default starter categories created on first login.

---

## Design notes

- Fonts: **Bebas Neue** for headings, **Inter** for body, both loaded from Google Fonts.
- Colours are CSS variables on `:root`, so you can change the palette in one place:
  - `--primary` (orange) for spending and main actions
  - `--credit` (green) for credit
  - `--gold` for today's date and highlights
  - `--danger` for destructive actions and errors
- Layout is a single column with a maximum width of 540px, using the same panel, chip and calendar components as GymLogs.
- Charts are plain SVG with no dependencies. The only external script is `supabase-js`, loaded from jsDelivr.