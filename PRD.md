<p align="center">
  <br/><br/>
  <span style="font-size:36px"><b>WEONGO</b></span><br/>
  <i>Product Requirements Document</i><br/><br/>
  <b>A Unified Competitive Programming Analytics & Growth Platform</b>
</p>

---

|  |  |
|---|---|
| **Team** | Suhan · Renish · Chintan |
| **Stack** | MERN + Socket.io + BullMQ + Redis + OpenRouter (free-tier LLM) |
| **Timeline** | 30 days |
| **Cost Constraint** | Zero paid services — every dependency must have a free tier sufficient for project scope |

---

# 1. Product Summary

WeonGO is a unified competitive programming analytics and growth platform that aggregates a user's LeetCode and Codeforces activity into a single dashboard, layering on AI-driven weak-topic analysis, peer comparison, contest tracking, and personal knowledge vaults (tricky problems + concepts). A background worker keeps data fresh even when the user is logged out, and a real-time layer pushes updates to connected clients the moment new data syncs in.

> [!IMPORTANT]
> **Core Pitch — "Spotify Wrapped for DSA addicts"**
>
> User enters their LeetCode and Codeforces handles once. WeonGO silently keeps their stats fresh, tells them what they're bad at, shows them how they compare to friends, and gives them a place to dump tricky problems and concepts they'd otherwise forget.

WeonGO is explicitly **not** a problem-solving platform — no judge, no code execution. It is a meta-layer on top of LeetCode and Codeforces that turns scattered profile data into structured self-knowledge and measurable growth.

---

# 2. Goals

- Give competitive programmers **one dashboard** instead of three tabs and a spreadsheet.
- Surface **weak topics** and a recommended roadmap using actual submission data, not vibes.
- Make consistency **visible and social** — streaks, heatmaps, friend leaderboards — to drive retention.
- Capture **tacit knowledge** (tricky problems, one-liner concepts) that normally gets lost in a notebook.
- Ship a system with **genuine backend depth**: scraping reliability, background workers, caching, and sockets. This is the component that matters most for resume and interview signal.

---

# 3. Non-Goals

- No in-app code execution or judging.
- No payments, subscriptions, or paid API tiers anywhere in the stack.
- No mobile app — responsive web only.
- No support for platforms beyond LeetCode and Codeforces in v1. AtCoder and CodeChef are explicitly deferred.

---

# 4. Users & Core User Story

## 4.1 Primary User

A competitive programmer — student or placement-prep candidate — who actively solves on LeetCode and/or Codeforces and wants to track progress, find weaknesses, and stay motivated via peers.

## 4.2 Core Story

> [!IMPORTANT]
> **User Journey**
>
> "I link my LeetCode and Codeforces handles once. From then on, WeonGO silently keeps my stats fresh, tells me what I'm bad at, shows me how I compare to my friends, and gives me a place to dump the tricky problems and concepts I'd otherwise forget."

---

# 5. System Architecture

## 5.1 Architecture Diagram

```
┌──────────────────────────────────────────────────────────┐
│                    CLIENT (React)                        │
│  Dashboard │ Network │ Smartsheet │ Contests │ Vaults    │
└────────────────────────┬─────────────────────────────────┘
                         │  REST (CRUD) + Socket.io (live updates)
┌────────────────────────┴─────────────────────────────────┐
│                  EXPRESS API SERVER                       │
│  Auth (JWT) · REST Controllers · Socket.io Gateway       │
│  Rate Limiter                                            │
└───┬──────────────────┬──────────────────┬────────────────┘
    │                  │                  │
┌───┴────┐    ┌────────┴────────┐   ┌────┴──────────────┐
│MongoDB │    │  Redis / BullMQ │   │  BullMQ Workers   │
│        │    │  Cache + Queue  │   │  - Sync Worker    │
│ Users  │    │  job backend    │   │  - AI Analysis    │
│ Stats  │    │                 │   │  - Prediction Job │
│ Subs   │    └────────┬────────┘   └────┬──────────────┘
└────────┘             │                 │
              ┌────────┴────────┬────────┴──────────┐
              │                 │                    │
        LeetCode GraphQL   Codeforces REST   OpenRouter (LLM)
```

> [!WARNING]
> **Critical Architectural Decision**
>
> Scraping and sync are fully decoupled from the user-facing request/response cycle. Nothing in the Express API ever calls LeetCode or Codeforces directly — only background BullMQ workers do. This is the single most important architectural decision in the system and is the primary interview talking point.

