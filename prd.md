# TourneyGen — Product Requirements Document (v1)

## 1. Overview

TourneyGen is a web application for creating and managing sports or gaming tournaments and leagues. Organizers can run single-elimination tournaments and double round-robin leagues. Guests can create quick throw-away tournaments without an account; registered organizers get persistent history, sharing, and PDF export. A future .NET MAUI integration is planned but out of scope for v1.

---

## 2. Tech Stack

| Layer | Technology |
|---|---|
| Backend | ASP.NET Core |
| Frontend | Blazor (C#) |
| ORM | Entity Framework Core + Pomelo MySQL provider |
| Database | MySQL (Docker container on Linode VPS) |
| Hosting | Linode (Akamai Cloud) VPS |
| File / PDF Storage | Linode Object Storage (S3-compatible) |
| Email | Gmail SMTP |
| Auth | ASP.NET Core Identity (email/password + Google OAuth) |

---

## 3. User Roles

### 3.1 Guest (no account)
- Can create a single tournament in a session without logging in.
- Session data is held in browser memory; data is lost permanently when the tab is closed.
- Cannot create leagues.
- Cannot share or export.
- If the guest logs in during an active session, the in-progress tournament is migrated to their account.

### 3.2 Organizer (registered account)
- Full access to tournaments and leagues.
- Persistent history accessible from any device at any time.
- Maximum of **5 active events** (tournaments + leagues combined) at any one time.
- Archived (completed/deleted) events do not count toward the 5-event limit.

---

## 4. Authentication & Account Management

- **Registration**: email/password via ASP.NET Core Identity. Email confirmation sent via Gmail SMTP before the account is activated.
- **OAuth**: Google OAuth as an alternative sign-in method.
- **Password reset**: Gmail SMTP email-based reset flow.
- **Participants do not need accounts.** Only the organizer requires an account.

---

## 5. Landing Page

- Minimal design.
- Two primary CTAs: **"Create Tournament"** and **"Create League"**.
- **Login** button in the top-right corner.
- Guests who click "Create League" are prompted to log in first.
- Guests who click "Create Tournament" proceed directly to guest/session mode.

---

## 6. Account Dashboard

- Displayed after login.
- Shows two lists:
  - **Active events** — tournaments and leagues currently in progress (max 5).
  - **Archived events** — completed events.
- Each event entry shows its name/title.
- Organizer can navigate into any event from the dashboard.

---

## 7. Tournaments

### 7.1 Format
Single-elimination bracket only.

### 7.2 Participant Count
Power-of-2 counts only: **4, 8, 16, 32, 64**.

### 7.3 Participant Entry
- Participants are entered one at a time before the event starts.
- The participant list is **locked once the event begins** — no additions or removals after that point.
- Participant display: name only (50-character maximum).
- Names are **editable at any time mid-event** by the organizer; changes are retroactive across all displayed results.
- The organizer is responsible for ensuring name uniqueness.

### 7.4 Bracket Seeding
- The app randomises the initial bracket seeding.
- The organiser can **reshuffle** the seeding as many times as desired before confirming and starting the event.

### 7.5 Match Results
- Win/loss only — no numeric scores or draws.
- Winners and losers are indicated manually via a button in the UI.
- **Editing a result**:
  - If no later rounds have been played: edit directly, no warning.
  - If later rounds have been played: warn the organiser that subsequent results may be affected; organiser confirms before the edit is applied.

### 7.6 Completion & Archiving
- When the final match is recorded the tournament is marked complete and moves to the archived events list.
- A winners PDF can be exported (account holders only — see Section 9).

### 7.7 Guest Limitations
- Guests can run a tournament through to completion within their session.
- No sharing link and no PDF export available to guests.
- On login during session, the active tournament is migrated to the account.

---

## 8. Leagues

Leagues are available to **account holders only**.

### 8.1 Format
Double round-robin: every participant plays every other participant twice (once at home, once away, or equivalent).

### 8.2 Participant Count
**4 to 64 participants**, any whole number in that range.

### 8.3 Participant Entry
Same rules as tournaments: one at a time, list locked once the league begins, names editable at any time with retroactive effect, 50-char max, organiser responsible for uniqueness.

### 8.4 Match Results
- Win / draw / loss only — no numeric scores.
- Results can be **edited or undone** at any time by the organiser.

### 8.5 Points
| Outcome | Points |
|---|---|
| Win | 3 |
| Draw | 1 each |
| Loss | 0 |

### 8.6 Standings & Tiebreakers
- Tied participants **share the same rank** in the standings table throughout the league.
- Tiebreaker logic is applied **only for first place, and only at the end of the league** (i.e., when all matches have been recorded).
- The organiser chooses one of four tiebreaker options:

| Option | Behaviour |
|---|---|
| **Auto-resolve (head-to-head)** | App automatically compares head-to-head results between tied participants and declares a winner. |
| **Generate tiebreaker bracket** | App generates a small single-elimination bracket for the tied first-place participants. |
| **Manual selection** | Organiser manually picks the winner from the tied participants. |
| **Coin flip** | App randomly selects a winner from the tied participants. |

### 8.7 Completion & Archiving
When all matches are recorded and the tiebreaker (if triggered) is resolved, the league is marked complete and archived.

---

## 9. Sharing (Account Holders Only)

- Each event (tournament or league) can generate a **shareable read-only link**.
- The link reflects the current live state of the event.
- State is updated on **manual refresh** by the viewer — no live push.
- A **"last updated" timestamp** is shown on the shared view.
- If the event is deleted, the shared link resolves to a generic **"not found"** page.

---

## 10. Export (Account Holders Only)

- When an event ends, the organiser can export a **winners PDF**.
- The PDF is generated server-side and stored on **Linode Object Storage** (S3-compatible).
- Export is not available to guests.

---

## 11. Deletion

- Any event (active or archived) can be deleted by the organiser.
- Deletion is a **hard delete**: all event data is permanently removed; no records are kept.
- A **confirmation warning** is shown before deletion is executed.
- Any active shared link for the deleted event returns a generic "not found" page.

---

## 12. Event Limits

| Constraint | Value |
|---|---|
| Max active events per organiser | 5 (tournaments + leagues combined) |
| Min tournament participants | 4 |
| Max tournament participants | 64 (power of 2) |
| Min league participants | 4 |
| Max league participants | 64 |
| Max participant name length | 50 characters |

---

## 13. Out of Scope for v1

The following features are explicitly deferred:

- Bulk participant import
- Category / group-stage feature
- Participant avatars
- Seed numbers displayed on bracket
- Numeric match scores
- .NET MAUI mobile/desktop app
- Any real-time / WebSocket push for the shared view (manual refresh only)

---

## 14. Data Model (Conceptual)

```
Organiser (ASP.NET Core Identity user)
  └── Events (max 5 active)
        ├── Tournament
        │     ├── Participants (4/8/16/32/64)
        │     ├── Rounds
        │     │     └── Matches (win/loss result, editable)
        │     └── SharedLink (optional)
        └── League
              ├── Participants (4–64)
              ├── Matches (win/draw/loss result, editable)
              ├── Standings (calculated)
              ├── Tiebreaker (optional, end-of-league, first place only)
              └── SharedLink (optional)
```

---

## 15. Key User Flows

### 15.1 Guest Tournament Flow
1. Landing page → "Create Tournament"
2. Enter participant names (power-of-2 count)
3. View randomised bracket → reshuffle if desired → confirm
4. Record match results round by round
5. Tournament completes → results visible in session
6. (Optional) Log in → tournament migrated to account

### 15.2 Organiser Tournament Flow
1. Log in → dashboard
2. "Create Tournament" → name event → enter participants → confirm bracket
3. Record match results; edit as needed
4. Tournament completes → archived → PDF export available

### 15.3 Organiser League Flow
1. Log in → dashboard (must have < 5 active events)
2. "Create League" → name event → enter participants → start league
3. Record match results (win/draw/loss) for each fixture; edit as needed
4. All matches complete → tiebreaker triggered if first-place tie → resolve
5. League completes → archived → PDF export available

### 15.4 Shared View Flow
1. Organiser copies shareable link from event page
2. Recipient opens link → read-only view of current state
3. Recipient refreshes to see latest state ("last updated" timestamp shown)
4. If event deleted → "not found" page shown
