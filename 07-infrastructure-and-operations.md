# Infrastructure and operations

How Clarvix is hosted, monitored and kept running. The whole backend is self-hosted: Supabase runs in Docker on one Oracle Cloud server, and the web apps run on Cloudflare.

[← Back to overview](README.md)

---

## Hosting at a glance

| Layer | What runs there |
|---|---|
| **Cloudflare Workers** | All web portals (school admin, teachers, parents, super-admin, website) |
| **Cloudflare R2** | Files: documents, receipts, report cards, app releases |
| **Oracle Cloud VM** | Ampere ARM, 2 vCPU, 12 GB RAM, Ubuntu 24.04 |
| **Docker on the VM** | Self-hosted Supabase in 11 containers |
| **Host services on the VM** | nginx, PDF renderer, reports service, email template service, metrics pusher (pm2) |
| **Database** | PostgreSQL 17 |
| **AI** | Claude on Azure AI Foundry |
| **Monitoring** | Grafana |

## Docker setup on the Oracle VM

```mermaid
flowchart TB
    CF[☁️ Cloudflare<br/>portals on Workers] --> NG[nginx]
    APPS[📱 Android apps] --> NG

    subgraph VM["Oracle Cloud VM · Ubuntu 24.04 · ARM"]
        NG --> EN

        subgraph DK["Docker · Supabase (11 containers)"]
            EN[Envoy<br/>API gateway]
            EN --> AU[GoTrue<br/>logins & 2-step]
            EN --> RS[PostgREST<br/>REST API]
            EN --> RT[Realtime<br/>live updates]
            EN --> ST[Storage<br/>files]
            EN --> FN[Edge Runtime<br/>webhooks · SMS · push]
            ST --> IM[imgproxy<br/>image resizing]
            RS & AU & RT & ST & FN --> PO[Supavisor<br/>connection pooler]
            PO --> DB[(PostgreSQL 17)]
            SD[Studio + Meta<br/>admin tools] --> DB
        end

        subgraph HS["Host services"]
            PDF[PDF renderer<br/>invoices · receipts · report cards]
            RP[Reports service]
            EM[Email template service]
            MP[Metrics pusher · pm2]
        end
        PDF & RP & EM --> DB
    end

    MP --> GF[📈 Grafana]
```

| Container | Job |
|---|---|
| **PostgreSQL 17** | All data, security rules, business logic and scheduled jobs |
| **GoTrue** | Sign-in, sessions, 2-step verification |
| **PostgREST** | Turns the database into a REST API that respects row-level security |
| **Realtime** | Private per-school channels for chat, notifications, live bus map and attendance logs |
| **Storage** | File uploads and downloads |
| **Edge Runtime** | Backend functions for payment webhooks, SMS, push notifications and the bus tracker |
| **Supavisor** | Connection pooling so the database isn't flooded |
| **Envoy** | Gateway that routes every request to the right service |
| **imgproxy** | Resizes and optimizes images on the fly |
| **Studio + Meta** | Database admin tools |

### Postgres extensions in use

| Extension | Used for |
|---|---|
| `pg_cron` | 52 scheduled jobs |
| `pg_net` | Outgoing HTTP calls from the database (webhooks, notifications) |
| `pgmq` | Message queues |
| `pgcrypto` | Hashing and encryption |
| `pg_trgm` | Fuzzy search for names |
| `supabase_vault` | Storing secrets encrypted |
| `pg_stat_statements`, `index_advisor`, `hypopg` | Finding slow queries and testing indexes |
| `wrappers` | Foreign data wrappers |

---

## Monitoring with Grafana

```mermaid
flowchart LR
    subgraph Sources
        VM[🖥️ VM metrics<br/>CPU · memory · disk · load · uptime]
        DC[🐳 Docker containers<br/>up / down per service]
        FE[🌐 Frontend routes<br/>every portal checked]
        DBH[🗄️ Platform health sync<br/>every minute]
    end
    VM & DC --> MP[Metrics pusher]
    MP --> GF[📈 Grafana dashboards]
    FE --> GF
    DBH --> SA[Super-admin<br/>Platform health page]
    GF --> AL[🚨 Alerts]
```