---

# 6. Data Model (MongoDB)

## 6.1 User

| Field | Type | Notes |
|---|---|---|
| `_id` | ObjectId | Primary key |
| `email`, `passwordHash` | String | Local auth |
| `googleId` | String? | Optional OAuth |
| `leetcodeUsername` | String? | Nullable — user may link only one platform |
| `codeforcesUsername` | String? | Nullable |
| `friends` | [ObjectId] | Mutual, accepted friend list |
| `friendRequests` | [ObjectId] | Pending incoming |
| `theme` | String | `light` / `dark` |
| `lastSyncedAt` | Date | Timestamp of last successful sync |
| `createdAt` | Date | Account creation |

## 6.2 PlatformStats

One document per user per platform; overwritten on each sync.

| Field | Type | Notes |
|---|---|---|
| `userId` | ObjectId | Ref: User |
| `platform` | String | `leetcode` / `codeforces` |
| `totalSolved`, `easySolved`, `mediumSolved`, `hardSolved` | Number | LeetCode only |
| `rating`, `maxRating`, `rank` | Number/String | Codeforces only |
| `ratingHistory` | [{contestId, ratingChange, newRating, date}] | CF contest history |
| `topicBreakdown` | [{tag, solvedCount, attemptedCount}] | Both platforms; normalised to shared taxonomy (Section 8) |
| `currentStreak`, `longestStreak` | Number | Derived from submission calendar |
| `submissionCalendar` | Map\<dateString, count\> | Powers the GitHub-style heatmap |

## 6.3 Submission (append-only)

Raw log; never deleted. Used for AI analysis, smartsheet, and weak-topic detection.

| Field | Type | Notes |
|---|---|---|
| `userId`, `platform`, `problemId`, `problemName`, `tags` | Mixed | Core identifiers |
| `verdict` | String | Accepted / Wrong Answer / TLE / etc. |
| `timestamp` | Date | Submission time |

## 6.4 TrickyProblem (User Vault)

| Field | Type | Notes |
|---|---|---|
| `userId`, `problemLink`, `title`, `tags` | Mixed | Problem identifiers |
| `hint`, `intuition`, `solution` | String | Three-stage progressive reveal |
| `cracked` | Boolean | Whether the user solved it |
| `createdAt` | Date | |

## 6.5 Concept (User Vault)

| Field | Type | Notes |
|---|---|---|
| `userId`, `title`, `body` | String | Concept content |
| `linkedProblems` | [String] | Plain URL strings — no resolution needed |
| `source` | String? | Blog/CF round reference |

## 6.6 FavouriteProblem

Simple join table: `userId`, `problemLink`, `contestId` (optional).

---

# 7. Feature Specifications

## 7.1 Real-Time Sync Engine (The Backbone)

> [!IMPORTANT]
> **Engineering Priority**
>
> This is the highest-engineering-value component and should be treated as the project's centrepiece, not a utility. It is the single feature most likely to generate substantive interview questions.

### Problem Statement

LeetCode has no official API. Codeforces rate-limits aggressively — 1 request per 2 seconds per the official docs. Calling either API on-demand inside a user-facing request is fragile and slow. Sync is therefore fully decoupled into BullMQ background jobs.

### Job Flow

- Repeatable job (BullMQ `repeat` option) runs every N minutes per user. Jobs are staggered via ID jitter so all users don't fire simultaneously.
- Job fetches LeetCode via the unofficial GraphQL endpoint (`leetcode.com/graphql`) and Codeforces via `codeforces.com/api/user.status` + `user.rating`.
- Diffs new submissions against the last-known submission ID/timestamp — only new submissions are appended to the Submission collection. Avoids reprocessing full history every cycle.
- Updates PlatformStats: topic breakdown, streak, heatmap, rating history.
- On a successful diff with one or more new submissions, emits a Socket.io `stats:updated` event to that user's room. Connected clients update live without a page refresh.
- Jobs run regardless of whether the user is logged in. The server-side cron does not care about client sessions — this is what "even when you're logged out" means architecturally.

### Failure Handling (Interview Differentiator)

- **Codeforces API downtime:** BullMQ exponential backoff retry (3 attempts).
- **LeetCode GraphQL schema changes:** wrap parsing in a schema-validation layer (zod/Joi). On validation failure, log and skip rather than crash the worker.
- **Rate-limit safety:** token-bucket limiter shared across all CF jobs via BullMQ's `limiter` option per queue. Non-negotiable — getting IP-banned from Codeforces kills the entire product.

