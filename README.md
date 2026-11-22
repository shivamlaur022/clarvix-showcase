<h1 align="center">Clarvix</h1>

<p align="center">
<b>An all-in-one school management platform.</b><br>
Fees, attendance, admissions, exams, timetable, transport, library, HR, payroll and an AI assistant.<br>
Built from scratch in 3 months, solo.
</p>

<p align="center">
<img src="https://img.shields.io/badge/TypeScript-3178C6?style=flat-square&logo=typescript&logoColor=white">
<img src="https://img.shields.io/badge/React-20232A?style=flat-square&logo=react&logoColor=61DAFB">
<img src="https://img.shields.io/badge/PostgreSQL_17-4169E1?style=flat-square&logo=postgresql&logoColor=white">
<img src="https://img.shields.io/badge/Supabase-3FCF8E?style=flat-square&logo=supabase&logoColor=white">
<img src="https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white">
<img src="https://img.shields.io/badge/Oracle_Cloud-F80000?style=flat-square&logo=oracle&logoColor=white">
<img src="https://img.shields.io/badge/Cloudflare-F38020?style=flat-square&logo=cloudflare&logoColor=white">
<img src="https://img.shields.io/badge/Grafana-F46800?style=flat-square&logo=grafana&logoColor=white">
<img src="https://img.shields.io/badge/Claude-D97757?style=flat-square&logo=anthropic&logoColor=white">
<img src="https://img.shields.io/badge/Kotlin-7F52FF?style=flat-square&logo=kotlin&logoColor=white">
<img src="https://img.shields.io/badge/Expo-000020?style=flat-square&logo=expo&logoColor=white">
</p>

> The source code is private because it runs real school data. This repo explains what the product does and how it's built.

---

## 📊 At a glance

| | |
|---|---|
| **Web portals** | 6 · school admin (~97 pages), teachers, students & parents, super-admin (36 pages), website (41 pages), parent app |
| **Android apps** | 5 · attendance scanner, gate guard, library desk, bus driver, parent app |
| **Database** | PostgreSQL 17 · 470+ tables · 1,700+ row-level security rules · 2,100+ functions |
| **Automation** | 52 scheduled jobs · auto invoices, reminders, alerts, scoring, monitoring |
| **Hosting** | Self-hosted Supabase in Docker on Oracle Cloud · portals on Cloudflare Workers |
| **Monitoring** | Grafana for the server, containers and frontend routes |
| **Releases** | 466 versioned migrations, all tested on a copy of production first |
| **Built in** | 3 months, from scratch, by one person |

---

## 🗺️ The whole product

```mermaid
flowchart TB
    CX((Clarvix))

    CX --> SA[🏫 School admin portal]
    CX --> TP[👩‍🏫 Teachers portal]
    CX --> PP[👨‍👩‍👧 Parent portal]
    CX --> SU[🛠️ Super-admin console]
    CX --> WB[🌐 Website & admission pages]
    CX --> AP[📱 Android apps]

    SA --> SA1[Academics<br/>marksheet · timetable · homework · syllabus · seating]
    SA --> SA2[Admissions CRM<br/>23 pages, lead to enrolment]
    SA --> SA3[Fees & money<br/>ledger · gateways · defaulters · expenses]
    SA --> SA4[Attendance<br/>QR · RFID · manual · alerts]
    SA --> SA5[Transport<br/>14 pages · live tracking]
    SA --> SA6[People<br/>HR · payroll · leave · access · ID cards]
    SA --> SA7[Clever AI<br/>asks before it changes anything]

    AP --> AP1[Attendance scanner]
    AP --> AP2[Gate guard]
    AP --> AP3[Library desk]
    AP --> AP4[Bus driver]
    AP --> AP5[Parent app]
```

## 📚 Detailed docs