| What's watched | How |
|---|---|
| Server CPU, memory, disk, load, uptime | Metrics pusher → Grafana |
| Every Docker container | Container status pushed to Grafana |
| Frontend portal routes | Route checks shown in Grafana |
| Scheduled jobs, workers, incidents | Platform health sync every minute → super-admin Platform health page |
| Payment gateways | Gateway incident monitor every 2 minutes |
| Live buses | Live transport monitor every 2 minutes |

---

## Automatic jobs (52, run by pg_cron)

```mermaid
flowchart TB
    CRON[⏱️ pg_cron<br/>52 jobs] --> BL[💳 Billing]
    CRON --> NT[📧 Emails & notifications]
    CRON --> CR[📣 Admissions CRM]
    CRON --> TP[🚌 Transport]
    CRON --> AT[✅ Attendance & people]
    CRON --> PL[🛡️ Platform & security]
```

| Area | Jobs | How often |
|---|---|---|
| **Billing & invoices** | Subscription billing cycle (auto invoices), dunning (payment reminders), school lifecycle (trials, renewals, locks), monthly student fee generation, payment gateway recovery | every 1–15 min |
| **Emails & notifications** | Comms automation, absent-student alerts to parents, homework publishing, parent learning alerts, email cleanup, notification retention | every 2–15 min, some daily |
| **Admissions CRM** | Follow-up, appointment, event and document reminders, offer expiry, email sequences, workflows, seat updates, nightly lead scoring, weekly forecast | every 2–30 min, nightly, weekly |
| **Transport** | Live bus monitor, stale driver check, document renewal alerts, daily trip automation, enrolment sync, GPS history cleanup | every 2–60 min |
| **Attendance & people** | Mark absent students, leave policy rules (nightly 11:30 PM IST), early pickup pass expiry, workforce automation | every 5 min, nightly |
| **Platform & security** | Device pairing sweep, privileged session expiry, ops watchdog, compliance, product automation, AI housekeeping, job log cleanup | every 5–60 min, daily |

## Subscription billing, fully automatic

```mermaid
stateDiagram-v2
    [*] --> Trial
    Trial --> Active: plan chosen
    Active --> Invoiced: billing cycle job
    Invoiced --> Active: paid
    Invoiced --> Overdue: due date passed
    Overdue --> Overdue: reminder emails (dunning)
    Overdue --> Locked: still unpaid
    Locked --> Active: owner pays inside the portal
    Locked --> Retention: long unpaid
    Retention --> [*]: data deleted after warnings + 2 approvals
```

- Invoices are created and emailed automatically, with PDF and GST details
- Reminders go out on a schedule before anything is locked
- A locked school can still pay from inside the portal, and unlocks right after payment

## Emails and notifications

```mermaid
flowchart LR
    EV[Something happens<br/>fee due · absent · new lead · invoice] --> Q[Notification<br/>queue]
    Q --> IA[🔔 In-app bell<br/>& parent inbox]
    Q --> PU[📲 Push<br/>parent & driver apps]
    Q --> EM{📧 Email}
    Q --> SM[💬 SMS<br/>payment links]
    EM --> SC[School's own mailbox<br/>admission emails · sequences · events]
    EM --> CX[Clarvix platform mail<br/>invoices · dunning · newsletters · notices]
    EM --> AU[Login emails<br/>branded templates]
```

| Channel | Used for |
|---|---|
| In-app | Staff bell and notification centre, parent inbox |
| Push | Parent app (tapping opens the exact item), driver schedule |
| Email (school's mailbox) | Admission status, follow-up sequences, event emails |
| Email (Clarvix) | Invoices, payment reminders, newsletters, notices, weekly digest, daily attendance summary |
| Email (login) | Branded sign-in, invitation and recovery emails |
| SMS | Payment links through the payment gateway |
| WhatsApp | Click-to-chat from the CRM with templates, sharing payment and referral links |

---

## Releases and backups

```mermaid
flowchart LR
    A[Write changes] --> B[Test on a copy<br/>of production]
    B --> C[Throw the copy away]
    C --> D[💾 Backup]
    D --> E[Apply migrations<br/>in fixed order]
    E --> F[Deploy Workers]
    F --> G[Live checks with<br/>temporary schools]
    G --> H[Delete test schools]
```

- 466 versioned migrations so far, each with a tested rollback where it changes existing behaviour
- A dated database backup is taken before every release and kept on the server
- Full disaster recovery tested at about 6.5 minutes