### Socket.io Design

Chosen over plain client-side polling because: (a) it is a stack requirement, (b) it gives the "feels alive" effect on the dashboard and friend leaderboard without wasteful polling. Important: this delivers **near-real-time** data (bounded by the worker poll interval, typically 5–15 minutes) pushed in **true real-time**. The liveness is in the delivery mechanism, not in the underlying data freshness, since neither upstream platform offers webhooks.

## 7.2 Dashboard

- **Pie/donut chart:** Easy/Medium/Hard split (LeetCode) — Recharts.
- **Rating-over-time line graph** (Codeforces `ratingHistory`).
- **Submission heatmap** (GitHub-style, from `submissionCalendar`).
- **Current streak counter** with "don't break it" framing.
- **Problem of the Day:** LeetCode exposes a daily challenge via GraphQL. Codeforces has no official POTD — surface a "recommended unsolved problem from your weak topic" instead (ties into Section 7.8). Do not fabricate a feature that doesn't exist upstream.

## 7.3 Network (Peer System)

- Friend request flow: send → accept/reject → mutual friends array update.
- Friend online/connected presence via Socket.io connection tracking. Use a **Redis-backed** `connectedUsers` map (not in-memory) so presence survives server restarts.
- Scoped as friends list + presence + leaderboard — not an open social network. No public discovery feed, no posts. This preserves the "network" framing while keeping scope controlled.

## 7.4 Smartsheet

- Aggregates the most-attempted problems within a user's friend group in the last 7 days. Queries across friends' Submission collections, groups by `problemId`, sorts by attempt count.
- Auto-refreshes when any friend's sync job completes via the same `stats:updated` socket event, scoped to a `smartsheet:<userId>` room.

## 7.5 Contests

- **Upcoming contests:** Codeforces exposes this via the `contest.list` API (free, official). LeetCode does not expose a contest API — either scrape the LeetCode contest page on a schedule, or scope "upcoming contests" to Codeforces only and show LeetCode past-contest performance only. Decision: be explicit about this gap early rather than discovering it in week 3.
- Past contest results pulled from `ratingHistory`.
- "Upsolve" button deep-links to the contest's problem list on the original platform. Do not mirror problem statements — copyright/ToS risk and unnecessary scope.
- Favourite button writes to `FavouriteProblem` collection.

## 7.6 Tricky Problems Vault

- Full CRUD on `TrickyProblem` collection.
- UI reveals **hint → intuition → solution** progressively via click-to-reveal for each stage individually, not all at once. Specify this now so the implementer does not default to a flat text dump.

## 7.7 Concept Vault

- Full CRUD on `Concept` collection.
- `linkedProblems` stored as plain URL strings — no resolution or validation against LeetCode/CF problem databases needed. Adds scope for no real benefit.

## 7.8 AI Weak-Topic Analysis & Roadmap

### Pipeline

- Triggered on-demand (user clicks "Analyze me") or weekly via BullMQ scheduled job.
- Backend aggregates `topicBreakdown` + recent Submission verdicts into a compact **structured JSON summary** — not a raw submission dump. Token budget matters on free models. Per-tag shape: `{tag, solved, attempted, failureRate, lastAttemptedDaysAgo}`.
- This structured summary is sent to OpenRouter with a system prompt constraining output to strict JSON. Instruct the model to return only JSON with no preamble or Markdown fences. Parse defensively; retry once on parse failure.
- Output schema:
  ```json
  {
    "weakTopics": [{"tag": "string", "reason": "string"}],
    "strongTopics": ["string"],
    "roadmap": [{"step": 1, "focusTag": "string", "suggestedProblems": ["string"]}]
  }
  ```
- Cache the result in Redis with a **24-hour TTL**. Re-running this on every dashboard load would burn free-tier rate limits fast.

### Free-Tier Constraint

Confirm the exact OpenRouter model with the team. Design the prompt to fit comfortably within that model's context window using the aggregated-summary approach above — never raw submission logs. Add a BullMQ request queue with `concurrency: 1` so concurrent users do not blow through the model's free-tier requests-per-minute limit.

## 7.9 Problem Recommendation Engine

### Tier 1 — Rule-Based (Ship This First)

For each weak tag, query a small static curated problem list. Maintain a JSON seed file mapping tag → 5–10 well-known problems per difficulty level, then recommend unsolved ones. Reliable, explainable, zero AI cost. This is the launch-ready version.

