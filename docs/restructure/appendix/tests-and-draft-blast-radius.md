# Test inventory + OrderDraft blast radius (read-only study)

Repo: `...\scratch-2026-11-16-d96b4e\AIresponse` (all paths below are relative to it; line numbers are 1-based as read on 2026-11-16).
Nothing in the repo was modified; no tests were run. Where something was inferred and not executed it is marked **UNVERIFIED**.

Method notes
- Test counts = number of lines matching `^\s*(it|test)(\.each)?\(` in each file. `it.each([...])` tables count as **1** (so true run-time counts are higher), and `describe.each` blocks in `theme-system.test.ts` multiply at run time (COMBOS x ~5 `it`s). Textual total: **99 files, 1355 `it(` calls**.
- "Imports" lists the `@/` (= `src/`) modules each file imports or `vi.mock`s (shortened: `m/` = `src/modules/`, `c/` = `src/components/`, `l/` = `src/lib/`, `core/` = `src/core/`).
- `tests/helpers/` also contains `json-schema.ts` (draft-07 mini validator), `oms-order-parser.ts` (OMS input parser used by contract tests), `theme-css.ts` (CSS-variable palette + contrast maths used by theme-system).
- AGENTS.md mentions `tests/contracts-*.test.ts`, `tests/p12-agent-door.test.ts`, `src/modules/oms/` and `src/modules/agent-door/`; **none of those exist** in this checkout (tests/ has 99 test files, none with those names; `src/modules/` has no `oms` or `agent-door`). Treat AGENTS.md as partially stale.

---------------------------------------------------------------------
## PART 1 — All test files

Legend for classification: UI = UI-STRUCTURE (reads source/CSS/markup), DRAFT = ORDER-DRAFT, ORD = ORDER-OTHER, AI = AI-PIPELINE, CH = CHANNELS, SEC = SECURITY/RBAC, AN = ANALYTICS, OTH = OTHER. "(+DRAFT n)" means the file is mainly something else but n of its tests touch OrderDraft/handleOrderIntent directly. "(UI part)" means only a minority of the file is structural.

