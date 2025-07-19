# ⏰ Time Zone Handling Guide for Full Stack Apps

> Tech Stack: **TypeScript (Frontend)** + **Drizzle ORM (Backend)** + **PostgreSQL (Database)**

> For a more detailed, story-driven version of this guide, check out my article on dev.to: [The Developer's Guide to Never Messing Up Time Zones Again](https://dev.to/jacksonkasi/the-developers-guide-to-never-messing-up-time-zones-again-a-typescript-drizzle-and-postgresql-4970)

## 📘 Table of Contents

1.  [Introduction](#1--introduction)
2.  [Core Principles](#2--core-principles)
3.  [Library Setup](#3--library-setup)
4.  [PostgreSQL + Drizzle ORM Setup](#4--postgresql--drizzle-orm-setup)
5.  [Frontend Time Handling (TypeScript)](#5--frontend-time-handling-typescript)
6.  [Handling Date Picker Input](#6--handling-date-picker-input)
7.  [**Solving the 'Broken Date Filter' Nightmare**](#7--solving-the-broken-date-filter-nightmare)
8.  [User Time Zone Config & Multi-Tenant Support](#8--user-time-zone-config--multi-tenant-support)
9.  [PDF Generation and Time Zone Safety](#9--pdf-generation-and-time-zone-safety)
10. [Standard Formatting Functions](#10--standard-formatting-functions)
11. [Example Time Zone Conversions](#11--example-time-zone-conversions)
12. [References & Libraries](#12--references--libraries)
13. [FAQ](#13--faq)

## 1. 🧩 Introduction

Time zone inconsistencies cause bugs in scheduling, reporting, and UI rendering. This guide ensures all time-related logic is safe, predictable, and localized — using UTC as the source of truth.

## 2. 📐 Core Principles

- ✅ Store all timestamps in **UTC**.
- ✅ Always convert timestamps to **user's preferred time zone** for display.
- ✅ Never rely on server/browser local time during rendering.
- ✅ Use strict global formatting functions.

## 3. 📦 Library Setup

**Install:**
```bash
npm install date-fns date-fns-tz
```

**Recommended Imports:**
```ts
import { formatInTimeZone, zonedTimeToUtc } from 'date-fns-tz';
```

> Optionally explore: [`Intl.DateTimeFormat`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat), [`Temporal`](https://tc39.es/proposal-temporal/)

## 4. 🏛 PostgreSQL + Drizzle ORM Setup

### PostgreSQL Migration:
```sql
created_at TIMESTAMPTZ DEFAULT (now() AT TIME ZONE 'UTC'),
updated_at TIMESTAMPTZ DEFAULT (now() AT TIME ZONE 'UTC')
```

### Drizzle Schema:
```ts
import { timestamp, varchar } from 'drizzle-orm/pg-core';

createdAt: timestamp('created_at', { withTimezone: true }).defaultNow(),
updatedAt: timestamp('updated_at', { withTimezone: true }).defaultNow(),
timeZone: varchar('time_zone', { length: 100 }).default('Asia/Kolkata'),
```

## 5. 🖥 Frontend Time Handling (TypeScript)

### Global Functions
```ts
export function formatDateOnly(date: Date, timeZone: string): string {
  return formatInTimeZone(date, timeZone, 'dd-MM-yyyy');
}

export function formatDateTime12Hr(date: Date, timeZone: string): string {
  return formatInTimeZone(date, timeZone, 'dd-MM-yyyy hh:mm a');
}
```

### Usage
```ts
const localTime = formatDateTime12Hr(new Date(utcDate), user.timeZone);
```

## 6. 📆 Handling Date Picker Input

### Problem:
Date pickers typically return values in the browser's local time zone. Without conversion, saving these directly will create wrong UTC timestamps.

### Solution:
- Convert the local date picker value into UTC **before saving to DB**.

### Conversion:
```ts
import { zonedTimeToUtc } from 'date-fns-tz';

const userInput = '2025-01-01T10:00';
const userTimeZone = user.timeZone;
const utcDate = zonedTimeToUtc(userInput, userTimeZone); // save this to DB
```

Ensure this logic happens in the **frontend form handler** before sending to backend.

## 7. 🎯 Solving the 'Broken Date Filter' Nightmare

This is where our principles fix the most common and frustrating time zone bugs. We'll solve two classic problems: filtering for a single day and filtering for a date range.

### Example 1: Filtering for a Single Day (Start & End of Day)

##### The Wrong Way ❌
This common approach creates a filter for a *UTC day*, not the user's local day, leading to missing data.

```ts
// This is a TRAP! It will not match the user's local day.
const selectedDay = "2025-07-19";

const startOfDayUTC = new Date(selectedDay);
startOfDayUTC.setUTCHours(0, 0, 0, 0); // Start of the day in UTC

const endOfDayUTC = new Date(selectedDay);
endOfDayUTC.setUTCHours(23, 59, 59, 999); // End of the day in UTC

const condition = and(
    gte(posts.createdAt, startOfDayUTC),
    lte(posts.createdAt, endOfDayUTC)
);
```

##### The Right Way ✅
We create a precise UTC range that perfectly matches the user's single local day.

```ts
import { zonedTimeToUtc } from 'date-fns-tz';
import { addDays } from 'date-fns';
import { and, gte, lt } from 'drizzle-orm';

const selectedDayStr = "2025-07-19";
const userTimeZone = "Asia/Kolkata";

const startOfDayInUtc = zonedTimeToUtc(selectedDayStr, userTimeZone);
const startOfNextDayInUtc = zonedTimeToUtc(
    addDays(new Date(selectedDayStr), 1),
    userTimeZone
);

const filterCondition = and(
    gte(posts.createdAt, startOfDayInUtc),
    lt(posts.createdAt, startOfNextDayInUtc)
);
```

### Example 2: Filtering a Date Range (From Date to To Date)

##### The Wrong Way ❌
This has the same flaw, using the start of the UTC "from" day and the end of the UTC "to" day.

```ts
// This is also a TRAP! It misses data at both ends of the range.
if (fromDate) {
  const fromDateObj = new Date(fromDate);
  fromDateObj.setUTCHours(0, 0, 0, 0);
  conditions.push(gte(posts.createdAt, fromDateObj));
}
if (toDate) {
  const toDateObj = new Date(toDate);
  toDateObj.setUTCHours(23, 59, 59, 999);
  conditions.push(lte(posts.createdAt, toDateObj));
}
```

##### The Right Way ✅
We find the start of the user's local "from" day and the start of the day *after* their "to" day.

```ts
import { zonedTimeToUtc } from 'date-fns-tz';
import { addDays } from 'date-fns';
import { and, gte, lt } from 'drizzle-orm';

const fromDateStr = "2025-07-01";
const toDateStr = "2025-07-31";
const userTimeZone = "Asia/Kolkata";

const startRangeUtc = zonedTimeToUtc(fromDateStr, userTimeZone);
const nextDayAfterToDate = addDays(new Date(toDateStr), 1);
const endRangeUtc = zonedTimeToUtc(nextDayAfterToDate, userTimeZone);

const filterCondition = and(
    gte(posts.createdAt, startRangeUtc),
    lt(posts.createdAt, endRangeUtc)
);
```

## 8. 👤 User Time Zone Config & Multi-Tenant Support

### In `users` table:
```ts
timeZone: varchar("time_zone", { length: 100 }).default('Asia/Kolkata')
```

### Preferred Display Logic:
```ts
const userTz = user.timeZone;
formatDateTime12Hr(new Date(timestamp), userTz);
```

✅ Multi-tenant mode: store org-level time zone fallback.
✅ Give UI setting for user to update their preferred time zone.

## 9. 📄 PDF Generation and Time Zone Safety

📌 **Problem:** Servers often run in different time zones. `new Date()` may produce unexpected results during PDF generation.

✅ **Solution:**
- Pre-format timestamps in user’s time zone before generating the PDF.
- Do **not** rely on server’s locale.

```ts
const displayTime = formatDateTime12Hr(new Date(data.createdAt), user.timeZone);
```

## 10. 🧪 Standard Formatting Functions

📌 Only 2 global functions allowed in codebase:

1. `formatDateOnly()` → `DD-MM-YYYY`
2. `formatDateTime12Hr()` → `DD-MM-YYYY HH:MM AM/PM`

➡️ All UI, exports, and PDFs must use these — no inline formatting allowed.

## 11. 🔁 Example Time Zone Conversions

### US → India

- US Central: `10:00 AM` (UTC-6)
- ➡ UTC: `10:00 AM + 6h = 4:00 PM UTC`
- ➡ IST: `4:00 PM + 5:30 = 9:30 PM IST`

```ts
const utc = zonedTimeToUtc('2025-01-01 10:00', 'America/Chicago');
const ist = formatInTimeZone(utc, 'Asia/Kolkata', 'dd-MM-yyyy hh:mm a');
```

### India → US

- IST: `10:00 AM` (UTC+5:30)
- ➡ UTC: `10:00 AM - 5:30 = 4:30 AM UTC`
- ➡ US Central: `4:30 AM - 6h = 10:30 PM (prev day)`

```ts
const utc = zonedTimeToUtc('2025-01-01 10:00', 'Asia/Kolkata');
const cst = formatInTimeZone(utc, 'America/Chicago', 'dd-MM-yyyy hh:mm a');
```

## 12. 📚 References & Libraries

- [`date-fns-tz`](https://github.com/marnusw/date-fns-tz)
- [`Intl.DateTimeFormat`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat)
- [`Temporal API`](https://tc39.es/proposal-temporal/)
- [`PostgreSQL Date/Time`](https://www.postgresql.org/docs/current/datatype-datetime.html)

## 13. ❓ FAQ

**Q: Why store time in UTC?**
> Ensures consistency across regions, avoids DST bugs, simplifies backend logic.

**Q: Should I ever store local time?**
> No. Always store UTC, format to local.

**Q: How do I change display format globally?**
> Update `formatDateOnly` and `formatDateTime12Hr` only. Never format inline.

**Q: What about scheduled jobs?**
> Always convert cron times to UTC before storing and scheduling.

### ✅ That’s it!
Keep your time logic predictable, safe, and local to your users. 🌏