### Tier 2 — AI-Augmented (Stretch Goal)

Feed weak topics into the same OpenRouter call from Section 7.8 and let the model suggest specific problems by name. Critical constraint: LLMs hallucinate problem names and numbers. Tier 2 output **must** be validated against the curated seed list before being shown to users — never shown raw.

## 7.10 Rating Prediction

Simple linear regression, or even simpler: a weighted moving average of the last N rating deltas over `ratingHistory`. Do not oversell this as ML. Frame it honestly as a "trend-based projection," not an "AI prediction." A recruiter who asks "what regression model, what features, what is your validation approach" will immediately expose an oversold simple average — and that is a negative signal.

## 7.11 Compare Profiles

Side-by-side PlatformStats for two users (self vs. any friend): ratings, topic breakdown, solve counts. Pure read/aggregate query — no new data model required.

## 7.12 Theme Toggle

Standard light/dark toggle persisted to `User.theme`, applied via CSS variables or Tailwind dark-mode class strategy. Lowest-risk feature in the entire spec. Implement last. Takes under a day.

---

# 8. Cross-Platform Tag Taxonomy

> [!WARNING]
> **Hidden Complexity — Address in Week 1**
>
> LeetCode tags (`"Dynamic Programming"`, `"Two Pointers"`) and Codeforces tags (`"dp"`, `"greedy"`, `"graphs"`) do not share a naming scheme. Weak-topic analysis is meaningless if `"dp"` and `"Dynamic Programming"` are treated as different topics. Build the normalization map early — this is a 1-day task in week 1 and a frustrating retrofit in week 3.

Build a normalization map:

```json
{
  "dp": "Dynamic Programming",
  "two-pointer": "Two Pointers",
  "greedy": "Greedy",
  "graphs": "Graph Theory",
  "...": "..."
}
```

All `topicBreakdown` entries and AI prompts must use the normalized taxonomy. Both LeetCode and Codeforces raw tags get mapped through this layer before touching the database.

---

# 9. Free-Tier Service Map

| Need | Service | Free-Tier Constraint |
|---|---|---|
| Database | MongoDB Atlas M0 (free cluster) | 512 MB storage cap — store aggregated stats, not raw submission history beyond the heatmap window |
| Cache + Queue | Redis Cloud free tier / Upstash free tier | ~30 MB / request-based limits — keep cached payloads small |
| LLM | OpenRouter free-tier model (team to confirm) | Shared low RPM cap on free models — must queue, never fire concurrently |
| API Hosting | Render free tier | Cold start issue — use cron-job.org keep-alive ping (same pattern as existing portfolio site) |
| Client Hosting | Vercel free tier | No relevant limits at this scale |
| LeetCode Data | Unofficial GraphQL endpoint | No published SLA — must degrade gracefully: show last-synced data on failure, never crash |
| Codeforces Data | Official REST API | 1 req / 2 sec hard limit — enforce via BullMQ token-bucket limiter across all CF jobs |

---

# 10. Feature-Based Team Split

Names are placeholder — confirm final assignment per pillar.

| Owner | Pillars | Notes |
|---|---|---|
| **Dev A** (Backend Heavy) | Real-Time Sync Engine (7.1) · Contests (7.5) · Theme Toggle (7.12) | Owns the worker/queue/socket infrastructure that everything else consumes. Critical dependency — must ship a skeleton + seed script by end of Week 1. |
| **Dev B** (Frontend + Realtime) | Dashboard (7.2) · Network/Peer System (7.3) · Smartsheet (7.4) | Primarily consumes data from Dev A's sync engine. Can build against seed data from day 1. |
| **Dev C** (AI + Algorithms) | AI Analysis (7.8) · Recommendation Engine (7.9) · Prediction (7.10) · Compare (7.11) · Vaults (7.6, 7.7) | Algorithmic and data-layer heavy. Vaults are CRUD-only and are the warmup; AI pipeline is the centrepiece. |

> [!CAUTION]
> **Critical Dependency Note**
>
> Dev B and Dev C's features both consume data produced by Dev A's sync engine. Dev A's sync engine and the shared data model (Section 6) must be functional in skeleton form by end of Week 1 — even with mocked/partial data. Dev A must ship a seed script that populates MongoDB with realistic fake data so Dev B and Dev C can build in parallel from day one without blocking on live scraping.