| # | File | `it` | Covers (one line) | src modules imported | Class |
|---|---|---|---|---|---|
| 1 | accounts-management.test.ts | 10 | createMember, resetMemberPassword, own-profile update | m/auth/password, m/auth/profile, m/members/service | SEC |
| 2 | ai-custom-endpoint-guard.test.ts | 4 | custom-provider private-network allow (server env only) | m/ai/endpoint-guard | SEC |
| 3 | ai-custom-provider.test.ts | 4 | workspace custom OpenAI-compatible provider | core/crypto, m/ai/provider, m/ai/router | AI |
| 4 | ai-mode-persona.test.ts | 19 | conversation stays AI_ACTIVE, page-level persona, FB name/avatar chain, delete isolation, HUMAN_ACTIVE blocks AI (#7+25) | channel-persona, orchestrator, provider, fb client/profile, conversations/service, handoff-service (createHandoff) | AI |
| 5 | ai-product-image-followup.test.ts | 1 | product photo sent on follow-up ask | orchestrator, provider, fb client, conversations/service | AI |
| 6 | ai-settings-connection.test.ts | 6 | AI settings connection modes + GLM preset | l/ai-connection | AI |
| 7 | api-handler.test.ts | 5 | createHandler + central error mapping | core/api/handler, core/errors | SEC |
| 8 | audit-fixes.test.ts | 19 | provider, pipeline+worker, memory, dedup+takeover, media, FB isolation fixes | guardrails, memory, orchestrator, provider, fb webhook/client, outbound, conversations, jobs, media, products | AI |
| 9 | brand-logo.test.ts | 7 | BrandLogo markup, theme hooks, icon.svg path parity, islamic/night CSS rules | c/brand-logo (+ reads src/app/icon.svg, globals.css) | **UI** |
| 10 | catalog-scoped-intake.test.ts | 7 | page catalog allow-list gates ORDER intake (draft + handleOrderIntent) | orders/draft-service, orders/intelligence, products/service | **DRAFT** |
| 11 | collection-intel-fields.test.ts | 10 | configurable collection fields + progress | conversations, intelligence/{fields,service,settings} | OTH |
| 12 | collection-intel-lab.test.ts | 17 | COLLECTION_ONLY mode, extraction without invention, isolation, OpenRouter errors | ai/provider, outbound, send-policy, intelligence/*, jobs/handlers | OTH (customer-intel; AI-adjacent) |
| 13 | conversation-purge.test.ts | 3 | purge thread + memory atoms; orders survive unlinked | conversations/service (+ db.orderDraft rows) | OTH (+DRAFT 1) |
| 14 | crypto.test.ts | 4 | encrypt/decrypt secrets | core/crypto | SEC |
| 15 | customer-media-send.test.ts | 8 | image selection once/conversation, send dispatch | crypto, fb/wa client, outbound, media/service | CH |
| 16 | durable-processing.test.ts | 21 | durable AI jobs, failure classification, crash/stale recovery, orphan sweeper, idempotent retry | orchestrator, provider, router, fb client, outbound, conversations, jobs/* | AI |
| 17 | empty-next.test.ts | 4 | EmptyNext component + 13 empty-state pages must carry `action=` + EmptyNext hrefs exist in nav | c/ui/empty-state, l/nav-model | **UI** |
| 18 | errors.test.ts | 4 | error hierarchy, auth zod schemas | core/errors, auth/schemas | SEC |
| 19 | facebook-oauth.test.ts | 3 | FB OAuth authorize URL | fb/oauth | CH |
| 20 | facebook-send-profile.test.ts | 10 | official send transport, per-page recipient, graph errors, profile name chain | crypto, fb client/profile, outbound, conversations | CH |
| 21 | facebook-subscription.test.ts | 6 | page webhook subscription | fb client/types | CH |
| 22 | facebook-webhook-batching.test.ts | 7 | per-message identity, batched POST loses nothing | fb normalizer/webhook, conversations | CH |
| 23 | faqs-list-route.test.ts | 2 | GET /api/faqs workspace-scoped (route-test pattern, see Part 4) | app/api/faqs/route (mocks prisma, auth/context, next/headers) | SEC/OTH |
| 24 | fluent-design.test.ts | 9 | design tokens + shell structure pinned by string match | (reads globals.css, tailwind.config.ts, app-sidebar, app-shell, ui/button, products/page) | **UI** |
| 25 | forensic-reliability.test.ts | 10 | FB transient/permanent classification, profile retry, orphan sweep window, media URL | core/env, provider, fb client/profile/types, conversations, jobs, media | CH |
| 26 | guided-tour.test.ts | 7 | tour steps = NAV groups + `data-tour` hooks; driver.js pinned; tour CSS vars only | l/nav-model, l/guided-tour (reads app-shell, app-sidebar, package.json, globals.css) | **UI** |
| 27 | hermes-endpoint-guard.test.ts | 16 | external-provider SSRF guard / DNS / redirects | ai/endpoint-guard, ai/provider | SEC |
| 28 | hermes-provider.test.ts | 25 | external provider normalization, resolution, mapping, fallback, "no invented prices/stock" | ai/context-builder, guardrails, provider, router | AI |
| 29 | inbox-conversations.test.ts | 15 | inbox identity, outbound delivery, soft delete, AI continuity | c/inbox/types, crypto, l/i18n, ai/memory, orchestrator, provider, fb client, outbound, conversations | AI |
| 30 | knowledge-e2e.test.ts | 16 | knowledge docs ingestion, extraction, semantic search, embeddings | errors, jobs/handlers, knowledge/* | OTH (knowledge) |
| 31 | labels-arabic.test.ts | 2 | every displayed Prisma enum value has Arabic label in `LABELS` (+ order audit labels) | l/i18n (reads prisma/schema.prisma) | **UI** (labels) |
| 32 | master-reliability.test.ts | 17 | orphan-sweeper starvation, AI_DISABLED reprocessable, job idempotency, partial batch failure, batching | channel-persona, orchestrator, persona-limits, prompts, provider, fb client/webhook, conversations, jobs/* | AI |
| 33 | media-channel-limits.test.ts | 6 | mediaFitsChannel per channel | media/channel-limits | CH |
| 34 | media-e2e.test.ts | 13 | upload/storage, product linking, AI prompt wiring | errors, ai/prompts, media/*, products/service | OTH (media) |
| 35 | media-public-url.test.ts | 10 | signed public media URLs + private endpoint | crypto, outbound, media/*, products | OTH (media) |
| 36 | media-request.test.ts | 5 | dialect "wants media" detection | media/request | OTH |
| 37 | multi-order-flow.test.ts | 7 | many orders per conversation; edit/cancel placed order from chat | orders/draft-service, orders/intelligence, orders/order-service, products/service | **DRAFT** |
| 38 | nav-search.test.ts | 9 | page search + keywords; app-shell combobox markup | l/nav-model, l/nav-search (reads app-shell.tsx) | **UI** |
| 39 | nav-structure.test.ts | 24 | NAV_GROUPS integrity vs `src/app/(dashboard)` filesystem, redirects, permissions, GroupTabs markup | c/group-tabs, c/nav-icons, l/nav-model, rbac/permissions | **UI** |
| 40 | ocr-wave1.test.ts | 19 | platform-key leak, order pricing in cents, batch failure, range validation, embeddings fail-closed, job timeout | analytics/time, crypto, errors, ai/router, inbound-job, jobs, embeddings, media, members, **orders/pricing** | OTH (+ORD 4) |
| 41 | ocr-wave2.test.ts | 11 | atomic tx helper, runPool, channel secrets at rest | crypto, secret-at-rest, webhook-tx, util/pool | CH/OTH |
| 42 | order-delete.test.ts | 6 | deleteOrder owner-only | errors, orders/order-service | **ORD** |
| 43 | p1-channel-registry.test.ts | 12 | channel registry, FB adapter, no silent mock | crypto, fb adapter/client/normalizer/webhook, mock-adapter, outbound, registry | CH |
| 44 | p1-pgvector-sql.test.ts | 3 | pgvector SQL quoting | knowledge/vector-store | OTH |
| 45 | p1-phone-formats.test.ts | 11 | phone spellings, context discrimination, reply-side strictness (privacy) | ai/guardrails, ai/privacy-gate, ai/provider | AI (privacy) |
| 46 | p1-privacy-gate.test.ts | 12 | token vault, external-provider boundary, full orchestrator wiring | crypto, guardrails, orchestrator, privacy-gate, provider, fb client, conversations, vector-store, products | AI (privacy) |
| 47 | p1-rate-limit.test.ts | 7 | rate limiter semantics + storage failure policy | core/api/rate-limit, errors | SEC |
| 48 | p10-analytics.test.ts | 30 | overview, tz/range, conversations, orders, products, AI analytics, agents, campaigns, CSV | analytics/*, rbac, sales/metrics | **AN** (+DRAFT 1: "draft is NOT an order", line 156) |
| 49 | p11-ui.test.ts | 6 | Arabic labels for enums, formatNumber/formatDate | l/i18n | **UI** (labels) |
| 50 | p12-attribution-consent.test.ts | 63 | ad attribution, referral normalizers, consent, retarget campaigns | crypto, campaigns/*, fb/ig/wa normalizers, inbound-job, conversations, customers, integrations/* | OTH (P12 ecosystem) |
| 51 | p12-capacity-compliance-ops.test.ts | 34 | capacity, compliance banned claims (AI guardrail), ops signals | crypto, ai/{context-builder,guardrails,orchestrator,provider}, campaigns, wa/fb client, integrations/*, vector-store, products | OTH (P12) + AI |
| 52 | p12-foundation.test.ts | 28 | signing, PII guard, URL guard, connections, outbox | crypto, customers, integrations/* | OTH (P12) |
| 53 | p12-merge-main.test.ts | 6 | system-1 rules on external-provider path and playground; no brand string in P12 UI sources (lines 168-196) | crypto, orchestrator, playground-service, provider, fb client, conversations, integrations/systems, products | AI (+UI 3, lines 190-196) |
| 54 | p12-orders-route.test.ts | 3 | GET/PATCH /api/orders/[id] route | app/api/orders/[id]/route, crypto, errors (mocks prisma/auth) | **ORD** |
| 55 | p12-v1-routes.test.ts | 19 | /api/v1 capacity, retarget campaigns, signed system webhooks | app/api/v1/*, rate-limit, customers, integrations/* | OTH (P12) |
| 56 | p12-whatsapp-statuses.test.ts | 4 | WA status webhooks + canonical verify | crypto, wa/statuses, wa/webhook, customers | CH |
| 57 | p3-authz.test.ts | 6 | session gates + member route permission pins | app/api/workspaces/*, auth/context, auth/session | SEC |
| 58 | p3-members.test.ts | 11 | invitations, roles, suspension, ownership floor, isolation | members/service, workspaces/service | SEC |
| 59 | p4-inbox.test.ts | 14 | inbox filters/pagination, internal notes, Contact 360, assignment, isolation | app/api/conversations/[id]/{assign,notes}, orchestrator, provider, conversations, customers, vector-store | CH/inbox (OTH) |
| 60 | p5-channel-adapters.test.ts | 18 | adapters IG/Telegram/WA/TikTok contract | channels/* | CH |
| 61 | p5-comments-connect.test.ts | 11 | FB/IG comments, connect/disconnect | crypto, channels/* | CH |
| 62 | p5-sends.test.ts | 10 | sends per channel | crypto, channel clients, outbound, conversations | CH |
| 63 | p5-webhooks.test.ts | 8 | webhooks per channel | crypto, channels/* | CH |
| 64 | p6-ai-studio.test.ts | 34 | model router, fallback gate, persona versions, playground isolation (zero prod writes incl. `orderDraft`, line 390), knowledge, privacy | many ai/*, routes, knowledge/* | AI (+DRAFT 1 indirect) |
| 65 | p7-workflows.test.ts | 25 | workflow schema, triggers, execution, idempotency, publish/rollback, RBAC; "no orderDraft created" (line 516) | app/api/workflows/*, automation/*, ai/*, fb client | OTH (automation) |
| 66 | p8-sales.test.ts | 36 | offers, sales plan from DB, cross-sell, objections, discount cap, wholesale never leaks; 2 orchestrator tests send ORDER_CREATE (lines 620, 645) | orchestrator, provider, conversations, rbac, sales/* | AI (sales) (+DRAFT 2 indirect) |
| 67 | p9-campaigns.test.ts | 49 | segments, eligibility, campaign gates/lifecycle/runner | campaigns/*, jobs/handlers, rbac, wa client | OTH (campaigns) |
| 68 | page-catalogs.test.ts | 22 | catalog CRUD, links, page assignment, resolution; "wiring" greps route source + extraction.ts + schema.prisma (lines 243-288) | m/catalog/service | OTH (+UI/static 4; one pins `ai/stages/extraction.ts`) |
| 69 | page-guides.test.ts | 22 | PAGE_GUIDES coverage/flow/visit logic/panel markup/app-shell wiring | c/page-guide, l/nav-model, l/page-guides | **UI** |
| 70 | password-input.test.ts | 12 | PasswordInput component + no raw `type="password"` anywhere in src | c/ui/password-input (scans whole src/) | **UI** |
| 71 | perf-msg-budget.test.ts | 12 | per-reply DB-op ceilings (product.findUnique<=1, catalog.count<=1, channel.findUnique<=1), inbound ordering | crypto, orchestrator, provider, fb client/webhook, inbound-job, conversations, jobs | AI (perf) |
| 72 | perf-msg-latency.test.ts | 2 | full Messenger pipeline latency baseline | orchestrator, provider, fb, outbound, conversations, jobs | AI (perf) |
| 73 | perf-msg-ops.test.ts | 1 | DB-op histogram for one reply (only asserts total > 0) | orchestrator, provider, fb, jobs/handlers | AI (perf) |
| 74 | persona-management.test.ts | 3 | persona routes open under external provider, playground same persona, static "no internal brand string" (line 181) | app/api/personas/*, ai/playground-service, provider | AI (+UI 1, lines 171-197) |
| 75 | phase2-pipeline.test.ts | 21 | incoming message pipeline, unread, assignment/takeover, isolation, pagination | errors, events/bus, conversations, customers | CH/inbox (OTH) |
| 76 | phase3-knowledge.test.ts | 18 | products, media upload validation, feedback, RAG, doc lifecycle, context builder | errors, ai/context-builder, feedback, jobs, knowledge/*, media, products | OTH |
| 77 | phase4-ai.test.ts | 29 | provider+structured output, intent heuristics+injection, orchestrator, guardrails, RAG, queue/cost | crypto, context-builder, guardrails, intent, orchestrator, prompts, provider, structured, fb client, conversations, feedback, jobs, knowledge/*, products | AI (+DRAFT 1 indirect: line 305 ORDER_CREATE) |
| 78 | phase5-orders.test.ts | 30 | primitives, state machines, pricing/delivery, draft, rules, order creation/idempotency, customer merge, full AI->Order pipeline, audit | **orders/{primitives,state-machine,pricing,delivery,draft-service,rules,order-service,intelligence}**, orchestrator, conversations, fb client, vector-store, products | **DRAFT** (mixed with ORD; see Part 3) |
| 79 | phase6-facebook.test.ts | 22 | webhook verify/ingest, policy, OAuth state, outbound, E2E scenarios | orchestrator, provider, fb *, outbound, conversations, jobs/handlers, vector-store, products | CH (+DRAFT 1: scenario 1-4, lines 360-390) |
| 80 | phase7-automation.test.ts | 26 | templates/conditions, automation engine, follow-ups, **human handoffs (createHandoff/claim/release/resolve/route)**, human replies via FB | orchestrator, provider, automation/*, handoffs/handoff-service, fb client, outbound, conversations | OTH (automation) / handoffs |
| 81 | phase8-analytics.test.ts | 23 | time ranges, template allowlist, AI cost, usage metering, historical revenue from snapshots, isolation, funnel | analytics/*, usage-service, ai/router, automation/template, **orders/draft-service**, orders/order-service, products | **AN** (+DRAFT 3: lines 130-150, 185-205 build orders via draft) |
| 82 | phase9-security.test.ts | 8 | tenant isolation sweep (13 resources incl. **orderDraft** via getOwnedDraft, line 74), RBAC matrix, webhook replay x100, HUMAN_ACTIVE x100 = 0 replies, zod boundaries | orchestrator, provider, automation, conversations, customers, feedback, handoffs, knowledge, media, **orders/draft-service**, order-service, products/schemas, rbac | SEC (+DRAFT 1) |
| 83 | playground-isolation.test.ts | 12 | playground never writes prod tables (incl. orderDraft); "order proposal never creates Order/OrderDraft" (#30, line 186) | ai/playground-service, provider, knowledge/*, jobs, products | AI (+DRAFT 1) |
| 84 | product-excel-import.test.ts | 6 | xlsx/csv product import | ai/provider, products/import-service | OTH |
| 85 | product-prices.test.ts | 5 | priceTiers on create/update | products/service | OTH |
| 86 | production-audit-fixes.test.ts | 26 | media filenames, signed delivery, FB outbound state machine, AI speed, profile consistency, memory | orchestrator, provider, fb client/profile/types, outbound, conversations, media, products | AI/CH |
| 87 | production-safety.test.ts | 13 | fail-closed AI provider resolution, no fake replies, prod storage path | core/env, orchestrator, provider, conversations, jobs, media/storage | AI |
| 88 | provider-labels.test.ts | 5 | provider display labels; no "hermes" visible; settings/ai sources use shared label map | l/provider-labels (reads settings/ai/page.tsx, ai-status-card.tsx) | **UI** (labels/brand) |
| 89 | pwa-manifest-og.test.ts | 13 | PWA manifest, OG/twitter metadata, brand assets exist in public/brand | app/manifest, app/layout (mocks next/font/google) | **UI** |
| 90 | rbac.test.ts | 10 | permission matrix | rbac/permissions | SEC |
| 91 | role-matrix.test.ts | 15 | RoleMatrix component vs ROLE_PERMISSIONS, role blurbs, team-view wiring | c/role-matrix, rbac/permissions (reads schema.prisma, team-view.tsx) | **UI** + SEC |
| 92 | setup-progress.test.ts | 27 | getSetupProgress from workspace data + settings.manage route pin + SetupChecklistView markup (lines 229-312) | m/setup/progress, c/setup-checklist, ai/provider, rbac | OTH (UI part ~9) |
| 93 | smooth-flow.test.ts | 13 | "no dead ends" greps on 8 page/component sources + SEGMENT_PRESETS validity | app/(dashboard)/campaigns/segment-dialog, campaigns/segment-schemas | **UI** |
| 94 | theme-system.test.ts | 38 (+describe.each) | theme resolve/apply/pre-paint script, CSS var colours, contrast per theme x scheme, theme assets, ThemePicker | l/theme, c/theme-picker, helpers/theme-css | **UI** |
| 95 | token-expiry-handoff-fix.test.ts | 7 | FB token expiry cleared on select-page; GET /api/handoffs include fix (2 source greps, lines 69, 154) | crypto, fb client/policy, outbound, conversations | CH |
| 96 | upload-route-forensics.test.ts | 7 | /api/media/upload route | app/api/media/upload/route, media/storage (mocks prisma/auth) | OTH (media) |
| 97 | wa-media-privacy.test.ts | 13 | CUSTOMER_INBOUND media never leaks; retention | crypto, context-builder, wa client/media-download/media-retention, media, products | CH (privacy) |
| 98 | webhook-payload-retention.test.ts | 13 | raw webhook payload retention sweep | channels/webhook-retention, integrations/scheduler | CH |
| 99 | whatsapp-inbound-media.test.ts | 13 | WA inbound media download/normalize | crypto, inbound-job, wa client/media-download/normalizer, media/storage | CH |

Class totals (primary): DRAFT 3 pure files (catalog-scoped-intake, multi-order-flow, phase5-orders) + ~9 files with indirect/partial draft coupling; UI 16-17 files (+ ~5 files with a few structural tests).

---------------------------------------------------------------------
## PART 2 — UI-STRUCTURE tests: exactly what they assert

Key idea: these tests read **source files, CSS and component markup**. For each: what breaks in a UI rebuild vs. what is a product rule worth keeping. "KEEP" = product rule; "REWRITE" = tied to current implementation; "RETIRE" = visual-only.

### 2.1 tests/nav-structure.test.ts (24)  [reads `src/app/(dashboard)/**/page.tsx`, `l/nav-model`, `c/nav-icons`, `c/group-tabs`]
- Walks every `page.tsx` under `src/app/(dashboard)` (lines 37-52) and requires each route to resolve through `pageForPath(r, NAV_GROUPS)`: line 74 `dashboardRoutes().filter((r) => !pageForPath(r, NAV_GROUPS))` must be `[]` ("صفحة بلا مجموعة"). **KEEP the rule "no orphan page"** (rewrite against the new nav model).
- Hard counts tied to today's IA: line 64 `NAV_GROUPS` length 7 + unique keys; line 98 total pages = **21**. **RETIRE/REWRITE** (any IA change breaks these numbers).
- Old URLs never break: `OLD_NAV_HREFS` (22 hrefs, lines 55-60) must still exist as routes (line 82) and be in nav or in `NAV_REDIRECTS` target (line 84). **KEEP in spirit** (deep links from notifications/favourites/agent-door) — AGENTS.md says "no route changes on regroup".
- Redirect pages: each key of `NAV_REDIRECTS` must not be in nav and its `page.tsx` must contain `redirectToMerged("<href>")` (lines 90-94). Implementation-specific; REWRITE.
- Every nav page exists as route (104); group icons unique and present in `NAV_ICONS` (108-111); nav `permission` values are real `PERMISSIONS` (117); each redirect target exists in nav (125). Structural, REWRITE.
- Path matching: `/settings/ai` -> label «التشغيل» group `ai` (134-135), `/settings` -> `/settings/account` (136), `/ai/studio` -> `/settings/ai` (137), `/workflows` -> `/automation`, `/order-warehouse` -> `/orders` group `sales` (154-156), prefix `/ordersX` rejected (160), query/trailing slash ignored (166-167). Labels asserted: «التشغيل», «المنتجات», «الكتالوجات». REWRITE (label/URL map), **KEEP** the prefix/query semantics of `pageForPath`.
- Permissions in nav (171-212): permission-less member sees only `settings` group with `/settings/account` (189-190); empty groups hidden; `groupLanding` = first allowed page (206); every role sees only pages whose permission it has (193-202); `JSON.stringify(visibleGroups(...))` contains no `"permission"` (211). **KEEP** (RBAC-in-nav product rules).
- Tabs bar (`GroupTabs`, 215-239): order of labels «الوارد، التحويلات، العملاء، مختبر التجميع» (222), exactly one `aria-current="page"` (233), `aria-label="صفحات المحادثات"` (238). Rebuild breaks it (tabs bar is a component this rebuild likely deletes). RETIRE/REWRITE; keep the a11y idea.

### 2.2 tests/nav-search.test.ts (9)  [reads `l/nav-search`, `src/components/app-shell.tsx`]
- Arabic query normalization (17-21); `searchPages` expected top hit table (26-44) e.g. «طلبات» -> `/orders`, «باسورد» -> `/settings/account`, «ضبط الذكاء» -> `/settings/ai`; AND semantics (54); limit 6 (49); **only permitted pages** (58-63, KEEP); merged-page keywords (71-73); every nav href has `PAGE_KEYWORDS` and vice versa (78-81).
- Structural source pin (line 86-93): `app-shell.tsx` must contain `searchPages(`, `role="combobox"`, `aria-activedescendant`, `role="listbox"`, `ArrowDown`, `Escape`, `e.key === "/"`, `"k"`, and `/conversations?q=`. **Breaks if app-shell.tsx is rebuilt** (REWRITE to the new search component).

### 2.3 tests/page-guides.test.ts (22)  [reads `l/page-guides`, `c/page-guide`, `src/components/app-shell.tsx`]
- Every nav page has a `PAGE_GUIDES` entry and no orphan guide (30-33); `what` 30..220 chars, `tip` <=170, `next.label` <=40 (38-41); no «hermes» in guides (46); each guide's `next.href` is a nav page and not itself (53-54); following `next` from any page cycles (never dead-ends) (59-67); onboarding chain `/channels -> /products -> /personas -> /knowledge -> /ai/playground -> /settings/ai` (71-74); `guideForPath` aliasing (80-98); `visitGuide` first-visit logic with corrupt storage (103-145); panel markup: «شو هاي الصفحة؟», `role="region"`, `aria-label="شرح الصفحة"`, `aria-label="إغلاق الشرح"`, overlay variant `data-guide="overlay"` + `absolute` (150-183).
- Shell source pins (187-200): `aria-label="شرح هذه الصفحة"`, `aria-expanded={guideOpen}`, `<div className="sticky top-0 z-20">`, `e.key === "Escape") setGuideOpen(false)`. Break on app-shell rebuild.
- Product rules worth keeping: "every page has a what/next guide, no dead-end journey, onboarding chain", `visitGuide` first-visit semantics (pure logic, no UI dependency).

### 2.4 tests/empty-next.test.ts (4)  [reads 13 page sources + all `src/**/*.tsx`]
- `EmptyNext` renders `href`, `data-empty-next`, `<svg`, `bg-primary` (24-29); secondary has `border-line` and not `bg-primary text-white` (35-35).
- Table `SITES` (40-54): for each of 13 pages (customers, handoffs, follow-ups, catalogs, feedback, settings/team/team-view, intelligence, analytics, orders, knowledge, media, automation, campaigns) the empty-state marker string (e.g. `title="لا يوجد عملاء بعد"`, `title={t.orders.emptyTitle}`) must exist and the `<EmptyState ... />` block must contain `action=` (56-63). Very coupled to current JSX & Arabic titles -> **REWRITE** (rule to keep: "an empty list always offers a next step").
- Line 65-73: scans every `.tsx` for `<EmptyNext href="...">` (>=6 of them) and requires `pageForPath(h, NAV_GROUPS)` truthy. Keep idea, rewrite regex.

### 2.5 tests/guided-tour.test.ts (7)  [reads app-shell.tsx, app-sidebar.tsx, package.json, globals.css]
- Tour steps for `group-*` targets equal nav group keys (21-22); `app-sidebar.tsx` contains ``tour={`group-${`` and `data-tour={tour}` (23-24); non-group steps must have `data-tour="<target>"` in app-shell.tsx (28-30); Arabic text, text<=120, title<=24, no "hermes" (35-39); autostart only on `/dashboard` first visit (45-47); `startGuidedTour(` in shell (51); `driver.js` pinned exact version (58); tour CSS block after `.zr-tour` uses `var(--surface)` and no hex (62-67). All shell/CSS pins break on rebuild; REWRITE or RETIRE if tour dropped. Keep: "no hard-coded colours".

### 2.6 tests/fluent-design.test.ts (9)  [reads globals.css, tailwind.config.ts, app-sidebar, app-shell, ui/button, products/page]
- **Pure visual pinning, RETIRE for a design-system rebuild.** Exact tokens: `--background: 243 243 243`, `--text: 36 36 36`, `--primary: 15 108 189`, `--primary-soft: 235 243 252` (lines 20-31); legacy names `--surface-1/2/3`, `--surface-hover`, `--brand:` (35); no `backdrop-filter` in CSS but `.glass-strong {` must exist (41-42); tailwind config has `soft: "rgb(var(--primary-soft) / <alpha-value>)"` (48-49).
- Sidebar must contain `bg-surface-2`, `border-e border-line`, `w-[280px]`, `bg-primary-soft` and not `backdrop-blur` (55-61); topbar `h-14`, `bg-surface-2`, `bg-surface-3`, `focus:border-primary`, not `lg:rounded-[20px]` (66-70); button `bg-primary`, `h-9`, no `bg-gradient` (74-77); products page `xl:grid-cols-3`, `object-contain`, `bg-surface-3/70`, categories from `new Set(products.map((p) => p.category)` (83-88).

### 2.7 tests/theme-system.test.ts (38 `it` + `describe.each`)  [reads layout.tsx, tailwind.config.ts, globals.css, auth-form.tsx, card.tsx, empty-state.tsx, app-sidebar.tsx, dashboard/page.tsx, public/brand/themes/*.webp]
- Logic (KEEP if themes survive): `resolveTheme`, `applyTheme/applyScheme`, pre-paint `THEME_INIT_SCRIPT` equals `resolveTheme` on every combination (lines 53-205); `layout.tsx` contains `THEME_INIT_SCRIPT` (137).
- Design-system contract: `tailwind.config.ts` contains **no `#rrggbb`** outside comments (line 214 `expect(code.match(/#[0-9a-fA-F]{6}\b/g) ?? []).toEqual([])`); primary/hover/soft/fg/accent/showcase read `rgb(var(--...))` and brand-50..950 read `rgb(var(--brand-N) / <alpha-value>)` (217-224); `textColor.primary` uses `--primary-fg` (226-228); default brand scale equals the legacy hex values exactly (230-243); dark `primary-soft` luminance <0.05 (245-253); `auth-form.tsx` has no `bg-[#hex]` (255-257).
- Contrast gates per theme x scheme (`describe.each(COMBOS)` line 271; ~line 276-320): all REQUIRED CSS roles defined (list at ~262: surface-1..3, line, text*, primary*, brand*, accent*, showcase, shadow-*, logo-*), text >=7:1, muted >=4.5, `text-primary` >=4.5 on surfaces, white on fill >=4.5, focus ring >=3. **KEEP the gate idea** (an accessibility product rule), but it is bound to `THEMES` + `[data-theme]` blocks in globals.css.
- Scoped palettes (`SCOPE_COMBOS`, line 324): contrast inside dark scopes (sidebar, hero).
- Asset/handle pins: every `url(/brand/...)` in CSS exists in `public/` (348-352); **design hooks must exist** (354-360): `card.tsx` contains `data-page-header`, `empty-state.tsx` `data-illustration`, `auth-form.tsx` `data-showcase-dots` and `<div data-showcase `, `app-sidebar.tsx` `data-pane="nav"`, `dashboard/page.tsx` `data-hero`; font loading regexes on `layout.tsx` (`Reem_Kufi({... variable: "--font-reem" ... preload: false`, body className `${arabic.variable} ${reem.variable} ${lalezar.variable}`) (363-385); WebP theme art ratio+<150KB (387-394); night theme CSS selectors/z-index (396-405); islamic `[data-showcase]/[data-hero]` background-image gradient before `url(` (407-421). 
- ThemePicker markup (425-463): `role="radiogroup"`, `aria-label="الثيم"`, `data-theme-card`, swatches `background-color:`, `data-scheme`, night badge "ليلي دائمًا", disabled light button.
- Rebuild impact: all hook-name, font, theme-art and CSS-selector assertions break; the pure functions survive only if `lib/theme.ts` is kept. If the rebuild drops multi-theme support, retire the whole file.

### 2.8 tests/brand-logo.test.ts (7)  [reads `src/app/icon.svg`, `globals.css`]
- Markup hooks `data-brand-logo`, `data-logo-tile`, `data-logo-bubble`, `data-logo-spark`, `aria-hidden` (17-22); colour classes `from-logo-from`, `via-logo-via`, `to-logo-to`, `var(--logo-bubble)`, `var(--logo-spark)`, no `sky-N|blue-N|#hex` (25-30); 5 distinct `h-N w-N` sizes (33-36); `BRAND_MARK_PATH` has exactly 2 `M` subpaths and `fill-rule="evenodd"` (38-41); `icon.svg` contains the same path constants (43-48, **KEEP: icon/component parity**); islamic tile clip-path polygon with 16 points + `drop-shadow` (50-58); night tile `border-radius: 9999px`, `mask: linear-gradient(`, `--logo-bubble: 26 11 46` (60-68). Rest RETIRE if logo/themes redone.

### 2.9 tests/password-input.test.ts (12)  [scans every file in `src/`]
- Component logic (`passwordToggle`, labels «إظهار/إخفاء كلمة المرور», «القيمة» for `kind="secret"`), markup (`type="password"`, `<button ... type="button"`, `pe-10`, `end-1`, `-ms-reveal`, `dir` passed to input and wrapper) (17-78).
- **Repo-wide guard (lines 80-94):** regex `/type\s*[=:]\s*[{]?\s*["']password["']/` must match **no file under `src/`** except `components/ui/password-input.tsx`. **KEEP — this is a product rule** ("every secret field uses PasswordInput") and survives a rebuild as long as the new design system keeps a PasswordInput; the file path in `OWN` (line 81) must be updated if the component moves. AGENTS.md: team page password field intentionally stays `type="text"`.

### 2.10 tests/role-matrix.test.ts (15)  [reads `prisma/schema.prisma` enum UserRole, `settings/team/team-view.tsx`]
- `ROLE_OPTIONS` equals schema roles minus `SUPER_ADMIN` (24-27); each role has permissions + blurb starting with its label (29-35); capabilities are real permissions, unique (39-43); OWNER/ADMIN have all, ADMIN set == OWNER set (45-51); blurbs truthful against `hasPermission` (55-82); table markup: `data-can="yes|no"` counts, `scope="col"`, `scope="row"`, `<caption`, «يقدر/لا يقدر», `overflow-x-auto`, `sticky` (86-116); team-view source pins: `<RoleMatrix`, `<RoleBlurbs`, `ROLE_OPTIONS as ROLES`, no `const ROLES`, `hint={ROLE_BLURB[inviteRole]}`, `value={inviteRole}`, `name="role"` (126-139). **KEEP** lines 24-82 (RBAC truth); REWRITE markup/team-view pins.

### 2.11 tests/setup-progress.test.ts (27; UI part = lines 229-312, ~9 tests)
- Lines 1-227: `getSetupProgress` derives steps from real workspace data (channel, products, persona, knowledge, playground, go-live with real provider key), isolation, read-only, response flags without ids; step hrefs exactly `["/channels","/products","/personas","/knowledge","/ai/playground","/settings/ai"]` (line 209-212); route `src/app/api/setup/progress/route.ts` contains `permission: "settings.manage"` (217-218); SALES_AGENT/VIEWER lack it (221-225). **KEEP.**
- Lines 229-312: `SetupChecklistView` markup — `خطوتك الجاية: ... أضف منتجاتك`, one `aria-current="step"`, `data-step/data-state` (done|next|todo), «اختياري» count, `role="progressbar"` + `aria-valuenow="1"` `aria-valuemax="5"`, "النظام جاهز" + `data-setup="ready"`, `<ol`. REWRITE with component.

### 2.12 tests/smooth-flow.test.ts (13)  [greps 9 source files; code stripped of comments]
Product rule = "no dead-end buttons". Concrete source greps: personas page must not `fetch("/api/ai/playground")` (24) and must contain `/ai/playground?persona=` (28); playground page `get("persona")` (32); handoffs page contains `/conversations?id=` and not ``href={`/conversations`}`` (38-40); inbox `get("id")` (44); customers page after `md:hidden` contains `<Link` (49-50); customer-panel contains ``href={`/orders/${o.id}`}`` (54) (**customer-panel.tsx is also where OrderDraftCard lives**); orders/[id]/page has no `>Subtotal<`, `>Discount<`, `>Delivery<`, `>Total<` (61), contains `/conversations?id=` and `href="/orders"` (66-67); campaigns segment-dialog contains `fetch("/api/segments"` and `method: "POST"`, campaigns page contains `SegmentDialog` and not «عبر واجهة الشرائح» (73-80); app-shell matches `/canTeam\s*&&/` (87); `SEGMENT_PRESETS` validate against `segmentCreateSchema` (95-100, KEEP: logic). Everything greps current JSX -> REWRITE per new pages, KEEP the deep-link contracts (`/conversations?id=`, `/ai/playground?persona=`, `/orders/:id`).

### 2.13 tests/pwa-manifest-og.test.ts (13)  [imports `app/manifest`, `app/layout`; reads `public/brand/*`]
- Manifest name contains "Zaki response", lang `ar`, dir `rtl`, `display: standalone`, `start_url: "/dashboard"`, `scope: "/"` (31-42); **`theme_color "#2563eb"`, `background_color "#f8fafc"`** (46-47 — will break if palette changes); maskable icon, all icons `src` == `/icon.svg` (50-56); `metadataBase` URL, openGraph/twitter, manifest link, `robots.index === false` (60-77); `viewport.themeColor` array with light+dark media and not duplicated in metadata (79-87); assets exist and size: `og.png` (PNG, 1200x630 linked), `login-art.webp`, `empty-{conversations,orders,products,media}.webp` (VP8L/VP8X transparent) (93-125). Keep: robots noindex, metadataBase, themeColor-in-viewport; update colours/assets on rebuild. Note start_url `/dashboard` is a route contract. `vi.mock("next/font/google")` lists every font used by layout.tsx (lines 8-12) — must be updated when fonts change.

### 2.14 tests/provider-labels.test.ts (5), tests/labels-arabic.test.ts (2), tests/p11-ui.test.ts (6)  (Arabic label/brand rules)
- provider-labels: no "hermes" in any provider display (20), `publicDetail()` translates/cleans error codes (26-31), `settings/ai/page.tsx` and `ai-status-card.tsx` import `@/lib/provider-labels` and contain no `const PROVIDER_AR` (35-39), card contains `publicDetail(t.compatibilityReason)` (44), `PROVIDER_OPTIONS` regex over `settings/ai/page.tsx` (49). **KEEP the owner decision (no brand name in visible text)**; REWRITE the file-content pins.
- labels-arabic: for 8 Prisma enums (HandoffStatus, CampaignStatus, CampaignRecipientStatus, CampaignEventType, MediaCategory, KnowledgeCategory, KnowledgeStatus, FollowUpStatus) every value has an Arabic `LABELS` entry; order audit actions `order.created|cancelled|state_changed|updated_from_chat|deleted` have Arabic labels (40-43). **KEEP.** Note `order_draft.*` audit keys (i18n.ts:488-492) are NOT asserted here, so removing them breaks nothing.
- p11-ui: triggers/operators/statuses have labels; `label()` fallback; `formatNumber/formatDate` (all pure, no file reads). **KEEP.**

### 2.15 Partial UI/static assertions inside non-UI files
- persona-management.test.ts #8 (lines 171-197): six sources (`personas/page.tsx`, `channels/page.tsx`, `settings/ai/page.tsx`, `ai-status-card.tsx`, `ai/playground/page.tsx`, `components/auth-form.tsx`) stripped of comments must not match `/hermes/i`; `settings/ai/page.tsx` has no `HERMES_`. Rebuild: update paths.
- p12-merge-main.test.ts (lines 168-196): `settings/page.tsx`, `settings/ai/page.tsx`, `components/products/compliance-card.tsx`, `components/inbox/marketing-card.tsx` no brand string; `SYSTEM_SPECS` Ar strings no hermes; `TOKEN_PRESETS["hermes-bridge"]` undefined.
- page-catalogs.test.ts lines 243-288 ("wiring"): route files contain `permission: "catalogs.read|write|assign"`, audit action names, `schema.prisma` Catalog/CatalogProduct blocks contain no `price`. **Line 273-279 pins `src/modules/ai/stages/extraction.ts`**: must contain `resolveCatalogForChannel`, regexp `/id: \{ in: allowedIds \}/`, and `allowedIds?: string[] | null`. This is the one test that source-greps an AI stage; any extraction.ts refactor must keep or update it.
- token-expiry-handoff-fix.test.ts lines 69-73 and 154-158: `select-page/route.ts` has >=2 `tokenExpiresAt: null`; `api/handoffs/route.ts` does not contain `include: { conversation: { select`. Not UI; backend source greps.

---------------------------------------------------------------------
## PART 3 — ORDER-DRAFT tests

### 3.0 Where draft coupling lives (summary)
- Pure draft files: `phase5-orders.test.ts` (30), `multi-order-flow.test.ts` (7), `catalog-scoped-intake.test.ts` (7).
- Partial/indirect: `phase6-facebook.test.ts` (1), `phase8-analytics.test.ts` (3), `phase9-security.test.ts` (1 row of an `it.each`), `p10-analytics.test.ts` (1), `conversation-purge.test.ts` (1), `playground-isolation.test.ts` (1), `p6-ai-studio.test.ts` (snapshot list), `p7-workflows.test.ts` (1 assertion), `p8-sales.test.ts` (2) and `phase4-ai.test.ts` (1) via orchestrator ORDER_CREATE.
- **No test hits `/api/order-drafts/*` routes** (grep for `order-drafts` in tests/ returns nothing). Those 4 route files (`route.ts`, `[id]/route.ts`, `[id]/confirm`, `[id]/cancel`) and `scripts/staging/concurrency.mjs` are untested by Vitest.
- **Implicit coupling (UNVERIFIED which tests actually write a draft):** `extraction.ts:220-224` calls `handleOrderIntent` for intent in `[ORDER_CREATE, ORDER_UPDATE, ORDER_CANCEL, UNKNOWN]` or `requestedAction === "CREATE_ORDER"`, so every orchestrator test whose mock intent is UNKNOWN or ORDER_* executes `handleOrderIntent` (reads `db.orderDraft` at intelligence.ts:70/100/301). Only the ORDER_CREATE ones create a row: phase4-ai.test.ts:305-318 (entities customer_name+phone), p8-sales.test.ts:620 and :645, phase5/phase6 flows. Removing the draft must keep these tests green with the new extraction path (they assert on memories/sales events/replies, not on draft rows).

### 3.1 tests/phase5-orders.test.ts (30) — every title, with what it pins
Primitives (imports `orders/primitives`)
1. L104 "normalizes Arabic and English phone numbers" — `normalizePhone` (٠٩٣..., +963, 00963 -> `0930000000`, "123" invalid), `toWesternDigits`. *Parser; business rule = phone canonicalisation (keep logic, location may change).*
2. L112 "parses quantities: digits, Arabic words, Arabic-Indic numerals" — `parseQuantity`: "بدي 3", "٣ قطع", "ثلاث قطع", "قطعتين", "خمسة" -> value; "0"/"-1" -> null; "2.5" -> `DECIMAL_NOT_ALLOWED`; `{max:100}` + 99999 -> `EXCEEDS_MAX`. *Parser table (retire with `parseQuantity`) but the cases are a useful regression corpus for the extraction prompt/validator; max-quantity is a business rule (default 100, env `MAX_ORDER_QUANTITY`, primitives.ts:58,74).*
3. L124 "resolves products: sku, name, normalized, ambiguity" — `resolveProduct` RESOLVED/AMBIGUOUS (two "ماء الكمأة" products)/NOT_FOUND. *Business rule: ambiguity -> ask, never guess. Survives in any design.*
4. L134 "parses confirmation only as a signal (context decided by caller)" — `parseConfirmation`: "نعم أكد" YES, "لا إلغاء" NO, "تمام" YES, chatter NONE. *Draft mechanism (retire).*
State machines (`orders/state-machine`)
5. L144 "enforces valid order transitions" — `canTransitionOrder`, `assertOrderTransition` throws AppError. **ORDER state machine — KEEP** (not draft).
6. L152 "enforces valid draft transitions" — `canTransitionDraft`. *Retire.*
Pricing & delivery (`orders/pricing`, `orders/delivery`) — all **business rules, KEEP**
7. L159 price from Product DB: 150000x3 + fee 50000 = 500000 SYP.
8. L170 free-over threshold -> fee 0, reason `FREE_OVER_THRESHOLD`.
9. L179 unavailable province `deliveryAvailable=false`; unknown province -> `NO_RULE_DEFAULT_FEE`.
10. L192 `formatOrderSummary` deterministic text («السعر: 450000 SYP», «الإجمالي», «التوصيل: مجاني») — no AI-generated prices.
Order draft (draft-service) — *mechanism, retire; the business facts noted*
11. L205 creates persisted draft (state COLLECTING) and `missingFields == ["quantity","phone","address"]`.
12. L215 collects across messages: product -> quantity -> phone (Arabic digits normalised) -> address; `becameReady` true at the end.
13. L229 contextual extraction via `handleOrderIntent` with all entities -> `REQUEST_CONFIRMATION`.
14. L242 invalid phone from AI "abc123" is dropped, not stored. *Business rule: never store unvalidated phone (keep as extraction validation).*
Rules engine (`orders/rules`)
15. L250 REQUIRE_INPUT for missing fields. 
16. L260 DENY for inactive product (`product_active` / `PRODUCT_INACTIVE`) — **KEEP rule.**
17. L271 DENY for max quantity (5000) — **KEEP rule.**
18. L284 REQUIRE_HUMAN for likely duplicate (re-activated draft after an order). *Rule KEEP (`detectDuplicate`), test setup uses draft rows -> rewrite setup.*
Order creation & idempotency (`order-service.createOrderFromDraft`)
19. L300 creates order with DB price snapshot: total 450000, currency SYP, second call `created:false` same orderId, 1 order, 1 orderItem with `productNameSnapshot` and `unitPrice` 150000. *Idempotency + snapshot = KEEP as rules; entry point `createOrderFromDraft` is draft-coupled (order-service.ts:30-36, 164).*
20. L317 price change after order does not change snapshot (unit 150000, total 450000). **KEEP.**
21. L327 "AI cannot inject a price" — draft + «نعم أكد» -> `ORDER_CONFIRMED`, total 450000 from DB. **KEEP rule** (price always from DB), mechanism = draft+confirmation.
22. L350 cross-workspace `getOwnedOrder`/`transitionOrder` fail (AppError). **KEEP.**
23. L360 state machine blocks illegal order transition via service (NEW->DELIVERED rejected, NEW->PROCESSING ok). **KEEP.**
24. L369 `detectDuplicate` same conversation+product -> `LIKELY`. **KEEP.**
25. L379 `mergeCustomerData` low-confidence phone does not overwrite (needsConfirmation `phone_mismatch`). **KEEP** (customer, not draft).
26. L388 high-confidence phone change accepted. **KEEP.**
27. L396 "order draft is persisted, not in-memory" (row has productId/quantity/phone, state READY). *Retire.*
28. L405 ambiguity -> `ASK_MISSING_FIELD`, `summaryText` contains «أي واحد تقصد». *Business rule (ask on ambiguity) KEEP, mechanism retire.*
Full pipeline & audit
29. L422 "customer order flow through the orchestrator ends with a DB order" — message 1 («بدي 3 من ماء الكمأة، رقمي 0930000000، عنواني دمشق دويلعة») -> status RESPONDED/CLARIFIED, draft READY with productId; message 2 «نعم أكد» -> exactly 1 order total 450000. **This is THE end-to-end scenario; rewrite for after-the-fact extraction (assert Order with correct product/qty/phone/address/total from the whole conversation).**
30. L452 audit: actions include `order_draft.updated`, `order.created`, `price.calculated`. *Keep `order.created`, `price.calculated`; drop `order_draft.updated`.*

### 3.2 tests/multi-order-flow.test.ts (7) — all mechanism-coupled (`handleOrderIntent`, draft rows)
1. L46 "a new order request after a confirmed order starts a SECOND order": order #1 confirmed via «نعم أكد»; then product B -> `REQUEST_CONFIRMATION`; draft state READY, `productId`=B, qty 2, `orderId` null (**draft recycling**: "one draft row per conversation is recycled", also unique(conversationId) in schema); confirm -> 2 orders, different orderIds. *Business rule to keep: a conversation can hold many orders.*
2. L103 "recycled draft keeps the customer contact from the previous order" — phone, address, province, name preserved. *Rule: reuse known contact data (keep, could be satisfied by extraction over whole conversation).*
3. L140 "duplicate rule no longer blocks a different-product second order" — same conversation+product is LIKELY duplicate; different product is not. *KEEP rule.*
4. L165 "quantity edit updates the placed order and reprices it": `ORDER_UPDATE` qty 5 -> `ORDER_UPDATED`, total 750000 (locked unit price 150000x5), single order/orderItem (edit never duplicates). **KEEP** (uses `updateOrderFromChat`, order-service.ts:366).
5. L195 "address edit updates delivery details on the placed order" (deliveryAddress, province).
6. L219 "delivered/terminal orders refuse edits" (OUT_FOR_DELIVERY -> action NONE, quantity untouched).
7. L247 "cancelling after a placed order cancels that ORDER (not just the draft)" (`cancelLatestOrderFromConversation`, status CANCELLED) and a new order is then possible.
No "multi-product draft" test exists: the draft holds a single `productId`/`quantity` (schema.prisma:1180-1209); multi-line orders are not pinned anywhere. **UNVERIFIED** whether `OrderItem` supports >1 line from the draft path (draft has one product).

### 3.3 tests/catalog-scoped-intake.test.ts (7) — page-catalog allow-list on order intake
1-4. product outside assigned catalog never enters draft (productId null, missing includes `product`, quantity merged); empty allow-list blocks everything; catalog product resolves; AI-provided `productId` outside catalog ignored. 5. undefined allow-list = legacy open behaviour. 6. `listResolvableProducts` honours allow-list. 7. `handleOrderIntent` passes `allowedProductIds` (draft productId stays null, `missingFields` has product).
*Business rule to keep:* "a page may only order products of its assigned catalog; ids supplied by AI are re-validated; no catalogs => legacy open". Needs a new home in the extraction/order-creation step (`extraction.ts:210-216` computes `allowedProductIds`; page-catalogs.test.ts:273 pins that).

### 3.4 Partial/indirect draft assertions
- phase6-facebook.test.ts:360-390 "scenario 1-4: greeting -> order collection -> confirm -> order created via Facebook": greeting (1 send), full-data message -> `db.orderDraft` for conversation has `state "READY"` (line 382-383), «نعم أكد» -> exactly 1 order, total 450000 ("DB price, never AI price"), 3 outbound sends total. **Rewrite** (assert on Order; decide whether the confirmation reply still happens, since `sentToMeta.length` is 3).
- phase8-analytics.test.ts:130-150 ("critical: product price change does NOT change historical revenue") and 185-205 (funnel with real rates) build orders via `updateDraft` + `createOrderFromDraft`; note line 10 imports `updateDraftFromExtraction` from `order-service`, which does not export it (harmless: unused alias, erased by esbuild; **UNVERIFIED** that vitest does not complain — suite is presumed green). Rules to keep: snapshot revenue (1350000 after price doubling, units 3), `funnel.ordersConfirmed`, `confirmRate`, `deliveryRate`. Only the seeding must change (create Order directly).
- p10-analytics.test.ts:156-163 "draft is NOT an order; stages separated": `orderDraft` COLLECTING row + 1 NEW order -> `getOrderAnalytics.created == 1`, `funnel.draftsStarted == 1`, `ordersConfirmed == 0`. Tied to `analytics/service.ts:393-411` (`draftsStarted`, `draftsReady`, `draftStartRate`, `readyRate`, `confirmRate = ordersConfirmed/draftsReady`) and `analytics/page.tsx:392-393`. phase8-analytics.test.ts:186-187 asserts `rates.draftStartRate` null on zero denominators. **Funnel definition changes if drafts go** — needs a product decision.
- phase9-security.test.ts:45 + 74: `orderDraft` row in WS_B and `["orderDraft", () => getOwnedDraft(db, WS_A, ids.draft)]` -> must reject (cross-tenant). Retire that row; tenant-isolation sweep otherwise **KEEP** (add the new extraction record if any).
- conversation-purge.test.ts:74-134: creates `db.orderDraft` (idempotencyKey `draft:d-1`, orderId) + order with `draftId`; after purge `_stores.orderDraft` length 0 and order survives with `conversationId` and `draftId` both null (lines 128-134). Tied to `conversations/threads.ts:623` (`orderDraft.deleteMany`) and `:637` (`draftId: null`). Business rule KEEP: orders survive purge, unlinked.
- playground-isolation.test.ts:186-193 (#30) and p6-ai-studio.test.ts:390 (`PROD` snapshot list includes `"orderDraft"`) and p7-workflows.test.ts:516: assert zero rows in `orderDraft` (+ `order`) after playground/workflow runs. **KEEP the invariant** ("playground/workflows never create real orders"); just drop `orderDraft` from the lists (the fake DB `_stores.orderDraft` access would be `undefined.length` TypeError if the store is removed before tests are edited).

### 3.5 Which behaviours are mechanism vs. rules that must survive
Retire with the mechanism (draft state machine):
- draft state graph EMPTY/COLLECTING/READY/CONFIRMING/CONFIRMED/CONVERTED/CANCELLED (`canTransitionDraft`, tests 6, 11-13, 27, `READY` assertions in phase5:435, phase6:383, multi-order:92).
- one row per conversation (`unique(conversationId)`), recycling after confirm/cancel (`recycleDraftForNewOrder`), `computeMissingFields` order of `["product","quantity","phone","address"]` (draft-service.ts:10).
- `parseConfirmation` YES/NO/NONE signal and bare-«نعم» semantics (intelligence.ts:104, 237), `REQUEST_CONFIRMATION` / `ORDER_CONFIRMED` / `ASK_MISSING_FIELD` action codes, `summaryText` reply («تم تأكيد طلبك ✅ رقم الطلب») that the response stage injects (`response.ts:76,131`).
- `/api/order-drafts` agent confirm/cancel flow and `order_draft.*` audit actions/labels.
- audit `order_draft.updated`; draft funnel stages in analytics.
Must survive in ANY design (each has an existing test to port):
- **Price always from DB, never from AI/customer text** (phase5 #7-10, #19-21, ocr-wave1 "order pricing in cents": `0.1x3 = 0.30`, `19.99x3 = 59.97`, negative discount clamped, integer money bit-identical — `orders/pricing`); price snapshot immune to later product price change (#20, phase8 revenue).
- **Rules engine verdicts**: required fields, product exists/active (`PRODUCT_INACTIVE`), availability (`OUT_OF_STOCK` for OUT_OF_STOCK/DISCONTINUED — rules.ts:55-56, **no orders test covers it**; only `isProductAvailable` in p8-sales:140-141 and sales plan p8-sales:174), quantity bounds (`NON_POSITIVE`, `EXCEEDS_MAX`, default 100 / env `MAX_ORDER_QUANTITY`), delivery availability (`PROVINCE_NOT_SERVED`, **untested**), duplicate (`LIKELY` -> REQUIRE_HUMAN, `POSSIBLE` -> REQUIRE_INPUT). Tested today: inactive, max qty, duplicate, missing fields only.
- **Delivery fee**: per-province rule, `freeOver`, unknown province default fee, unavailable province flag.
- **Idempotency**: second create returns same order, 1 Order + 1 OrderItem (phase5 #19); today implemented via `draft.orderId` + `idempotencyKey` (order-service.ts:42-55,164). The fake DB does **not** enforce `OrderDraft.idempotencyKey`/`Order` uniqueness (only listed uniques in `inmemory-db.ts` uniqMap) so a new key must be enforced by code or added to uniqMap.
- Ambiguous product -> ask, never guess; products limited to page catalog; AI-provided product ids revalidated; invalid phone dropped; `mergeCustomerData` confidence gates.
- Order lifecycle state machine + workspace isolation + audit (`order.created`, `price.calculated`, `order.state_changed`...), edits from chat reprice at the locked unit price and refuse terminal states, cancel cancels the ORDER, multiple orders per conversation, contact reuse.
- **HUMAN_ACTIVE means no automated replies** (ai-mode-persona #7+25 line 171; phase9 "HUMAN_ACTIVE x100 inbound -> 0 AI replies" line 137; phase6 scenario 5 / takeover test; p8-sales:641 no sales events; durable-processing:415; master-reliability:210). Any after-the-fact extractor must also not run/ reply under HUMAN_ACTIVE (decision needed: extraction itself is not a reply).
- **Privacy masking**: p1-privacy-gate (external provider never sees raw phone/name; token vault), p1-phone-formats, `ai/privacy-gate` — relevant because after-the-fact extraction over the whole conversation needs the customer's name/phone/address; AGENTS.md rule: no customer data to external models (`privacy-gate.ts`, `masking.ts`). Extraction against an external provider will be blocked/masked by these tests' rules.
- Playground/workflow never write real orders; conversation purge keeps orders unlinked; tenant isolation of orders.

### 3.6 Source-grep pins that touch order-intent code
- page-catalogs.test.ts:273-279 pins `extraction.ts` strings (see 2.15).
- `extraction.ts` order-intent surface to be edited: 29 (import), 214-231 (catalog allow-list + `handleOrderIntent` call), 281-285 (`plan.goal = "complete_order"`, `needsClarification`, `missingFields`), 318 (`needsHumanReview` branch), 346 (`orderResult` into state), 440-462 (`ORDER_CREATE: "complete_order"`, `ORDER_UPDATE: "clarify"`, `ORDER_CANCEL: "handoff"` goal map; `needsClarification` for ORDER_CREATE with missing fields). No test asserts `plan.goal` strings directly (grep of tests for `complete_order` — not run; **UNVERIFIED**).

---------------------------------------------------------------------
## PART 4 — Helpers and fixtures

### tests/setup.ts (vitest `setupFiles`, vitest.config.ts)
- Imports `@/core/db/prisma` first (so dotenv side effect happens before scrubbing), sets `NODE_ENV=test`, `DATABASE_URL ??=` dummy, **forces** `JWT_SECRET` and `ENCRYPTION_KEY` to fixed test values, deletes AI/provider/Meta env keys (`AI_PROVIDER`, `OPENAI_API_KEY`, `OPENROUTER_API_KEY`, `ANTHROPIC_API_KEY`, `GOOGLE_API_KEY`, `HERMES_*`, `ALLOW_MOCK_AI`, `AI_DISABLE_AUTO_HANDOFF`, `META_REDIRECT_URI`, `META_GRAPH_VERSION`), and installs a fake public DNS lookup (`setIntegrationDnsLookup` -> 93.184.216.34) so no test touches the network. `vitest.config.ts`: `include tests/**/*.test.ts`, `environment: node`, alias `@ -> src`, `esbuild.jsx = "automatic"` (so components render with `renderToStaticMarkup`). **No jsdom/DOM**: UI tests are string/markup based only.
- AGENTS.md: without a Prisma engine expect one `PrismaClientInitializationError` per file (use `--dangerouslyIgnoreUnhandledErrors`); CI has none.

### tests/helpers/inmemory-db.ts (845 lines) — `createFakeDb()`
- Generic `makeModel(name, store, defaults)` offering: `findUnique` (fast path by `id`, else `matchesWhere`, compound keys like `a_b` split on `_`), `findFirst` (honours `orderBy`), `findUniqueOrThrow` (plain `Error`, not NotFound), `findMany` (where/orderBy/`take`/limited `cursor`/`include` stub), `create`, `update` (by id or where), `updateMany`, `delete`, `deleteMany`, `count`, `upsert`, `groupBy` (single `by`, only `_count`), `createMany` (`skipDuplicates` for catalog models). **No `aggregate`, no `$transaction`, no `$queryRaw`/`$executeRaw`, no `select` projection (returns full clone), no `distinct`, no `skip` except tiny cursor handling, `include` returns `[]`/`{}` placeholders unless relation hand-attached.** (Services take `db` typed `Prisma.TransactionClient`, so tests pass `db as Db`.)
- Where operators supported: equality, `null`, `Date` equality, `not`, `contains` (+`mode: "insensitive"`), `gt/gte/lt/lte` (ANDed), `equals`, `startsWith`, `in`, `notIn`, `hasSome`, `hasEvery`, `AND`, `OR`, `NOT`, relation filter `some`/`none`, nested relation object filter (e.g. `channel: { type: ... }`), compound-unique keys. Unsupported ops are silently ignored (matchesCondition loop only handles listed ops) — a new operator such as `endsWith`, `isEmpty`, `has`, `search`, `array_contains`, `mode: "default"` will silently match everything.
- `update`/`updateMany` support `{ increment }` only (no `decrement`, `set`, `push`).
- Default row values per model (e.g. `isActive: true, tags: [], deletedAt: null`, product `currency "USD"`, `stockStatus "IN_STOCK"`, order `status NEW`, orderDraft `state COLLECTING, missingFields []` (**note: fake default is COLLECTING; prisma default is EMPTY**)).
- Models present in `db`/`_stores` (61): customer, conversation, message, channelIdentity, channel, workspaceMember, auditLog, product, productVariant, productPrice, productMedia, mediaAsset, feedback, knowledgeDocument, knowledgeChunk, faq, backgroundJob, aISettings, persona, personaVersion, customerMemory, knowledgeCitation, channelAccount, humanHandoff, automation, automationVersion, automationExecution, user, followUp, workspaceUsage, dailyWorkspaceAnalytics, webhookEvent, playgroundSession, playgroundMessage, aIUsageLog, order, orderItem, **orderDraft**, deliveryRule, salesEvent, audienceSegment, campaign, campaignContentVersion, campaignRecipient, campaignEvent, discountApproval, workspace, attributionTouch, integrationConnection, integrationToken, integrationDelivery, agentDraft (agent-door draft, unrelated to OrderDraft), agentCallLog, productCompliance, omsConnection, catalog, catalogProduct, catalogPageAssignment, customerIntel, customerIntelRun, collectionFieldSetting. (Declared at ~lines 377-440 and 460-505.)
- Built-in constraint simulation (P2002 with `err.code = "P2002"`): `uniqMap` in `create` (automationExecution, automationVersion, personaVersion, customerMemory, followUp.idempotencyKey, customer workspace+phone, campaignContentVersion, campaignRecipient, attributionTouch.dedupeKey, integrationConnection/Token/Delivery, agentDraft, productCompliance, omsConnection, campaign.externalIdempotencyKey, catalog workspace+name) and `createMany` map (+ catalogProduct); extra wrappers: message (conversationId+externalEventId / +externalMessageId), conversation (workspaceId+channelConversationId), channel partial unique (workspace,type) when `externalId` null, channelIdentity (channelType+externalId), webhookEvent (source+externalEventId), backgroundJob **partial** unique on idempotencyKey (active only) with special `findUnique`, automationExecution. Wrappers/hydrators: conversation `findUnique`/`findMany` hydrate `channel(+account)` and `customer(+identities)`; `customer.create` nested `identities.create`; `order.create` nested `items.create` -> `orderItem` rows; `orderItem.findMany` hydrates `.order`; knowledgeDocument `include.chunks`. `RELATIONS` map auto-attaches `customer.{orders,conversations,attributionTouches}` and `conversation.channel` for relation filters.
- **Not enforced:** `OrderDraft` unique(conversationId) and unique(idempotencyKey), `Order.orderNumber` unique, `Order.idempotencyKey`. (So tests do not prove DB-level uniqueness for orders/drafts.)

### What a test for a NEW Prisma model needs in the fake DB
1. Add a `const xRows: Row[] = []` next to the others, add it to `_stores` (use the same key as the model delegate: `db._stores.myModel`), and add `myModel: makeModel("myModel", xRows, { ...defaults })` to the `db` object (delegate name = Prisma client accessor, lower-camel; note `aIUsageLog`/`aISettings` oddities).
2. Add column defaults that the Prisma schema has (`@default(...)`: enums, arrays `[]`, booleans, `Json` `{}`) - otherwise `undefined` surfaces in code under test.
3. If the schema has `@@unique`/`@unique`, add an entry to `uniqMap` in `create` (and `createMany` if used) or tests will never see P2002.
4. If code uses relations (`include`, relation filters, `select: { relation: ... }`), hand-attach rows (like the order/orderItem and conversation wrappers) or add to `RELATIONS`; the fake never joins.
5. If code uses `aggregate`, `groupBy` with several keys/`_sum`, `$transaction`, `$queryRaw`, `skip`, `distinct`, array operators other than `hasSome/hasEvery`, or `decrement`, extend the helper first.
6. Tests that mock `@/core/db/prisma` build the db inside the mock factory (see below), so a new model is available to route tests automatically once it exists in `createFakeDb`.
7. For tenant isolation add a row to `phase9-security.test.ts` (`ids` map + `it.each` sweep at lines 62-77), to `playground-isolation.test.ts:27` store list and `p6-ai-studio.test.ts:390` `PROD` list if the playground must not write it, and (if shown) to `labels-arabic.test.ts` `DISPLAYED` enum list for Arabic labels.

### Route-test pattern (mock auth + prisma) — example `tests/faqs-list-route.test.ts` (lines 18-45)
```
vi.mock("@/core/db/prisma", async () => { const { createFakeDb } = await import("./helpers/inmemory-db");
  const db = createFakeDb(); (globalThis as any).__faqsRouteDb = db; return { prisma: db }; });
vi.mock("@/modules/auth/context", async () => { ... return { getAuthContext: async () => ({ userId, workspaceId: WS, role: "SALES_AGENT",
  memberPermissions: [], can: p => p === "knowledge.read", require: p => { if (p !== "knowledge.read") throw new AuthorizationError(...) } }) }; });
vi.mock("next/headers", () => ({ cookies: async () => ({ get: () => undefined, set: () => {}, delete: () => {} }) }));
```
then `import { GET } from "@/app/api/faqs/route"`, call `GET(new NextRequest("http://localhost/api/faqs?limit=200"), { params: {} })`, and clear stores in `beforeEach` by setting every `_stores[k].length = 0`. Other examples: `p12-orders-route.test.ts` (orders/[id]), `p4-inbox.test.ts` (assign/notes), `p6-ai-studio.test.ts`, `p7-workflows.test.ts`, `persona-management.test.ts`, `upload-route-forensics.test.ts`; `p3-authz.test.ts` uses the **real** `getAuthContext` (mocks only prisma + next/headers) to test session gates. Service-level tests skip mocks entirely: `createFakeDb()` and pass `db as Prisma.TransactionClient` (phase5 pattern, lines 20-74, with `setFacebookClientFactory(...)` and `encryptSecret` to fake the channel; `processIncomingMessage` -> `processAIMessage(db, {...}, { vectorStore })`; AI provider mocked via `setMockAIProvider(new MockAIProvider())` with `intentHandler`/`responseHandler`).

---------------------------------------------------------------------
## PART 5 — Blast radius outside tests (grep of src/, prisma/, scripts/, docs/, root md)

### 5.1 OrderDraft / orderDraft / draft-service / order-drafts
Prisma / DB
- prisma/schema.prisma:1169-1178 `enum OrderDraftState`; 1180-1209 `model OrderDraft` (unique conversationId, unique idempotencyKey, indexes); 1055 `Order.draftId String?` (scalar, no relation).
- prisma/migrations/20260922000000_init/migration.sql:82 (enum), 556-578 (table), 967/970/973/976 (indexes). Per AGENTS.md schema changes are additive-only migrations (no drops) -> removal needs an explicit exception.
API routes (all under src/app/api/order-drafts)
- `route.ts:17` `prisma.orderDraft.findMany` (list by conversationId for inbox).
- `[id]/route.ts:3,11` imports `getOwnedDraft` (GET one).
- `[id]/confirm/route.ts:3-4,14,15,19-20,26-27` `getOwnedDraft`, rejects CANCELLED/CONVERTED, calls `createOrderFromDraft({draftId})`, audit `order_draft.confirmed_by_agent`.
- `[id]/cancel/route.ts:3,12` `cancelDraft`.
Modules
- src/modules/orders/draft-service.ts (275 lines): whole file (`getOrCreateDraft` 25, `getOwnedDraft` 36, `updateDraftFromExtraction` 47, `computeMissingFields` 156, `recycleDraftForNewOrder` 181, `setDraftState` 224, `cancelDraft` 238, `buildDraftSummary` 245; events `OrderDraftCreated/Updated`, `OrderReadyForConfirmation`).
- src/modules/orders/intelligence.ts (337 lines): `db.orderDraft` at 70, 72, 100, 106, 301; audits `order_draft.cancelled` 75, 112; imports draft-service at 5; `handleOrderIntent` 46, `listResolvableProducts` 320 (also used by catalog gating), re-exports `evaluateOrderRules, aggregateVerdict` at 337.
- src/modules/orders/order-service.ts: 10 (type import), 30-55 `createOrderFromDraft` (reads `db.orderDraft`), 122 (audit entityType "OrderDraft"), 164-166 (marks draft CONVERTED + `idempotencyKey`). Also contains non-draft functions to keep: `transitionOrder` 313, `getOwnedOrder` 345, `updateOrderFromChat` 366, `cancelLatestOrderFromConversation` 481, `deleteOrder` 556, `mergeCustomerData` 607.
- src/modules/orders/state-machine.ts:1,57-79 draft transitions (`DRAFT_TRANSITIONS`, `canTransitionDraft`, `assertDraftTransition`); order transitions (lines 20-47) stay.
- src/modules/orders/primitives.ts: `normalizePhone` 26, `parseQuantity` 70, `parseConfirmation` 112, `inferProvince` 129, `resolveProduct` 172, `DEFAULT_MAX_ORDER_QUANTITY` 58 (also reused by rules engine, and phone normalization may be reused elsewhere — **UNVERIFIED**, run `grep normalizePhone src` before deleting).
- src/modules/analytics/service.ts:393-411 funnel counts `db.orderDraft.count` (non-EMPTY; READY|CONFIRMING|CONFIRMED|CONVERTED) -> `draftsStarted`, `draftsReady`, `draftStartRate`, `readyRate`, `confirmRate = ordersConfirmed/draftsReady` (395-396 are the queries); UI `src/app/(dashboard)/analytics/page.tsx:392-393` («مسودات طلب», «مسودات جاهزة»).
- src/modules/conversations/threads.ts:576 (comment), 623 `orderDraft.deleteMany` in purge, 637 `order.updateMany({ draftId: null })`.
- src/modules/sales/coach.ts:72-73 `orderDraft.findFirst` fallback to get the product of the current conversation (typed ad-hoc cast; untested — no test references coach + orderDraft).
- src/modules/ai/playground-service.ts:30-31 comment only ("never runs handleOrderIntent"), `orderProposal` 66, 409-429, 520, 530 (playground builds a *proposal* without DB writes; UI `ai/playground/page.tsx:38,357-361`; i18n `playground.orderProposal` `lib/i18n.ts:306`).
- src/core/events/bus.ts:31-33 event names `OrderDraftCreated`, `OrderDraftUpdated`, `OrderReadyForConfirmation`; :66 `p.draftId` in entity-id fallback.
- src/lib/i18n.ts:488-492 labels `order_draft.cancelled|confirmed_by_agent|recycled|state_changed|updated`; 116 `draftMissing` («الحقول الناقصة»).
UI
- src/components/inbox/customer-panel.tsx:15,19,126 (`orderDraft` prop, renders `OrderDraftCard`), 157-192 `OrderDraftCard` (state badge, product, qty, phone/address ticks, missing fields); src/components/inbox/types.ts:84-93 `DraftInfo`; src/components/inbox/inbox.tsx:28 (type import), 87 state, 129-134 `fetch('/api/order-drafts?conversationId=...')`, 361 reset, 804/812 pass to panel (desktop + mobile).
Scripts / docs
- scripts/staging/concurrency.mjs:153 (`GET /api/order-drafts?conversationId=`), 163 (`POST /api/order-drafts/:id/confirm`) — staging concurrency probe depends on the confirm endpoint.
- docs/IMPLEMENTATION_PLAN.md:86, docs/PROGRESS.md:25 and 504, docs/PROJECT-REVIEW.md:139 and 318, docs/سيستم_7_التوافق_مع_OMS.md:40 and 75 (OMS doc says «من OrderDraft المكتمل ← طلب بالـ OMS» i.e. the OMS handoff contract is described in terms of the draft), SECURITY-AUDIT.md:138 (matched on pattern, workflow context — likely unrelated; **UNVERIFIED**). `docs/` was only grepped, not read in full.

AI-stage order intent (non-draft wording but part of the same removal)
- src/modules/ai/stages/extraction.ts:29 import; 214-231 `handleOrderIntent` call (intents ORDER_CREATE/UPDATE/CANCEL/UNKNOWN or `requestedAction === "CREATE_ORDER"`) + `allowedProductIds`; 281-285; 318-346; 440-462.
- src/modules/ai/stages/state.ts:11 (type import), 82 `orderResult` field.
- src/modules/ai/stages/response.ts:57, 76-78 (`summaryText` digits are exempt/validated), 131 `orderSummary` in the prompt.
- src/modules/ai/stages/gating.ts: **no** reference to draft/order (grep clean).
- src/modules/ai/intent.ts:11-14 heuristic intents ORDER_CANCEL/ORDER_UPDATE («أكد», «موافق», «أوكي» -> ORDER_UPDATE)/ORDER_CREATE; src/modules/ai/prompts.ts:35-37 intent enum, :261 missingFields rule for ORDER_CREATE; src/app/api/handoffs/route.ts:7 and handoff-service.ts:9,19 reason `ORDER_BLOCKED`; extraction.ts:338 `metadata: { reason: "ORDER_BLOCKED" }`; src/app/(dashboard)/handoffs/page.tsx:31, components/inbox/inbox.tsx:48 labels.

### 5.2 `parseConfirmation`, `parseQuantity`, `OrderDraftState`, HumanHandoff / createHandoff
- `parseConfirmation`: defined primitives.ts:112; used intelligence.ts:4 (import), 104, 237. Tests: phase5-orders.test.ts:4,135-139. Nothing else in src/.
- `parseQuantity`: defined primitives.ts:70; used intelligence.ts:176, 303. Tests: phase5-orders.test.ts:113-121. (draft-service imports `normalizePhone`/`resolveProduct`, not `parseQuantity`.)
- `OrderDraftState`: type from `@prisma/client`; draft-service.ts:1,183,224,226; state-machine.ts:1,57,67,71; prisma/schema.prisma:1170,1185. No UI/test usage other than string states `"READY"` etc. in tests.
- `HumanHandoff` (model prisma/schema.prisma:1435): src/modules/handoffs/handoff-service.ts (createHandoff :28; claim/release/resolve/route; entityType at 73,107,131,151,211; event `HumanHandoffCreated` 78 / `HumanHandoffResolved` 155); src/app/api/handoffs/route.ts:19 (comment: no Prisma relation to Conversation) and `[id]/route.ts:3` (claim/release/resolve/route); src/modules/automation/workflow-engine.ts:87 maps `HumanHandoffCreated` -> `HANDOFF_CREATED`; src/modules/conversations/threads.ts:53 (OPEN-handoff review flag), :628 purge deletes `humanHandoff`; src/modules/analytics/agents.ts:14,163; src/modules/ai/playground-service.ts:30 (comment: never writes HumanHandoff); stages/extraction.ts:328 and stages/response.ts:253 (comments: AI path creates NO HumanHandoff, every message gets an AI reply). **Important: the AI order path does not call `createHandoff` directly** — `orderResult.needsHumanReview` (extraction.ts:318) only produces an alert/`ORDER_BLOCKED` reason; see lines 318-345 (**UNVERIFIED** whether it writes a handoff row in non-hermes mode; comments say no).
- `createHandoff` tests: ai-mode-persona.test.ts:16,144; phase7-automation.test.ts:8,295-352 (create/claim race/resolve/route/least-loaded); `humanHandoff` fake model used in phase9-security.test.ts:51/57/77, token-expiry-handoff-fix.test.ts, p4-inbox, p8/p9. Handoffs are independent of OrderDraft (removing drafts does not touch them, except the `ORDER_BLOCKED` reason label if the order-blocked path disappears).
- Docs: docs/PROGRESS.md:24,631,735; docs/PROJECT-REVIEW.md:144,187; docs/IMPLEMENTATION_PLAN.md:88,261; docs/سيستم_7_التوافق_مع_OMS.md:58; docs/CODE-REVIEW-OCR-2026-10-16.md:80.

### 5.3 What remains if the draft goes: surfaces that still need an "order from conversation" entry point
`createOrderFromDraft` is the only creator of AI orders and it reads a draft row. Orders also carry attribution/OMS metadata created inside it (P8 ad-source inheritance + `recordSalesEvent` after create, order-service.ts ~170-215; automation events `OrderCreated`, `orders.ordersCreated` usage meter). A replacement must re-implement: rules check, `calculatePricing`, order+items snapshot, audit `order.created`/`price.calculated`, `WorkspaceUsage.ordersCreated`, events `OrderCreated`, attribution snapshot, sales events, idempotency. **UNVERIFIED** (lines 170-300 of order-service.ts were only skimmed, not read in full).

---------------------------------------------------------------------
## Recommendations for the plan (derived, not requirements)
- KEEP/PORT (business rules): phase5 #5,#7-10,#16-18,#19-26 (pricing/delivery/rules/idempotency/snapshot/isolation/merge), multi-order #3-#7 (edit/cancel/duplicate), ocr-wave1 pricing-in-cents (4), order-delete, p12-orders-route, phase8 revenue snapshot + isolation, conversation-purge (orders unlinked), playground/workflow zero-write invariants, HUMAN_ACTIVE and privacy-gate suites, catalog-scoped allow-list rule.
- REWRITE: phase5 #29 (e2e flow), phase6 scenario 1-4, multi-order #1-#2, catalog-scoped (7) against the new extractor, p10 funnel test, phase8 seeding, phase9 sweep row, conversation-purge setup, playground/p6/p7 store lists.
- RETIRE with the mechanism: phase5 #4,#6,#11-#13,#15,#27,#30(audit key), parts of #2 (parser table; reuse as prompt-eval corpus), primitives `parseConfirmation`, state-machine draft tests.
- UI rebuild: RETIRE fluent-design, theme visual pins (CSS selectors/art/fonts), brand-logo CSS, pwa colours, guided-tour/app-shell pins, nav counts (7 groups / 21 pages), empty-next JSX markers, smooth-flow JSX greps; KEEP-as-rules: no orphan pages, old URLs keep working, nav visibility == RBAC, every page has a guide/next step, empty states have next action, no raw `type="password"`, no hard-coded colours in tailwind/tour CSS, contrast gates, no brand string in visible text, Arabic labels for every enum, deep-link contracts.


=====================================================================
# PART 6 — LIVE DELTA (deployed tree: `...\scratch-2026-11-16-d96b4e\AIresponse-live`)
=====================================================================
Method: `diff -rq` of the two `tests/` folders + reading new/changed files and the live `src/` (read-only). Paths are relative to the live tree; line numbers are live-tree lines. Where PART 6 contradicts Parts 1-5 (old tree), PART 6 wins for the deployed system.

Headline: live `tests/` has 148 entries (vs 99 test files before). **No test file was deleted.** 47 files are NEW (46 `.test.ts` + `p16-benchmark-reply-queue.ts`, a benchmark helper that vitest does not pick up), 18 are MODIFIED, plus `helpers/inmemory-db.ts`. `tests/setup.ts` unchanged.

Tags used in 6(b): **[MECH]** pins the draft mechanism (retire/replace); **[RULE]** business rule that must survive any redesign; **[RULE\*]** rule whose assertion is valuable but whose setup is draft-coupled (port assertion, rewrite setup).

## 6(a) NEW or MODIFIED test files

### MODIFIED (18)
| File | it (old->live) | What changed | Class |
|---|---|---|---|
| accounts-management.test.ts | 10 -> 10 | bcrypt test timeout raised to 20s | SEC |
| catalog-scoped-intake.test.ts | 7 -> 7 | line 46-48: off-catalog product -> `missingFields` now ALSO contains `quantity` (quantity is per item, P22) | DRAFT |
| collection-intel-lab.test.ts | 17 -> 17 | AUTO_REPLY now also enqueues an independent `extract_customer_intel` job (key `^intel:`); analysis runs in AUTO_REPLY (status DONE, zero outbound) | OTH |
| durable-processing.test.ts | 21 -> 22 | orphan test deletes ALL jobs of the message (reply + intel); new "16b" same-millisecond burst ordering test | AI |
| empty-next.test.ts | 4 -> 4 | drops follow-ups / automation / campaigns empty-state sites; EmptyNext hrefs `>=5` | UI |
| facebook-oauth.test.ts | 3 -> 4 | authorize URL must NOT request `pages_user_profile` | CH |
| helpers/inmemory-db.ts | - | see 6(e) | helper |
| labels-arabic.test.ts | 2 -> 2 | `"OrderDraftState"` added to DISPLAYED enums (line 33-34) | UI(labels)+DRAFT |
| multi-order-flow.test.ts | 7 -> 7 | draft asserts via `findMany` + `sequence`; a **placed order is no longer editable from chat** (action NONE + audit `order.edit_requested`) | DRAFT+ORD |
| nav-search.test.ts | 9 -> 9 | removes «أتمتة/اتمتة» rows and «سير عمل» assertion | UI |
| nav-structure.test.ts | 24 -> 24 | 6 groups / 18 pages / `/workflows`->`/dashboard` (see 6(c)) | UI |
| p10-analytics.test.ts | 30 -> 30 | agent-analytics seed anchored 2h back (flake fix); draft-funnel test unchanged | AN (+DRAFT 1) |
| page-catalogs.test.ts | 22 -> 24 | new: deleting the last page-assigned catalog is refused without `force` (/آخر كتالوج/) | OTH |
| page-guides.test.ts | 22 -> 22 | `guideForPath("/workflows")` -> `/dashboard` (line 87) | UI |
| perf-msg-budget.test.ts | 12 -> 13 | new "newer turn defers while a MIDDLE older sibling is still live" | AI |
| perf-msg-latency.test.ts | 2 -> 2 | responseHandler differentiates "بكم" prompts | AI |
| phase4-ai.test.ts | 29 -> 29 | line 427-432: inbound enqueues exactly 1 `process_ai_message` AND 1 `extract_customer_intel`; runJobs processes 2 | AI |
| phase5-orders.test.ts | 30 -> 36 | see 6(b) | **DRAFT** |
| phase6-facebook.test.ts | 22 -> 22 | OAuth-state tamper test made deterministic | CH |
| smooth-flow.test.ts | 13 -> 10 | removes campaigns-segment tests and SEGMENT_PRESETS test | UI |

### NEW (46 tests + 1 benchmark)
| File | it | Covers | src imports (shortened) | Class |
|---|---|---|---|---|
| ai-fast-probe.test.ts | 6 | provider link probe | ai/provider | AI |
| ai-fast-reply-path.test.ts | 3 | extraction fast path: short no-signal msg = 0 extraction calls + real reply; price Q / digits still extract | crypto, orchestrator, provider, fb client, conversations | AI |
| ai-provider-switch-not-locked.test.ts | 6 | broken custom-provider URL must not lock settings (pure local fn) | none | AI/SEC |
| ai-stage-latency.test.ts | 5 | stage latency percentiles, pipeline writes stage timings | analytics/*, orchestrator, provider | AN/AI |
| arabic-product-name-match.test.ts | 23 | Arabic product-name normalisation/matching from customer text | products/name-match | OTH (feeds drafts) |
| audit-followups.test.ts | 21 | catalog filter, product-match, wholesale hidden, atomic order+product writes, **order-from-draft variants + Telegram retry job**, knowledge docs by page products | crypto, ai/product-match, catalog/schemas, telegram client, jobs/handlers, knowledge/*, **orders/{draft-service,order-service,telegram-delivery}**, products | ORD/OMS (+DRAFT 5) |
| catalog-boundary.test.ts | 5 | catalog-boundary prompt text, FAQ filter, reply never mentions removed product | ai/catalog-boundary, orchestrator, provider, knowledge, products | AI |
| conversation-memory-and-telegram-badge.test.ts | 4 | resetConversationMemory; telegramNotifiedOrderIds | customers/memory, orders/telegram-status | OMS |
| customer-intel-sync.test.ts | 10 | planCustomerSync -> one profile from 3 messages | conversations, intelligence/* | OTH |
| customer-profile-refresh.test.ts | 4 | real customer name refresh | fb client/profile, jobs | CH |
| draft-fills-every-message.test.ts | 19 | draft fills from ANY message; «اي» = confirm; `UNCONFIRMED_ORDER_CLAIM`; contact redaction; stage source pins | ai/guardrails, orders/{draft-service,intelligence,primitives}, products | **DRAFT** (+AI guardrail) |
| every-confirmed-order-lands.test.ts | 12 | hourly sweep re-enqueues confirmed orders missing OMS/Telegram; never duplicates | crypto, orders/delivery-sweep | **OMS** |
| intel-stays-on-your-server.test.ts | 8 | source greps: intelligence uses workspace provider, not fixed OpenRouter | reads intelligence/{service,provider}.ts | AI(privacy)+grep |
| media-on-request.test.ts | 33 | media send-once, model told truth, plural requests | orchestrator, provider, catalog, outbound, media, products | AI/CH |
| media-request-idioms.test.ts | 2 | «شكل» idiom is not a media request | media/request | OTH |
| message-stamps.test.ts | 1 | sender stamp | conversations/threads | OTH |
| never-ignore-a-message.test.ts | 14 | no inbound dropped (2 WA numbers/batch, Messenger button, no-text voice/sticker/TikTok, batch w/o workspace fails loudly) + **last-resort reply** | crypto, ai/last-resort, normalizers, inbound-job, wa webhook | AI/CH |
| oms-product-fields.test.ts | 9 | OMS v1 product fields (price, currency, stock, offers, complements, aliases...) | oms/client | **OMS** |
| oms-v1-link.test.ts | 12 | v1 client, encrypted connection, OMS products = replies products, **every confirmed order hands off once** | crypto, context-builder, oms/*, orders/{draft-service,order-service}, products | **OMS** (+DRAFT seeding) |
| order-confirm-race.test.ts | 7 | concurrent edit while confirming = 409; duplicate confirm = same order; re-confirm on edit; recipient snapshot | orders/{draft-service,order-service}, products | **DRAFT** |
| order-draft-customer-prefill.test.ts | 6 | draft pre-fills phone/address from customer profile | orders/draft-service, products | **DRAFT** |
| order-draft-multi-item.test.ts | 7 | one independent draft PER PRODUCT (sequence) | orders/{draft-service,intelligence,order-service}, products | **DRAFT** |
| p14-customer-linking.test.ts | 5 | customer link by external identity | conversations, customers | OTH |
| p14-manual-correction-audit.test.ts | 4 | manual city correction audit | conversations, customers, intelligence | OTH |
| p14-reply-extraction-independence.test.ts | 9 | reply independent of extraction job; extraction creates no order/draft; HUMAN_ACTIVE -> AI_DISABLED | crypto, orchestrator, provider, fb client, conversations, customers, jobs | AI (+DRAFT 1) |
| p15-customer-field-history.test.ts | 10 | staff corrections / field history | conversations, customers, intelligence, rbac | OTH/SEC |
| p16-knowledge-embedding.test.ts | 3 | skip embedding when vector store empty | knowledge/* | AI |
| p16-reply-pipeline.test.ts | 4 | same-cycle job pickup; slow conversation doesn't block another; per-conversation order | crypto, provider, fb client, jobs | AI |
| p17-reply-speed.test.ts | 1 | first message doesn't wait on Graph profile | crypto, provider, fb client, jobs | AI/CH |
| page-product-boundary-and-price-accuracy.test.ts | 65 | open drafts can't outlive catalog; cross-sell inside page; recall guard; media fallback; price guardrail; product prompt block; price validation; catalog page null-safe (source grep) | many ai/*, catalog, orders/{draft-service,intelligence}, products, sales | AI (+DRAFT ~5) |
| persona-binding.test.ts | 22 | persona routes (mock prisma/auth), model fallback, versioned edit | app/api/personas/route, ai/router | AI |
| price-fidelity.test.ts | 7 | reply price/currency must equal product | ai/guardrails | AI |
| product-price-validation.test.ts | 5 | price schema (<=2 decimals, compareAt>price) | products/schemas | OTH |
| product-save-error.test.ts | 5 | friendly save error text | lib/product-save-error | UI(msg) |
| product-wholesale-redaction.test.ts | 3 | wholesale/discount cap hidden from API | products/service, rbac | SEC |
| realtime-events.test.ts | 5 | SSE event build/publish/workspace filter | realtime/events | OTH |
| reliability-ops.test.ts | 7 | worker heartbeat; Telegram notification privacy; order reports per currency | analytics, jobs/worker-health, orders/notify | OMS/AN |
| reliability-repair.test.ts | 15 | HUMAN_ACTIVE stops every automated sender; concurrent confirm = 1 order; cancel event; province/discount edit; confirmed order not editable from chat; quantity parsing | events/bus, send-policy, orders/*, products | **DRAFT**+SEC |
| reliability-speed.test.ts | 12 | maintenance scheduling, same-cycle jobs, reply-job priority | analytics, outbound, jobs | AI(infra) |
| reply-every-message.test.ts | 13 | every customer message gets a reply (sequential/burst/price/follow-up), guard-rejection doesn't silence thread, timeouts don't block | crypto, env, provider, fb, jobs, knowledge, products | AI |
| reply-length-guard.test.ts | 7 | source greps: settings/ai max-token warning; health route 24h failures | reads settings/ai/page.tsx, api/health/route.ts | **UI/grep** |
| reply-speed-safe.test.ts | 18 | DEFERRED retry, inbound on reply path, extraction concurrency, orphan sweep | crypto, orchestrator, provider, fb, jobs, products | AI |
| telegram-order-ready.test.ts | 8 | Telegram order delivery once + badge; per-page required fields; HTML escape | crypto, telegram client, integrations/telegram-orders, orders/*, products | **OMS** (+DRAFT 3) |
| why-no-reply.test.ts | 7 | failure reasons grouped, isolated, sanitized | analytics/service | AN(privacy) |
| worker-capacity.test.ts | 4 | source grep of docker-compose.yml worker/postgres blocks | reads docker-compose.yml | OTH(grep) |
| worker-fanout-and-processes.test.ts | 5 | empty automation jobs, worker processes | automation/active-check, jobs/service | OTH |
| p16-benchmark-reply-queue.ts | 0 | benchmark script (not a test) | jobs/service | - |

## 6(b) ORDER-DRAFT / OMS / Telegram / guardrail tests — titles and what they pin

### draft-fills-every-message.test.ts (19)
- "بيانات الطلب بأي رسالة": "كمية بلا منتج مع مسودة مفتوحة = بيانات طلب" [MECH, `hasOrderData` intelligence.ts:51]; "رقم أو عنوان = بيانات طلب" [MECH]; "سؤال سعر بلا أي بيانات ما بيفتح طلب" [RULE*].
- "المسودة بتكمل مهما كان تصنيف الرسالة": "كمية برسالة مصنّفة سؤال بتنكتب بالمسودة" [MECH]; "الرقم والعنوان برسالة مصنّفة سؤال بينكتبوا بالمسودة" [MECH]; "بيانات الطلب بتوصل المسودة حتى لو ما في مسودة مفتوحة قبلها" [MECH]; "رسالة بلا أي بيانات طلب ما بتفتح مسودة" [RULE*]; "الطلب بيتأكد بـ«اي» بعد ما تكتمل المسودة من رسائل مصنّفة أسئلة" [RULE*] (real incident 2026-11-14: bot said "ثبّتنا الطلب" but nothing reached the OMS).
- "«اي» لحالها تأكيد": "«اي» و«أي» لحالهم = موافقة"; "«أي» جوّا سؤال بتضل «which» مش موافقة" [MECH parseConfirmation; keep the dialect corpus].
- "ما نقول «ثبّتنا الطلب» بلا رقم طلب" (`validateResponse`, `UNCONFIRMED_ORDER_CLAIM`, ai/guardrails.ts:175): "ادعاء التثبيت بلا طلب = رفض"; "صيغ تانية للادعاء كمان مرفوضة" (تم تأكيد طلبك / سجّلنا طلبك / تم تسجيل الطلب / أكدنا الطلب); "مع طلب فعلي الادعاء مسموح"; "بلا القاعدة السلوك القديم ما بيتغيّر"; "سؤال عن الناقص مسموح وهي بتجمع" — **all [RULE]**: the bot may claim confirmation only when `orderPlaced` is true. In an after-the-fact design `orderPlaced` must come from the new extraction/creation result.
- "رقم الزبون وعنوانه ما بيوصلوا الموديل": "ملخص المسودة بينشال منه الرقم والعنوان قبل الموديل" (`redactSummaryContacts`, draft-service.ts:646 -> «الهاتف: محفوظ», «العنوان: محفوظ») [RULE privacy]; "مسار الرد بيستعمل النسخة المنقّاة" = **source grep** on `ai/stages/response.ts` for `/orderSummary:\s*orderResult\?\.summaryText\s*\?\s*redactSummaryContacts/` [MECH pin].
- "مسار الذكاء بينادي منطق الطلب": "بوابة النداء بتعتمد على بيانات الطلب مش بس على التصنيف" = source grep `extraction.ts` for `/hasOrderData\(extraction\)/` [MECH]; "الحارس بياخد حالة الطلب من منطق الطلب" = source grep `response.ts` for `/orderPlaced/` [pin on mechanism name].

### order-draft-multi-item.test.ts (7) — owner decision «كل منتج مسودة مستقلة (لا سلة)»
[MECH]: "منتجان في رسالة واحدة = مسودتان، كل واحدة عنصر واحد وجاهزة"; "منتجان برسالتين + كمية بلا اسم تُحدّث آخر مسودة فقط"; "نفس المنتج مرة ثانية: نفس المسودة (آخر كمية تفوز)"; "«نعم» مع مسودتين جاهزتين = سؤال، وبعد التسمية يتأكد المطلوب" (ASK_MISSING_FIELD naming both products).
[RULE*]: "كل مسودة تُنشئ طلبها المستقل برقم ومجموع مستقل" (100x2=200 and 50x1=50, own `productNameSnapshot`); "منتج غير متوفر: طلبه يُرفض ولا ينشأ، وطلب المنتج التاني يمشي" (`ORDER_RULES_DENIED` for OUT_OF_STOCK — closes the Part-3 gap, stock rule IS tested live); "مسودة قديمة مفردة (items فارغة) لسا تنشئ طلبًا صحيحًا" [MECH legacy row]. Product question for the redesign: does "one order per product" survive?

### order-draft-customer-prefill.test.ts (6)
"زبون قديم: الاتصال يُكمَّل من ملفه والمسودة تصير READY" [RULE* prefill from profile; READY is MECH]; "زبون جديد بلا ملف: الهاتف والعنوان يضلوا ناقصين (لا اختراع)" [RULE]; "هاتف ملف غير صالح ما بينحفظ بالمسودة" [RULE]; "قيمة موجودة بالمسودة ما بتنستبدل بملف العميل" [RULE: explicit value in conversation beats profile]; "معرّف منتج من مساحة تانية ما بينقبل بالمسودة" [RULE tenant isolation]; "معرّف منتج داخل كتالوج الصفحة بينقبل وبينحفظ اسمه" [RULE catalog].

### order-confirm-race.test.ts (7)
"تعديل المسودة أثناء التأكيد = رفض 409 وما بينشأ طلب بقيم قديمة" (`DRAFT_CHANGED`, 409, 0 orders) [RULE*; claim at order-service.ts:218]; "تأكيد عادي بلا تعديل ينجح وينشئ طلباً واحداً" [RULE]; "تأكيد مكرر = نفس الطلب (created=false) بلا طلب ثانٍ" [RULE idempotency]; "تغيير الكمية على مسودة READY بيطلب تأكيداً جديداً (becameReady)", "نفس الكمية بلا تغيير ما بيعيد طلب التأكيد", "تغيير العنوان بيطلب تأكيداً جديداً" [MECH]; "الطلب بيحفظ هاتف واسم المستلم من المسودة حتى لو اختلف عن ملف العميل" [RULE recipient snapshot].

### every-confirmed-order-lands.test.ts (12) — all [RULE]; works on Order rows only, survives a redesign unchanged
OMS (`sweepOrdersMissingOms`, delivery-sweep.ts:76): "طلب انعمل قبل ما الـ OMS ينربط بينزل بعد ما ينربط"; "ما بيكرّر: الدورة الجاية ما بتعمل مهمة تانية"; "طلب نزل بالـ OMS أصلًا ما بينبعت مرة تانية"; "طلب ملغي ما بينزل"; "طلب لسا ما تأكد ما بينزل"; "الإرسال التلقائي مطفي = ما في كنس"; "الـ OMS مش مربوط = ما في كنس"; "طلب أقدم من نافذة الكنس ما بينسحب" (72h). Telegram (`sweepOrdersMissingTelegram`, :132): "طلب انعمل قبل ما البوت ينضبط بيوصل بعدها"; "طلب وصل المجموعة أصلًا ما بينبعت مرة تانية"; "البوت مش مضبوط = ما في مهام فاضية"; "ما بيكرّر: الدورة الجاية ما بتعمل مهمة تانية". Scheduled at integrations/scheduler.ts:78-80.

### oms-v1-link.test.ts (12) + oms-product-fields.test.ts (9) — [RULE], draft only in seeding
oms-v1-link: "عميل باب v1: بيقرأ منتج الـ OMS ... وبيكتب جسم الطلب حسب v1OrderSchema"; "التوكن بينحفظ مشفّر وما بيرجع للواجهة · الإرسال التلقائي مفعّل من أول حفظ"; "رابط http عام أو داخلي بينرفض قبل ما يطلع التوكن"; "نفس الرمز بيتحدّث · نفس الاسم بياخد رمز الـ OMS ... المحلي بيتوقف (مش بينحذف)"; "رد فاضي أو صفحة ممتلئة = ولا منتج بيتوقف"; "منتج من الـ OMS: أسعار مسمّاة قديمة عندنا ما بتوصل للذكاء (السعر والعروض من الـ OMS بس)"; "توكن مرفوض = ولا تغيير بالمنتجات"; "إنشاء الطلب بيحط مهمة إرسال بمفتاح فيه المساحة — والمهمة بتنزّله بالـ OMS مرة وحدة" (job `handoff_order_to_oms`, key `autoHandoffKey(WS, orderId)`, header `Idempotency-Key` = `omsIdempotencyKey`, POST `/api/v1/orders`, body `source: "replies"`, `items[{product_code, quantity, unit_price "15.00"}]`; order gets `omsOrderId/omsOrderNumber/handoffMode`; repeat = no 2nd POST); "الإرسال التلقائي مطفي = ولا مهمة (والطلب بينبعت يدويًا)"; "رفض نهائي من الـ OMS (منتج مش عنده) = ولا إعادة، والسبب بينحفظ" (`oms.auto_handoff_refused`, `OMS_REJECTED`); "الـ OMS واقف (500) = المهمة بتفشل وبتنعاد" (`OMS_ERROR`); "السعر المعدّل من الـ OMS بينحفظ بالسجل" (`oms.order_handed_off`). **Coupling:** helper `confirmedOrder()` (~203-223) builds the order via `updateDraftFromExtraction` + `createOrderFromDraft`; the job is enqueued inside order creation (`order-service.ts:439-440 maybeEnqueueAutoHandoff`) — any new creation path must call it.
oms-product-fields: price/currency/stock; offers; complements; aliases cleaned; no aliases = []; description/category/image empty today and read when sent; alternate field names; empty text never wipes a hand-written description.

### telegram-order-ready.test.ts (8), conversation-memory-and-telegram-badge.test.ts (4)
"مسودة READY ما بترحّل: لا مهمة ولا إرسال" [RULE*: only real orders are sent]; "إنشاء الطلب يرسل مرة وحدة للمجموعة/التوبيك + يسجّل الشعار" [RULE]; "إعدادات مطفية = لا إرسال ولا شعار (الطلب ينشأ)" [RULE]; "فشل تلجرام = الطلب ينشأ بلا شعار وبلا كسر" [RULE]; "حقول الصفحة: بدون عنوان تبقى المسودة جاهزة إذا العنوان غير مطلوب"; "حقل الاسم مطلوب: ما بتصير جاهزة قبل توفّر الاسم"; "تغيير حقول الصفحة بعد الجاهزية: الإنشاء يعيد التحقق ويرفض الناقص" [RULE*: per-page required fields `Channel.orderRequiredFields`, recheck at creation]; "يهرّب HTML ويمنع حقن الوسوم" [RULE].
conversation-memory-and-telegram-badge: "resetConversationMemory: يحذف ذاكرة هذه المحادثة فقط + الملخّص، ويترك الباقي"; "محادثة مساحة أخرى = غير موجودة"; "telegramNotifiedOrderIds: يرجّع فقط الطلبات اللي إلها سجل إرسال ناجح، وبعزل المساحة"; "بلا طلبات = مجموعة فاضية بلا استعلام" [RULE, no draft].
audit-followups (order part, lines 86-246): "فشل حذف الطلب ما بيمسح بنوده"; "فشل حفظ الأسعار الإضافية ما بيترك منتج ناقص"; "سباق على نفس SKU بيرجع 409 مش خطأ 500"; "تعديل الكمية من المحادثة: فشل تحديث الطلب بيرجّع البند كما كان"; "تغيير الهاتف من المحادثة بيتحدّث على الطلب نفسه"; "المتغير الصحيح بيتسعّر بسعره"; "متغير مش تابع للمنتج بيتجاهل وبيرجع لسعر المنتج"; "فشل تلجرام بيحفظ مهمة إعادة محاولة، والإعادة بترسل مرة وحدة"; "نجاح تلجرام من أول مرة ما بيضيف مهمة" — [RULE/RULE*]. **UNVERIFIED conflict:** "تعديل الكمية/تغيير الهاتف من المحادثة" vs the new rule that confirmed orders are NOT edited from chat (multi-order-flow, reliability-repair:290) — check which order statuses each applies to.

### multi-order-flow.test.ts (7, modified)
1 "a new order request after a confirmed order starts a SECOND order" — reads `findMany`, picks open draft with highest `sequence`: [MECH]; 2 orders with distinct ids: [RULE]. 2 "recycled draft keeps the customer contact from the previous order" — new draft inherits phone/address/province/name [RULE*]. 3 "duplicate rule no longer blocks a different-product second order" [RULE]. 4 **changed** "a quantity edit on a placed order is refused and only logged for review" (`action NONE`, total unchanged, quantity stays 3, 1 order, audit `order.edit_requested`) [RULE — the old "quantity edit reprices" rule is gone]. 5 **changed** "an address edit on a placed order is refused and only logged for review" [RULE]. 6 "delivered/terminal orders refuse edits" [RULE]. 7 "cancelling after a placed order cancels that ORDER (not just the draft)" + new order possible after [RULE].

### catalog-scoped-intake (7, modified), catalog-boundary (5), page-product-boundary-and-price-accuracy (65)
catalog-scoped-intake titles as Part 3.3; only change: off-catalog product -> `missingFields` also contains `quantity`. [RULE catalog boundary, MECH draft].
catalog-boundary: "كتالوج فاضي: منع أي ذكر لمنتج"; "كتالوج فيه منتجات: يسمّيها ويمنع ما عداها"; "يتجاهل الأسماء الفارغة ويحدّ العدد"; "أسئلة المنتج المحذوف ما بترجع، والعام والمسموح بيرجعوا"; "سؤال عن منتج خارج الكتالوج: تعليمات الرد فيها الحدود وبلا حقائق المنتج المحذوف" — [RULE] (`ai/catalog-boundary.ts`).
page-product-boundary: "Goal 1 item 1: open drafts cannot outlive the catalog": "a READY draft for a removed product is not confirmed, and keeps the contact fields" [RULE*, `retireOutOfCatalogDraftItems` draft-service.ts:600]; "an unassigned page (empty allowed set) cannot confirm either"; "the which-order question never names the removed product"; "no catalogs (undefined allowed set) keeps confirming as before". Then cross-sell inside the page; recall guard (memory/summary/history never reach the model for a removed product, current message kept); media fallback respects boundary; active-only allowed set; name-list cap (>40 names); price guardrail (numeric compare, thousand-scale forms, currency tokens, FAQ-grounded amounts); prompt product block; price validation — [RULE]. One **source grep**: `catalogs/[id]/page.tsx` must not contain `p.product.id`, `{pg.channel.displayName}`, `pg.channel.id` and must contain `p?.name` (lines 642-657).

### phase5-orders.test.ts (30 -> 36)
New: "parses tens/hundreds and composites (Phase 0.2)" (عشرين=20، مية=100، «مية وخمسين»=150، «خمسة وعشرين»=25) [MECH, keep as eval corpus]; "never reads a quantity from inside words, codes or day names (Phase 0.2)" (الاثنين/استفسار/LG50/TW-500 -> null) [MECH]; "refuses to guess when two unrelated quantities appear (Phase 0.2)" (`بدي 3 و 5` -> null, `AMBIGUOUS_QUANTITY`) [MECH]; "never turns a question or greeting into a decision (Phase 0.1)" (UNKNOWN for أي لون عندكم؟ / أهلا / «نعم لا» / «بلا مشكلة بدي أطلب») [MECH]; "keeps a negation of another verb from becoming a refusal (Phase 0.1)" [MECH]; "keeps a permissive default for workspaces with no delivery rules at all" (`NO_RULE_DEFAULT_FEE`) [RULE].
Changed: `parseConfirmation` non-confirmation = `"UNKNOWN"` (lines 162-163; was NONE); "marks unavailable provinces and unknown provinces": a workspace WITH delivery rules now treats an unconfigured province as undeliverable (`deliveryAvailable false`, `NO_DELIVERY_RULE`, Phase 1.5) [RULE changed]; draft tests: profile phone already on the draft so `missingFields` = `["quantity","address"]` (~272) / `["address"]` (~273); invalid AI phone now leaves the profile phone "0930000000" (~298). All other titles as Part 3.1.

### Reply / never-ignore / last-resort / HUMAN_ACTIVE / privacy tests (all [RULE])
- never-ignore-a-message (14): "رقمين واتساب بنفس الدفعة: ولا رقم بيضيع" ("رسالة كل رقم بتوصل — مش بس الأول", "إعادة تسليم نفس الدفعة بتضل مكرر"); "كبسة زر بالماسنجر = رسالة زبون" (button text; same click twice = same key; echo dropped); "رسالة بلا نص بتوصل كمان" (Telegram voice/sticker recorded; bot reply and no-content update dropped; TikTok no-text recorded); "دفعة رسائل بلا مساحة ما بتنحسب «تمّت» -> بتفشل بصوت عالي"; "خلصت محاولات الرد: الزبون بياخد رد آمن بدل السكوت": "بيبعت النص الآمن", **"ما بيبعت إذا الموظف مستلم المحادثة"**, "ما بيبعت إذا الزبون أصلًا أخد رد بعدها" (`ai/last-resort.ts:33`, called from `jobs/service.ts:595-596`).
- reply-every-message (13): three sequential, three back-to-back, price/quantity messages «قديش سعره»/«بدي 2», follow-up after price reply, greeting -> product Q -> «تمام بدي اطلب»; guard-rejected message -> `AI_FAILED` and next answered immediately; regenerate on the model that answered; embedding failure/timeouts don't block; stale retry gives no late reply.
- why-no-reply (7): reasons grouped with counts; workspace-isolated; none when no failures; sanitized (strips phone and ids); empty reason readable; long text cut.
- p14-reply-extraction-independence (9): "الرد يُرسل ومهمة الاستخراج مازالت PENDING"; "مزوّد الاستخراج لا يُنادى على مسار الرد إطلاقًا"; "runJobs يعالج الرد والاستخراج بنفس الدورة بلا إرسال مزدوج"; "فشل استخراج العميل لا يفقد الرد"; "تأخير متعمد للاستخراج: الرد يخرج قبل انتهائه"; "الاستخراج البطيء لاحقًا لا ينتج إرسالًا مزدوجًا"; **"استخراج بيانات العميل لا ينشئ طلبًا ولا مسودة ولا يغيّر سعر المنتج"** (a background order extractor needs an explicit separate path; this test must be REWRITTEN, not just retired); **"HUMAN_ACTIVE يمنع الرد الآلي (AI_DISABLED، بلا إرسال)"**; "ملف العميل يقرأ البيانات المحفوظة فعليًا". Closest existing pattern to the planned after-the-fact extraction: separate `extract_customer_intel` job (key `intel:...`) enqueued with the reply job (phase4-ai 427-432, collection-intel-lab 200-213).
- p16-reply-pipeline (4), p17-reply-speed (1), ai-fast-reply-path (3): same-cycle pickup; A doesn't block B; same-conversation ordering; disabled-workspace AI no hot-loop; first message doesn't wait for Graph profile; short no-signal message = 0 extraction calls, price question / digits still extract (`ai-fast-reply-path.test.ts:91-147` — interacts with `hasOrderData`).
- reliability-repair (15): "يقرأ الوضع والحالة باستعلام واحد"; "HUMAN_ACTIVE يوقف كل رد آلي (AI · أتمتة · متابعة · نظام)"; "HUMAN_ACTIVE يسمح بالرد اليدوي ويرفض الحملة..."; "الحملة تُبثّ على سلسلة يملكها النظام حتى لو HUMAN_ACTIVE"; "COLLECTION_ONLY يوقف الآلي وحتى بثّ الحملة"; "وضع غير معروف = رفض آمن"; isolation; "طلبان متزامنان ينتجان طلبًا واحدًا فقط (مفتاح عدم التكرار)"; "ينشر OrderCancelled ولا ينشر ConversationUpdated"; province edit keeps discount; undefined province with delivery rules = unavailable; "طلب مؤكد لا يُعدّل من المحادثة ويُسجّل طلب تعديل"; quantity parsing in/out of order context ("عشرين بلا نية طلب وبلا مسودة = ما بتنشئ طلب ولا مسودة") — send-guard group [RULE], concurrency [RULE*], parsing [MECH].
- price-fidelity (7): reply price/currency must equal the product [RULE].

### Mechanism vs rules — consolidated (live)
Retire with the draft: per-product `sequence`/`items` (multi-item 1-4,7), `hasOrderData` gate and the source pins (draft-fills grep tests; page-catalogs.test.ts:273-279 still greps `extraction.ts`), READY/COLLECTING asserts, re-confirm-on-edit (order-confirm-race 4-6), `parseConfirmation` YES/NO/UNKNOWN, `parseQuantity` (keep corpus), `order_draft.*` labels, `OrderDraftState` label test, draft funnel, `/api/order-drafts` + inbox draft cards.
Must survive (port tests): price from DB/OMS only (phase5 #7-10,19-21; audit-followups variants; oms-v1-link; price-fidelity; page-product-boundary Goal 2); catalog boundary per page; idempotent creation and concurrent-confirm single order (phase5 #19, order-confirm-race, reliability-repair 1.1; fake DB now enforces Order `idempotencyKey`/`orderNumber` uniqueness); rules verdicts incl. OUT_OF_STOCK and `NO_DELIVERY_RULE`; confirmed order never rewritten from chat + `order.edit_requested`; confirmed order reaches OMS exactly once and Telegram exactly once (oms-v1-link, every-confirmed-order-lands, telegram-order-ready, audit-followups); `UNCONFIRMED_ORDER_CLAIM`; never ignore a message + last-resort reply; HUMAN_ACTIVE = no automated reply (reliability-repair 96-109, p14 236, phase9:137, never-ignore "ما بيبعت إذا الموظف مستلم"); privacy masking (`redactSummaryContacts`, why-no-reply sanitation, Telegram no name/phone by default, p1 privacy-gate); per-page required order fields; playground/workflow zero-write; tenant isolation.
Gaps live: no test hits `/api/order-drafts/*` or `/api/channels/by-id/[id]/order-fields`; `sales/coach.ts:72` draft fallback untested.

## 6(c) Part 2 UPDATE — UI pins that changed live
- **nav-structure.test.ts**: line 63-65 `it("٦ مجموعات بمفاتيح فريدة")` -> `expect(NAV_GROUPS).toHaveLength(6)` and unique keys `toBe(6)`; line 97-98 `it("... القائمة ١٨ تبويب")` -> `expect(NAV_GROUPS.flatMap((g) => g.pages)).toHaveLength(18)`; line 138 `expect(hit("/workflows")?.page.href).toBe("/dashboard")` (was `/automation`). `OLD_NAV_HREFS` (55-60) unchanged and still lists `/automation`, `/workflows`, `/follow-ups`; live `NAV_REDIRECTS` (nav-model.ts:112-121) maps `/campaigns`, `/automation`, `/follow-ups`, `/workflows` -> `/dashboard` and the `(dashboard)` folder still contains those four directories (test lines 88-95 `redirectToMerged("<href>")` apply). Live groups: home(/dashboard), chats(/conversations,/handoffs,/customers,/intelligence), sales(/orders,/products,/catalogs), ai(/personas,/knowledge,/media,/feedback,.../ai/playground,/settings/ai), reports(/analytics), settings(/channels,/settings/team,/settings/account) = 6 groups / 18 pages; the «الأتمتة» group is gone. A rebuild must keep 6/18 or edit these numbers; the "no orphan page" and "old links keep working" rules are unchanged.
- **nav-search.test.ts**: removed `["أتمتة","/automation"]`, `["اتمتة","/automation"]` and the «سير عمل» assertion; keyword-coverage and app-shell combobox pins unchanged.
- **page-guides.test.ts** line 87: `guideForPath("/workflows")?.href` -> `"/dashboard"`; rest unchanged.
- **empty-next.test.ts**: SITES loses follow-ups, automation, campaigns (11 left); line 68 `expect(hrefs.length).toBeGreaterThanOrEqual(5)`.
- **smooth-flow.test.ts**: campaigns "شريحة ← حملة" describe and `SEGMENT_PRESETS` describe removed (13 -> 10 tests).
- **labels-arabic.test.ts** (new pin): DISPLAYED now includes `"OrderDraftState"` (line 34); the helper does `expect(m, "enum OrderDraftState").not.toBeNull()` — **fails if the enum leaves `prisma/schema.prisma` before the test is edited**, and requires Arabic `LABELS` for EMPTY/COLLECTING/READY/CONFIRMING/CONFIRMED/CONVERTED/CANCELLED while it exists.
- **NEW source-grep pins**: (1) draft-fills-every-message: `response.ts` `/orderSummary:\s*orderResult\?\.summaryText\s*\?\s*redactSummaryContacts/` and `/orderPlaced/`; `extraction.ts` `/hasOrderData\(extraction\)/`. (2) reply-length-guard: `settings/ai/page.tsx` must not contain «كل ١٠٠ ≈ ٧٠ كلمة», must match `/بالعربي كل ١٠٠/`, contain `SAFE_ARABIC_MAX_TOKENS` (>=300), `RiskNote`, `FixNote`, `/form\.maxTokens < SAFE_ARABIC_MAX_TOKENS/`; `api/health/route.ts` contains `RECENT_FAILURE_WINDOW_MS = 24 * 60 * 60 * 1000`, `/status:\s*"FAILED",\s*finishedAt:\s*\{\s*gte:/`, `failedJobsAllTime`. (3) page-product-boundary:642-657 on `catalogs/[id]/page.tsx`. (4) intel-stays-on-your-server: `intelligence/service.ts` must call `getWorkspaceExtractionProvider(` not `getExtractionProvider(`, contain `params.options ??`; `provider.ts` `/aISettings[\s\S]{0,20}findUnique/`, `getWorkspaceProvider(`, `resolveModel(settings, "EXTRACTION")`, no "openrouter" in that function. (5) worker-capacity: `docker-compose.yml` worker block has `WORKER_PROCESSES: ${WORKER_PROCESSES:-4}`, `AI_JOB_CONCURRENCY: ${AI_JOB_CONCURRENCY:-16}`, no `scale:`; postgres `max_connections=${POSTGRES_MAX_CONNECTIONS:-200}`. **No new source pins on the inbox, orders or channels pages**; inbox/customer-panel are still touched only by smooth-flow (customer-panel `href={`/orders/${o.id}`}`) and persona-management (channels/page.tsx brand check).
- Byte-identical live: fluent-design, theme-system, brand-logo, password-input, role-matrix, pwa-manifest-og, provider-labels, p12-merge-main, guided-tour.

## 6(d) Part 5 UPDATE — live-tree blast radius
Grep terms: OrderDraft|orderDraft|order_draft|draft-service|order-drafts|createOrderFromDraft|handleOrderIntent|hasOrderData|maybeEnqueueAutoHandoff|parseConfirmation|parseQuantity|OrderDraftState|redactSummaryContacts|UNCONFIRMED_ORDER_CLAIM|listOpenDrafts|updateDraftFromExtraction|getOwnedDraft|cancelDraft|required-fields.

**Prisma**
- prisma/schema.prisma:1223 `enum OrderDraftState`; 1233-1264 `model OrderDraft` (`sequence Int @default(1)`, `items Json @default("[]")`, `@@unique([conversationId, sequence])`, `idempotencyKey @unique`, `orderId`); :188 `Channel.orderRequiredFields String[] @default(["product","quantity","phone","address"])`; `Order.draftId` still exists (conversations/threads.ts:688).
- Migrations: 20260922000000_init, **20261101142509_order_draft_items**, **20261106090000_order_draft_per_product** (additive; dropping the table would be the first destructive migration — AGENTS.md says additive-only).
**API routes**
- src/app/api/order-drafts/route.ts:4 (`normalizeOrderFields`), :18 `prisma.orderDraft.findMany`, :36 and :49 (per-page `orderRequiredFields`); [id]/route.ts:3,11 `getOwnedDraft`; [id]/cancel/route.ts:3,12 `cancelDraft`; [id]/confirm/route.ts:3,4,14,19-20 (`createOrderFromDraft({draftId})`), :26-27 audit `order_draft.confirmed_by_agent`/`OrderDraft`.
- src/app/api/channels/by-id/[id]/order-fields/route.ts:6,22,28,46 (per-page required-fields GET/PATCH); src/app/api/channels/facebook/route.ts:7,29 (returns `orderRequiredFields`).
**Modules**
- src/modules/orders/draft-service.ts: `REQUIRED_DRAFT_FIELDS` 15, `parseDraftItems` 50, `OPEN_DRAFT_STATES` 95, `listOpenDrafts` 98, `getOrCreateDraft` 172, `getOwnedDraft` 224, `updateDraftFromExtraction` 235, `computeMissingFields` 457, `getRequiredOrderFields` 496-513, `recycleDraftForNewOrder` 531, `setDraftState` 575, `cancelDraft` 589, `retireOutOfCatalogDraftItems` 600, `redactSummaryContacts` 646, `buildDraftSummary` 652; events `OrderDraftCreated` 160, `OrderDraftUpdated` 324/443/564, `OrderReadyForConfirmation` 431.
- src/modules/orders/intelligence.ts: `hasOrderData` 51, `handleOrderIntent` 80, `listResolvableProducts` 419; direct `db.orderDraft` writes/reads at 123, 133, 174, 288.
- src/modules/orders/order-service.ts: 15 imports `computeMissingFields, getRequiredOrderFields, parseDraftItems`; 39 `createOrderFromDraft`; 44, 73, 82 draft reads/updates; 201 audit entityType "OrderDraft"; 218 `tx.orderDraft.updateMany` (claim = concurrency guard, `DRAFT_CHANGED` 409); 264 `tx.orderDraft.update`; 279-282 refetch; **407 `publishRealtime`, 422 `deliverOrderTelegram`, 439-440 `maybeEnqueueAutoHandoff` (dynamic import of `@/modules/oms/auto-handoff`)**; 551 audit `order.edit_requested`.
- src/modules/orders/primitives.ts: `parseQuantity` 111, `parseConfirmation` 209 (YES/NO/UNKNOWN); state-machine.ts:1,57,67,71 draft transitions; orders/required-fields.ts (whole file; used by draft-service and the order-drafts/order-fields APIs).
- OMS: oms/auto-handoff.ts (`autoHandoffKey` 26, `maybeEnqueueAutoHandoff` 39, `handleAutoHandoffJob` 65, `sweepStuckOmsHandoffs` 100, `enqueueHourlyProductSyncs` 147, `handleProductSyncJob` 169), oms/{client,connection,handoff,sync}.ts; jobs/handlers.ts:746 `handoff_order_to_oms`, :750 `sync_oms_products`; jobs/service.ts:42-43; settings/ai/why-no-reply-card.tsx:32-33 labels. These start from an **Order**, not a draft (only hook: order-service.ts:439).
- Telegram: orders/telegram-delivery.ts (`deliverOrderTelegram` :10, realtime :82), orders/notify.ts, orders/telegram-status.ts, integrations/telegram-orders.ts; callers order-service.ts:422 and jobs/handlers.ts:630. Sweeps: orders/delivery-sweep.ts:76/:132, scheduled at integrations/scheduler.ts:78-80.
- Realtime: realtime/events.ts + `/api/events`; `publishRealtime` callers: channels/outbound.ts:328,613,864; conversations/inbound.ts:283; conversations/threads.ts:521,540,554,610; handoffs/handoff-service.ts:81,117,147,176; orders/order-service.ts:407; orders/telegram-delivery.ts:82.
- AI stages: ai/stages/extraction.ts:33 imports `handleOrderIntent, hasOrderData`; :300 `orderResult`; :303-308 gate (`ORDER_CREATE|ORDER_UPDATE|ORDER_CANCEL|UNKNOWN` or `requestedAction === "CREATE_ORDER"` or `hasOrderData(extraction)`); :311 call; :406 `orderResult?.needsHumanReview`; :426 `reason: "ORDER_BLOCKED"`. ai/stages/response.ts:27 imports `redactSummaryContacts`; :171 `orderSummary: ... redactSummaryContacts(...)`. ai/stages/state.ts:11,82. ai/guardrails.ts:175 `UNCONFIRMED_ORDER_CLAIM`. ai/last-resort.ts:33 (jobs/service.ts:595-596). ai/catalog-boundary.ts, ai/product-match.ts, products/name-match.ts (draft product resolution). ai/playground-service.ts:30-31 comments only.
- Analytics: analytics/service.ts:428-429 `db.orderDraft.count` (draftsStarted/draftsReady funnel).
- Conversations: conversations/threads.ts:674 `orderDraft.deleteMany`; :688 `order.updateMany({ draftId: null })`.
- Sales: sales/coach.ts:72-73 `orderDraft.findFirst` (still untested).
- Events: core/events/bus.ts:31-32 `OrderDraftCreated`, `OrderDraftUpdated` (declaration of `OrderReadyForConfirmation` not matched in live grep although emitted at draft-service.ts:431 — UNVERIFIED).
- i18n: lib/i18n.ts:495-499 `order_draft.*`; `OrderDraftState` values need labels (labels-arabic.test.ts:34).
**UI**: components/inbox/customer-panel.tsx:17,23 (`orderDrafts: DraftInfo[]`, plural), :175-178, :233 `OrderDraftCard`; components/inbox/inbox.tsx:88, :130 (`fetch('/api/order-drafts?conversationId=...')`), :137-139, :384, :855, :865; `DraftInfo` in components/inbox/types.ts (line UNVERIFIED).
**Scripts**: scripts/staging/concurrency.mjs:153,163 (list + `POST /api/order-drafts/:id/confirm`); **scripts/db-counts.ts:22 lists `"orderDraft"`** (new vs old tree).
**Docs** mentioning drafts/route: docs/HANDOVER-2026-10-27.md, IMPLEMENTATION_PLAN.md, PROGRESS.md, PROJECT-REVIEW.md, docs/سيستم_7_التوافق_مع_OMS.md, AGENTS.md, SECURITY-AUDIT.md (not read in full).
**`parseConfirmation`/`parseQuantity`**: defined primitives.ts:209/:111; only src consumer is orders/intelligence.ts (line numbers shifted, UNVERIFIED). **`maybeEnqueueAutoHandoff`**: def oms/auto-handoff.ts:39; sole caller order-service.ts:439-440. **`createOrderFromDraft` callers**: order-service.ts:39, order-drafts/[id]/confirm/route.ts:4,19, intelligence.ts (confirm path), plus tests (phase5, multi-order-flow, order-draft-multi-item, order-confirm-race, oms-v1-link, telegram-order-ready, audit-followups, reliability-repair, phase8-analytics).

## 6(e) Fake DB differences (live tests/helpers/inmemory-db.ts)
- NEW model `telegramOrderSettings` (`makeModel(..., { enabled: false })`, `_stores.telegramOrderSettings`). `omsConnection`, `catalog*`, `customerIntel*`, `collectionFieldSetting` already existed.
- NEW unique in `create` uniqMap: `order: (r, w) => (w.idempotencyKey != null && r.idempotencyKey === w.idempotencyKey) || (w.orderNumber != null && r.orderNumber === w.orderNumber)` -> P2002. **Still not simulated:** `OrderDraft @@unique([conversationId, sequence])` and `OrderDraft.idempotencyKey`.
- NEW defaults: `orderDraft: { state: "COLLECTING", sequence: 1, missingFields: [], items: [], phone/productId/quantity/rawAddress/province/customerName/orderId: null, ... }`; `channel` default `orderRequiredFields: ["product","quantity","phone","address"]`.
- NEW `$transaction(fn)` (P15): interactive only; snapshots every store with shallow row copies, runs `fn(db)` with `$transaction` hidden (no nesting), restores all stores on throw. Not supported: array form, isolation levels, deep rollback of nested JSON (e.g. `items` mutated in place would not roll back — UNVERIFIED impact on order creation, which uses `tx.orderDraft.updateMany` at order-service.ts:218).
- IDs zero-padded (`c-${name}-${String(n).padStart(6,"0")}-${rand}`) so id ordering is stable beyond 9 rows (relates to durable-processing 16b).
- Otherwise identical to Part 4 (same where operators, `increment` only, no `aggregate`/`$queryRaw`, no `select` projection). `findUnique({ where: { conversationId } })` on orderDraft now returns the FIRST matching row (many per conversation); old-style `draftRow()` helpers (phase5-orders lines 94-99, unchanged live) silently pick the lowest-sequence draft.

## 6(f) Updated recommendations
- The live draft is a much bigger surface: per-product rows, `items` JSON, per-page required fields, optimistic claim in `createOrderFromDraft`, and three side effects at order creation (realtime, Telegram, OMS auto-handoff job). Safest seam for the redesign = **order creation**: one function doing rules + pricing + snapshot + idempotency + `order.created` audit + `OrderCreated` event + `deliverOrderTelegram` + `maybeEnqueueAutoHandoff`, and port the [RULE]/[RULE*] tests above to call it directly (seed from the new extraction record instead of a draft).
- Tests that fail immediately when `OrderDraft` goes: phase5-orders (draft sections + e2e), multi-order-flow, catalog-scoped-intake, draft-fills-every-message, order-draft-multi-item, order-draft-customer-prefill, order-confirm-race, oms-v1-link (seeding), telegram-order-ready, audit-followups (variants/telegram), reliability-repair, page-product-boundary (Goal 1), phase6-facebook (scenario 1-4), phase8-analytics, phase9-security, p10-analytics, conversation-purge, playground-isolation, p6-ai-studio, p7-workflows, p14-reply-extraction-independence (store asserts), labels-arabic (enum), and the fake DB defaults. Tests that survive untouched: every-confirmed-order-lands, oms-product-fields, conversation-memory-and-telegram-badge, never-ignore-a-message, reply-every-message, why-no-reply, p16/p17/ai-fast-*, price-fidelity, catalog-boundary, realtime-events, reliability-ops.
