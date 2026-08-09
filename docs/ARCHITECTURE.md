# Samplerepo — Architecture

> **Living documentation** — Generated from your connected repository and kept in sync via Arcobay webhooks. Commit this file for a repo-local snapshot; the [live diagram](http://localhost:3000/share/RKOV9nnw-T9U9cV4o-uGvkhseXvYlPKf) always reflects the latest codebase.

_Exported Aug 9, 2026, 9:16 AM. Re-export from Arcobay after major changes to refresh this file._

## Overview

Next.js on Vercel with Supabase data/auth, Inngest workers for repo scans and exports, Claude for AI, and Paddle billing — derived from project documentation.

[![Samplerepo architecture — live architecture diagram](https://img.shields.io/badge/Arcobay-live%20diagram-111?label=Architecture)](http://localhost:3000/share/RKOV9nnw-T9U9cV4o-uGvkhseXvYlPKf)

![Samplerepo architecture — build animation](architecture.gif)

![Samplerepo architecture](architecture.png)

[Open interactive diagram in Arcobay](http://localhost:3000/share/RKOV9nnw-T9U9cV4o-uGvkhseXvYlPKf)

## Components

### End User

**Type:** external

Shopper

### Webhooks

**Type:** gateway

Push and merge event…

### pgvector

**Type:** database

Semantic search exte…

### Next.js App

**Type:** frontend

Web app

### Supabase Auth

**Type:** security

Authentication

### Inngest

**Type:** queue

Background jobs, ret…

### Background Workers

**Type:** service

Repo scans, diagram…

### Supabase Storage

**Type:** storage

Exported diagrams, v…

### Anthropic Claude API

**Type:** external

LLM for diagrams, Q&…

### Paddle

**Type:** external

Payments

### Resend

**Type:** external

Transactional email…

### Sentry

**Type:** external

Error monitoring and…

### Anthropic

**Type:** external

### Supabase

**Type:** database

Postgres

## Connections

- End User → Next.js App — _Uses app_
- Next.js App → Supabase Auth — _Authenticates via_
- Supabase Auth → Supabase — _Manages users_
- Next.js App → Supabase — _Reads/writes_
- Next.js App → Supabase Storage — _Stores exports_
- Next.js App → pgvector — _Semantic search_
- Next.js App → Anthropic Claude API — _AI requests_
- Next.js App → Paddle — _Billing_
- Webhooks → Next.js App — _Verified webhook_
- Next.js App → Inngest — _Enqueues jobs_
- Inngest → Background Workers — _Runs jobs_
- Background Workers → Anthropic Claude API — _Analyzes code_
- Background Workers → Supabase — _Saves version_
- Next.js App → Resend — _Transactional email_
- Inngest → Resend — _Digest emails_
- Next.js App → Sentry — _Reports errors_
- Next.js App → Background Workers — _calls_
- Background Workers → Paddle — _creates transaction_
- Background Workers → Anthropic — _integrates_
- Background Workers → Supabase

## Design decisions

_No decision notes captured yet. Add notes in Arcobay to document why components exist._

---

Maintained with [Arcobay](https://arcobay.com) — living architecture documentation aligned with your codebase.
