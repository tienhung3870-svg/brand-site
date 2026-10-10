# Building the SEO Topic Cluster Planner: Architecture, 272 Tests, and 3 Live Data Sources

*Author: Hung Pham*  
*Date: October 2026*  
*Project: [SEO Topic Cluster Planner](https://seo-topic-cluster-planner.pages.dev/)*  
*Site & Portfolio: [phamtienhung.me](https://phamtienhung.me)*  

---

## 1. Why I Built It

Planning content for search engines usually starts in a chaotic spreadsheet. Writers and strategists list target keywords, guess internal linking structures, and paste URLs into rows without any way to see how articles actually connect. Traditional SEO suites solve part of this, but they come with heavy baggage: costly monthly subscriptions, closed ecosystems, and the requirement to upload confidential editorial roadmaps to external third-party databases months before publication.

I wanted a specialized, local-first browser tool where creators can visually construct topic clusters, enforce strict internal linking architectures (Pillar ↔ Spoke, parent ↔ child), and evaluate keyword potential—all without leaking unpublished content strategies.

Key design principles from day zero:
- **Local-First & Zero-Leakage:** All documents, keywords, and relationship graphs are stored directly in the browser's IndexedDB. No user content or unpublished drafts ever leave the user's machine unless explicitly exported.
- **Structural Integrity:** Enforce strict link validation rules (preventing circular parent-child dependencies, warning about missing links to children, and ensuring self-canonical hygiene) before a single word of copy is written.
- **Fast Interactive Visualization:** A full visual graph canvas capable of rendering multi-tier topic topologies with automatic layout.

The result is the **SEO Topic Cluster Planner**, deployed on Cloudflare Pages with client-side persistence and zero requirement for cloud logins to start planning.

---

## 2. The Stack

The architecture was chosen to balance extreme client-side speed, strict type safety, and zero backend maintenance overhead:

| Layer | Technology | Rationale |
|---|---|---|
| **Framework & Build** | Vite + React 19 + TypeScript | Instant HMR during development, sub-second production builds, and end-to-end type safety across domain entities. |
| **Interactive Canvas** | `@xyflow/react` (React Flow) + Dagre | Flexible node-based canvas for exploring topic clusters, paired with Dagre's directed graph layout engine for automated hierarchical arrangement. |
| **State & Command Pattern** | Zustand | Lightweight, decoupled reactive state stores. Every canvas edit and article modification is dispatched through a command pattern, giving users full multi-level undo/redo. |
| **Schema & Validation** | Zod 4 | Strict runtime schema parsing for `ProjectDocument`, `Article`, `InternalLink`, and `ValidationIssue`. Catches document corruption and schema drift before persistence. |
| **Persistence** | IndexedDB via `idb` | Zero-latency local storage supporting full versioned snapshot backups, project duplication, and offline recovery. |
| **Edge Functions** | Cloudflare Pages Functions | Lightweight serverless proxy endpoints (`/api/suggest`, `/api/paa`, `/api/bing`) handling rate limiting, CORS headers, and secure API key isolation. |
| **Testing** | Vitest + `@playwright/test` | Lightning-fast unit tests with native V8 coverage reporting, paired with robust browser-level end-to-end test execution. |

---

## 3. Test-Driven Verification: 249 Unit Tests + 23 E2E Tests

I built the planner following Test-Driven Development (TDD). Because the application manages complex directed acyclic graph (DAG) invariants, link checklists, and multi-source keyword math, testing wasn't an afterthought—it drove every interface contract.

### Unit & Domain Test Suite (249 Tests)
The unit test suite covers core domain mechanics across 23 test suites:
- **URL & Domain Invariants:** Strict HTTP(S) validation, preventing domain mismatches, rejecting malformed userinfo inputs, and validating canonical rules.
- **Graph Topology & Structural Checks:** Automated cycle detection, hierarchy tier calculation (Pillar = Tier 1, child = parent depth + 1), and validation of missing parent-to-child reciprocal links.
- **Merge Algorithms & Scoring:** Comprehensive testing of keyword deduplication, provenance tracking, and confidence score calculation.
- **Edge Functions & Parsers:** Unit tests verifying JSON and XML response parsers for Google Autocomplete, People Also Ask (PAA), and Bing Webmaster API endpoints under varying network payloads and HTTP status codes.

### Browser End-to-End Suite (23 Playwright Tests)
The 23 Playwright E2E specs verify complete end-to-end workflows in real Chromium environments:
1. **Keyword Research & CSV Import:** Uploading CSV keyword batches, inspecting provenance, and asserting table rendering.
2. **Interactive Canvas Operations:** Adding articles, connecting edges, expanding/collapsing subtrees, and triggering automated Dagre re-layouts.
3. **Multi-Source Keyword Research Panel:** Submitting queries, viewing multi-source metrics, inspecting provenance drawers, and exporting aggregated CSVs.
4. **Validation Checklist & Issues Drawer:** Triggering structural warnings and verifying interactive resolution links.
5. **Brief & Markdown Export:** Generating and downloading formatted article briefs and content guidelines.
6. **Cloud Fallback & Offline Resilience:** Testing graceful degradation when cloud backend services are unavailable or unconfigured.

All 23 E2E tests execute deterministically against the preview build.

---

## 4. Three Live Volume & Keyword Signals

Search volume from a single source is notoriously unreliable or heavily delayed. To provide actionable intelligence while respecting user privacy, the planner combines three live data signals:

```
+-------------------------------------------------------------------+
|                        Client-Side Planner                        |
|   (src/features/keywords/sources/research.ts + MultiSourcePanel)   |
+---------------------------------+---------------------------------+
                                  |
               +------------------+------------------+
               |                  |                  |
               v                  v                  v
       [Google Suggest]    [Google Trends]     [Bing Proxy]
       /api/suggest        Direct / Relative   /api/bing (Cloudflare)
               |                  |                  |
               +------------------+------------------+
                                  |
                                  v
                    +---------------------------+
                    |  MultiSource Merge Engine  |
                    |   (src/keywords/merge.ts)  |
                    +---------------------------+
                                  |
                                  v
                      Unified Keyword Decision
                      - Weighted Volume Score
                      - Provenance & Intent
                      - Disagreement Flags
```

### 1. Google Suggest (Autocomplete API)
Queried via `/api/suggest` through Cloudflare Pages Functions. It captures real-time searcher query completion patterns and prefix expansions directly from Google's suggestion endpoint, returning immediate long-tail variants.

### 2. Google Trends
Analyzes normalized search momentum, seasonal interest fluctuations, and relative topic velocity. This reveals whether a topic is surging, evergreen, or decaying.

### 3. Bing Webmaster Proxy
Implemented in `functions/api/bing.ts`. Because calling the Bing Webmaster API directly from the browser would expose private credentials and trigger CORS errors, Cloudflare Pages Functions acts as an authenticated proxy:
- Injects the server-side `BING_WEBMASTER_API_KEY` secret.
- Enforces strict IP-based rate limiting (10 requests per minute per IP).
- Enforces locale constraints (`en*` queries validated; missing credentials return graceful 503 status codes).

### The Multi-Source Heuristic Merge Engine
The core logic in `src/features/keywords/merge.ts` unifies these feeds into an actionable consensus:
- **Base Weighting:** Bing direct query counts receive a high baseline weight (`0.8`).
- **Relative Trends Score:** Trends-only signals receive a relative confidence weight (`0.4`).
- **Co-occurrence Boost:** Keywords confirmed by both Autocomplete and Trends (score > 30) are boosted to `0.6` and flagged as `likely_searched`.
- **Automated Disagreement Flagging:** If Bing indicates zero volume while Google Trends indicates high momentum (> 50), the system flags a `disagreement`, prompting human editorial review rather than masking the discrepancy.

---

## 5. Lessons Learned

Building and testing this tool surfaced several non-obvious engineering challenges:

### Lesson 1: Cache Hygiene — Never Cache Transient Failures
In our initial implementation of client-side caching (`src/features/keywords/sources/cache.ts`), API responses were stored in IndexedDB with a 7-day TTL. When testing an environment where the Bing API key had not yet been provisioned, the endpoint returned HTTP 503. The caching layer mistakenly cached this 503 response. As a result, even after the valid API key was configured, downstream queries remained locked in an "UNKNOWN" state until cache expiry.  
**Rule established:** Only cache verified HTTP 200 responses. Never persist 5xx errors or transient rate-limit responses into multi-day storage.

### Lesson 2: Build-Time Environment Leakage into Preview Bundles
During local development, `vite build` bundled environment variables from `.env.local` into the production output bundle. When Playwright launched `vite preview` to run the `cloud-fallback.spec.ts` test (which asserts that the UI shows appropriate fallback messaging when cloud sync is unconfigured), the test unexpectedly failed because the app detected the baked-in credentials and considered itself configured.  
**Fix:** We decoupled E2E testing into a dedicated test mode (`mode: e2e`) that guarantees an empty environment configuration, ensuring offline and fallback assertions remain 100% reproducible.

### Lesson 3: Mocking 3rd-Party Endpoints in E2E Suites
Relying on live external search engine APIs during CI/E2E test runs leads to flaky builds, IP throttling, and non-deterministic assertions. We decoupled browser tests by using Playwright's `page.route` to mock edge API responses with realistic fixtures, while reserving real network testing for isolated unit and smoke tests. This delivered fast, 100% reliable test runs.

---

## 6. What's Next

The tool is live at [seo-topic-cluster-planner.pages.dev](https://seo-topic-cluster-planner.pages.dev/). Current work focuses on expanding keyword provenance filters, adding deeper content brief export templates, and integrating semantic clustering algorithms directly in WebAssembly.

All code and architecture decisions are governed by verified local test evidence.