| Doc | What's inside |
|---|---|
| [School admin portal](01-school-admin-portal.md) | Every sidebar section and page, with flow charts for admissions, attendance, fees, transport and payroll |
| [Teachers portal](02-teachers-portal.md) | Teaching pages, and how the owner grants access page by page |
| [Parent portal](03-parent-portal.md) | Learn, fees, inbox, transport and more |
| [Super-admin console](04-super-admin-console.md) | Running the platform: schools, billing, product, support, compliance |
| [Android apps](05-android-apps.md) | All 5 apps, screen by screen |
| [Website & public pages](06-website-and-public-pages.md) | Marketing site and each school's admission pages |
| [Infrastructure & operations](07-infrastructure-and-operations.md) | Docker setup, Grafana monitoring, 52 jobs, emails, billing automation, releases |

---

## 🏗️ How it's built

```mermaid
flowchart LR
    subgraph Users
        U1[School staff]
        U2[Parents]
        U3[Android apps]
    end

    subgraph CF["☁️ Cloudflare"]
        W[Web portals<br/>React on Workers]
        R2[(R2 files)]
    end

    subgraph OR["Oracle Cloud VM"]
        NG[nginx] --> SB[Supabase in Docker<br/>auth · API · realtime · storage · functions]
        SB --> DB[(PostgreSQL 17)]
        HS[PDF · reports · email services]
    end

    AI[🤖 Claude<br/>Azure AI Foundry]
    GF[📈 Grafana]

    U1 & U2 --> W
    U3 --> NG
    W --> NG
    W --> R2
    W --> AI
    OR -. metrics .-> GF
```

Full container-by-container diagram: [Infrastructure & operations](07-infrastructure-and-operations.md).

---

## 🔒 Every school's data stays separate

Every row belongs to one school. Row-level security checks that the person asking is part of that school and has access to that page. Staff permissions go down to single pages, and the database checks the same permission on every read and every change.

```mermaid
flowchart LR
    R[Request] --> L{Logged in?}
    L -- no --> X[Refused]
    L -- yes --> S{Member of<br/>this school?}
    S -- no --> X
    S -- yes --> P{Has this page?<br/>view / edit}
    P -- no --> X
    P -- yes --> K{School locked?}
    K -- yes, and it's a change --> X2[Read-only]
    K -- no --> OK[✅ Allowed]
```

## 🔐 School lock

If a school stops paying, **the database itself** blocks every change on around 300 tables, so it can't be worked around through the API. Paying the bill, exports, password reset and support still work on purpose.

| Who | What they see |
|---|---|
| School owner | Portal becomes read-only, with a pay button |
| Staff and parents | Lock screen with a 1-minute countdown, then logged out |
| Device apps | Pause and keep their data, carry on after unlock |
| Clever AI | Makes no changes |

## 🤖 Clever AI

An assistant inside the admin portal that can answer anything about the school and make changes. It **never changes anything without asking first.**

```mermaid
sequenceDiagram
    actor Admin
    participant AI as Clever AI
    participant DB as Database

    Admin->>AI: "Send fee reminders to Class 8"
    AI->>DB: Find Class 8 dues (same checks as the page)
    DB-->>AI: 23 students with pending fees
    AI-->>Admin: Plan: remind 23 parents. Approve?
    Admin->>AI: Approve
    AI->>DB: Run once
    DB-->>AI: Done
    AI-->>Admin: ✅ 23 reminders sent
```

- It only sees what the logged-in person can see
- Every change is shown as before → after, then approved
- Risky changes (deletes, permissions, money going out) also need a typed confirmation and a 2-step code from the last 5 minutes, checked on the server
- It searches for the few tools it needs out of hundreds, which made each request about 78% smaller
- Tested with hundreds of real questions, real changes and attempts to trick it

---

<p align="center">
Shivam Kumar · <a href="mailto:proffesorrk@gmail.com">proffesorrk@gmail.com</a> · <a href="https://www.linkedin.com/in/shivam-kumar-28b40b306/">LinkedIn</a> · <a href="https://clarvix.co">clarvix.co</a>
</p>
