# GVM — System Dossier & Detailed Flow

**One autonomous machine that finds local businesses to sell to, qualifies them, and hands you a ready-to-call CRM — and also *is* the product those businesses buy.**

*Version 1.0 · 2026-06-20 · Built and operated on a single Google Cloud VM. This document is written for two readers at once: a non-technical person (investor, partner) and a technical person (engineer, diligence). Every major section starts with a plain "In one line" before the detail. No passwords, API keys, or customer data appear anywhere in this file.*

---

## Table of contents

1. [The 30-second version](#1-the-30-second-version)
2. [The problem](#2-the-problem)
3. [The solution at a glance](#3-the-solution-at-a-glance)
4. [Master system map](#4-master-system-map)
5. [★ The Autonomous Lead Engine — step-by-step flow](#5--the-autonomous-lead-engine--step-by-step-flow)
6. [The scoring brain — how a lead is judged](#6-the-scoring-brain--how-a-lead-is-judged)
7. [The CRM output — the Google Sheet](#7-the-crm-output--the-google-sheet)
8. [The sales & field workflow](#8-the-sales--field-workflow)
9. [The product being sold — GVM WhatsApp Front Desk](#9-the-product-being-sold--gvm-whatsapp-front-desk)
10. [The platform & infrastructure](#10-the-platform--infrastructure)
11. [Hermes — the autonomous brain](#11-hermes--the-autonomous-brain)
12. [Economics & cost](#12-economics--cost)
13. [Market & expansion](#13-market--expansion)
14. [Compliance reality (India)](#14-compliance-reality-india)
15. [Proof & traction](#15-proof--traction)
16. [Roadmap](#16-roadmap)
17. [Appendix — full inventory, webhooks, glossary](#17-appendix)
18. [Frequently asked questions](#18-frequently-asked-questions)

---

## 1. The 30-second version

**In one line:** You type *"get me diagnostic labs in Nashik"* into a chat app, and a few minutes later a Google Sheet fills itself with the best labs to call — names, owners' numbers, why each one will (or won't) buy, and a ranked "call this week" list — with **no human doing the research**.

GVM is two engines that feed each other:

- **Engine 1 — the customer-finding machine (the "lead engine").** An autonomous agent + an automation pipeline that discovers real businesses from Google Maps, researches each one, applies our qualification rules, and writes a finished sales CRM. This is how GVM finds its *own* customers, cheaply and at scale.
- **Engine 2 — the product we sell (the "Front Desk").** A WhatsApp (and, later, voice) assistant that sits on a business's own number and answers every customer message 24×7 — bookings, prices, report-status, after-hours capture — so small businesses stop losing leads to missed calls.

Both run on **one rented cloud computer** that currently costs us **₹0/month** (Google's free trial credit), and both are already live and demonstrated.

---

## 2. The problem

**In one line:** Small Indian businesses lose paying customers every day simply because nobody picks up the phone — and the usual fix (hiring a receptionist) is expensive and still leaves nights and weekends uncovered.

- A busy diagnostic lab, clinic, or coaching centre in a Tier-2 city takes **dozens of calls a day**: "Is my report ready?", "What's the price of a CBC?", "Do you do home collection?", "Are you open now?".
- When the line is busy or it's after hours, **15–30% of those calls go unanswered** — and an unanswered call is usually a customer who simply calls the next lab.
- The traditional fix is a **front-desk receptionist at ₹15,000–22,000/month** — a real cost for an SMB, and they still go home at night, take breaks, and can't be in two places at once.
- Meanwhile, the **sellers** of software to these businesses have their own problem: the businesses aren't in any clean database. They're pins on Google Maps. Finding, qualifying, and prioritising them by hand is slow, and most are independents that email-list vendors miss entirely.

GVM attacks both sides of that gap with one system.

---

## 3. The solution at a glance

**In one line:** A single autonomous system that (a) finds and qualifies the businesses worth selling to, and (b) is the always-on front desk those businesses pay for.

```
                       ┌─────────────────────────────────────────────┐
   "Find me labs   →   │   GVM  (one cloud machine)                │   →  Ready CRM sheet
    in Nashik"         │                                             │      + ranked call list
                       │   Engine 1: finds & qualifies customers      │
                       │   Engine 2: the Front Desk product they buy  │
                       └─────────────────────────────────────────────┘
                                          │
                              sells the Front Desk to
                                          ▼
                          independent labs / clinics / SMBs
```

The rest of this document walks through **exactly how each hand-off works**, then the platform it runs on, then the money.

---

## 4. Master system map

**In one line:** Everything below runs on one Google Cloud computer; the founder talks to it through a chat app, and it talks to Google Maps and Google Sheets on its own.

```
   ┌──────────────┐   plain-English request ("labs in Nashik")
   │  FOUNDER     │ ───────────────────────────────────────────────┐
   │ (Telegram)   │ ◄──── "Done ✅ — sheet ready: <link>" ──────────┤
   └──────────────┘                                                 │
                                                                    ▼
 ╔══════════════════════════ ONE GCP VM  (e2-standard-4 · 4 vCPU · 16 GB · Mumbai) ══════════════════════════╗
 ║                                                                                                            ║
 ║   ┌───────────────────┐        triggers        ┌──────────────────────────────────────────────┐          ║
 ║   │  HERMES            │ ─────────────────────► │  n8n  (automation engine)                    │          ║
 ║   │  autonomous agent  │   webhook call         │                                              │          ║
 ║   │  brain = Gemini    │ ◄───────────────────── │  • Lead Engine   • Sales workflows           │          ║
 ║   │  steered by SOUL.md│   sheet link back      │  • Front-Desk bot (WhatsApp)                 │          ║
 ║   └───────────────────┘                         └───────┬───────────────┬──────────────┬───────┘          ║
 ║                                                         │               │              │                  ║
 ║                                              scrape      │       store    │      message  │                 ║
 ║                                                         ▼               ▼              ▼                  ║
 ║                                              ┌────────────┐   ┌────────────────┐  ┌──────────┐            ║
 ║                                              │ ScrapingDog│   │ Google Sheets  │  │  WAHA /  │            ║
 ║                                              │ (Maps data)│   │   = the CRM    │  │ Meta WA  │            ║
 ║                                              └────────────┘   └────────────────┘  └──────────┘            ║
 ║                                                                                                            ║
 ║   ── all of the above is published to the internet by ──►  Dokploy + Traefik  (hosting + HTTPS) ──►        ║
 ║      n8n.rohinisonawane.com · pharmacare.rohinisonawane.com · dokploy dashboard                           ║
 ║                                                                                                            ║
 ║   (also on the same box, as proof the platform is versatile: PharmaCare PWA + a clinic booking bot)        ║
 ╚════════════════════════════════════════════════════════════════════════════════════════════════════════════╝
```

**The five building blocks (plain English):**

| Block | What it is | Job in GVM |
|------|------------|---------------|
| **Hermes** | An AI agent you chat with on Telegram | The "operator" — understands plain requests and pulls the right levers |
| **n8n** | A visual automation engine (think: programmable plumbing) | Does the actual work — scrape, score, write the sheet, send messages |
| **ScrapingDog** | A Google-Maps data service | Supplies the raw list of real businesses |
| **Google Sheets** | A spreadsheet | *Is* the CRM — the human-readable output the founder works from |
| **WAHA / Meta WhatsApp** | WhatsApp gateways | Carries the Front-Desk product's messages to/from customers |

All of it sits behind **Dokploy + Traefik**, which is the "hosting + automatic HTTPS" layer that makes these private services reachable at clean web addresses.

---

## 5. ★ The Autonomous Lead Engine — step-by-step flow

**In one line:** This is the heart of the business — a fully autonomous loop where a one-sentence request becomes a finished, scored CRM with no human touching the data in between.

### 5.1 The flow as a sequence

```
 STEP 0   Founder (Telegram):  "find diagnostic labs in Nashik"
            │
            ▼
 STEP 1   HERMES reads its instruction file (SOUL.md), recognises this as a
          "find prospects" request, and decides which lever to pull.
          (Brain = Google Gemini. No web-search, no guessing — it calls the engine.)
            │  fires a webhook:  POST /webhook/gvm-leads  { city, industry, tab }
            ▼
 STEP 2   n8n  ►  BUILD QUERY      turns the request into a Maps search string
                                   e.g. "Diagnostic Labs in Nashik, India"
            │
            ▼
 STEP 3   n8n  ►  SCRAPINGDOG      calls the Google-Maps data API, returns ~20 real
                                   businesses (name, phone, website, rating, reviews,
                                   address, map URL).  Auto-retries if ratings missing.
            │
            ▼
 STEP 4   n8n  ►  NORMALIZE        cleans every record into standard columns
            │
            ▼
 STEP 5   n8n  ►  DEDUP            removes repeats (same place) by map identity
            │
            ▼
 STEP 6   n8n  ►  SCORE & FILTER   ★ the "brain" (see §6): for each lab it checks the
                                   website live for WhatsApp/booking/chat, applies the
                                   qualification rules, and decides:
                                     • is it a chain? (skip)   • is it a vet? (skip)
                                     • how much call volume?   • how big is its digital gap?
                                     • Call-This-Week / Nurture / Skip
                                     • owner name + WhatsApp number
                                     • 6 "GVM-fit" scores + an overall fit
            │
            ▼
 STEP 7   n8n  ►  WRITE TO CRM     appends every scored lab into the right tab of the
                                   Google Sheet (one tab per city), matched by place-id
                                   so re-runs update in place instead of duplicating.
            │
            ▼
 STEP 8   n8n  ►  TELEGRAM PING    sends the founder "✅ Done — N leads ready: <sheet link>"
            │
            ▼
 RESULT   A finished CRM sheet, ranked and reasoned, in ~1–2 minutes.  No human in the loop.
```

### 5.2 The "no human in the loop" point (why it matters)

Between STEP 1 and STEP 8 there is **no manual research, no copy-paste, no analyst**. The founder's only actions are *asking* (one sentence) and later *calling* the businesses the system surfaced. That is the leverage: the cost of producing a qualified, city-specific prospect list drops to near-zero and a few minutes of wall-clock time.

### 5.3 The engine's nodes (technical)

Workflow **`GVM Lead Engine`** (n8n id `lboUXTH8g7B5qQuf`, **active**, trigger `POST /webhook/gvm-leads` with `{city, industry, tab?}`):

| # | Node | Type | What it does |
|---|------|------|--------------|
| 1 | Webhook | trigger | Accepts `{city, industry, tab}`. `tab` lets each city write to its own sheet tab while keeping `industry` for scoring. |
| 2 | Build Query | code | Builds the Maps search text + carries `industry`/`city`/`tab` forward. |
| 3 | ScrapingDog | code | Calls the ScrapingDog Google-Maps API; retries up to 3× until ratings are present; returns ~20 results. |
| 4 | Normalize | code | Maps raw fields → standard columns (Business, Phone, Phone_E164, Website, Rating, Reviews, Address, MapsURL, PlaceID, …). |
| 5 | Dedup | code | Drops duplicates by PlaceID. |
| 6 | Enrich + AI Pitch | code | The scoring brain (§6). Fetches each website to detect WhatsApp/booking/chat/contact-form; computes tier, fit, owner, reasons. **Facts-only — no per-lead LLM call** (pitches are generated on demand by a separate workflow). |
| 7 | Append to Industry Tab | Google Sheets | `appendOrUpdate` keyed on PlaceID into `tab` (falls back to `industry`); re-runs update rows in place. |
| 8 | Notify → Telegram Ping | code + HTTP | Posts "Done — N leads, <link>" to the founder's Telegram. |

The same engine is **multi-industry by design** — it carries chain-blocklists and positioning for Diagnostic Labs, Hospitals, Hotels, Insurance, Real Estate, Coaching Classes, and Travel Agencies — and **multi-city** via the `tab` parameter.

---

## 6. The scoring brain — how a lead is judged

**In one line:** The engine doesn't just dump a list — it applies the same rules a sharp salesperson would, so the sheet arrives pre-sorted into "call these now", "warm up these later", and "don't bother".

This logic lives in the engine's "Enrich + AI Pitch" node and a matching "Re-Score" workflow. The rules, in plain English:

1. **Kill the chains.** National brands (Apollo, Metropolis, Suburban, SRL/Agilus, Redcliffe, Thyrocare, Dr Lal, Neuberg, Vijaya, Orange Health, etc.) are flagged `IsChain = Yes` and set to **Skip** — they have central call-centres and no local decision-maker, so they're a waste of a founder's call. *(This list was hardened after a real run in Bangalore surfaced chains the original list missed.)*
2. **Drop out-of-scope businesses.** A veterinary/pet diagnostics centre is not our customer → **Skip** (`out-of-icp`). *(Added after a "for Pets" lab slipped into a Bangalore run.)*
3. **Score the call volume.** Review count is a proxy for how many calls a business fields. More reviews = more inbound = more pain = better fit. Independents with **300+ reviews** get full volume credit. *(Originally the score decayed above 1,500 reviews on the assumption "too busy = already automated" — but for an independent, very high volume is the strongest signal. That decay was removed, so the busiest independents now correctly rank top.)*
4. **Rating as a quality gate.** Very low ratings dampen the score.
5. **Measure the digital gap = the need.** The engine fetches the business's website (if any) and checks for an online-booking tool, a WhatsApp link, a live chat widget, a contact form. **The *more* they're missing, the *more* they need GVM** — so a busy, phone-only, no-website lab is the ideal target.
6. **Decide the tier.** Combining volume + rating + digital-gap into a single pay-probability score:
   - **Call-This-Week** — strong score, real volume (300+ reviews), independent.
   - **Nurture** — decent but smaller, or already fairly digital.
   - **Skip** — chains, out-of-scope, or too small to bother.
7. **Find the human.** Extracts an owner name ("Dr. …") where present and normalises the phone to a WhatsApp-ready number, plus a decision-maker label (owner vs chain-HQ).
8. **Score product fit (6 dimensions).** Each lab gets 1–10 on **ReportStatusFit, HomeCollectionFit, AfterHoursFit, WhatsAppFit, AutomationFit**, rolled into an **OverallFit (0–100)** — a quick read of *which* parts of GVM will land.
9. **Say why.** Every lead carries a plain-English **WhyWillTheyBuy** (e.g. "416 reviews (busy); 5★; phone-only, no website") and the founder later fills **WhyWillTheyNotBuy** from real conversations.

**The moat:** these rules are codified, auditable, and tunable per market. They were validated to reproduce the founder's own hand-picked top-10 for the first city, then sharpened against live runs in two more — i.e. the judgement improves as the system meets reality, not in a vacuum.

---

## 7. The CRM output — the Google Sheet

**In one line:** The deliverable is a normal Google Sheet anyone can read — the founder owns it, works from it, and edits it; the engine only ever fills the *computed* columns so the founder's own notes are never overwritten.

- **Central sheet** (Google Sheets id `1KyUmledYCVHInHw0FxOkPI6bhsU_H5kCau3p7_F_0rI`, access-controlled).
- **One tab per city/industry** — e.g. `Diagnostic Labs` (Aurangabad), `Diagnostic Labs - Nashik`, `Diagnostic Labs - HSR Bangalore`. New tabs are created on demand by a helper workflow.
- **Two tiers of data:**
  - **LEADS** (the dialling list) — identity, owner + mobile, the tier (`OutreachPriority`), the reasons (`WhyWillTheyBuy` / `WhyWillTheyNotBuy`), pain signals, fit scores, plus founder-filled fields like `ContactStatus`, `BuyingEvidence`, `OwnerInterviewCompleted`, and `ObservedReality` (what was seen on a visit).
  - **DEALS** (the deep CRM) — a ~70-column record a lead is "promoted" into once it's qualified: operations, tech stack, objections, pilot metrics, money (what they pay a receptionist, estimated missed calls), commercials, and outcome.
- **Safe by construction:** the engine and the re-scorer emit **only computed columns**, matched by PlaceID, so re-running on a city refreshes scores **without touching** the human's manual notes. Founder edits and machine scores coexist.

---

## 8. The sales & field workflow

**In one line:** Once the CRM exists, a second set of one-tap tools turns it into daily action — today's call list on your phone, log a visit by texting the agent, and promote a hot lead into the deep CRM.

```
   CRM sheet ──► CALL LIST ──► TELEGRAM ──► founder calls/visits ──► LOG VISIT ──► PROMOTE ──► (RE-SCORE)
                (ranked)      (on phone)                          (one message)  (→ DEALS)   (refresh tiers)
```

- **Call List** (`5y8iTy9WxWY5TPrM`, `/webhook/gvm-calllist`) — returns the ranked "Call-This-Week" labs as data, excluding already-won/lost.
- **Call List → Telegram** (`R4ip6F9wSbQlxcK1`, `/webhook/gvm-calllist-tg`) — formats that list and pushes it to the founder's phone, with the owner number, why-they'll-buy, and three pain questions to open with.
- **Log Visit** (`W4lXr2CmhpGpcMwO`, `/webhook/gvm-logvisit`) — the founder texts Hermes a natural sentence after a visit ("*visited MEDCIS, met the owner, 2 receptionists, ~11 patients waiting, reports printed by hand, no WhatsApp; says he already has staff — send 2-min demo*"); Hermes extracts the fields and updates that lab's row (contact status, what was observed, buying evidence, objection, next action) **without overwriting its scores**. This is the "stop scraping, start talking to owners" loop.
- **Promote** (`8zIbOTh90LsXzrhR`, `/webhook/gvm-promote`) — copies a qualified lead into the 70-column DEALS tab.
- **Re-Score** (`w4cIEryWouVOGaTd`, `/webhook/gvm-rescore`) — recomputes tiers on an existing tab (no re-scrape) when the rules change — e.g. after the chain/volume tuning, it correctly promoted a 4,639-review independent that the old rules had buried.
- **Housekeeping helpers** — Create Tab, Patch Rows (safe column writer), and a Dedup tool (kept off unless needed) that backs up a tab before merging duplicate rows.

---

## 9. The product being sold — GVM WhatsApp Front Desk

**In one line:** The thing a lab actually pays for: a WhatsApp assistant on the lab's own number that books walk-ins, answers prices and timings, tells patients when reports are ready, and captures every after-hours message — 24×7.

This is a separate, already-built demo system (its own workflows + its own Git repo). It is **deliberately deterministic**: the AI understands what the patient *wants* (in Hindi/Marathi/English), but the actual booking/lookup runs as code — so it **cannot invent a price or a slot**.

### 9.1 What it does (the flows)

```
  Patient on WhatsApp ─► GVM-LABS BOT (the "Brain") ─► decides intent ─► runs the matching action
                                                          │
   ┌──────────────────────────────────────────────────────┼───────────────────────────────────────┐
   ▼                 ▼                ▼                ▼    ▼                ▼                ▼
 walk-in          home            report           price/FAQ          after-hours       cancellation
 token booking    collection      status            + timings          capture           (find & cancel)
 (Token #, VOC-id) request        (look up sender)   (only real data)   (tag, reply,      open booking)
                                                                         follow up next AM)
                                              ▲
                                              │  staff post a patient's number in the lab's
                                              │  WhatsApp group ─► bot marks report "Ready" +
                                              └─ texts the patient automatically  (highest-ROI flow)
```

### 9.2 The workflows (technical)

| Workflow | n8n id | Role |
|---------|--------|------|
| GVM-Labs Bot — Inbound Router | `LpnFG9kwWAdQOkiw` (active) | The "Brain": Gemini-based intent + per-phone conversation memory, with a numbered-menu fast path; routes to booking / report / price / after-hours / cancel. |
| GVM-Labs Lookup | `QWRu5KjmAes26om0` (active) | Looks a patient up by last-10-digits of phone. |
| GVM-Labs Setup | `lrftZj7qwGjCdJHj` (off) | One-time seeder/reset for the demo. |
| GVM-Labs Reminders | `ycLwHgDbI4qqnY8C` (off, cron) | After-hours morning nudge + 2-day follow-up. |

### 9.3 The honest reality (investor-relevant)

- **WhatsApp front desk = built and demonstrated end-to-end** (booking, report status, after-hours, cancellation). This is the **wedge** — low risk, fast to deploy.
- **Demo runs on WAHA** (an unofficial WhatsApp gateway) — fine for demos, **not** for production at scale. The **production path is Meta's official WhatsApp Cloud API** (compliant, no ban risk), which is a known, documented switch.
- **Voice receptionist = on the roadmap, gated** on validating Marathi/Hindi latency and quality before we promise it. We sell what we can deliver today and add voice when it's proven.

---

## 10. The platform & infrastructure

**In one line:** Everything runs on a *single* rented Google Cloud computer — one box doing the work of a small stack — which keeps cost near zero and the whole system portable.

### 10.1 The machine

| Attribute | Value |
|-----------|-------|
| Provider / type | Google Cloud Compute Engine, **e2-standard-4** |
| Power | **4 vCPU, 16 GB RAM**, 96 GB disk |
| Region | **asia-south1 (Mumbai)**, zone `-c` |
| Public IP | `34.93.159.255` |
| OS | Ubuntu 24.04 LTS |
| Live since | 2026-06-07 · ~40% disk, light load (plenty of headroom) |

### 10.2 What's running on it

| Service | What it is | Public address |
|---------|------------|----------------|
| **Hermes** | The autonomous AI agent (see §11) | Telegram `@Livehermes_bot` |
| **n8n** + Postgres | The automation engine running all GVM workflows | `n8n.rohinisonawane.com` |
| **WAHA** | WhatsApp gateway for the Front-Desk demo | internal only |
| **Dokploy** | Self-hosted hosting platform (deploys + manages the apps) | dokploy dashboard |
| **Traefik** | Reverse proxy: routes traffic + issues HTTPS certificates automatically (Let's Encrypt) | ports 80/443 |
| **PharmaCare** web+api | A pharmacy PWA — *adjacent proof the same box ships real products* | `pharmacare.rohinisonawane.com` |

### 10.3 Why one box (the design choice)

- **Cost:** one VM, currently on free credit, replaces a fistful of paid SaaS subscriptions.
- **No per-message platform tax:** n8n is open-source and self-hosted, so automations cost compute, not per-run fees.
- **Data stays with the customer:** the CRM is the founder's own Google Sheet — no vendor lock-in, nothing trapped in a proprietary database.
- **Portable & reversible:** the whole stack is containerised; it can be lifted to a bigger machine, or to another cloud, without a rewrite. Every experimental change (e.g. the WhatsApp demo) is built to be switched off cleanly.

---

## 11. Hermes — the autonomous brain

**In one line:** Hermes is the always-on AI agent you talk to like a colleague; it understands plain requests, knows which tools to run, and improves at your repetitive work over time — without a human writing code each time.

- **What it is:** a self-hosted autonomous agent (`nousresearch/hermes-agent`) whose "brain" is the **Google Gemini API** (so the heavy thinking is a cheap API call, not a giant model on our box).
- **How you use it:** entirely through **Telegram** — natural messages, no UI to learn.
- **How it's steered:** a single instruction file, **`SOUL.md`**, that defines its capabilities and the exact tools/commands to run for each. It's loaded fresh on every message, so its behaviour can be updated instantly — no restart, no redeploy. This is how new skills get added: describe the capability + the command, and Hermes can do it.
- **What it can do today** (its current capabilities):
  - **GVM Prospect Finder** — "find <business type> in <city>" → runs the lead engine, returns the sheet.
  - **Today's Call List** — "who should I call?" → pushes the ranked list to Telegram.
  - **Log a Field Visit** — natural "I visited X…" → records the visit into the CRM.
  - **Clinic Sales-Lead Lookup** — medical-specialty lead lookups from an existing database.
- **The vision:** Hermes is the single, conversational control surface for the whole business — the founder describes intent; the machine plans and executes. It accrues skills as the work repeats, turning manual operations into one-sentence commands.

---

## 12. Economics & cost

**In one line:** It costs essentially nothing to run today (Google's free credit), a few thousand rupees a month once paid, and each customer costs a few hundred rupees a month to serve against a price point that replaces a ₹15,000–22,000 receptionist — so gross margins are 80–90%.

### 12.1 What it costs to run the platform

| Item | Today | Fully paid (steady state) | Notes |
|------|-------|---------------------------|-------|
| GCP VM (e2-standard-4, Mumbai) | **₹0** | **₹8,000–9,000/mo** | Currently on the GCP **$300 / 90-day** free trial credit (the figure is $300, not $100), plus an always-free tier. |
| ScrapingDog (Maps data) | low/seed | **~$40/mo (~₹3,300)** | A Maps search ≈ 5 credits; covers many city sweeps. |
| Google Gemini (Hermes + on-demand pitches) | **~₹0** | **<₹100/mo** | Generous free tier; 2.5 Flash is very cheap per token. |
| n8n (automation engine) | **₹0** | **₹0** | Open-source, self-hosted. |
| Google Sheets (the CRM) | **₹0** | **₹0** | Free. |
| WhatsApp (demo via WAHA) | **₹0** | n/a | Production uses Meta Cloud API, billed per customer (below). |
| **Platform total** | **≈ ₹0/mo** | **≈ ₹4,500–9,000/mo** | One platform serves *all* customers. |

### 12.2 What it costs to serve one customer (the Front Desk)

| Item | Per customer / month |
|------|----------------------|
| WhatsApp Cloud API messages (utility ≈ ₹0.115/msg) | ₹150–400 |
| Storage + Gemini FAQ answers | ₹50–300 |
| Amortised VM share | ₹300–600 |
| **COGS per customer** | **≈ ₹500–1,300/mo** |

### 12.3 Pricing & margin

- **Anchor:** the customer's alternative is a **₹15,000–22,000/month receptionist** who still can't cover nights.
- **GVM price:** entry **₹1,499–3,000/month** (WhatsApp wedge), scaling to higher tiers with voice/booking bundles.
- **Gross margin: ~80–90%.** Most customers recover the subscription within weeks from recovered missed-call revenue.
- **Customer acquisition** is the lead engine itself (near-zero marginal cost to produce a qualified, reasoned prospect list), plus founder-led calls/visits — no paid ads required to start.

### 12.4 Competitive price context

| Alternative | Price | Gap GVM fills |
|-------------|-------|------------------|
| Cloud lab-report apps | ₹400–800/mo | report delivery only — no live front desk |
| Enterprise lab CRMs (e.g. CrelioHealth tier) | ₹8,000–25,000/mo | enterprise-priced; WhatsApp gated behind top tiers |
| Metered IVR/voice (e.g. MyOperator) | per-minute | no WhatsApp, no booking logic |
| US-centric AI receptionists | ~$49–499/mo | not built for Indian SMBs / vernacular / price points |
| **GVM** | **₹1,499–3,000/mo** | WhatsApp-first, instant setup, 24×7, India-compliant |

*(Cost figures web-verified June 2026 against GCP, ScrapingDog, Gemini, Meta WhatsApp, and n8n public pricing; treat as planning estimates, not contractual quotes.)*

---

## 13. Market & expansion

**In one line:** Start with diagnostic labs in Tier-2 Maharashtra where independents are plentiful, prove it, then replicate city-by-city and industry-by-industry — the engine already supports seven verticals.

- **Beachhead:** independent **diagnostic labs** in **Aurangabad**, then **Pune / Nashik / Nagpur**.
- **A real, data-backed finding from live runs:** **Tier-2 cities are a far richer market than metros.** A Nashik run returned **18 of 20 independent** labs; a Bangalore (HSR Layout) run returned only **8 of 20** (the rest national chains). The system *measures* this automatically, so we can point sales at the densest independent supply.
- **Other verticals already wired in** (chain blocklists + positioning ready): Hospitals/clinics, Hotels, Insurance agencies, Real estate, Coaching classes, Travel agencies.
- **Closing dynamics** (from GTM research): labs and coaching show the strongest near-term close probability; the wedge is the WhatsApp front desk because the pain (missed calls, report-status questions) is immediate and measurable.

---

## 14. Compliance reality (India)

**In one line:** We deliberately operate inside the rules — no cold AI voice spam, no grey-market WhatsApp at scale — which turns compliance into a selling point rather than a risk.

- **TRAI / TCCCPR:** cold AI-voice broadcasting is restricted. GVM's sales outreach is **human-led 1:1 calls**, and the *product* is an **inbound** receptionist (the customer's own customers message in) — both compliant.
- **WhatsApp:** unofficial gateways are demo-only; production runs on **Meta's official WhatsApp Business / Cloud API**, which is the sanctioned channel.
- **DPDP Act 2023:** prospect data is sourced from businesses' **own public listings**; provenance is recorded. Personal data a business itself published is treated accordingly, and every lead row carries its source.
- **Net:** "we're built compliant" is a trust signal to SMB owners and a moat against fly-by-night competitors.

---

## 15. Proof & traction

**In one line:** This isn't a slide deck — the autonomous loop runs today, has been pointed at three cities, and the WhatsApp product has been demonstrated end-to-end.

- ✅ **The autonomous lead→CRM loop is live** and produces scored sheets in minutes.
- ✅ **Three markets scored** through the engine: Aurangabad (beachhead), Nashik, HSR Bangalore — with the metro-vs-Tier-2 insight surfaced automatically.
- ✅ **Scoring validated** against the founder's own hand-picked top-10, then hardened against two live runs (chain + veterinary exclusions added, volume model recalibrated, duplicate rows cleaned).
- ✅ **WhatsApp Front Desk demonstrated** end-to-end (booking, report status, after-hours, cancellation) on a real WhatsApp number.
- ✅ **Field-capture loop live** — visits can be logged into the CRM from the phone by texting the agent.
- ✅ **31 automations** running on the platform (21 active), all on one ₹0-today VM.

---

## 16. Roadmap

**In one line:** The build is largely done; the next phase is conversations with real owners and disciplined scale-up.

| Status | Item |
|--------|------|
| ✅ Done | Lead engine, scoring brain, multi-city tabs, Sales CRM, call-list-to-phone, log-visit, duplicate cleanup, chain/vet hardening, volume recalibration |
| ▶ Now | Owner visits in Nashik/Aurangabad; capture real pains + objections; convert the first paying labs onto Meta Cloud API |
| ⏭ Next | Scale city-by-city; turn on production WhatsApp billing; validate and add the **voice** receptionist; expand to the next vertical |
| 🔭 Later | More Hermes skills (the agent automates more of the operation); analytics on conversion by city/industry |

---

## 17. Appendix

### 17.1 GVM workflow inventory (n8n)

| Workflow | id | Status | Trigger / webhook |
|----------|-----|--------|-------------------|
| **GVM Lead Engine** (core) | `lboUXTH8g7B5qQuf` | active | `POST /webhook/gvm-leads` |
| Re-Score | `w4cIEryWouVOGaTd` | active | `/webhook/gvm-rescore` |
| Call List | `5y8iTy9WxWY5TPrM` | active | `/webhook/gvm-calllist` |
| Call List → Telegram | `R4ip6F9wSbQlxcK1` | active | `/webhook/gvm-calllist-tg` |
| Pitch (on-demand) | `6Cw0YAn598JuYeN3` | active | `/webhook/gvm-pitch` |
| Promote (LEADS→DEALS) | `8zIbOTh90LsXzrhR` | active | `/webhook/gvm-promote` |
| Log Visit | `W4lXr2CmhpGpcMwO` | active | `/webhook/gvm-logvisit` |
| Create Tab | `CAu5AIvnJMJjayhk` | active | `/webhook/gvm-createtab` |
| Patch Rows | `bp3WGhl8paJXuBG1` | active | `/webhook/gvm-patch` |
| Setup – Create Tabs | `BUzBi2ktkGR199Dp` | — | `/webhook/gvm-create-tabs` |
| Dedup Tab | `yWjxRR1DbeecqunH` | off | `/webhook/gvm-dedup` |
| Rebuild LEADS (one-shot) | `FrYVfEy5seW07LBJ` | off | one-shot |
| GVM-Labs Bot — Inbound Router | `LpnFG9kwWAdQOkiw` | active | `/webhook/gvm-labs-in` |
| GVM-Labs Lookup | `QWRu5KjmAes26om0` | active | `/webhook/gvm-labs-lookup` |
| GVM-Labs Setup | `lrftZj7qwGjCdJHj` | off | `/webhook/gvm-labs-setup` |
| GVM-Labs Reminders | `ycLwHgDbI4qqnY8C` | off | cron |

**Adjacent / earlier automations on the same instance** (not core GVM): clinic lead-discovery (`mKifdBsFGCFYOQHp`), website enrichment (`LFjXaSIg85ctzy47`), leads lookup+rank (`Cis0X2ErThustWQc`), apply-QA-flags (`1qUhXRjZtF9qJn7j`), reset-leads (`5YAUvp3QYDEQyUZk`), user-record lookup (`cmo5eqCtd9lTM66r`), clinic appointment reminders cron (`tyaAd4At5c0rTR23`), a clinic WhatsApp booking bot (`tKf2dsjVyGunlhr7`, off), and an experimental lead MCP server (`DOmJKgtseAhgCv6i`). *Instance total: 31 workflows, 21 active.*

### 17.2 Services & URLs

| Service | URL |
|---------|-----|
| n8n automation | `https://n8n.rohinisonawane.com` |
| PharmaCare PWA | `https://pharmacare.rohinisonawane.com` |
| Dokploy dashboard | dokploy dashboard (admin) |
| Hermes agent | Telegram `@Livehermes_bot` |
| CRM | Google Sheet `1KyUmledYCVHInHw0FxOkPI6bhsU_H5kCau3p7_F_0rI` (access-controlled) |

### 17.3 Glossary (for the non-technical reader)

- **VM (virtual machine):** a rented computer in Google's data centre.
- **n8n:** a tool that lets you wire up automations visually (like a flowchart that actually runs).
- **Webhook:** a private web address that, when "pinged", kicks off an automation.
- **ScrapingDog:** a service that fetches Google-Maps business listings for us.
- **Hermes:** our AI agent you chat with on Telegram; it runs the tools for you.
- **Gemini:** Google's AI model — the "brain" Hermes (and the pitch writer) calls.
- **WAHA / Meta Cloud API:** ways to send & receive WhatsApp messages (demo vs official-production).
- **Dokploy / Traefik:** the hosting + automatic-HTTPS layer that publishes our services to the web.
- **CRM:** Customer Relationship Management — here, simply the Google Sheet of leads/deals.
- **Lead / tier:** a prospect business; "Call-This-Week / Nurture / Skip" is how urgently to pursue it.

### 17.4 Source documents (deeper detail)

- `GVM-AURANGABAD-GTM.md` — full GTM strategy, scoring rubric, market research.
- `gvm-labs/DOCUMENTATION.md` — the WhatsApp Front Desk, node-by-node.
- `DIAGNOSTIC-LABS-DEMO.md` — the lab demo plan + as-built.
- `WHAT-WE-BUILT.md` — plain-English platform walkthrough.
- `WHATSAPP-PRODUCTION-SETUP.md` — the Meta Cloud API production path.

---

## 18. Frequently asked questions

**In one line:** Honest answers to the questions a VC, an engineer, a cautious lab owner, and a compliance lawyer all ask — including the gaps we're still closing, with the mitigation stated plainly rather than spun away.

### 18.1 Product & how it works

**Q: What is the business, exactly — am I buying the lead engine or the WhatsApp bot?**
You're buying a vertical SaaS product: an always-on WhatsApp "Front Desk" for independent Indian SMBs (beachhead: diagnostic labs in Tier-2 Maharashtra) that answers patient messages 24×7 — bookings, prices, report-status, after-hours capture — so they stop losing customers to missed calls. The revenue line is the Front Desk subscription. The autonomous lead engine is not a separate product; it's our internal go-to-market machine that finds and pre-qualifies the businesses to sell to at near-zero marginal cost — a CAC-reduction asset, not a second revenue line. Both are built and running on one cloud VM today.

**Q: What is the wedge versus the vision?**
The wedge is the WhatsApp Front Desk — low-risk, fast to deploy, built and demonstrated end-to-end, with measurable ROI (recovered missed calls). The vision is to become the conversational front-office layer for Indian SMBs: WhatsApp first, then a vernacular voice receptionist, then more verticals (the engine is already wired for hospitals, hotels, insurance, real estate, coaching, travel). The honest gap is that voice — the biggest part of the vision — is not yet validated; it's gated on proving Marathi/Hindi latency and quality on real calls. We deliberately sell only what we can deliver today.
  - *Follow-up:* If the real money is in voice, is WhatsApp just a low-value foot in the door? — No. The highest-ROI flow (staff post a number in their group, the bot marks the report ready and auto-texts the patient) solves a daily pain at ₹1,499–3,000/month against a ₹15,000–22,000/month receptionist. WhatsApp lands the logo and the trust; voice is the expansion revenue with a higher ARPU ceiling. The honest risk: if voice never clears the quality bar, we're a WhatsApp-automation company with a lower ARPU ceiling and the TAM math changes.

**Q: Can it really replace a receptionist?**
No, and we won't sell it that way. It replaces the repetitive, after-hours, and overflow part of the job — "is my report ready", "what's the price", "are you open" — the calls that come when the one staff member is already on the line or has gone home. The judgement, the warmth with a worried patient, the messy edge cases stay with your people. It's a tireless extra hand for the boring, high-volume stuff for about a tenth of a salary, so your staff spend their time on patients in front of them.

**Q: How is this different from a receptionist already sending reports on WhatsApp manually?**
It's automatic and two-way. Today a staff member has to remember, find the number, and type it out, which is why it slips at peak hours. With GVM, when staff drop the patient's number in the lab's WhatsApp group, the bot marks the report ready and texts the patient itself, instantly, even at 8pm — and answers the patient's follow-up without anyone touching the phone.

**Q: Do patients have to install anything, and does it work on my existing number?**
Patients install nothing — they message your number on the WhatsApp they already have. For production we move you onto Meta's official WhatsApp Business API, the sanctioned, ban-safe channel; depending on how your number is set up we either enable it for the API or use a dedicated business number, sorted during setup. The first demo you see runs on an unofficial gateway that's fine for showing the system but not for production — the real deployment uses the official Meta channel.

**Q: Does it actually work in Marathi and Hindi?**
Yes — the WhatsApp assistant understands Hindi, Marathi, and English for what patients type, because we're built for Tier-2 Maharashtra. The AI brain handles vernacular intent and the answers come from the lab's fixed data, so the reply is correct regardless of language. We'd rather you test it in Marathi yourself during the demo than take our word for it. (Voice in Marathi is a separate, not-yet-validated roadmap item — see §18.10.)

### 18.2 The autonomous lead engine

**Q: Is "autonomous, no human in the loop" real, or is the founder quietly doing the work?**
It's real for the research-to-CRM portion specifically. Between the one-sentence Telegram request and the finished, scored Google Sheet there is no manual research, copy-paste, or analyst: Hermes recognises the intent, fires the n8n webhook, and the pipeline scrapes ~20 businesses, normalises, dedups, scores against codified rules, writes the right city tab, and pings back — all in 1–2 minutes. What is emphatically not autonomous is the selling: the founder calls and visits the businesses surfaced. The accurate claim is "autonomous prospect generation", not "autonomous revenue".

**Q: How is the lead scoring validated — not just "it sounds plausible"?**
Two ways, with stated limits. First, the rules were tuned until they reproduced the founder's own hand-picked top-10 for Aurangabad — a ground-truth check against expert human judgement. Second, they were hardened against two more live runs (Nashik, HSR Bangalore), where reality exposed gaps. The honest ceiling: "reproduces the founder's top-10" is validation against one expert's opinion, not against actual buying outcomes — we don't yet have a who-bought-vs-who-scored-high dataset because first customers are still being closed. The scoring is validated for face-correctness and expert agreement; not yet against revenue.
  - *Follow-up:* Couldn't reproducing one person's top-10 just mean you overfit to that person? — That's exactly why we didn't stop there. The top-10 is a sanity floor; the harder evidence came from the two live runs surfacing cases the founder hadn't anticipated (a veterinary lab, chains the original blocklist missed, an under-ranked high-volume independent), each forcing a rule change. The only validation that truly matters is win-rate by score tier, which our own GTM notes flag as a hypothesis to test against the first 10–15 real conversations. We're at that step, not past it.
  - *Follow-up:* What concrete rule changes came out of the live runs? — Three. Chains: a Bangalore run surfaced national brands the original list missed, so the blocklist was hardened (Apollo, Metropolis, SRL/Agilus, Thyrocare, Dr Lal, Neuberg, etc.). Scope: a "for Pets" veterinary lab slipped in, so an out-of-ICP skip was added. Volume: the original score decayed above 1,500 reviews on a "too busy = already automated" assumption, wrong for independents — that decay was removed, and on re-score it correctly promoted a 4,639-review independent the old rules had buried.

**Q: How do you catch scoring drift over time?**
Today, drift is caught by the human, not by a monitor. A Re-Score workflow recomputes tiers on an existing tab with no re-scrape, so when a rule changes we re-run the whole history and see what moves. But there's no automated alarm that the score distribution or tier mix has shifted — the signal currently arrives as the founder noticing a bad call list. As real outcomes accumulate, the right next step is the conversion-by-city/industry analytics that sit on the "Later" part of the roadmap. We don't pretend that loop is closed yet.

**Q: How does the engine avoid mis-classifying a chain as an independent?**
Chain detection is a maintained blocklist of known national brands matched against the business name, setting `IsChain = Yes` and tier Skip — explicit and auditable rather than a model guess. Its weakness is that it only catches chains it knows; false-negatives (an unlisted chain slipping through) are the live failure mode, caught when a founder calls and hits a central call-centre, then adds the brand. The cost of a miss is one wasted call, and Tier-2's low chain density (Nashik: 18/20 independent) makes the reactive list acceptable in the beachhead.
  - *Follow-up:* If chain detection is imperfect, can you trust the 18/20 vs 8/20 ratios? — They're directional, not an audited census. But the direction of any error helps us: an imperfect blocklist undercounts chains, which would make a metro look more independent-rich than it is — yet Bangalore still came out chain-heavy at 8/20. So detection error, if anything, understates the Tier-2 advantage, which strengthens the conclusion rather than weakening it.

**Q: When you re-run a city you only get ~20 results — isn't that a small, biased slice?**
Correct, and worth being clear about. One ScrapingDog Maps query returns ~20 businesses, a top-slice ranked by Google's own relevance, not a full census. That's fine for the immediate job — a high-quality ranked call list this week — but it means market-size claims and chain ratios are sampled, not exhaustive. The pipeline accumulates coverage across runs (dedup keyed on PlaceID), and the GTM plan describes deliberate per-vertical passes to convert directory ranges into deduped counts. Treat per-run output as "the best ~20 to call" and any total-market number as an estimate needing multiple sweeps.
  - *Follow-up:* Does Google's ranking bias which 20 you see — favouring businesses with websites? — Almost certainly some, and it cuts against our thesis if unmanaged: Maps relevance can favour richer digital presence, the opposite of our ideal phone-only target. We don't have a measured study of that bias. The partial counter is that our scoring then actively rewards the digital gap, re-sorting toward phone-only independents rather than taking Google's order as the answer. Quantifying and correcting for Maps' ranking bias is a coverage-quality improvement we'd want before strong TAM claims.

**Q: Could you sell the lead engine itself as a product?**
Possibly as a future option, but we wouldn't underwrite the investment on it. It's a generic lead-discovery-and-scoring pipeline (ScrapingDog + n8n + Gemini + Sheets) competing in a crowded sales-intelligence market; selling it would commoditise us. The differentiated, defensible thing is the vertical product plus the India-specific scoring rules and compliance posture. For now it's a cost lever — the funnel that makes founder-led GTM efficient — not a second revenue line.

### 18.3 Business model, pricing & unit economics

**Q: Is this expensive — and what's the price?**
Entry price is ₹1,499–3,000/month. The thing it stands in for, a front-desk receptionist, costs ₹15,000–22,000/month and still goes home at night — so it's roughly a tenth of a salary for the repetitive, round-the-clock part. Most owners recover the subscription within weeks from missed-call revenue they used to lose to the lab down the road. There's no big setup fee and no per-message tax from us; one platform serves everyone, which keeps the price this low.
  - *Follow-up:* Is there a long contract or a setup fee? — We're early-stage and closing our first paying labs now, so there's no rigid price sheet to pretend otherwise. The model is a low monthly subscription, no large upfront fee, simple terms in writing, and you can stop if it's not working. We'd rather earn the renewal each month than lock you in.
  - *Follow-up:* What do I pay Meta for WhatsApp itself? — Meta bills per conversation on the official API, around ₹0.115 for a utility message, roughly ₹150–400/month for a typical lab. That's built into how we price — no surprise separate bill you have to manage. The whole cost to serve one lab is a few hundred rupees a month, which is why the price sits where it does.

**Q: Walk me through unit economics — COGS and gross margin, and how confident are you?**
Per-customer COGS for the Front Desk is roughly ₹500–1,300/month: WhatsApp Cloud API utility messages (~₹0.115 each) at ₹150–400, Gemini answers plus storage at ₹50–300, and an amortised VM share at ₹300–600. Against a ₹1,499–3,000 price, that's an 80–90% gross margin. The platform itself runs one VM serving all customers — about ₹4,500–9,000/month fully paid. The confidence caveat: these are web-verified planning estimates against published GCP/Meta/Gemini/ScrapingDog pricing, not yet validated against a real paying-customer message-volume distribution.
  - *Follow-up:* You target the busiest businesses — doesn't a high-volume lab blow up the margin under flat pricing? — A real tension we've already flagged. The resolution is capped bundles with metered overage: most front-desk traffic is free-form replies inside Meta's 24-hour service window (free), and only proactive templates cost ~₹0.115. Voice is where metering matters more (₹2.5–4/minute COGS), so voice will be explicitly metered or capped, never flat.

**Q: What are your CAC, payback, and LTV assumptions?**
Today CAC is dominated by founder time, not cash: the engine produces a qualified, reasoned list at near-zero marginal cost (a Maps search ≈ 5 ScrapingDog credits; ScrapingDog ~$40/month), and acquisition is founder-led calls and visits with no paid ads. So early CAC is essentially selling hours per close. At 80–90% margin each customer contributes roughly ₹1,000–2,500/month, so payback should be months not years. The honest gap: we have no real close-rate, sales-cycle, or churn data yet — LTV is a projection, not a measurement, and producing those numbers is exactly what the next phase is for.
  - *Follow-up:* "Near-zero CAC" — what's the catch? — The catch is that lead cost is not CAC. The data layer is genuinely near-zero, but the real acquisition cost is human time: the SDR/founder hours to call, visit, demo, and close across ~600–900 worked prospects per ~20 customers, plus white-glove onboarding. CAC is dominated by sales labour, which scales with headcount you control rather than auction-priced ad markets — favourable, but it still means sales productivity per rep must be managed.
  - *Follow-up:* Founder-led selling doesn't scale — what's the CAC with a hired team? — That's the key unknown. The bet is that the engine plus a tight playbook (ranked list to phone, three pain questions, log-visit by text) makes a junior rep productive fast, keeping CAC low. We won't know rep-led CAC until the founder closes the first cohort and documents what converts. If rep-led CAC turns out high, the city-by-city replication thesis weakens.
  - *Follow-up:* Where does acquisition cost go up as you scale, not down? — In the crowded markets and the higher verticals. Pune is the largest TAM but most competitive, so deals cost more sales effort. Hospitals and real estate carry 1–3 month committee cycles, so their per-deal cost is high (justified only for flagship logos). The cheap-CAC zone is the beachhead — owner-led, fast-cycle labs and coaching in Tier-2 — which is exactly why the sequencing protects that segment first.

**Q: Where does expansion revenue come from?**
Three vectors: (1) tier upgrades — adding a booking engine and, once validated, a vernacular voice receptionist, pushing deals toward the ₹4,999–19,999/month band our GTM research cites; (2) usage — capped bundles with overage on high-volume accounts; (3) multi-vertical, multi-location land-and-expand within an owner's other businesses. The net-revenue-retention story is land on WhatsApp, expand to voice plus booking — which is also where the ARPU for venture scale comes from.
  - *Follow-up:* How much projected ARPU depends on voice, which isn't validated? — A meaningful chunk of the upside (the move from ~₹2–3k to ~₹5–20k/month) leans on voice and bundles. If voice never validates, our defensible ARPU is the WhatsApp-plus-booking tier, which is lower. We'd model two scenarios: a WhatsApp-only base case and a voice-unlocked upside case — and be candid the upside is contingent on a real-call Marathi/Hindi pilot we haven't run yet.

### 18.4 Market, TAM & competition

**Q: How big is this really — TAM, SAM, and the path to a venture-scale outcome?**
Bottom-up and honest: Aurangabad alone has a deduped serviceable market of roughly 600–900 high-call-volume businesses across our seven verticals (labs ~120–180, coaching ~150–300, hospitals ~80–150, etc.). Using MahaRERA-registered projects as a district-level proxy, replication cities multiply that — Nashik ~2.2×, Nagpur ~1.5×, Pune ~7.3× versus Aurangabad. A four-city Maharashtra cluster is on the order of low tens of thousands of serviceable SMBs across verticals; all of India is far larger. At ₹1,499–3,000/month that's a real but not obviously billion-dollar SAM from labs alone. These are directory-derived ranges, not a validated census — we verify each city with one cheap ScrapingDog pass (~5 credits) before spending.
  - *Follow-up:* Labs in four cities caps out well under a venture return — what's the honest scale story? — Agreed: labs-only in four cities is a lifestyle-to-small-business outcome, not a fund-returner. The scale story rests on three multipliers stacking: vertical (the same engine and bot generalise to coaching, hospitals, hotels, real estate — coaching is the largest pool), geographic (all-India Tier-2), and ARPU (voice plus booking bundles). If only one multiplier works, this is a good small business. You're underwriting the bet that at least two of the three compound.
  - *Follow-up:* What's your realistic SOM in the next 12–18 months? — Our stated near-term target is 20 paying customers in 90 days in Aurangabad at roughly ₹2 lakh MRR, then Nashik as overflow from Month 2. That's a plan, not achieved traction — today we have demos and three scored cities, with first customers being closed founder-led. A credible 12–18 month SOM is low-hundreds of accounts across Aurangabad and Nashik if close rate and retention hold, which is what the next milestones are designed to prove.

**Q: Why now? What changed?**
Three things converged. First, capable cheap vernacular LLMs (Gemini 2.5 Flash at well under a rupee per conversation, plus a free tier) make an Indian-language intent layer economically viable — personalising ~400 leads costs under a dollar. Second, WhatsApp is now the default business channel in India and Meta's official Cloud API made compliant programmatic messaging accessible (~₹0.115/utility message). Third, India's compliance regime (TRAI/TCCCPR, DPDP 2023) is tightening, which punishes grey-market WhatsApp and cold-AI-voice players and rewards a built-compliant inbound product. None of this stack was cheap or compliant enough two to three years ago.

**Q: What's the moat — what stops a competitor, or the customer, from DIY-ing this?**
We'll be straight: there's no single hard moat yet, and that's the biggest risk in the business — the components are commodity, and the differentiator is not the tech but local trust, vernacular quality, and done-for-you service. What we have are compounding soft moats: (1) codified, auditable, per-vertical scoring rules hardened against real live runs; (2) a built-compliant posture (inbound-only, Meta API, public-listing data with recorded provenance) that is a trust signal to owners and a barrier to grey-market players; (3) a data flywheel — every owner visit and outcome logged back sharpens the scoring and the pitch; and (4) switching costs once the bot sits on the customer's own WhatsApp number. The DIY risk for an individual lab owner is low (no time, no Meta API setup, no vernacular-AI know-how), but a funded competitor could replicate v1 — so our real defensibility is speed-to-density and time-to-trust.
  - *Follow-up:* Why don't a bigger lab CRM (CrelioHealth) or a horizontal WhatsApp platform (Wati, Gallabox, AiSensy) just win? — They might, in their tier. Enterprise lab CRMs are priced ₹8,000–25,000/month and gate WhatsApp behind top tiers — built for chains, not the phone-only independent we target, and a downmarket move dilutes their ARPU. Horizontal WhatsApp platforms sell generic broadcast/chatbot tooling, not a vertical-specific, deterministic, India-compliant receptionist with report-status logic. Our defensibility is being the focused, vernacular, Tier-2-priced, vertically-deep option — but we won't pretend a determined incumbent couldn't enter, which is why landing density fast matters.
  - *Follow-up:* What stops a well-funded competitor out-spending you in Pune? — In Pune, little stops raw spend — which is exactly why the plan does not lead with Pune and says compete there on integration depth and local service, not price. The defensible play is to win the Tier-2 markets metros-focused players ignore (independents are densest there: 18/20 in Nashik vs 8/20 in Bangalore), build local case studies and trust, then enter Pune with a sharper wedge and reference customers.
  - *Follow-up:* How durable is the compliance moat, really? — It's a moat against the lazy and the fly-by-night, not a serious competitor — anyone can choose the official Meta API and inbound-only design. Its real value is twofold: it lets us close regulated-vertical owners (diagnostics, insurance) who ask about consent and data, and it keeps us off the ban-and-blowback treadmill grey-market vendors live on. It's table stakes done early and turned into a sales asset, not a permanent barrier.

### 18.5 Technology, architecture & reliability

**Q: Everything runs on one VM — what happens if that box dies?**
Today this is the system's biggest single point of failure and we won't pretend otherwise. One e2-standard-4 in Mumbai runs Hermes, n8n + its Postgres, WAHA, Dokploy/Traefik, plus the adjacent PharmaCare PWA; if it's lost, both engines go dark until rebuilt, with no automatic failover. Genuine mitigations: (1) the stack is fully containerised under Dokploy/Docker, so it's reproducible on a new VM, not a hand-tuned snowflake; (2) the business data does not live on the box — the CRM is a Google Sheet, so a VM loss doesn't destroy leads or deals; (3) the product's customer records are also in Sheets. The honest gap is recovery time: a rebuild today is a manual gcloud/Dokploy exercise, not a one-click restore, and that's exactly the work to do before paying customers are on it.
  - *Follow-up:* How long would a rebuild actually take? — Realistically hours, not minutes, and it's manual. The VM is driven over `gcloud compute ssh`, Dokploy apps are compose files, and several deploy quirks live only in build notes (e.g. PharmaCare deploys by git-archive + scp + rebuild, and DNS A-records must be grey-cloud in Cloudflare or Let's Encrypt HTTP-01 breaks). The fix is infra-as-code provisioning plus a documented runbook so recovery is repeatable by anyone — not yet done.
  - *Follow-up:* What's your actual backup story? — Mixed. The CRM and the product's operational records are Google Sheets, which Google versions and durably stores offsite by design. n8n workflows are exported as JSON and kept in Git, so workflow logic is recoverable. What we do not have is automated, tested, offsite backups of the n8n Postgres (execution history, credentials, session state) or scheduled GCP disk snapshots — so a disk loss would mean rebuilding workflows from JSON but possibly losing execution history and in-flight conversation state. Scheduled snapshots plus an automated Postgres dump to object storage is small, high-priority work.

**Q: How does the lead pipeline handle failures — what stops it silently producing garbage or nothing?**
There's partial resilience and one specific silent-failure mode we've already found. The ScrapingDog node retries up to 3× when ratings come back missing (~20% of calls miss the whole ratings block). Writes are idempotent: the sheet append is `appendOrUpdate` keyed on PlaceID, so a failed-and-retried run is safe and re-runs update in place. The known landmine: when the ScrapingDog key hits its credit limit, the engine treats the errored response as zero results and finishes "success" with an empty sheet. So a 0-lead run usually means credits exhausted, not a bug — but nothing currently alerts on it. That's a monitoring gap we need to close with an explicit credit/empty-result alarm.
  - *Follow-up:* So a quota problem looks like success? — Yes, and it's the most important reliability honesty point. Two dependencies fail quietly: ScrapingDog credit exhaustion yields an empty "success" run, and Gemini's free-tier quota (~100 calls/key/day) returns RESOURCE_EXHAUSTED. We mitigated Gemini two ways — 4-key rotation that self-heals on 429, and moving per-lead pitch generation out of the core engine (now facts-only, pitches generated on-demand separately), so the core run no longer depends on Gemini quota. ScrapingDog we mitigated with a backup key. The real production fix is active monitoring with alerts on empty results and low credit balance, plus paid tiers so quota isn't the failure mode.
  - *Follow-up:* Is there any dead-letter or alerting if a run partially fails? — Not yet formally. n8n records every execution (inbound payload, node outputs, errors), so post-hoc debugging is good, and the completion path sends a Telegram "Done — N leads" ping. But there's no automated alert on a failed or empty run, no dead-letter queue, and no retry-the-whole-run-later mechanism. Acceptable for a founder-operated tool; not for production-grade autonomy, and it's on the list.

**Q: Google Sheets as the CRM and as the product's per-lab database — when does that break?**
Sheets is a deliberate, defensible early choice with a known ceiling. A Sheet's hard limit is 10M cells (Google is doubling to 20M in a 2026 beta), with practical slowdown past ~100,000 rows. At ~1,500 leads/month with ~20 columns we won't strain it for years on lead volume. The benefits are real: the founder owns and edits the data, no lock-in, zero cost, and a lab owner already understands a spreadsheet. The planned migration trigger is explicit — move to Postgres on the same VM (or n8n data tables) when the master sheet exceeds a few thousand active rows or we need a kanban pipeline, status automations, and concurrent multi-user editing.
  - *Follow-up:* What about concurrency — the engine writing while the founder edits? — This is the more realistic near-term limit than cell count. The design reduces clobber risk: the engine and re-scorer emit only computed columns, matched by PlaceID, so machine writes never overwrite the founder's manual notes. But Sheets is not transactional and has no row-level locking, so concurrent writes plus a human editing the same tab can race. Fine for one founder running one city at a time; once multiple operators or many concurrent customer writes are involved, that's precisely when the DB migration is warranted.
  - *Follow-up:* Does the Front Desk's Sheets-as-database scale per customer? — For a single lab's daily volume, yes — bookings and report-status rows are low-cardinality, and the production model is a daily tab merged monthly to a master, keeping each tab small. The scaling concern is multi-tenancy: every customer on its own sheet/tab, every customer's OAuth and config, multiplied across tenants. Real multi-tenancy is where Sheets-as-DB starts to creak, and it's a known production-path item.

**Q: n8n does orchestration, AI glue, the customer bot, and crons on one instance — what's the throughput ceiling?**
There are 31 workflows (21 active) on one self-hosted n8n instance backed by one Postgres, on a 4 vCPU/16 GB box at ~40% disk and light load. n8n self-hosted is single-instance by default — not queue-moded here — so concurrency is bounded by the box and n8n's default execution model. At founder-and-demo volume this is comfortable. The throughput risks at scale are concrete: the customer bot is synchronous per inbound message and calls Gemini in-line; reminder crons run every 15 minutes reading whole sheets; and all of this contends for CPU with the lead engine and PharmaCare. The scale path is well-trodden (n8n queue mode with Redis and multiple workers, lift onto a bigger VM, split services), but we haven't needed it yet.
  - *Follow-up:* Is there session-state that doesn't survive a restart or scale-out? — Yes, and it matters. The bot's conversation memory (per-phone rolling history, booking sub-states, pendingCancel) lives in n8n workflow static-data, in-process. So a container restart can drop in-flight context, and you can't naively run multiple workers because they wouldn't share static-data. There's also a documented minor static-data race when messages arrive <4 seconds apart (seen only in automated tests). Moving session state to Redis/Postgres is a prerequisite for both horizontal scaling and clean restarts — a known, bounded piece of work.
  - *Follow-up:* How do you isolate the revenue-bearing customer product from the internal lead engine on the same box? — Today they're logically separated (distinct workflows, prefixes, sheets) but share the same n8n instance, Postgres, CPU, and memory — so a runaway sweep could in principle starve the bot. We've been disciplined about not touching each other's workflows, but that's process, not hard resource isolation. Production-grade separation means splitting the revenue product onto its own instance/VM with its own resources and uptime budget.

**Q: What's your vendor lock-in and portability exposure?**
Portability is genuinely one of the stronger parts of the story. n8n is open-source and self-hosted (no per-run platform tax, workflows exported as JSON); the data lives in Google Sheets the founder can export anytime; the whole stack is containerised under Dokploy/Docker, so it lifts to a bigger VM or another cloud without a rewrite. The realistic lock-in points are: Google (Sheets as datastore, Gemini as model — both swappable behind a Code-node interface); ScrapingDog as the single discovery source (mitigated by the identified Outscraper alternative); and, for the product, Meta's WhatsApp Cloud API (a deliberate compliance choice, not accidental lock-in). None are one-way doors.
  - *Follow-up:* Is Hermes a lock-in or a liability? — It's the most replaceable component. Hermes is `nousresearch/hermes-agent` with Gemini as the brain, steered by a single SOUL.md instruction file loaded fresh each message; every capability it exposes ultimately just calls an n8n webhook. If Hermes were removed you'd lose the natural-language Telegram front door but keep 100% of the actual machinery. Low lock-in, by design.

**Q: What monitoring and observability do you have?**
This is one of the weakest areas, named plainly. What exists: n8n stores full execution history queryable via API (good post-hoc debugging), the lead engine sends a Telegram completion ping, and Dokploy gives basic container status. What does not exist: proactive alerting on failed/empty runs, uptime monitoring on public endpoints, ScrapingDog-credit and Gemini-quota alarms, log aggregation, or any on-call signal. The consequence is the silent-failure modes above surface only when the founder notices. Going production-grade here is a finite checklist: uptime checks, n8n execution error-alerts, credit/quota threshold alarms, and basic log/metric collection.
  - *Follow-up:* If the bot stopped answering at 2am, how would you find out today? — Honestly, maybe not until morning. There's no synthetic monitoring on the inbound path and no alert if WAHA disconnects or the OAuth token lapses. The bot degrades gracefully rather than hard-failing (Gemini key rotation, plus deterministic keyword routing if Gemini is fully down), but a transport-level outage has no automated alarm. A WhatsApp-uptime synthetic check and a credential-health check are top of the hardening list and cheap to add.

**Q: That weekly Google OAuth expiry — explain it.**
It's the most acute recurring reliability bug we have, and we want it on the record. The shared Google Sheets OAuth credential is in Google's OAuth "Testing" mode, where refresh tokens expire roughly every 7 days. When it lapses, every workflow that reads or writes Sheets fails — the CRM and the customer-facing Front Desk (bookings, report-status writes). The current remedy is manual: reconnect the credential in n8n. The durable fix is known and small — publish the OAuth consent screen to "In production" so tokens stop expiring — but it hasn't been done yet. Manageable as a weekly chore for a demo; unacceptable for paying customers and a must-fix before go-live.
  - *Follow-up:* Has it actually caused an outage? — It's a documented, recurring known-issue with a documented workaround in our runbook. We won't claim a specific customer-facing outage because there are no paying customers on it yet — which is exactly why now is the time to fix it. We found it in operation, not in theory.

**Q: What does it take to get from a hobby-grade box to production-grade — the punch list?**
Grouped by priority. Reliability/DR: scheduled GCP disk snapshots plus automated offsite Postgres dumps; infra-as-code provisioning and a tested recovery runbook; move bot session-state out of in-process static-data into Redis/Postgres. Security: rotate every plaintext credential (DB-first), move secrets to an encrypted store/vault, add auth/HMAC on all n8n webhooks (and Meta signature verification on production inbound). Availability: publish the Google OAuth consent screen; enable billing on Gemini and ScrapingDog so quota/credits stop being silent failure modes; isolate the revenue product onto its own instance. Observability: uptime/synthetic checks, alerting on failed/empty runs, credit/quota alarms. Product: complete the WAHA → Meta Cloud API cutover. None are research problems — a few weeks of disciplined engineering, doable incrementally because the stack is containerised and the data is in Sheets.
  - *Follow-up:* Top three first? — (1) Publish the Google OAuth consent screen — small, removes a weekly outage that would hit bookings. (2) Rotate plaintext credentials and move secrets to an encrypted store — highest-blast-radius security gap and cheap. (3) Add basic monitoring/alerting on uptime, empty/failed runs, and low quota — it converts our two known silent-failure modes into things we catch automatically. Backups/DR and session-state externalisation come right after.
  - *Follow-up:* Would you keep the single-VM design or replatform? — Keep the philosophy (one platform serving all customers is what gives the 80–90% margins) but evolve the topology: split the revenue product onto its own instance, add backups plus a warm-recovery path, and move n8n to queue mode with externalised state when concurrency demands. Containerisation means each is a migration, not a rebuild. We're not architecturally cornered — we're under-hardened, which is a more fixable problem.

### 18.6 AI, data quality & accuracy

**Q: Does the AI hallucinate — could it invent businesses, owners, prices, or facts?**
The design deliberately keeps the LLM out of any path where it could fabricate a fact. In the lead engine, business records come from ScrapingDog's Google-Maps API, never a model; the enrich step is facts-only with no per-lead LLM call, and scores and reasons are computed in code. The only place an LLM writes prose is the on-demand Pitch workflow and the bot's reply text, and even there the prompt forbids invented facts, uses low temperature, computes ROI deterministically, and stores raw output for audit. The standing rule we wrote down is blunt: treat any LLM text as Tier-3 opinion, never as fact in the master data. A hallucination becomes a badly-phrased sentence, not a wrong number or a fake lead.
  - *Follow-up:* The owner name and "why they'll buy" read like model output — what if they're wrong on a call? — The owner name is extracted from the listing text (a "Dr." pattern), not invented, and is blank where absent rather than guessed. WhyWillTheyBuy is templated from computed facts (e.g. "416 reviews (busy); 5★; phone-only, no website"), restating measured signals. The residual risk is a stale name on the listing itself — a Maps data-quality issue — and the founder verifies the human on the first call, since outreach is human-led 1:1, so a wrong name is caught in 30 seconds, not blasted at scale.

**Q: How accurate is the scraped Google-Maps data — phones, ratings, reviews, addresses?**
The data is exactly as good as the business's own public Google listing, no better. ScrapingDog returns name, phone, website, rating, review count, address, map URL, and place-id, with up to 3 retries if ratings are missing — but it can't correct an owner's wrong number or a months-old rating. We treat Maps as the source of truth for "does this business exist and roughly how busy is it", which is what the scoring needs, not as a verified contact database. Honest framing: a place-id, name, and review count are high-confidence; a phone number is medium-confidence until a human dials it; a scraped owner name is the weakest field.
  - *Follow-up:* What's your actual error rate on phone numbers? — We don't have a measured rate; that would require dialling a sample and logging hits, which we haven't done as a formal study — a genuine gap. The mitigation is structural: every wrong number surfaces on the first human-dialled call and is logged back via Log-Visit, so the CRM self-corrects. Quantifying a baseline hit-rate on a 100-lead sample is a validation step our own GTM notes flag before scaling spend.
  - *Follow-up:* What about businesses that have closed or moved? — A closed-but-still-listed business is the classic stale-data failure and the scrape will include it; we don't yet pull Google's "permanently closed" flag as a hard filter, so a dead lab can appear. In practice it's caught at call time and marked lost. Reading and honouring the closed/temporarily-closed status during normalise is a concrete, small improvement, not yet shipped.

**Q: How does the WhatsApp bot avoid quoting a wrong price, a non-existent slot, or a false report status?**
By a deliberate, architecturally-enforced split: the LLM (Gemini) reads intent and phrases the reply, but deterministic code does every fact-bearing action — generating the token/VOC-id, writing the sheet row, running the patient lookup, executing the booking. So the model decides "this person wants a thyroid test" but it cannot invent a price, token, or slot, because those come from code reading the lab's config and booking system. Prices and timings are answered strictly from the lab config with a hard "never invent prices" rule; an unpublished individual test's price returns "confirm at the desk/phone" rather than a fabricated figure. Our test suite has explicit cases for this (a thyroid-rate ask must not produce a fabricated rupee figure; "do you do X-ray?" must not falsely say yes for a pathology-only lab).
  - *Follow-up:* Gemini still writes the reply sentence — couldn't it mis-state a real config price? — It could mis-state a number that exists in the config — the residual risk of letting a model phrase anything. Two things bound it: the config prices are the only numbers in scope, and the higher-stakes facts (token, report status, VOC reference) are inserted by code, not the model. For the safety-critical path we'd want the price string itself code-inserted rather than model-rephrased — a tightening we can make. Today the worst realistic failure is a mis-typed package price caught by the patient or desk, not a fabricated medical result.
  - *Follow-up:* What about report status — could it say a report is ready when it isn't? — The bot doesn't decide a report is ready; your staff do. It only marks a report ready and notifies the patient after your own people post that patient's number in the lab WhatsApp group. The trigger is a human action on your side; the bot just removes the manual typing and delay. It can't get ahead of your actual lab process.

**Q: Could the bot book the wrong thing — mis-read intent, name, or test in Hindi/Marathi/Hinglish?**
Intent and field extraction is where the model genuinely operates, so mis-extraction is possible — and it's the area we've tested most heavily because that's where past bugs lived. The test catalogue covers exactly these failure modes: "report ready?" followed by "but I never gave a sample" must not create a phantom booking; a mid-flow switch from walk-in to home collection must carry the name/test and book exactly once; the lab's own address must never auto-fill as the patient's. The bot keeps a rolling ~12-turn memory to interpret "11 am" or "actually home collection" in context. Even when intent is read, consequences are bounded: a booking is a token and a row a human can see and fix, and the patient sees the confirmation immediately and can correct it.
  - *Follow-up:* How do you know the Marathi/Hindi understanding is good and not just demo-good? — We don't have a measured accuracy figure across vernacular inputs and won't claim one. What we have is a structured test catalogue exercised in Hinglish ("mujhe thyroid test karwana hai", "mera test cancel kar do" → "haan") and a keyword-routing fallback if comprehension fails. The bigger honesty is on the roadmap: voice is explicitly gated on validating Marathi/Hindi quality precisely because we don't consider vernacular quality proven yet. Text is more forgiving than voice, but a rigorous multilingual accuracy benchmark is a to-do, not a done.

**Q: How dependent is the system on Google Gemini — cost, rate limits, outages, model changes?**
Gemini is the brain of both Hermes and the bot's intent layer, so the dependency is real but bounded. Cost is low: 2.5 Flash is cheap per token with a generous free tier, projected under ₹100/month fully paid. Rate limits are the live constraint — free tier is ~100 calls/key/day — which is why the bot rotates over multiple keys (~400/day) and self-heals on a 429. For outages, the bot falls back to deterministic keyword routing so it never hard-fails. Model changes are the least-managed risk: if Google alters 2.5 Flash's behaviour, prompts could need re-tuning, and we don't have an automated regression suite against the live model (we have a written test catalogue to re-run manually).
  - *Follow-up:* Rotating free-tier keys sounds like a grey-area workaround — is that production-grade? — It's a demo-grade reliability hack and we'll call it that. For real volume the production path is simply to enable billing on one Gemini key, which removes the per-key daily cap; Flash is cheap enough that paid usage still lands under ₹100/month at our scale. Key-rotation is a free-tier convenience for demos, not the production architecture.
  - *Follow-up:* Could you swap the model out to reduce single-vendor risk? — Architecturally yes — the LLM is called over an HTTP API and only does NLU/phrasing, returning strict JSON, so it's a relatively contained swap to another provider that supports constrained JSON. Not zero work: prompts are tuned to Gemini and the Marathi/Hindi/Hinglish quality would need re-benchmarking, especially for voice. We haven't run a portability test, so model-swap is feasible-but-unproven, not press-a-button.

**Q: On re-runs, do scores and notes get overwritten with stale or wrong values?**
Re-runs are safe by construction. Every write is `appendOrUpdate` keyed on PlaceID, so re-scraping a city updates rows in place rather than duplicating. The engine and re-scorer emit only computed columns, so the founder's manual fields (ContactStatus, BuyingEvidence, ObservedReality, WhyWillTheyNotBuy) are never touched — human ground-truth always survives a re-scrape. The flip side: scraped fields (rating, reviews, phone) refresh to whatever Google currently shows, so a re-run reduces staleness but only pulls ~20 results per query and isn't a scheduled full-market refresh.
  - *Follow-up:* So a re-run could overwrite a good phone number with a now-wrong listing number? — On the scraped Phone column, yes — it reflects the current listing, for better or worse. But the founder's verified contact actions live in the human-owned columns that re-runs don't touch, and a bad number confirmed on a call is logged there. The computed Phone column tracks Google; the verified reality tracks the human's notes; the two coexist by design rather than the machine clobbering the human.

**Q: Where would you most expect this to be wrong in the field, and what's the single most important thing you haven't measured?**
Five places, ranked by impact: (1) phone numbers and closed businesses (stale data, no measured hit-rate yet); (2) chains slipping the reactive blocklist; (3) scoring not predicting actual buying (validated against an expert, not revenue); (4) vernacular intent mis-reads (heavily tested, no formal benchmark); (5) single-VM concentration and finite free credit. None are hidden in the architecture — the LLM is kept out of fact-bearing paths, writes are deterministic, human notes survive re-runs, and the human is the final reviewer today. The single most important unmeasured thing is win-rate by score tier — does a "Call-This-Week" lead actually close more than a "Nurture" one? Until that loop closes, the scoring is well-engineered and expert-validated but commercially unproven, and we'd rather say that than overclaim.

### 18.7 Security, privacy & compliance

**Q: Walk me through your security posture — webhook auth, secrets, what's exposed.**
We'll be direct because we've audited ourselves. The honest baseline: secrets are currently in plaintext — API keys, tokens, and credentials sit in build notes and .env files on the VM — and we have a pending action to rotate every one, prioritised by blast radius (DB passwords first, then Gemini/Telegram/n8n API key/Google OAuth/WAHA/VAPID). On exposure: services are behind Traefik with automatic Let's Encrypt HTTPS, and sensitive dashboards (Dokploy, Hermes, n8n admin) are reached via SSH tunnel rather than public ports — that part is good. The weak spot is webhook authentication: the n8n `/webhook/gvm-*` endpoints are obscure URLs but obscurity is not authentication, and they need a shared-secret/HMAC check (and Meta's signed-payload verification on the production inbound path) before they're truly production-safe.
  - *Follow-up:* Could someone trigger a lead run or poison the CRM via the webhooks today? — It's a real gap to close. An attacker who learned a path could trigger runs (burning ScrapingDog credits) or hit the patch/log-visit/promote endpoints that write to the CRM. Partial mitigations: writes are PlaceID-keyed upserts touching only computed columns (limiting blast radius), and the highest-risk dashboards aren't publicly exposed. Adding a required auth header or HMAC signature on every webhook is straightforward and should be done before customer-facing scale.
  - *Follow-up:* You mentioned an admin123 password and 365-day tokens — is that this system? — That's the PharmaCare PWA on the same VM, not the GVM engine or the Front Desk, but it's relevant because it shares the box and reveals our security maturity. The audit found real issues (admin123 bootstrap password, no login rate limiting, 365-day refresh tokens in localStorage, cross-tenant service-worker cache leakage, non-idempotent background-sync POSTs), all documented with prioritised fixes. We surface it rather than hide it: capable architecture, hobby-grade hardening that needs a deliberate pass before production.
  - *Follow-up:* Can you rotate credentials without downtime? — Partially. Gemini keys are an array we redeploy and self-heal on quota (low risk); n8n is updated via REST API. The painful ones are stateful: rotating JWT secrets logs everyone out, DB-password rotation touches live data, and the Google OAuth credential is the worst — in "Testing" mode it expires roughly weekly and breaks CRM and bot writes. The durable fix (publishing the OAuth consent screen) is known but not yet done, so rotation today is manual and partly disruptive.

**Q: Is scraping business listings off Google Maps actually legal in India?**
GVM doesn't scrape Google directly; it buys Maps listing data through a third-party API (ScrapingDog), pulling business identity the business itself published (name, public phone, website, rating, address). The honest legal posture is that this reduces risk rather than eliminating it. Under DPDP Act 2023, Section 3(c)(ii) exempts personal data the principal themselves made public — covering a lab listing its own number on its own profile — but that exemption is narrow and contested, and there's no GDPR-style "legitimate interest" ground in India. We tag every record with provenance (source, URL, capture time) so each lead's self-published status is defensible per row.
  - *Follow-up:* Doesn't a third-party scraper just move the problem to the vendor? — Partly, and we're honest about that. ScrapingDog's and Google's terms are between us and the vendor, and MeitY's August-2024 position keeps scraping within the IT Act's ambit regardless of who fetches. We treat ScrapingDog as reducing the operational footprint, not as a legal shield. The mitigations that matter are downstream: we only retain self-published business contact data, record provenance per row, and drop or flag any number not self-published by the business.
  - *Follow-up:* Worst case if a business complains? — A DPDP grievance or a cease-and-contact request once enforcement is fully live (phased through ~mid-2027). We mitigate by treating the sheet as a research/routing layer, never a blast list; honouring opt-out and erasure on request across all channels; and naming a grievance contact in a published privacy notice. The GTM plan budgets a one-time India data-protection counsel review before scaling outreach.

**Q: A sole proprietor's mobile is personal data even in a business context — how do you handle the owner numbers you extract?**
This is the sharpest edge in the lead engine and we acknowledge it directly. The scoring brain extracts an owner name and normalises the listed phone into a WhatsApp-ready number, and our GTM analysis explicitly flags that a sole proprietor's personal number is personal data with no B2B carve-out under DPDP. Our defence is the self-published exemption: we only retain a number the business itself published on its own listing/site, with the provenance URL proving it; where a number was harvested from a third-party directory the owner didn't publish, the documented rule is to drop or flag it.
  - *Follow-up:* Can you always tell self-published from directory-pulled? — Not perfectly, which is why this is a managed risk, not a solved one. A number on a business-managed verified Google profile is a strong self-published signal but not airtight. We store the provenance URL so the basis is auditable, and for high-value enterprise targets the policy is to stop relying on the exemption and obtain explicit consent before contact.
  - *Follow-up:* Does the engine auto-contact these numbers? — No, deliberately. The engine produces a research sheet; it does not auto-dial or auto-message. First touch is a human, manually-dialled 1:1 call within the 9am–9pm window, plus email/LinkedIn. Automated WhatsApp only fires after explicit opt-in. The pipeline never blasts the scraped list.

**Q: TRAI/TCCCPR restricts commercial calling and targets AI telemarketing — how is your outbound compliant?**
Our outbound sales calls are genuine human 1:1 consultative calls to the business's own published number, within the 9am–9pm IST window, with no autodialer and no AI-voice cold campaigns. The TCCCPR regime requires promotional/telemarketing calls from the 140-series via a DLT-registered telemarketer, DND-scrubbed, with no B2B exemption — and automated AI-voice cold-calling of scraped lists would be illegal without that registration. We avoid the entire outbound DLT/DND regime by keeping cold outreach human-led and routing numbers to a human call list, not an auto-dialer.
  - *Follow-up:* Five spam complaints in ten days can blacklist a number for two years — what stops your callers tripping that? — That tightened threshold is exactly why outreach stays human and low-volume per caller, with a 9am–9pm calling-window guard and opt-out honoured instantly across channels. We also lead with WhatsApp and email, which are cheaper and more opt-in-friendly. The structural protection is that the volume engine is inbound product adoption, not outbound dialling, so we're not running the kind of high-frequency campaign that generates complaints.
  - *Follow-up:* Is the AI voice receptionist itself a TRAI problem? — Only if used for outbound cold calls, which we won't do. The voice receptionist is sold strictly as the customer's inbound product, answering their own incoming calls, which sidesteps the outbound 140-series/DLT/DND regime entirely. Voice is also gated on a separate latency/quality validation and not yet promised.

**Q: Does WhatsApp Business or AI voice require TRAI DLT registration?**
The answer splits by channel. WhatsApp Business API does not require DLT registration — DLT is an SMS-and-voice regime, and "no DLT for WhatsApp" is a well-established point. Outbound commercial voice is different: it falls under the DLT/140-series/DND regime, which is exactly why we keep voice as an inbound-only product the customer uses on their own line, not an outbound tool we operate.
  - *Follow-up:* If you ever added outbound voice, what would you need? — DLT registration as a principal entity (and/or via a registered telemarketer), the 140-series for promotional traffic, DND scrubbing, template registration, and the 9am–9pm window — none of which we have today. Because it's a heavy regime, the documented decision is not to build outbound auto-dialling; outbound calls stay human within the window, and registration would only be a deliberate, counsel-led step.

**Q: The product handles patient health data — names, phones, test types, home-collection addresses, report status. How is that handled under DPDP, and who's the controller?**
This is the most sensitive data in the system. The Front Desk stores patient name, phone, the test/package requested, home-collection address, and report status in a Google Sheet the lab owns, with a consent_flag set to Y when the patient messages in (inbound = opt-in for that chat, plus a one-line consent at booking). Test type ordered is effectively health information and warrants careful treatment. DPDP 2023 doesn't create a separate "sensitive personal data" tier like GDPR, but this is unquestionably personal data; the controls are minimal PII, consent to store, an access-controlled sheet, and deletion on request, with STOP honoured instantly. On roles: the lab is the primary data fiduciary (its own customers, its own number, its own Sheet) and GVM acts as a data processor — an allocation that must be papered in a customer agreement as part of the pre-scale legal sign-off, not yet fully executed.
  - *Follow-up:* A Google Sheet for health data sounds under-secured — is it adequate? — It's access-controlled and adequate for a demo and early pilots, but we won't overclaim it as a hardened health-data store. We capture minimal data, restrict sheet access, capture consent, and delete on request. For scale, the open items are formal access controls, a documented retention/deletion policy, and counsel sign-off on retention. We don't store clinical report contents in the demo — report PDFs are only delivered in production via Meta Cloud API; the demo is status-only.
  - *Follow-up:* Is there a data-residency requirement, and where does the data live? — DPDP 2023 imposes no blanket localisation; it allows transfers except to government-blacklisted countries. Compute runs on a Google Cloud VM in Mumbai (asia-south1), so the processing layer is in-India. The honest caveat: the CRM/patient records sit in Google Sheets and Gemini is called for intent, so some data touches Google's broader infrastructure — to be reflected in the privacy notice and confirmed with counsel for any sector with stricter residency than baseline DPDP.

**Q: What consent do you capture on the product side, and is "they messaged us" enough under DPDP?**
Consent is captured as a one-line consent at booking plus the principle that an inbound message is opt-in for that chat, with a consent_flag stored per record. WhatsApp's own rules reinforce this: free-form replies only within 24 hours of the patient's last message; outside that, only pre-approved UTILITY templates, with a STOP opt-out line honoured instantly. For the immediate service interaction, "the patient initiated contact for a specific purpose" is a reasonable basis — but DPDP expects consent to be specific and purpose-limited, so we should not over-rely on inbound-equals-consent for anything beyond servicing that request. We don't repurpose the data for marketing, and getting counsel to sign off the exact consent-notice wording is a named pre-scale task.
  - *Follow-up:* How does opt-out propagate and can a patient be deleted? — STOP is honoured instantly with suppression across all channels and an OPTOUT/STOP terminal status, and the privacy commitment includes a one-click opt-out/erasure path consistent with the DPDP right to erasure. The practical gap to close at scale is making erasure across the Sheet plus any cached state fully systematic and logged — part of the retention-policy work counsel needs to review.

**Q: What's your liability if the bot gives a patient wrong information?**
We engineered the product specifically to shrink this exposure: it's deterministic by design. Gemini only reads intent and phrases replies; deterministic code does the booking, token generation, lookups, and writes — so the bot cannot invent a price, test, or slot, and the most dangerous failure mode (a hallucinated clinical or commercial fact) is structurally prevented. Liability ultimately sits primarily with the lab as operator and data fiduciary, with appropriate limitation-of-liability and non-medical-advice disclaimer language in the customer contract — which needs to be papered properly. The bot never gives clinical interpretation and routes anything outside scope to a human.
  - *Follow-up:* The report-ready flow flips a row when staff post a number — what if staff post the wrong number? — A real residual risk, but it's a human-input failure, not a model failure. The lookup matches by last-10-digits, so a mistyped number could notify the wrong patient. Mitigations: the bot only proceeds on a positive lookup match (else "no patient found"); the message is a status notification, not the clinical report in the demo; the production PDF goes through the audited Cloud API path; and the contract makes the lab responsible for what its staff post.

**Q: You market "built compliant" as a feature — isn't that overclaiming given the open items?**
A fair challenge, and the honest answer is that the posture is real but not yet fully papered. Genuinely true today: outreach is human-led 1:1 and inbound-first, production WhatsApp uses Meta's official API, the product is inbound (no cold AI voice), prospect data is from public listings with recorded provenance, and the bot is deterministic so it can't invent facts. Not yet done: a counsel-signed consent notice and retention policy, executed processor agreements with customers, and full operationalisation of erasure and audit. So the accurate claim is "compliant by design, with the legal paperwork as a named pre-scale task" — not "fully certified compliant."
  - *Follow-up:* What's the single biggest unmitigated legal risk you'd flag in diligence? — The DPDP exposure on the lead engine's handling of owner personal numbers: the self-published exemption is narrow and contested, enforcement is still phasing in through ~2027, and there's no legitimate-interest fallback in India. We mitigate with provenance tagging, no-blast discipline, and consent for high-value targets, but it's the item most dependent on counsel sign-off and on enforcement direction we don't control. The second is patient health-data handling at scale, needing the processor agreements and retention policy executed before we hold real records in volume.

### 18.8 The WhatsApp gateway: WAHA → Meta Cloud API

**Q: Your product runs on WAHA, an unofficial gateway — isn't that a ban risk and a credibility problem?**
WAHA is demo-only — built and tested end-to-end, then paused, never intended for production. Our documented risk figure is that proactive cold messaging via unofficial gateways runs roughly a 15–30% ban over twelve months (versus under 2% inbound-only), with bans usually permanent for that number; the free NOWEB engine also can't send PDFs. For any real customer traffic the path is Meta's official WhatsApp Cloud API — sanctioned, ban-safe, native PDF delivery, billed per conversation (~₹0.115/utility message, free within the 24-hour window). The ban risk applies to the demo, not the production product, which is why we never demo as if WAHA is the shipping platform.
  - *Follow-up:* Why not just move to the official API now and skip WAHA? — Because Meta onboarding is per-number, owner-login-gated, and only worth doing once a customer signs — it needs the customer to free their number from the consumer app, complete Business Verification (2–10 days, capped at 250 conversations/24h until cleared), submit UTILITY templates, and generate a permanent system-user token. WAHA lets a prospect see the full Front Desk live, today, with zero onboarding friction. The cutover is a documented ~30–60-minute-per-lab transport swap — a deliberate sequencing choice, not forgotten debt.
  - *Follow-up:* How big is the cutover and what could go wrong? — It's a clean transport swap, not a rewrite: the n8n send path changes to POST `graph.facebook.com/.../messages` (templates for proactive sends, free-form replies inside the 24-hour window), plus a new inbound webhook for Meta's verify handshake and payload shape. Cal.com, the sheets, and the cron logic stay identical. The real risks are timeline (verification delays) and getting templates approved in the right UTILITY category (Meta can re-categorise a borderline one as MARKETING, costing more) — not architectural risk. We start verification early in the close so it doesn't block a sale.
  - *Follow-up:* If a WAHA ban is permanent, what happens to a demo lab? — The demo runs on our own test/demo number, not a customer's live business number, so a ban is contained to us. The hard rule: no real customer ever goes live on WAHA; production cutover to Meta Cloud API happens on a new dedicated number before any client volume.

**Q: Cloud API can't read group chats — doesn't that break your highest-ROI report-intake flow in production?**
It changes it, and the docs are explicit about the split. The highest-ROI flow — staff posting a patient's number in a WhatsApp group to mark a report ready — relies on reading a group, which Cloud API can't do. So in production the intake side moves to WAHA (strictly inbound/test) or a Google Form for the internal staff group, while the patient-facing sending (including the actual PDF) goes through Cloud API. It's a deliberate two-channel design rather than one number doing everything — documented but not yet run with a live lab.

### 18.9 Customer objections

**Q: Why do I need this — I already have reception staff?**
Your staff are great when they're free and at the desk, but they handle one call at a time, go home at night, and take breaks. In a busy lab, 15–30% of calls go unanswered when the line is busy or after hours, and an unanswered call is usually a patient who dials the next lab. GVM doesn't replace your people for the human work — it catches what they physically can't: the second and third caller, the 9pm "is my report ready", the Sunday "what's the price of a CBC". Think of it as a tireless second receptionist on WhatsApp that never says "please hold".
  - *Follow-up:* But most of my patients just walk in — do they call that much? — The busiest independent labs we score have hundreds to thousands of Google reviews, a direct proxy for inbound volume, and the most common questions (report-status, price) are exactly the repetitive ones that tie up your desk. Even if walk-ins are your bulk, every report-status call the bot handles is one your receptionist didn't have to interrupt a patient at the counter to take. We'd rather show you on your own number for two weeks than argue the point.

**Q: What if it gives a patient a wrong answer — a wrong price or a false report status?**
This is the part we're most careful about and it's the core design decision: the AI is only allowed to understand what the patient is asking — it cannot make up the answer. Prices, slots, and report status come from your real data through fixed code, so the bot literally cannot invent a price or a slot; report status is pulled by matching the patient's phone number, not guessed. If something isn't in the data, it says so and routes to your staff rather than bluffing. That deterministic design exists specifically to protect your reputation.
  - *Follow-up:* How do I know the prices it quotes are mine and stay current? — We load your actual price list and timings at setup, and the bot reads only from that. When you change a price we update the source and every answer reflects it — there's no AI "remembering" an old price. You can test it yourself before it ever talks to a real patient.

**Q: Is my patients' data safe — and will you sell or share it?**
We take it seriously and it's built into how we operate. Production runs on Meta's official WhatsApp API, not a grey-market gateway; we operate with India's DPDP Act in mind; and the design is minimal — the bot looks a patient up by phone to fetch their own report status, it isn't broadcasting health data around. We do not sell or share your patients' data — it exists to serve your lab, full stop. The only data we gather more broadly is public business listing information about prospects (from public Google listings, with the source recorded), never from your patients. As a young company we're still formalising every policy, so we'd rather walk you through exactly what's stored and where than hand you a glossy claim.
  - *Follow-up:* Where is it stored — some foreign server? — The platform runs on a Google Cloud machine in the Mumbai region, so the compute sits in India. We'll be straight that today everything runs on a single cloud machine — efficient and portable but a concentration point we're actively planning to harden as we take on more labs. We're happy to walk you or your IT person through the exact data flow.

**Q: How long does setup take, and what do you need from me?**
The WhatsApp front desk is deliberately the fast, low-risk option — no hardware, no install at your counter, no change to how staff already use WhatsApp. The real setup is on our side: loading your price list, timings, and FAQs, and getting your number onto the official WhatsApp API (the Meta verification step takes a few days but is a known, documented process). We need your price list, timings and services, the common questions patients ask, and access to set up your business WhatsApp number — about an hour or two of your time, then you check the demo before a single real patient talks to it.

**Q: Can I see it work before I commit, and have you got other labs using it?**
We'll be completely honest: we're early. The WhatsApp front desk is built and we can demo it end-to-end on a real number right now — booking, price/FAQ, report status, after-hours capture, cancellation. We've scored three cities of labs and we're closing our first paying labs now; you'd be among the first, which is exactly why the price is a wedge and the attention is personal. We won't invent customer names or revenue to impress you. What we will do is put a working demo on your own prices and timings in front of you — you and your receptionist can hammer it in Marathi, Hindi, and English before you pay anything.
  - *Follow-up:* Being your first customer makes me nervous — what's in it for me? — Being early should come with an advantage: founder-level attention, pricing at the low end, and direct influence on how the product fits a lab like yours. The risk is mitigated by it being overflow-catching rather than your whole front desk (so it can't break your operation) and by there being no big upfront cost. If it doesn't earn its keep in a pilot, you walk.

**Q: What if I want to stop — do I lose my number or my data?**
You can stop, and you keep what matters. There's no large upfront investment to strand you, and the design avoids lock-in: your WhatsApp number belongs to your lab, not us, and your patient records live in your own Google Sheet, not a proprietary box. We'd hand back control cleanly. Since we start with a low monthly subscription rather than a long contract, leaving is straightforward — we'd honestly rather lose a customer who isn't getting value than trap one, because at this stage word of mouth in your city is everything. (Exact data-export and offboarding mechanics will be spelled out in the agreement so it's fair and documented, not figured out under pressure.)

**Q: Who fixes it when it breaks, and what's your uptime?**
Right now, the founder, directly — sales and support are founder-led, so you get the person who built it, not a ticket queue. We won't quote a five-nines SLA we can't back up. The honest picture: it's a single Mumbai cloud VM with light load and headroom, containerised so it can be moved or recovered without a rewrite, and removing that single point of failure with redundancy is a top priority as we add labs. Crucially, because your front desk is overflow-catching, an outage degrades gracefully back to your staff and existing phone rather than stopping your lab. The bot itself runs 24×7 to capture after-hours messages; a formal round-the-clock support desk is part of the scale plan, not something that exists yet.

**Q: How did you even find me — I never gave you my number?**
The same way any patient finds you: your public Google Maps listing. Part of what we built reads public business listings — name, rating, reviews, whether you have a website or WhatsApp — and flags independent labs that look like a good fit, then the founder calls them personally. We record where every piece came from, and it's all information you've already published publicly. There's no purchased list and no data from anywhere private, and the outreach is a person calling you, not an AI spam machine.
  - *Follow-up:* Is that legal? — We use public listing data — the same data anyone sees searching for your lab — and record its source per the DPDP Act's approach to publicly available business information. The outreach is deliberately human-led 1:1 calls, not cold automated voice, because cold AI-voice broadcasting is restricted in India and we won't do it. The whole approach is built to sit inside the rules — that's a point of pride, not a loophole.

**Q: What happens when a patient asks something the bot has never seen?**
It doesn't bluff. The bot understands intent, but when something falls outside what it can answer from your real data, it captures the message, replies politely, and flags it for your staff to follow up — after hours, it sends a morning nudge so nobody's dropped. The deterministic design means it would rather hand off to a human than make something up, so an odd question becomes a captured lead for your staff, not a wrong answer to your patient. Over time, the common ones tell us which FAQs to add to your configuration.

### 18.10 Voice: the gated upsell

**Q: Voice is the premium upsell and the bigger ticket — but it isn't built. How does that affect the GTM and the revenue story?**
Honestly and deliberately, voice is gated and not yet promised. The GTM leads with the WhatsApp-plus-chatbot-plus-booking wedge because it's built, demoed, near-zero-risk, and cash-generating now; voice is positioned as a Month-2+ upsell that goes on the price sheet only after a recorded Marathi/Hinglish demo clears an end-to-end latency bar (about <800ms, ideal <600ms) on the real stack. The single biggest unbuilt gap is a real-time voice loop — Marathi has thinner model coverage that degrades on the Marathi+Hindi+English code-switching Aurangabad callers actually use, so it must be piloted on real calls before any quality guarantee. The near-term provable revenue is the WhatsApp wedge; voice is upside that lifts ACV once validated, but no revenue model should depend on it until the de-risk experiment clears.
  - *Follow-up:* What has to be true before you quote a single voice customer? — One end-to-end voice loop built and measured on real Marathi/Hinglish calls against the <800ms turn-latency bar, with a recorded demo that clears it. Until that recording exists, the rule is: don't sell voice, sell the WhatsApp wedge. And voice must be sold as metered minute-bundles with overage, never uncapped — because an uncapped flat fee on a high-volume customer is the one way the margins invert.
  - *Follow-up:* If voice never works in Marathi, is the business still viable? — Yes. On the WhatsApp wedge the economics already stand: ~80–90% gross margin at ₹1,499–3,000/month against ₹500–1,300 COGS, replacing part of a ₹15,000–22,000 receptionist. Voice expands ACV and TAM, but the WhatsApp-plus-booking-plus-chatbot bundle is a complete, sellable product on its own — voice is upside, not the foundation.
  - *Follow-up:* Are there compliance or accuracy-liability concerns specific to a vernacular voice receptionist? — Two. On compliance, an inbound voice receptionist on the customer's own line stays outside the outbound DLT/DND regime, so regulatory exposure is limited. On accuracy-liability, Marathi degrades on code-switching, so real-call accuracy may be worse than benchmark, raising the stakes of a misunderstanding on a clinical-adjacent call. The same containment applies (model interprets, code executes, ambiguity routes to a human, non-medical-advice disclaimer) — but the most important mitigation is simply not shipping voice until accuracy is validated.

### 18.11 Scaling, team & operations

**Q: What exactly is the sales motion today, and is any of it repeatable beyond the founder?**
Today the motion is founder-led and consultative: the engine produces a scored, ranked CRM; Hermes pushes today's call list to the founder's phone with the owner number, the why-they'll-buy line, and three pain questions; the founder calls or visits, then logs the visit by texting Hermes. The genuinely repeatable, non-founder-dependent half is the top of funnel — discovery, qualification, scoring, the personalised opener, and the daily ranked list are all codified, so a new rep inherits a pre-sorted list and a script. What is not yet repeatable is the close: there are no signed customers yet, so the close motion is unproven and still embodied in the founder. The honest status: lead-gen is productised; closing still needs to become a documented, teachable playbook.
  - *Follow-up:* You have demos but no paying customers — why believe the motion closes at all? — We won't dress that up: the product is demoed end-to-end and three cities are scored, but first customers are still being closed, so close-rate is an estimate, not a fact. The reason to believe it closes is the ROI gap is owner-obvious and quantifiable — replacing/backstopping a ₹15,000–22,000 receptionist for ₹1,499–3,000 — with a deterministic demo the owner watches book a slot on WhatsApp. The "Now"-phase milestone (owner visits in Nashik/Aurangabad, converting the first labs onto Meta Cloud API) is what de-risks it.
  - *Follow-up:* What's the single biggest reason a deal stalls? — Predictable stalls: "I already have staff" (the demo must reframe around nights/overflow, not headcount), trust in a brand-new unproven name, and the production-readiness gap (a WAHA demo isn't a Meta-official production line, so there's setup friction before go-live). Mitigations: case studies named after the first labs close, the inbound/compliant framing as a trust signal, and a documented Meta cutover.

**Q: How do you scale sales beyond one person, and what's the first hire?**
The plan is to make the lead engine the funnel so a hired rep starts productive on day one instead of prospecting. The engine hands a rep a city tab pre-sorted into Call-This-Week/Nurture/Skip, each row carrying owner name, WhatsApp number, a plain-English why-they'll-buy, fit scores, and pain-questions. Training collapses to: learn the deterministic demo, follow the written opener/ROI calculator per vertical, and log visits by texting Hermes. The funnel math sizes the load: roughly 600–900 worked prospects over 90 days to land ~20 customers, which is the trigger to hire a dedicated dialer. What's still missing for a clean hire is a proven, written close playbook and the first case studies — both come from the first cohort of real deals.
  - *Follow-up:* What must be true before you hire for sales? — A written, proven close playbook and at least a couple of named case studies. Right now the close lives in the founder's head and there are no signed customers; hiring an SDR against an unproven close just burns the engine's output. The first cohort converts founder intuition into a teachable script, an objection guide, and ROI proof — that's the unlock.
  - *Follow-up:* How long to ramp a new rep? — Faster than a typical SDR because they don't prospect — they inherit a scored, reasoned list and a written opener. The real ramp items are product fluency and objection judgement, which sharpens once real conversations are logged. We'd treat "first case study in hand" as the prerequisite for a clean ramp; before that, the founder is still discovering the objection set.

**Q: Walk me through the path from founder-led to a real team — what roles, in what order?**
The build is largely done, so the team you add is go-to-market, not engineering. Sequence: (1) founder closes the first cohort personally to extract the objection set and produce 3–5 named case studies (the "Now" phase); (2) first hire is a field/inside SDR working the engine's ranked list within the compliant window and running the demo, teachable once a close playbook exists; (3) an onboarding/customer-success person to absorb the Meta verification, content setup, and retention-watching as customer count grows; (4) only later, more engineering to harden the platform, split services, migrate the datastore, and build voice. Hermes is the lever that delays headcount — operational tasks become one-sentence commands — so the team you add is for the human-judgement work (closing, relationships) the system can't automate.

**Q: How many customers can one VM and one person realistically support?**
Two different ceilings, and it's important not to conflate them. The VM ceiling is high and not the near-term constraint: an e2-standard-4 at ~40% disk and light load, and the platform is multi-tenant — one n8n serves all customers, so adding a lab is mostly more WhatsApp conversations (COGS ~₹500–1,300/month), not more infrastructure. The binding constraint is the human: sales capacity, onboarding hand-holding (especially the Meta verification step), and content setup cap a single operator long before the VM does. So the first scaling lever is people and process; the VM only becomes a question at much larger scale.
  - *Follow-up:* Rough number — 20, 100, 500 customers on one VM? — We won't invent a benchmark since the load hasn't been stress-tested at volume. Qualitatively: at a few hundred customers the binding limits are human support and WhatsApp conversation throughput, not compute, since the same workflows serve everyone. Somewhere into the thousands, Google Sheets as the datastore becomes the first technical wall (slowdown past ~100,000 rows; Postgres on the same VM is the migration), and n8n concurrency and single-box concentration start to matter. People and the Sheets datastore break before the CPU does.

**Q: What breaks first at 50, at 500, and at 5,000 customers?**
At 50, people and process break, not technology — the VM is barely loaded, but the founder as single point of contact for onboarding (especially Meta verification) and support doesn't fit alongside selling; you'd feel the absence of an onboarding runbook, a per-customer health view, and a second human. At 500, the technical seams matter: Google Sheets as system-of-record approaches its practical limit (migrate to Postgres on the same VM), the single VM becomes a genuine reliability concern for 500 live front-desks (execute the containerised portability and split lead-engine from product), and WhatsApp conversation throughput/cost and Meta tier limits need managing — alongside a real support/success function and a multi-rep, by-city sales team. At 5,000 you're a different company: the datastore is long past Sheets, the VM is a multi-node or managed footprint, WhatsApp is at high Meta tiers, voice (if shipped) adds telephony and metered-minute billing, and the deepest break is organisational — churn management at scale, a real success org, management layers, and probably partner/reseller channels rather than pure direct sales. Ironically the lead engine scales most gracefully of all; it's the human-heavy close-onboard-retain functions that must become an organisation.
  - *Follow-up:* Is 50 customers a real business? — On unit economics the trajectory is strong (80–90% margin, ₹1,499–3,000 price vs ₹500–1,300 COGS, working per-customer from early on). The plan models ~20 customers reaching roughly ₹1.6–2 lakh MRR at a blended tier mix. 50 on the WhatsApp wedge alone is a healthy proof of repeatability and the base to raise on or fund the first hires from — an early-stage business, not yet a scaled one.
  - *Follow-up:* What's the margin risk at scale and how is it controlled? — The named risk is voice: a flat fee against a genuinely high-volume customer is the one way the 80–90% margin inverts (a 4,000-minute hospital at a low tier can cost more in voice than it pays). The hard guardrail is that voice is sold only as metered minute-bundles with overage, never uncapped. WhatsApp cost is per-conversation and small, so the chatbot wedge stays comfortably above 80% even at scale; pricing tiers and template discipline (utility vs marketing) protect margin.

**Q: What's the onboarding and support load per customer — and can it be self-serve?**
Onboarding has a light part and a heavy part. Light: the assistant and its flows are already built and the same brain serves every customer, so a new lab is configuration (prices, FAQs, timings, staff group), not a new build. Heavy and founder-gated: moving the customer onto Meta's Cloud API requires Business Verification (~2–10 days), a dedicated number, and template pre-approval — real per-customer friction and the main onboarding cost today. Ongoing support should be low because the bot is deterministic (no class of "it invented a price" tickets) and the data lives in the customer's own Sheet, but none of this is measured at volume yet. Self-serve isn't built — the Meta steps are owner-gated — so for the first cohort, assume founder-led, white-glove onboarding, which is also valuable for learning the objection and content set.

**Q: Doesn't generating leads cheaply just create a pile of unworked leads?**
A real failure mode the system is built against. The engine doesn't dump a list — it ranks into Call-This-Week/Nurture/Skip and excludes already-won/lost, so a rep always works the top of a prioritised queue. The risk is generating faster than humans can work, which is why the GTM discipline is one-city-per-weekly-sweep with dedup-upsert (refresh existing rows rather than pile on new geography) and gating new sweeps to human capacity. Generation is throttled to match the dialer, not run open-loop.

**Q: The Aurangabad funnel math is tight — is there enough top-of-funnel to hit 20 in 90 days?**
The math is tight and the plan says so honestly. To land ~20 customers you need ~70–90 qualified demos, from ~180–220 positive replies, from ~600–900 worked prospects. The Aurangabad labs-plus-coaching pool is ~270–480 real prospects, which the plan flatly admits "may not fully fill 20 if close rates run low." The resolution is breadth, not more scraping of one city: keep real-estate and hotels in the mix and start Nashik by Month 2 as overflow (~2.2× the volume). The engine produces qualified supply cheaply; the constraint is human touches against that supply — which is why SDR capacity is a named open decision.
  - *Follow-up:* If close rates come in below the 25–30% estimate, what's the lever? — Three, in order: widen the geo earlier (pull Nashik forward, then Pune) since lead supply is cheap and abundant; broaden verticals worked in parallel (real-estate, hotels) rather than re-touching a tapped-out pool; and tighten the demo/ROI calculator from real conversation data. The close-prob numbers are explicitly flagged as reasoned estimates, not surveyed, to be revised against the first 10–15 real conversations.

**Q: What's the geographic and vertical expansion sequence, and why that order?**
Geography first within one vertical, then verticals. Geo: Aurangabad now (prove the playbook, collect case studies), then Nashik (the cleanest clone — same Marathi, lowest competition, ~2.2× the lead volume, copy-paste config), then Pune (largest TAM at ~7.3× but most crowded, so enter with a sharper wedge and compete on integration depth and local service, not price), then Nagpur (strong growth but operationally distant and Hindi-leaning, needing a Hindi-default and a local referral first). Vertical: lead with diagnostic labs and coaching (highest close probability, shortest 1–4 week cycles, instantly quantifiable missed-call ROI); run 2–3 hospitals or real-estate developers in parallel as flagship logos despite their 1–3 month cycles; treat insurance as opportunistic. A new vertical or city is mostly a config change — the engine already carries chain-blocklists and positioning for all seven verticals.
  - *Follow-up:* Why labs first, not the bigger hospital market? — Hospitals are the best technical fit (highest call volume) but the worst near-term business: procurement, IT, committee sign-off, and data-privacy scrutiny stretch the cycle to 1–3 months. Labs are owner/pathologist-led, so there's a single decision-maker, every missed call is a lost test with a known rupee value, and the report-status flow is an obvious win — close in 1–3 weeks. The discipline is explicit: pursue a couple of hospitals for case-study value but don't let their long cycles absorb the bandwidth that closes the high-velocity deals.
  - *Follow-up:* What actually changes per new city — is it really copy-paste? — Mostly yes, and that's the leverage. One stack, one Google Sheet partitioned by a City column, and discovery driven by changing the {city, industries} config. Enrichment, scoring, the lookup webhook, and the WhatsApp booking bot are reused unchanged. The one genuine per-city fork is the voice agent's default language (Marathi+English for Aurangabad/Pune/Nashik, Hindi-default for Nagpur) — and voice isn't even shipping yet. For the WhatsApp wedge today, a new city is essentially a config change plus local outreach.

### 18.12 Risks, traction, roadmap & the ask

**Q: How real is the traction — separate what's done from the plan.**
Done and verifiable: the autonomous lead-to-CRM loop is live and produces scored sheets in minutes; three cities scored (Aurangabad, Nashik, HSR Bangalore); scoring validated against the founder's hand-picked top-10 and hardened across two live runs (chain and veterinary exclusions added, volume model recalibrated); the WhatsApp Front Desk demonstrated end-to-end on a real number; field-capture live; 31 automations (21 active) on one VM. Also proven by operation — the failure modes we've been candid about (silent zero-result runs, the weekly OAuth expiry, Gemini quota) — we know because we hit them, not because we theorised them. Not yet done: zero paying customers (first being closed founder-led); no production WhatsApp traffic; no real close-rate, churn, or ARPU data; voice unbuilt. Strong technical and engine traction with the commercial proof still ahead — which is the right risk profile for the stage, but you should not credit revenue that doesn't exist.
  - *Follow-up:* The case-study deck — is that a real customer? — No, and we want to be precise: it's an anonymised "working demonstration" deck describing a busy ~600-review Aurangabad lab generically, using a fictional brand ("City Pathology Lab") for the chat visuals. It's a sales trust-builder, not a paying-customer reference, and we won't represent it as traction.
  - *Follow-up:* How much of the reliability story is proven versus theoretical? — The engineering is real and exercised; the production-grade hardening and at-scale endurance are the unproven, scoped work ahead. Proven by operation: the lead loop, the scoring validation and hardening, the end-to-end Front Desk demo, the field-capture loop. Not proven: sustained uptime under real customer load, the Meta Cloud API path with a live customer (documented, not cut over), disaster recovery (never executed), and voice.

**Q: What churn do you expect, and how do you retain SMBs?**
There's no observed churn number yet because there are no paying customers — any figure would be invented. The retention logic the product is built around: it's load-bearing and measurable. Once a lab's daily report-status traffic and after-hours capture run through GVM — especially the staff-post-a-number, bot-auto-texts flow — ripping it out reverts them to missed calls and manual work, and most customers recover the subscription within weeks from recovered missed-call revenue. The structural risks are real: the low price makes a churn event cheap, the value is invisible if we don't surface it, and a single ban or downtime event could break trust. Mitigations: move to the Meta API so the number can't get banned, make ROI visible (missed calls recovered, reports auto-sent), and keep the customer's data in their own Sheet and number so it's their asset, not ours.
  - *Follow-up:* What's the leading indicator you'd watch to catch churn early? — Message volume and the report-ready flow firing. If staff stop posting numbers in the group and inbound patient messages drop, the product has fallen out of the daily workflow — that's the canary. The platform already logs every interaction through n8n, so a simple per-customer activity/usage view is the natural early-warning dashboard, the retention-side equivalent of the conversion analytics on the roadmap.
  - *Follow-up:* Doesn't a ₹1,499–3,000 price make customers churn casually? — It cuts both ways. Cheap means low switching pain, but it also makes the decision to keep it trivial if it's delivering — at that price, recovered revenue from even a few saved missed calls dwarfs the fee. The defensive move is operational stickiness (wired into their number and daily report workflow) plus upsell into stickier tiers (booking, then validated voice), so a customer who started on the wedge has more embedded and more to lose by leaving.

**Q: Tell me about key-person and single-VM concentration risk.**
That's the most acute risk and we won't minimise it. Today the entire system — sales, the lead engine, product builds, infrastructure — is founder-operated, the autonomous loop is steered by a single SOUL.md file, and one e2-standard-4 in Mumbai runs everything with no automatic failover. Mitigations in motion: the whole stack is containerised and portable (liftable to a bigger machine or another cloud without a rewrite), the CRM is a plain Google Sheet the customer owns (no proprietary lock-in), customer data lives off the box, and the operating playbook is being codified so it can transfer to a hire. Concretely, de-risking this requires the next raise to fund a first sales hire and a second engineer — a milestone we'd hold ourselves to.
  - *Follow-up:* If the founder is hit by a bus tomorrow, does the company survive? — Not gracefully today — it's a single-founder, single-operator system. What survives is recoverable: the code is in Git, the infrastructure is containerised and documented, the CRM data lives in Google Sheets the customer controls, and the production path is written down. So the assets are recoverable by a competent engineer, but the selling motion and roadmap are in one head. Reducing that is the explicit purpose of the first two hires post-raise.

**Q: You're on free GCP credits — how dependent is the business on that?**
The dependence is cosmetic, not structural, and we won't present ₹0 as the steady state. Free credit (GCP's $300/90-day trial plus the always-free tier) makes today's run-cost essentially zero, but the fully-paid platform is only ₹4,500–9,000/month — trivial relative to even a handful of paying customers. When the credit ends it's a small, known monthly bill, not a cliff. The more important point: "we run at zero today" should not be read as traction or product-market fit — it just means our burn is low while we prove the sales motion.
  - *Follow-up:* Isn't the zero-cost framing a bit of theatre? — Fair, and we'd rather own it: zero-cost-to-run is a real near-term cash advantage but not a moat and not evidence customers want this. The genuinely defensible cost point is the architecture — one self-hosted, open-source n8n platform with no per-message tax serving all customers — which keeps gross margin high regardless of who pays the cloud bill. Weight the 80–90% margin architecture, not the temporary free credit.
  - *Follow-up:* What breaks first when the GCP free trial ends? — Nothing functionally — it just starts costing ~₹8,000–9,000/month for the VM. The subtler dependency is Gemini's free tier: the lead engine no longer needs it (pitches moved off the hot path), but the bot's NLU calls Gemini per message, so at customer volume we must enable billing on a key before ~100 calls/day throttles real patients. That's a one-time billing-enable, cheap per token, and a known pre-scale step.

**Q: What milestones in the next 12–18 months de-risk the next round, and which single one proves the company?**
The milestones map to the gaps: close the first paying cohort founder-led in Aurangabad/Nashik on the Meta Cloud API and produce real close-rate, sales-cycle, ARPU, and early-churn numbers; prove rep-led selling by codifying the playbook and getting a first non-founder rep productive; demonstrate net revenue retention via a WhatsApp-to-booking upgrade; and run a real-call Marathi/Hindi voice pilot that clears a quality bar (the biggest single ARPU value-unlock). The single most important is retention of the first paying cohort — lead supply, scoring, demo quality, and margins are already in hand; the one thing never tested is whether an SMB owner pays month two and month three. If even 10–20 labs stay and ideally expand, the core PMF question is answered; without retention, none of the rest matters.
  - *Follow-up:* What would make you walk away from this yourself? — Two killers: a first cohort churning fast (the missed-call ROI doesn't translate to perceived value once the novelty fades), or rep-led CAC so high that only the founder can sell it (capping us at a lifestyle business). A narrower third: the voice pilot failing badly on real Marathi/Hindi calls and the WhatsApp-only ARPU proving too thin to support city-by-city expansion economics. Any of those would say the venture-scale thesis is broken even if the product "works."

**Q: Is this venture-backable or a great bootstrapped SMB-tools business — and what round makes sense?**
On today's evidence it's closer to a capital-efficient, potentially-great SMB-tools business with venture optionality, not yet a proven venture-scale company. The high margins, near-zero CAC engine, and low burn make it genuinely fundable at pre-seed altitude; whether it earns venture scale depends on the three multipliers (vertical, geographic, ARPU/voice) stacking. We'd frame the ask as a small pre-seed sized to fund exactly the de-risking milestones — first paying cohort with retention data, a first sales hire to test rep-led CAC, the Meta production cutover at scale, and a voice pilot — not a big round priced on a vision. Staying lean (the architecture supports near-zero run cost) means we don't need much capital to reach the inflection that prices the next round on demonstrated PMF; over-raising now would just dilute on a demo-stage story.

**Q: What are the realistic exit and scale paths?**
The most probable outcome is a strategic acquisition once we have regional density and proven retention: buyers include horizontal Indian WhatsApp/CPaaS platforms (Wati, Gallabox, AiSensy, or a Gupshup/Exotel-type) wanting vertical depth and SMB distribution, enterprise vertical-SaaS players (a lab-CRM moving downmarket), or an SMB-software roll-up wanting the Tier-2 footprint and the lead engine. A standalone IPO path would require the multi-vertical, multi-city, voice-unlocked version at real scale — possible but the lower-probability branch. We'd underwrite this as a capital-efficient business that becomes an attractive tuck-in once it proves repeatable retention and a rep-led motion across a couple of cities, with optionality on a larger outcome if voice plus multi-vertical compounds.

---

*End of dossier. This file contains no credentials and is safe to share. For a live walkthrough, the system can be driven end-to-end from a single Telegram message.*