---

# 11. Build Sequence (4 Weeks)

## Week 1 — Foundations

- Auth (JWT + optional Google OAuth), DB schema live in MongoDB Atlas.
- Sync engine skeleton (mocked data acceptable — the shape matters, not the data source yet).
- Seed script: populates DB with realistic fake users, stats, and submissions.
- Basic Socket.io connection handshake confirmed end-to-end.

*Goal: by Friday of Week 1, every team member can build against real data shapes without waiting on live scraping.*

## Week 2 — Core Features (Parallel)

- Dashboard charts (Recharts: pie, line, heatmap).
- Network/friend system (request flow, presence, leaderboard).
- Vaults CRUD (both Tricky Problems and Concept).
- Rule-based recommendation engine (Tier 1) — seed the tag → problems JSON file.
- Tag taxonomy normalization layer built and integrated.

## Week 3 — Integration + AI Layer

- Real sync engine fully wired to LeetCode GraphQL and Codeforces REST.
- AI weak-topic pipeline (OpenRouter call, structured JSON output, Redis cache).
- Smartsheet (depends on friends + submissions both being live — ship after Week 2 features land).
- Contests view (upcoming CF contests + past performance).
- Rating prediction (moving average over ratingHistory).

## Week 4 — Hardening + Polish

- Error handling on scraping failures; graceful degradation to last-known-good data.
- Rate-limit safety audit: confirm BullMQ limiter is working under concurrent load.
- Deployment: Render (API) + Vercel (client) + cron-job.org keep-alive.
- Theme toggle implementation.
- Demo script and screen recording prep.

---

# 12. Risk Register

| Risk | Likelihood / Impact | Mitigation |
|---|---|---|
| LeetCode unofficial endpoint breaks or changes response shape | High likelihood, Medium impact | Schema validation layer (zod/Joi) on all LeetCode responses. On validation failure: log, skip, surface last-known-good data to user. Never crash the worker. |
| Codeforces rate-limit ban | Low likelihood, Critical impact | Centralised token-bucket limiter via BullMQ `limiter` option. Never call CF outside the worker. One IP ban kills the entire product. |
| Free LLM rate limits choke AI feature under demo load | Medium likelihood, Medium impact | 24h Redis cache on AI results. BullMQ queue with `concurrency: 1`. Fallback: show cached result or a canned "Analysis unavailable" message — never surface a raw API error to the user. |
| Redis/Mongo free-tier storage caps hit | Low likelihood, Low impact | Store only aggregated stats and recent submission window (last 90 days). Do not store full raw submission history indefinitely. |
| Scope creep back to all features at 100% simultaneously | High likelihood, High impact | Tier 1 (rule-based) versions are the default target. Tier 2 (AI-augmented) enhancements are explicit stretch goals, not launch blockers. Track per-feature completion weekly. |

---

# 13. Key API Contracts

## 13.1 Auth

| Endpoint | Request Body | Response |
|---|---|---|
| `POST /api/auth/register` | `{ email, password }` | `{ token, user }` |
| `POST /api/auth/login` | `{ email, password }` | `{ token, user }` |
| `POST /api/auth/link-platform` | `{ platform, username }` + JWT | `{ success, message }` |

## 13.2 Stats & Dashboard

| Endpoint | Auth/Params | Response |
|---|---|---|
| `GET /api/stats/:userId` | JWT | `PlatformStats[]` |
| `GET /api/stats/:userId/heatmap` | JWT | `{ dates: Map<string, number> }` |
| `POST /api/stats/:userId/sync` | JWT (triggers manual sync job) | `{ jobId }` |

## 13.3 Network

| Endpoint | Request | Response |
|---|---|---|
| `POST /api/friends/request` | `{ targetUserId }` + JWT | `{ success }` |
| `POST /api/friends/respond` | `{ requesterId, accept }` + JWT | `{ success }` |
| `GET /api/friends/leaderboard` | JWT | `[{ userId, username, totalSolved, rating }]` |

## 13.4 AI Analysis

| Endpoint | Request | Response |
|---|---|---|
| `POST /api/analysis/trigger` | JWT (queues AI job) | `{ jobId }` |
| `GET /api/analysis/:userId` | JWT | `{ weakTopics, strongTopics, roadmap }` or `204` if not yet available |

## 13.5 Vaults

