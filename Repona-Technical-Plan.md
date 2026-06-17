# Repona — Complete Technical Plan & Stack

This document covers every technical decision required to build Repona end to end: architecture, every service involved, how each of the 14 product features actually gets built, data modeling, security, and a phased build order. Every tool chosen is free (or has a free tier sufficient for launch and early growth), with the reasoning for each choice and an honest note on its limits, so nothing is a silent surprise later.

A guiding rule used throughout: **prefer one platform doing three things over three platforms doing one thing each**, because a solo developer's scarcest resource is integration time, not money. Where that tradeoff isn't clear-cut, both options are noted.

---

## 1. High-Level Architecture

Repona is a single web application (no separate native mobile app needed at launch — Section 25 of the user-flow document already specifies the web app must be responsive) with four broad layers:

1. **Frontend** — what the user sees and interacts with.
2. **Backend / API layer** — auth, business logic, orchestration of AI calls, webhooks.
3. **Data layer** — relational database, file/diagram storage, search index.
4. **Background job layer** — anything slow (repository scanning, diagram generation, video rendering) that cannot block a user's request.

A simplified flow: the frontend talks to the backend API; the backend reads/writes the database directly for fast operations, and hands slow operations to a background job queue; the background workers call external services (the code host's API, the LLM API, the video renderer) and write results back to the database, then notify the frontend (via polling or a lightweight realtime channel) when done.

---

## 2. Frontend

### 2.1 Framework: Next.js (React)
**Choice and why:** Next.js is the dominant framework for exactly this kind of product — a marketing site, an authenticated dashboard, and API routes, all in one codebase. Vercel's free tier includes 100GB bandwidth, 100 hours of function execution, and unlimited personal projects, which comfortably covers an early-stage SaaS. It also has the deepest free-tier integration with the database choice below.
**Limit to know:** the free tier is generous but is explicitly for "personal projects" per Vercel's terms — once Repona has real paying customers, budgeting for Vercel's paid tier (or migrating the API layer to a different free host) should be planned for, not assumed free forever. This is a "when it succeeds" problem, not a launch-blocking one.

### 2.2 UI Component System: shadcn/ui + Tailwind CSS
**Choice and why:** this directly answers the "premium SaaS feeling, but free" requirement. shadcn/ui has become the default component system for React and Next.js projects, with 75,000+ GitHub stars, and critically, it's not a runtime UI library you install — it's a set of source files copied directly into your project, meaning there's no vendor lock-in, no usage limits, and no paid tier at all for the core library. It produces a clean, modern look that doesn't read as a generic free template if used thoughtfully.
**Starter point:** rather than building every screen from a blank canvas, start from a free, MIT-licensed shadcn dashboard starter. shadcn-admin is a complete free starting point with 2,800+ stars and 10+ production-ready pages, which covers the sidebar, settings pages, and data table patterns Repona needs (project list, team management, billing) out of the box, leaving the actual diagram canvas and prompt interface as the main custom-built pieces.
**Premium-feel layer:** add subtle motion using Framer Motion (free, open-source, MIT-licensed) for page transitions, panel reveals, and the staged-progress states described throughout the user-flow document (Section 4.3, 5.2) — small, purposeful animation is one of the cheapest ways to make a free-stack product feel expensive, and it costs nothing but development time.

### 2.3 Diagram Canvas Rendering
**Choice and why:** the underlying diagrams are SVG-based (consistent with the open-source reference project explored earlier), so the canvas itself is rendered as inline SVG manipulated directly via React state — no paid diagramming library needed. For pan/zoom/node-dragging interactions specifically (Section 8.1, 8.4 of the user-flow document), use `react-zoom-pan-pinch` (free, MIT) for pan/zoom, and plain SVG drag handlers (or `@dnd-kit`, free, MIT) for manual node repositioning. This avoids depending on a paid diagramming SDK (e.g., GoJS, which has free trial limits, not a free production tier) entirely.

### 2.4 Animated Video Preview (Client-Side)
For the live "preview thumbnail reflecting playback settings" described in Section 14.2 of the user-flow document, use CSS/SVG animation directly in the browser (since the diagrams are SVG, animating `stroke-dashoffset`, opacity, and transform properties achieves the step-by-step/continuous-flow effect natively, with zero extra library cost) — this preview does not need to produce the final exported file, only a representative live preview; the actual export file is generated server-side (Section 8 below).

---

## 3. Backend & API Layer

### 3.1 Framework: Next.js API Routes / Route Handlers
**Choice and why:** keeping backend logic inside the same Next.js project (using Route Handlers, Next's built-in backend capability) avoids running and maintaining a second separate server/framework, which matters enormously for a solo developer's time. This is sufficient for Repona's needs — authentication, CRUD operations on projects/diagrams, and triggering background jobs.

### 3.2 Background Job Queue
**Choice and why:** repository scanning, diagram generation, and especially video rendering are all too slow to run inside a normal web request (which typically times out within seconds to a couple of minutes depending on host). A dedicated job queue is needed.
- **Recommended: Inngest (free tier)** — purpose-built for exactly this pattern (background jobs, retries, scheduled/recurring tasks like the drift-detection digest in Section 15.3 of the user-flow document), has a generous free tier suitable for an early-stage product, and integrates directly with Next.js with minimal setup.
- **Alternative if avoiding any third-party job service entirely:** a simple Postgres-backed job queue (e.g., using the `pgmq` extension or a hand-rolled "jobs" table polled by a worker process) — fully free with no external dependency, but requires you to build retry/failure logic yourself rather than getting it for free from Inngest. Recommended only if minimizing the number of external services is a higher priority than development speed.

### 3.3 Authentication
**Choice and why:** authentication needs to support email/password, GitHub OAuth, GitLab OAuth, Google OAuth, and 2FA (all specified in Section 2 of the user-flow document) without building this fragile, security-sensitive code from scratch.
- **Recommended: Supabase Auth** (bundled with the database choice below, see Section 4) — supports all the required providers, email verification flows, and session management out of the box, free at the tier Repona needs for a long time.
- This single choice also directly enables the separation required in Section 2.2 of the user-flow document — logging in via GitHub for *account identity* versus requesting GitHub *repository read access* later as a separate, explicit consent step — by treating the initial OAuth login and the later repository-scoped OAuth grant as two distinct authorization flows, not silently reusing one token for both purposes.

---

## 4. Data Layer

### 4.1 Primary Database: PostgreSQL via Supabase
**Choice and why, given the research:** Supabase is the better platform if your application needs three or more of auth, storage, realtime, and edge functions alongside Postgres, since it saves significant integration work — and Repona genuinely needs all four: auth (Section 3.3), file storage (Section 4.3, for exported diagrams/videos), realtime (for live notification delivery, Section 21), and a relational database for everything else. Choosing Supabase over assembling Neon + a separate auth provider + a separate storage provider directly serves the "one platform doing several things" principle for a solo developer.
**Honest tradeoff acknowledged:** Neon has the stronger integration story specifically for Next.js/Vercel and offers instant database branching even on its free tier, which is genuinely a better fit if the team ever grows and adopts a heavier CI/CD workflow with per-pull-request preview databases. For a solo developer at launch, this is a "nice to have later" rather than a launch-blocking need, so Supabase's bundled convenience wins for now. If Repona grows a multi-person engineering team later, revisit this decision specifically for the branching workflow benefit.
**Free tier limit to know:** Supabase's free tier includes 500MB of database storage, and the free tier database pauses after a week of inactivity (it un-pauses automatically on next access, so this mainly matters for a project that goes completely unused for a week, not for active users). 500MB is small but sufficient for thousands of users' worth of project metadata, decision notes, and Q&A history, since the heavy files (diagrams, videos) live in object storage, not the database itself (Section 4.3).

### 4.2 Database Schema (Core Tables)
A complete schema isn't fully fleshed out here since this document intentionally avoids low-level implementation, but the core entities every feature in the user-flow document maps onto are:
- `users`, `organizations` (or `teams`), `team_members` (with `role` enum: admin/editor/viewer, per Section 17.3)
- `projects` (with `type` enum: connected/prompt-based/multi-repo, per Section 7.2)
- `repositories` (linked to projects, storing the OAuth-derived access reference, not the raw token itself — see Section 9 on security)
- `diagrams` (linked to projects, with `diagram_type` enum: architecture/sequence/data-flow/lifecycle/dependency-graph, per Section 8.2)
- `diagram_versions` (full version history per Section 8.5, each row referencing the diagram it belongs to and an optional commit/PR reference)
- `decisions` (linked to a specific diagram component, with edit history per Section 10.3)
- `qa_messages` (conversation history per project, per Section 9)
- `comments` (linked to diagram components, per Section 17.4)
- `exports` (a record of generated static/video exports, their format, and their storage location)
- `notifications` (per Section 21, with `read` status and `category` for digest batching)
- `audit_logs` (for the AI-code-audit feature, Section 12, storing structural diffs and reviewer decisions)

### 4.3 File & Object Storage
**Choice and why:** Supabase Storage (bundled with the same platform as the database) for storing exported PNG/JPEG/WebP/SVG/PDF files and rendered MP4/WebM/GIF videos. Using the same platform avoids a second storage vendor and a second set of credentials/SDKs to manage.
**Free tier limit to know:** Supabase's free storage allowance is modest; video files in particular are large, so a practical mitigation is to set a retention policy (e.g., auto-delete exports older than 30 days unless explicitly pinned/saved by the user) rather than keeping every export forever — this is a sensible product decision anyway, not just a cost workaround, since most exports are shared once and not revisited.

### 4.4 Search
**Choice and why:** Section 18 of the user-flow document requires plain-language search across diagrams, decisions, Q&A history, and comments. Two viable free approaches:
- **Recommended for launch: PostgreSQL full-text search** (`tsvector`/`tsquery`, built directly into Postgres, so it's entirely free with no additional service) combined with `pgvector` (a free Postgres extension, supported natively by Supabase) for semantic/plain-language matching beyond exact keywords — this directly supports the "search works in plain language, not just exact keyword matches" requirement without adding a separate search infrastructure service.
- **If search quality with pgvector proves insufficient as the product grows:** Meilisearch (free, open-source, self-hostable) is the natural upgrade path, but this is realistically a post-launch optimization, not a v1 requirement.

### 4.5 Caching / Rate-Limiting Layer
**Choice and why:** Upstash Redis (free tier, serverless, pay-per-request beyond the free allowance) for short-lived data: rate-limiting API calls to external services (the code host, the LLM provider), caching expensive repeated computations, and managing realtime presence indicators (Section 22's "another team member is viewing this project"). Chosen specifically because it's HTTP-based and serverless-friendly, meaning it works cleanly from Vercel's serverless functions without the connection-pooling complications of a traditional Redis instance.

---

## 5. AI / LLM Layer (The Core Engine)

This is the layer that actually understands code and prompts, and generates diagrams, Q&A answers, decision suggestions, and audits. Every one of these features routes through this layer.

### 5.1 LLM Provider
**Choice and why:** Anthropic's Claude API (or another frontier LLM API) is the core dependency for nearly every intelligent feature: reading a codebase and describing its structure, interpreting a natural-language prompt into a diagram structure, answering Q&A questions, generating stakeholder-friendly summaries, and producing structural audit diffs.
**Honest cost note, since "free" has a real limit here:** unlike the rest of this stack, LLM API calls are not free at any meaningful volume — this is a genuine, ongoing operating cost that scales with usage, not a one-time setup choice. This is the single most important number to track once Repona has real users, since it directly determines per-user gross margin. Mitigate this with aggressive caching (don't re-analyze unchanged files on every diagram regeneration; only re-analyze what the webhook diff indicates actually changed) and by using a smaller/cheaper model tier for simpler tasks (e.g., a lightweight model for short Q&A answers) while reserving the most capable model tier for the harder task of full repository structural analysis.

### 5.2 Code-to-Diagram Pipeline (Feature 4.1 in the product document)
1. Webhook fires on push/merge (Section 4.4 of the user-flow document) → background job triggered (Section 3.2).
2. Worker fetches the diff (only changed files, not the whole repository every time, to control LLM cost) via the code host's API.
3. Worker sends the diff plus relevant existing structural context to the LLM, requesting a structured JSON output describing components, relationships, and changes (not raw SVG/HTML directly from the LLM — this is the more defensible, structured approach flagged earlier when evaluating the open-source reference project, since a stable JSON intermediate representation allows precise, non-drifting edits rather than the LLM re-describing the whole diagram from scratch every time).
4. A deterministic renderer (custom-built, not LLM-generated) takes that JSON and produces the actual SVG diagram, applying the established visual/color system (Section "Semantic configuration" patterns referenced in the original reference repo) consistently — this guarantees visual consistency across regenerations, which an LLM directly generating raw SVG every time cannot reliably guarantee.
5. Result saved as a new `diagram_version` row (Section 4.2) and the frontend is notified (Section 21).

### 5.3 Prompt-to-Diagram Pipeline (Feature 4.14)
Same JSON-intermediate-representation approach as above, but the input is the user's natural-language prompt (plus prior conversation history for iterative refinement, Section 5.3 of the user-flow document) instead of a code diff. This shared intermediate format is also exactly what makes the "merge point" (Section 6 of the user-flow document, upgrading a prompt-based diagram into a connected one) clean to implement — both pipelines converge on the same JSON shape, so reconciling or comparing a prompt-based diagram against a code-derived one is a structural comparison, not an apples-to-oranges problem.

### 5.4 Plain-Language Q&A (Feature 4.2)
Standard retrieval-augmented generation: the user's question, relevant diagram JSON, relevant decision notes, and relevant code excerpts (fetched via the search layer in Section 4.4) are assembled into context and sent to the LLM for an answer. The "I don't see anything related to that" honest-uncertainty behavior specified in Section 9.3 of the user-flow document is implemented by explicitly instructing the model to say so when retrieved context is empty or irrelevant, rather than letting it guess.

### 5.5 AI-Generated Code Auditing (Feature 4.5)
The structural diff described in Section 12.2 of the user-flow document is computed in two layers: a deterministic layer (parsing the actual diff for new import statements, new external API calls, new environment variables — language-aware static analysis, not LLM-based, for accuracy on factual structural facts) combined with an LLM layer that interprets *significance* (which changes are risky vs. trivial) and writes the plain-language summary. Keeping the factual detection deterministic and only using the LLM for interpretation avoids the LLM hallucinating structural facts that a simple parser can determine reliably and for free.

### 5.6 Stakeholder View Generation (Feature 4.6)
A separate LLM prompt, given the same diagram JSON, specifically instructed to simplify and remove jargon, producing both a simplified diagram JSON variant and an accompanying plain-language summary paragraph.

---

## 6. Code Host Integrations

### 6.1 GitHub, GitLab, Bitbucket APIs
All three offer free-tier API access sufficient for Repona's needs (reading repository contents, commit history, registering webhooks) — there is no paid tier required here at any realistic usage volume for an early-stage product, since these are standard developer-facing APIs with generous rate limits for OAuth-authenticated apps.
**Security note (ties into Section 9):** store only the OAuth access/refresh tokens needed for API calls, scoped to the minimum permissions required (read-only repository access plus webhook management), never broader account permissions than necessary, directly supporting the per-repository revocation requirement in Section 20.3 of the user-flow document.

### 6.2 Webhooks
Standard webhook registration via each provider's API, pointed at a dedicated Repona API route that verifies the webhook signature (each provider supports this, preventing forged webhook calls), then enqueues a background job (Section 3.2) rather than processing inline.

---

## 7. Notifications & Realtime

### 7.1 In-App Notifications
Supabase Realtime (bundled with the database choice in Section 4.1, built on Postgres logical replication, included free) — when a new row is inserted into the `notifications` table, connected clients receive it instantly without polling, powering the notification bell described in Section 21.1 of the user-flow document.

### 7.2 Email Notifications
**Choice and why:** Resend (generous free tier, modern API, straightforward Next.js integration) for transactional emails — verification emails, password resets, digest summaries, team invites. Chosen over older providers (e.g., SendGrid) specifically for its developer experience and because its free tier is sufficient for an early-stage product's email volume.

### 7.3 Digest Batching
Implemented as a scheduled background job (via the job queue in Section 3.2, using its recurring/cron-style scheduling capability) that runs daily/weekly per user preference (Section 15.3, 21.3 of the user-flow document), querying unread notifications in batchable categories and sending a single summary email rather than one email per event.

---

## 8. Export Pipeline (Static + Animated Video)

### 8.1 Static Exports (PNG, JPEG, WebP, SVG, PDF)
Since diagrams are SVG-based, static raster exports (PNG/JPEG/WebP) are generated using the same client-side canvas-rendering approach already proven in the open-source reference project explored earlier (serializing the SVG, rendering it onto an HTML canvas at high resolution, then exporting as a blob) — this can run entirely client-side for on-demand exports, costing nothing server-side. PDF export additionally uses a lightweight free library (e.g., `jsPDF`, MIT-licensed) to wrap the rendered image or vector content into a PDF container.

### 8.2 Animated Video Export — The One Genuinely Hard Piece
This is the most technically involved feature in the entire product, and deserves the most careful tool choice, given what the research surfaced.
**Important finding to flag directly:** Remotion is the most well-known React-based video rendering tool, but Remotion is free to use for individuals and companies up to three people, while a Company License is required for teams of four or more, starting at $100/month minimum spend. Since Repona is being built as a SaaS intended to grow, this is a real constraint to plan around honestly rather than discover later — adopting Remotion now means accepting a future paid obligation once the team (not the customer base — the company itself) grows past three people, or once render volume is automated at scale, since the Automators tier (for automated, customer-facing rendering) is priced at $0.01 per render with a $100/month minimum regardless of team size, and this *does* apply to Repona's use case, since Repona would be rendering videos programmatically on behalf of its own customers, not for internal one-off use.
**Recommended approach instead: FFmpeg-based rendering, fully free regardless of scale.** The pipeline:
1. The diagram's animation (step-by-step reveal or continuous flow, per Section 14.2 of the user-flow document) is defined the same way the live browser preview works (Section 2.4) — as a sequence of SVG states or a continuous SVG animation timeline.
2. A headless browser (Playwright, free and open-source) loads the animated SVG/HTML and captures it frame-by-frame at the target framerate, OR captures it as a screen recording over the animation's real-time duration.
3. FFmpeg (free, open-source, no licensing fee regardless of company size or usage volume) assembles the captured frames into the final MP4/WebM/GIF, handles the loop option, and burns in captions (Section 14.2's caption requirement) using FFmpeg's subtitle-overlay capability.
4. This entire pipeline runs as a background job (Section 3.2), given it's the slowest operation in the product, consistent with the "rendering is a background process, user is notified when ready" requirement in Section 14.2 of the user-flow document.
**Honest tradeoff:** this approach requires more custom engineering than simply learning Remotion's API, since you're assembling the frame-capture-and-encode pipeline yourself rather than using a framework built for exactly this. But it has zero licensing ceiling tied to company size or render volume, which directly matters for a SaaS product whose whole purpose is offering this feature to many customers at scale — the cost that does scale (server compute time for headless browser rendering) is a hosting cost, not a licensing fee, and is controllable by the same job-queue infrastructure already in place.

---

## 9. Security & Compliance Considerations

- **OAuth token storage:** encrypted at rest (Supabase supports column-level encryption, or encrypt before storage using a server-side secret) — repository access tokens are sensitive and must never be exposed to the frontend or logged in plaintext anywhere.
- **Webhook signature verification:** mandatory on every incoming webhook (Section 6.2) to prevent forged requests from triggering fake scans or job floods.
- **Role-based access enforcement:** every API route that touches project data must check the requesting user's role (Section 17.3 of the user-flow document) server-side — never trust a frontend-only permission check, since that's trivially bypassable.
- **Rate limiting:** on both Repona's own API (preventing abuse) and on outbound calls to the code host APIs and the LLM provider (preventing accidental cost spikes from a runaway loop or a malicious actor hammering the prompt endpoint) — implemented via the Redis layer in Section 4.5.
- **Data export & deletion compliance:** Sections 20.4 and 23 of the user-flow document (data export, account deletion) aren't just UX niceties — they're the baseline expected for handling user data responsibly (and are required outright if Repona ever has EU users, under GDPR's right to access and right to erasure). Building these flows in from the start is far easier than retrofitting them after launch.
- **Least-privilege repository access:** request only read access to repository contents and webhook management — never request write access, since Repona never needs to modify a user's code, which is itself both a security best practice and a trust-building detail worth stating clearly to users during the OAuth consent flow (Section 4.1 of the user-flow document already specifies this kind of transparency).

---

## 10. Payments & Billing

### 10.1 Provider: Paddle
**Choice and why:** Paddle is the best Stripe-like option for global subscription SaaS when you want a hosted, polished billing experience plus far less tax/compliance overhead. It supports the subscription billing model (Section 19 of the user-flow document) with built-in dunning/retry handling for failed payments (covering the "grace period" requirement in Section 19.2) and self-serve cancellation through its hosted customer billing portal (covering Section 19.3 with minimal custom billing UI work).
**Integration note:** use Paddle's hosted checkout and its customer billing portal rather than building subscription management UI from scratch. Treat Paddle webhooks as the source of truth for subscription state changes (new subscription, renewal, payment failure, cancellation, plan changes), and update Repona entitlements in the database accordingly.

---

## 11. Analytics & Monitoring (Operational, Not Optional)

A solo developer needs to know what's actually happening in production without manually checking — these are not "extra polish," they're how you'll know if something breaks at 2am or if a feature nobody uses is wasting your LLM budget.

- **Error tracking:** Sentry (free tier sufficient for early-stage volume) — catches and reports unhandled errors in both frontend and backend, directly supporting the "clear retry option rather than silently failing" requirement throughout Section 22 of the user-flow document, since you can't fix a silent failure you don't know happened.
- **Product analytics:** PostHog (free tier, generous event volume) — tracks which features are actually used (critical for deciding what to build next, and for noticing if, say, video export is rarely used despite being expensive to run) and supports basic funnel analysis (signup → first diagram → first export → upgrade).
- **Uptime monitoring:** a free tier of a simple uptime checker (e.g., UptimeRobot's free plan) pinging the production app periodically, so you find out about an outage from a monitor, not from a customer complaint.

---

## 12. Mapping Every Feature to Its Technical Implementation

A direct cross-reference, so nothing from the product document is left unaddressed:

| Feature (from product doc) | Primarily implemented via |
|---|---|
| 4.1 Self-updating diagrams | Webhooks (Section 6.2) + job queue (3.2) + LLM pipeline (5.2) |
| 4.2 Plain-language Q&A | RAG pipeline (5.4) + Postgres/pgvector search (4.4) |
| 4.3 Decision capture | Core Postgres tables (4.2), no AI needed — pure CRUD |
| 4.4 Onboarding mode | Sequencing logic generated once via LLM (5.2's JSON output ordering), then stored and tracked as plain progress data (4.2) |
| 4.5 AI code audit | Deterministic static analysis + LLM interpretation (5.5) |
| 4.6 Stakeholder view | Dedicated LLM prompt (5.6) on existing diagram JSON |
| 4.7 CI/workflow integration | Webhooks (6.2) + structural-change-without-decision-note checks (background job logic) |
| 4.8 Searchable knowledge base | Postgres full-text + pgvector (4.4) |
| 4.9 Team collaboration | Core tables + role checks (4.2, 9) + Supabase Realtime for live comment updates (7.1) |
| 4.10 Drift detection | Webhook-triggered comparison job (3.2, 15) |
| 4.11 Multi-repo support | Multiple `repositories` rows per `project` (4.2) + LLM cross-repo relationship inference (5.2 extended) |
| 4.12 Export & sharing | Static: client-side canvas (8.1). Video: FFmpeg/Playwright pipeline (8.2) |
| 4.13 First-run onboarding | Pure frontend flow (Section 2, 3 of user-flow doc), no special backend |
| 4.14 Prompt-based generation | Same LLM + JSON-intermediate pipeline as 4.1, fed a prompt instead of a code diff (5.3) |

---

## 13. Recommended Build Order (Phased)

Building everything simultaneously isn't realistic for a solo developer. This order prioritizes reaching a demonstrable, shareable product as early as possible.

**Phase 1 — Foundation (no AI yet):**
Auth (3.3), core database schema (4.2), basic project dashboard (Section 7.1 of user-flow doc), billing skeleton (Section 10) wired but not necessarily enforced yet.

**Phase 2 — Prompt-to-diagram, the fastest path to something demoable:**
The prompt input UI (Section 2.4), the LLM prompt-to-JSON pipeline (5.3), the deterministic JSON-to-SVG renderer (5.2 step 4), static export (8.1). This alone is a complete, shareable product slice with zero repository/webhook complexity — deliberately sequenced first because it has the fewest moving parts and the fastest path to user feedback.

**Phase 3 — Connect-a-repository path:**
OAuth repo-scoped consent (4.1 of user-flow doc), repository selection and initial scan (4.2–4.3), webhook registration (6.2), the code-diff-to-diagram pipeline (5.2).

**Phase 4 — The features that create retention and justify a subscription:**
Plain-language Q&A (5.4), decision capture (4.2's `decisions` table + UI), version history (Section 8.5 of user-flow doc), drift detection (15).

**Phase 5 — Differentiation and premium-tier features:**
Stakeholder view (5.6), AI code audit (5.5), animated video export (8.2 — deliberately last, since it's the most engineering-intensive single feature and not needed to validate the core product).

**Phase 6 — Scale-readiness:**
Multi-repo support (4.11), team collaboration and roles enforcement (9, Section 17 of user-flow doc), onboarding mode (4.4), analytics/monitoring hardening (Section 11).

---

## 14. Honest Summary of Ongoing (Non-Free) Costs

Everything in this stack is free to start, but two categories will eventually cost real money as Repona grows, and pretending otherwise would be misleading:

1. **LLM API usage** (Section 5.1) — scales directly with user activity. This is the most important number to monitor from day one.
2. **Hosting/compute beyond free tiers** (Vercel function execution, Supabase database/storage, background job/video-rendering compute) — scales with user count and usage, but free tiers (per the research throughout this document) comfortably cover early-stage usage before any of this becomes a real expense.

Everything else in this document — the frontend framework, UI components, database, auth, search, payments processing setup, error tracking, and the FFmpeg-based video pipeline specifically chosen to avoid Remotion's company-size licensing trigger — has no inherent cost ceiling tied to your success. The two costs above are the only ones to actively watch as Repona grows.
