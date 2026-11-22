<h1 align="center">Clarvix</h1>

<p align="center">
<b>School management platform. Fees, attendance, admissions, exams, transport, library, HR, payroll and an AI assistant, all in one place.</b><br>
Built from scratch in 3 months, by me.
</p>

<p align="center">
<img src="https://img.shields.io/badge/TypeScript-3178C6?style=flat-square&logo=typescript&logoColor=white">
<img src="https://img.shields.io/badge/React-20232A?style=flat-square&logo=react&logoColor=61DAFB">
<img src="https://img.shields.io/badge/PostgreSQL-4169E1?style=flat-square&logo=postgresql&logoColor=white">
<img src="https://img.shields.io/badge/Supabase-3FCF8E?style=flat-square&logo=supabase&logoColor=white">
<img src="https://img.shields.io/badge/Cloudflare_Workers-F38020?style=flat-square&logo=cloudflare&logoColor=white">
<img src="https://img.shields.io/badge/Claude-D97757?style=flat-square&logo=anthropic&logoColor=white">
<img src="https://img.shields.io/badge/Kotlin-7F52FF?style=flat-square&logo=kotlin&logoColor=white">
<img src="https://img.shields.io/badge/Expo-000020?style=flat-square&logo=expo&logoColor=white">
</p>

<p align="center">
<a href="#-at-a-glance">At a glance</a> ·
<a href="#-screenshots">Screenshots</a> ·
<a href="#-architecture">Architecture</a> ·
<a href="#-clever-ai">Clever AI</a> ·
<a href="#-school-lock">School lock</a> ·
<a href="#-releases">Releases</a>
</p>

![Dashboard](images/dashboard-top.png)

> The source code is private because it runs real school data. This repo shows what it does and how the main parts work.

---

## 📊 At a glance

| | |
|---|---|
| **Web portals** | 6 · school admin (105 pages), teachers, students & parents, super-admin, school website, parent app |
| **Android apps** | 5 · QR attendance scanner, gate guard, library desk, bus driver (live tracking), parent app |
| **Database** | PostgreSQL · 470+ tables · 1,700+ row-level security rules · 52 scheduled jobs |
| **Migrations** | 466 versioned, each one tested on a copy of production first |
| **AI** | Clever AI on Claude · answers questions and makes changes after approval |
| **Built in** | 3 months, from scratch, solo |

---

## 📸 Screenshots

**Fees and money**

![Fees section](images/dashboard-money.png)

| Attendance (dark mode) | On mobile |
|:---:|:---:|
| <img src="images/dashboard-attendance-dark.png" width="100%"> | <img src="images/dashboard-mobile.png" width="55%"> |

<sub>Screenshots use demo data.</sub>

---

## 🏗️ Architecture

```mermaid
flowchart LR
    subgraph Users
        A[School admin]
        T[Teachers]
        P[Students & parents]
        D[Android apps<br/>scanner · guard · library · bus]
    end

    subgraph Cloudflare
        W[Web portals<br/>React on Workers]
        R2[(R2 file storage)]
    end

    subgraph Oracle_Cloud[Oracle Cloud VM]
        AUTH[Supabase Auth]
        DB[(PostgreSQL<br/>row-level security)]
        EF[Edge functions<br/>webhooks · SMS · push]
        CRON[52 scheduled jobs]
    end

    AI[Claude<br/>Azure AI Foundry]

    A & T & P --> W
    D --> AUTH
    W --> AUTH
    W --> DB
    W --> R2
    W --> AI
    D --> DB
    EF --> DB
    CRON --> DB
```

Every school's rows carry the school they belong to. Row-level security checks that the person asking is part of that school and can open that page. Anything that needs more power goes through a database function that reads the school from the user's login, never from what the browser sends.

---

## 🤖 Clever AI

An assistant inside the admin portal. Ask it anything about the school, or ask it to change something. It can only see what the logged-in admin can see, and it **never changes anything without asking first.**

```mermaid
sequenceDiagram
    actor Admin
    participant AI as Clever AI
    participant DB as Database

    Admin->>AI: "Send fee reminders to Class 8"
    AI->>DB: Look up Class 8 dues (same checks as the page)
    DB-->>AI: 23 students with pending fees
    AI-->>Admin: Plan: remind 23 parents by SMS. Approve?
    Admin->>AI: Approve
    AI->>DB: Run once, through the page's own function
    DB-->>AI: Done, 23 reminders sent
    AI-->>Admin: ✅ Reminders sent
```

**Extra protection for risky changes**, like deleting records, changing permissions or money going out:

```mermaid
flowchart LR
    S[Risky step] --> C{Typed<br/>confirmation?}
    C -- no --> X[Blocked]
    C -- yes --> M{2-step code<br/>in last 5 min?}
    M -- no --> X
    M -- yes --> O{Approved on<br/>its own?}
    O -- no --> X
    O -- yes --> R[Runs once<br/>and gets logged]
```

A few more details:
- The 2-step check happens on the server, not in the browser.
- If the data changed since the plan was made, the step is refused.
- There are hundreds of possible actions, so the AI searches for the few tools it needs instead of loading all of them. Each request got about 78% smaller.
- It's tested with real questions, real changes and attempts to trick it: instructions hidden in a student's name, asking for another school's data, a staff member trying to do more than allowed.

---

## 🔒 School lock

If a school stops paying, hiding buttons isn't enough, because someone could still call the API directly. So **the database itself** blocks changes for that school.

```mermaid
flowchart TD
    W[Any change to any of ~300 tables] --> L{School locked?}
    L -- no --> OK[Change saved]
    L -- yes --> E{Allowed anyway?<br/>pay bill · export · reset password · support}
    E -- yes --> OK
    E -- no --> B[Blocked<br/>HTTP 423]
```

On the apps, people can still sign in, but they see a lock screen and get logged out after a minute. A test runs on every release and fails if a new table is missing this check.

```sql
-- Simplified idea, not the real code
create function block_writes_when_locked() returns trigger as $$
begin
  if school_is_locked(current_school()) and not action_is_allowed() then
    raise exception 'school locked' using errcode = 'PT423';
  end if;
  return null;
end $$ language plpgsql;
```

---

## 🚢 Releases

```mermaid
flowchart LR
    A[Write changes] --> B[Test on a copy<br/>of production]
    B --> C[Throw the copy away]
    C --> D[Backup]
    D --> E[Apply changes<br/>in fixed order]
    E --> F[Deploy portals]
    F --> G[Live checks with<br/>temporary test schools]
    G --> H[Delete test schools]
```

Every change also has a tested rollback.

---

<p align="center">
Shivam Laur · <a href="mailto:proffesorrk@gmail.com">proffesorrk@gmail.com</a> · <a href="https://www.linkedin.com/in/shivam-kumar-28b40b306/">LinkedIn</a> · <a href="https://clarvix.co">clarvix.co</a>
</p>