| Endpoint | Request | Response |
|---|---|---|
| `GET /api/vault/tricky` | JWT | `[TrickyProblem]` |
| `POST /api/vault/tricky` | `{ problemLink, title, hint, intuition, solution, tags }` + JWT | `TrickyProblem` |
| `PUT /api/vault/tricky/:id` | Partial update + JWT | `TrickyProblem` |
| `DELETE /api/vault/tricky/:id` | JWT | `{ deleted: true }` |
| `GET /api/vault/concepts` | JWT | `[Concept]` |
| `POST /api/vault/concepts` | `{ title, body, linkedProblems, source }` + JWT | `Concept` |

---

# 14. Socket.io Event Reference

| Event | Description |
|---|---|
| `stats:updated` | **Server → Client.** Emitted when a sync job completes with new submissions for a user. Payload: `{ userId, platformStats, submissionCount }`. Client re-fetches dashboard data on receipt. |
| `smartsheet:updated` | **Server → Client.** Emitted when any friend's sync completes. Scoped to `smartsheet:<userId>` room. Payload: `{ userId }`. Client re-fetches smartsheet on receipt. |
| `friend:online` | **Server → Client.** Emitted to a user's friends when that user connects. Payload: `{ userId }`. |
| `friend:offline` | **Server → Client.** Emitted to a user's friends when that user disconnects. Payload: `{ userId }`. |
| `analysis:ready` | **Server → Client.** Emitted when the AI analysis job for a user completes. Payload: `{ userId }`. Client fetches `/api/analysis/:userId` on receipt. |

---

# 15. AI Prompt Design (Weak-Topic Analysis)

## 15.1 System Prompt

> You are a competitive programming coach. Analyse the submission data provided and return ONLY a JSON object with no preamble, no explanation, and no Markdown fences. The JSON must exactly match the schema:
> ```json
> {
>   "weakTopics": [{"tag": "string", "reason": "string"}],
>   "strongTopics": ["string"],
>   "roadmap": [{"step": 1, "focusTag": "string", "suggestedProblems": ["string"]}]
> }
> ```

## 15.2 User Message Template

> Submission summary (last 90 days): `[JSON array of {tag, solved, attempted, failureRate, lastAttemptedDaysAgo}]`. Identify the 3 weakest topics, the 2 strongest, and provide a 5-step roadmap to address the weaknesses.

## 15.3 Parsing & Caching

- Wrap `JSON.parse` in try/catch. On parse failure: retry the API call once with a stricter system prompt.
- On second failure: return a `503` to the client with message "Analysis temporarily unavailable." Never surface a raw API error.
- Cache successful results in Redis with a **24-hour TTL** keyed by `userId`.
- Queue all OpenRouter calls through a BullMQ queue with `concurrency: 1` to avoid blowing the free-tier RPM limit under demo load.

---

# 16. Definition of Done

| Feature | Done When… |
|---|---|
| **Sync Engine** | Worker runs on schedule, diffs correctly, updates DB, emits socket event. Survives a simulated Codeforces outage (exponential backoff tested). Rate limiter confirmed active. |
| **Dashboard** | All four widgets (pie, line graph, heatmap, streak) render with real synced data. POTD shows for LeetCode; weak-topic recommendation shows for Codeforces. |
| **Network** | Friend request → accept → leaderboard full flow works. Presence (online/offline) updates in under 2 seconds. |
| **Smartsheet** | Shows top 10 most-attempted problems among friends in last 7 days. Updates when a friend's sync completes. |
| **Contests** | Upcoming CF contests listed with countdown. Past contest results visible. Upsolve link works. |
| **Vaults** | Full CRUD for both Tricky Problems and Concepts. Three-stage progressive reveal implemented. |
| **AI Analysis** | Structured JSON returned and rendered. Redis cache confirmed (second trigger within 24h returns cached). Queue concurrency confirmed. |
| **Recommendation** | Tier 1 rule-based recommendations shown per weak topic. No hallucinated problem names surfaced. |
| **Prediction** | Rating projection shown on Dashboard. Framed as "trend projection", not "AI prediction". |
| **Compare** | Side-by-side stats render correctly for any two friends. |
| **Theme Toggle** | Light/dark persists across sessions. Implemented last. |
| **Deployment** | API on Render, client on Vercel, keep-alive ping active, no cold-start failures during demo. |

---

<p align="center"><b>End of Document</b></p>
<p align="center"><i>Pending: final dev-to-pillar name mapping  |  confirmed OpenRouter model for Sections 7.8 & 7.9</i></p>