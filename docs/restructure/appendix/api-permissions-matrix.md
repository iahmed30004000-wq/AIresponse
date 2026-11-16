# RBAC inventory — AIresponse (Next.js 14 / Prisma / zod)

Read-only inventory. Nothing in the repo was modified. All paths are relative to the repo root (`...\AIresponse`). Line numbers refer to the file at the time of reading. "UNVERIFIED" = not confirmed in code.

> Public copy. Section 2d and a few remarks in Parts 3–4 about account-management rules are withheld because this repository is public; the owner holds the full copy.

Legend for UI caller column: `D/` = `src/app/(dashboard)/`, `C/` = `src/components/`. "dyn" = path built dynamically (template variable), matched by reading the call site. `L<n>` after a permission = line of the `permission:` option (or `export` line when single-line).

Counting method: route files = 134 (`find src/app/api -name route.ts`). Exported HTTP handlers = 194 (sum of the methods below).

---------------------------------------------------------------------------

# PART 1 — Route matrix (134 routes)

Auth modes: `P:x` = createHandler `permission: x` (auth + active membership + permission via `auth.require`, handler.ts:89-92). `AUTH` = createHandler without `permission` and not `public` (any active member of the session workspace). `PUBLIC` = `public: true`. Rate limit = createHandler `rateLimit` (per IP+path). CSRF Origin check applies to every createHandler POST/PATCH/PUT/DELETE when an Origin header is present (handler.ts:65-76).

## 1.1 /api/ai (7)

| # | Path | Methods → required permission | Extra guards | Purpose | UI callers |
|---|---|---|---|---|---|
| 1 | /api/ai/playground | POST → P:ai.configure (L13) | none | Retired stub: always returns 410 GONE | NONE |
| 2 | /api/ai/playground/sessions/[id]/messages | POST → P:ai.configure (L29) | RL 20/min | Send a message in a playground session (AI reply) | D/ai/playground/page.tsx:187 |
| 3 | /api/ai/playground/sessions/[id] | GET → P:ai.configure (L8); DELETE → P:ai.configure (L15) | – | Read / delete a playground session | D/ai/playground/page.tsx:99 (GET), :150 (DELETE) |
| 4 | /api/ai/playground/sessions | GET → P:ai.configure (L15); POST → P:ai.configure (L22) | POST RL 30/min | List / create playground sessions | D/ai/playground/page.tsx:82,:141 (GET), :120 (POST) |
| 5 | /api/ai/providers/test | POST → P:ai.configure (L25) | RL 5/min | Live test of an AI provider key/endpoint | D/settings/ai/page.tsx:68 |
| 6 | /api/ai/settings | GET → P:ai.configure (L41); PATCH → P:ai.configure (L56) | – | Read / update workspace AI settings | D/settings/ai/page.tsx:198 (GET), :278 (PATCH) |
| 7 | /api/ai/status | GET → P:ai.configure (L21) | – | AI status card (presence flags, no secrets) | D/settings/ai/ai-status-card.tsx:82 |

## 1.2 /api/analytics (18) — `analyticsRoute` default = analytics.read (`_shared.ts:28`)

UI callers for all `analytics/<name>` GETs: `D/analytics/page.tsx:242` (dyn, `endpoints` map at :204-213); `overview` also `D/analytics/page.tsx:250` and `D/dashboard/page.tsx:81`.

| # | Path | Methods → permission | Extra guards | Purpose | UI callers |
|---|---|---|---|---|---|
| 8 | /api/analytics/agents | GET → P:analytics.agents (route L7, via analyticsRoute 2nd arg) | – | Agent performance report (sensitive) | D/analytics/page.tsx:242 (dyn, tab "agents") |
| 9 | /api/analytics/ai | GET → P:analytics.read (L4) | – | AI analytics | D/analytics/page.tsx:242 (dyn) |
| 10 | /api/analytics/automations | GET → P:analytics.read (L4) | – | Automation analytics | D/analytics/page.tsx:242 (dyn) |
| 11 | /api/analytics/campaigns | GET → P:analytics.read (L6) | – | Campaign analytics | D/analytics/page.tsx:242 (dyn) |
| 12 | /api/analytics/conversations | GET → P:analytics.read (L4) | – | Conversation analytics | D/analytics/page.tsx:242 (dyn) |
| 13 | /api/analytics/export | GET → P:analytics.read (L160) | **inline** `auth.can("analytics.agents")` required when `type=agents` (L162-164); audit log `analytics.exported` | CSV export of any analytics tab | D/analytics/page.tsx:269 |
| 14 | /api/analytics/follow-ups | GET → P:analytics.read (L4) | – | Follow-up analytics | D/analytics/page.tsx:242 (dyn) |
| 15 | /api/analytics/funnel | GET → P:analytics.read (L4) | – | Funnel | D/analytics/page.tsx:242 (dyn) |
| 16 | /api/analytics/handoffs | GET → P:analytics.read (L4) | – | Handoff analytics | D/analytics/page.tsx:242 (dyn) |
| 17 | /api/analytics/orders | GET → P:analytics.read (L4) | – | Order analytics | D/analytics/page.tsx:242 (dyn) |
| 18 | /api/analytics/overview | GET → P:analytics.read (L4) | – | Overview KPIs | D/dashboard/page.tsx:81; D/analytics/page.tsx:242,:250 |
| 19 | /api/analytics/products | GET → P:analytics.read (L4) | – | Product analytics | D/analytics/page.tsx:242 (dyn) |
| 20 | /api/analytics/queue | GET → P:analytics.read (L5) | – | Background-job queue metrics | NONE |
| 21 | /api/analytics/rebuild | POST → **no `permission` option** (AUTH only) | **inline** `can("workspace.manage") \|\| can("settings.manage")` (L22); then `isSuperAdmin` OR any `WorkspaceMember` row in `body.workspaceId` (L26-32); RL 5/min; audit | Rebuild daily analytics for a range (comment L15 says "SUPER_ADMIN only" — code does not enforce that) | NONE |
| 22 | /api/analytics/sales | GET → P:analytics.read (via analyticsRoute, L8) | – | Upsell/discount sales metrics | D/analytics/page.tsx:242 (dyn) |
| 23 | /api/analytics/sla | GET → P:analytics.read (via analyticsRoute, L10) | – | SLA metrics | NONE |
| 24 | /api/analytics/trends | GET → P:analytics.read (L6) | – | Trends | D/analytics/page.tsx:242 (dyn) |
| 25 | /api/analytics/usage | GET → P:usage.read (L5) | – | Workspace AI usage summary | NONE |

## 1.3 /api/auth (4)

| # | Path | Methods → permission | Extra guards | Purpose | UI callers |
|---|---|---|---|---|---|
| 26 | /api/auth/login | POST → PUBLIC (L15) | RL 10/min fail-closed (`rateLimitStrict`); timing shield | Login; session picks first ACTIVE membership (L20-35) | C/auth-form.tsx:31 (dyn `/api/auth/${mode}`) |
| 27 | /api/auth/logout | POST → PUBLIC (L5) | – | Clear cookie, 303 → /login | C/app-shell.tsx:378, C/app-sidebar.tsx:145 (form POST) |
| 28 | /api/auth/me | GET → PUBLIC (L9, manual `readSession`); PATCH → AUTH (L33) | PATCH needs an active workspace (uses getAuthContext) | Current user + workspaces; self profile update (name/email/password) | D/settings/account/page.tsx:53 (GET), :72,:89,:110 (PATCH) |
| 29 | /api/auth/register | POST → PUBLIC (L14) | RL 5/min fail-closed | Create user + workspace (user becomes OWNER) | C/auth-form.tsx:31 (dyn) |

## 1.4 /api/automations, /api/workflows (7)

| # | Path | Methods → permission | Extra | Purpose | UI callers |
|---|---|---|---|---|---|
| 30 | /api/automations/[id] | GET → P:automation.read (L17); PATCH → P:automation.write (L31); DELETE → P:automation.write (L48); POST → P:automation.write (L68, actions enable/disable/duplicate) | – | Quick-rule CRUD + toggle/duplicate | D/automation/page.tsx:116 (POST enable/disable), :176 (DELETE). GET, PATCH, `duplicate`: NONE |
| 31 | /api/automations | GET → P:automation.read (L8); POST → P:automation.write (L20) | – | List / create quick rules | D/automation/quick-rule-dialog.tsx:46 (POST). GET: NONE (page lists via /api/workflows, same table) |
| 32 | /api/workflows/[id]/preview | POST → P:automation.write (L97) | – | Dry-run a workflow | D/automation/page.tsx:154 |
| 33 | /api/workflows/[id] | GET → P:automation.read (L11); PATCH → P:automation.write (L31); POST → P:automation.write (L65, publish/pause/stop/rollback); DELETE → P:automation.write (L88) | – | Workflow detail / lifecycle | D/automation/page.tsx:101 (GET), :134 (POST). PATCH, DELETE: NONE (delete goes through /api/automations/[id]) |
| 34 | /api/workflows | GET → P:automation.read (L8); POST → P:automation.write (L26) | – | List / create DRAFT workflow | D/automation/page.tsx:83 (GET); D/automation/workflow-builder-dialog.tsx:104 (POST) |

(Rows 30-34 = 5 routes; sequence numbers continue below.)

## 1.5 /api/campaigns, /api/segments (11 + 2)

| # | Path | Methods → permission | Extra | Purpose | UI callers |
|---|---|---|---|---|---|
| 35 | /api/campaigns/[id]/approve | POST → P:campaigns.approve (L9) | service re-checks `ctx.can("campaigns.approve")` (campaign-service.ts:215) | Approve content | D/campaigns/page.tsx:139 (dyn `${path}`) |
| 36 | /api/campaigns/[id]/dry-run | POST → P:campaigns.read (L9) | read-level on a POST | Preview send without sending | D/campaigns/page.tsx:105 |
| 37 | /api/campaigns/[id]/lifecycle | POST → P:campaigns.write (L11) | – | pause / resume / cancel | D/campaigns/page.tsx:119 |
| 38 | /api/campaigns/[id]/recipients | GET → P:campaigns.read (L9) | – | Recipients list | NONE |
| 39 | /api/campaigns/[id] | GET → P:campaigns.read (L9); PATCH → P:campaigns.write (L29) | – | Detail / edit | D/campaigns/page.tsx:92 (GET). PATCH: NONE |
| 40 | /api/campaigns/[id]/schedule | POST → P:campaigns.write (L10) | – | Schedule send | D/campaigns/page.tsx:139 (dyn) |
| 41 | /api/campaigns/[id]/validate | POST → P:campaigns.write (L9) | – | Validate audience/content | D/campaigns/page.tsx:139 (dyn) |
| 42 | /api/campaigns/ai-draft | POST → P:campaigns.write (L17) | RL 10/min | AI-drafted campaign copy | NONE |
| 43 | /api/campaigns | GET → P:campaigns.read (L8); POST → P:campaigns.write (L22) | – | List / create | D/campaigns/page.tsx:71 (GET), :172 (POST) |
| 44 | /api/segments/[id] | GET → P:campaigns.read (L9); PATCH → P:campaigns.write (L17); POST → P:campaigns.read (L36, segment preview) | – | Segment detail / edit / preview | NONE |
| 45 | /api/segments | GET → P:campaigns.read (L8); POST → P:campaigns.write (L18) | – | List / create audience segment | D/campaigns/page.tsx:71 (GET); D/campaigns/segment-dialog.tsx:100 (POST) |

## 1.6 /api/catalogs (4), /api/channels (11), /api/comments (6)

| # | Path | Methods → permission | Extra | Purpose | UI callers |
|---|---|---|---|---|---|
| 46 | /api/catalogs/[id]/pages | POST → P:catalogs.assign (L16); DELETE → P:catalogs.assign (L37) | – | Assign / unassign catalog to a FB page | D/catalogs/[id]/page.tsx:194 (POST), :214 (DELETE); D/channels/catalog-page-section.tsx:63 (POST) |
| 47 | /api/catalogs/[id]/products | POST → P:catalogs.write (L16); DELETE → P:catalogs.write (L33) | – | Add / remove products in catalog | D/catalogs/[id]/page.tsx:153, :176 |
| 48 | /api/catalogs/[id] | GET → P:catalogs.read (L14); PATCH → P:catalogs.write (L22); DELETE → P:catalogs.write (L39) | – | Catalog CRUD | D/catalogs/[id]/page.tsx:84 (GET), :119 (PATCH); D/catalogs/page.tsx:101 (PATCH), :120 (DELETE) |
| 49 | /api/catalogs | GET → P:catalogs.read (L21); POST → P:catalogs.write (L29) | – | List / create | D/catalogs/page.tsx:57, :77; D/channels/catalog-page-section.tsx:42 (GET) |
| 50 | /api/channels/[type] | POST → P:channels.connect (L23); DELETE → P:channels.connect (L41) | – | Connect / disconnect IG / Telegram / WhatsApp / TikTok channel | D/channels/multi-channel-section.tsx:78 (POST), :94 (DELETE) |
| 51 | /api/channels/by-id/[id]/catalog | GET → P:catalogs.read (L14) | – | Catalog bound to a page | D/channels/catalog-page-section.tsx:41 |
| 52 | /api/channels/by-id/[id]/persona | PATCH → P:channels.connect (L19) | – | Bind persona to channel | NONE |
| 53 | /api/channels/by-id/[id]/send-mode | PATCH → P:channels.connect (L18) | – | Set channel send mode | D/intelligence/page.tsx:257 |
| 54 | /api/channels/facebook/callback | GET → PUBLIC (plain `export async function GET`, no createHandler) | signed OAuth state (`verifyOAuthState`) L30 | Meta OAuth redirect target | NONE (Meta redirect; UI links to /connect only) |
| 55 | /api/channels/facebook/connect | GET → P:channels.connect (L16) | – | Start OAuth (redirect to Meta) | D/channels/page.tsx:210,:290 (anchor href) |
| 56 | /api/channels/facebook/disconnect | POST → P:channels.connect (L15); PUT → P:channels.connect (L65) | – | Disconnect page; PUT = token smoke-test | D/channels/page.tsx:117 (POST), :150 (PUT) |
| 57 | /api/channels/facebook | GET → P:channels.read (L13); PATCH → **P:ai.configure** (L45) | – | List FB connections; assign persona to page | D/channels/page.tsx:49 (GET), :137 (PATCH); D/ai/playground/page.tsx:80; D/catalogs/[id]/page.tsx:103 |
| 58 | /api/channels/facebook/select-page | POST → P:channels.connect (L25) | – | Finish OAuth: pick page | D/channels/page.tsx:96 |
| 59 | /api/channels | GET → P:channels.read (L7) | – | All channels | D/channels/multi-channel-section.tsx:56; D/intelligence/page.tsx:178; C/inbox/inbox.tsx:109 |
| 60 | /api/comments/facebook/hide | POST → P:channels.send_human (L15) | – | Hide FB comment | NONE |
| 61 | /api/comments/facebook/private | POST → P:channels.send_human (L15) | – | Private reply to FB comment | NONE |
| 62 | /api/comments/facebook/reply | POST → P:channels.send_human (L15) | – | Public reply to FB comment | NONE |
| 63 | /api/comments/instagram/hide | POST → P:channels.send_human (L14) | – | Hide IG comment | NONE |
| 64 | /api/comments/instagram/private | POST → P:channels.send_human (L16) | – | Private reply IG | NONE |
| 65 | /api/comments/instagram/reply | POST → P:channels.send_human (L19) | – | Reply IG comment | NONE |

## 1.7 /api/conversations (9), /api/customers (3), /api/discount-approvals (2)

| # | Path | Methods → permission | Extra | Purpose | UI callers |
|---|---|---|---|---|---|
| 66 | /api/conversations/[id]/assign | POST → P:conversations.assign (L9) | – | Assign conversation to agent | NONE |
| 67 | /api/conversations/[id]/coach | POST → P:conversations.read (L22) | none (no RL); wholesale/discount info gated inside `sales/coach.ts:53-54` via `ctx.can` | AI sales-coach suggestion (AI call on a read permission) | C/inbox/inbox.tsx:418 |
| 68 | /api/conversations/[id]/messages/[messageId]/retry | POST → P:channels.send_human (L15) | – | Retry failed send | C/inbox/inbox.tsx:315 |
| 69 | /api/conversations/[id]/messages | GET → P:conversations.read (L11); POST → P:channels.send_human (L27) | – | Thread / send human message | C/inbox/inbox.tsx:199,:224 (GET), :283 (POST) |
| 70 | /api/conversations/[id]/notes | POST → P:conversations.reply (L15) | – | Internal note | C/inbox/inbox.tsx:334 |
| 71 | /api/conversations/[id]/read | POST → P:conversations.read (L9) | write (mark read) on read perm | Mark read | C/inbox/inbox.tsx:246 |
| 72 | /api/conversations/[id]/release-ai | POST → P:conversations.reply (L8) | – | Hand back to AI | C/inbox/inbox.tsx:374 (dyn; `action("release-ai")` :616) |
| 73 | /api/conversations/[id] | GET → P:conversations.read (L9); PATCH → P:conversations.reply (L18, status/priority/tags); DELETE → P:conversations.delete (L34, full purge) | – | Conversation detail/update/purge | C/inbox/inbox.tsx:225 (GET), :395 (PATCH), :354 (DELETE) |
| 74 | /api/conversations/[id]/takeover | POST → P:conversations.reply (L8) | – | Human takeover | C/inbox/inbox.tsx:374 (dyn; `action("takeover")` :887) |
| 75 | /api/conversations | GET → P:conversations.read (L7) | – | Inbox list | C/inbox/inbox.tsx:151 |
| 76 | /api/customers/[id]/consent | GET → P:customers.read (L10); POST → P:customers.write (L15) | – | Marketing consent | C/inbox/marketing-card.tsx:35 (GET), :49 (POST) |
| 77 | /api/customers/[id] | GET → P:customers.read (L9); PATCH → P:customers.write (L18) | – | Customer detail / edit | C/inbox/inbox.tsx:237 (GET). PATCH: NONE |
| 78 | /api/customers | GET → P:customers.read (L7) | – | Customers list | D/customers/page.tsx:38 |
| 79 | /api/discount-approvals/[id] | POST → P:orders.discount.approve (L12) | service re-check `ctx.can` (pricing-policy.ts:143); requester cannot approve own request | Decide a discount approval | NONE |
| 80 | /api/discount-approvals | GET → P:orders.read (L11); POST → P:orders.discount (L29) | – | List / request discount approval | NONE |

## 1.8 /api/faqs, /api/feedback, /api/follow-ups, /api/handoffs, /api/health, /api/intelligence, /api/jobs (13)

| # | Path | Methods → permission | Extra | Purpose | UI callers |
|---|---|---|---|---|---|
| 81 | /api/faqs/[id] | PATCH → P:knowledge.write (L9) | – | Edit FAQ | D/knowledge/page.tsx:223; D/products/[id]/page.tsx:197 |
| 82 | /api/faqs | GET → P:knowledge.read (L18); POST → P:knowledge.write (L27) | – | List / create FAQ | D/knowledge/page.tsx:85 (GET), :213 (POST); D/products/[id]/page.tsx:187 (POST) |
| 83 | /api/feedback/[id] | GET → P:feedback.read (L9); PATCH → P:feedback.write (L18) | – | Feedback item | D/feedback/page.tsx:95,:109 (PATCH). GET: NONE |
| 84 | /api/feedback | GET → P:feedback.read (L7); POST → P:feedback.write (L24) | – | List / create feedback | D/feedback/page.tsx:53, :72 |
| 85 | /api/follow-ups/[id]/cancel | POST → P:followups.cancel (L9) | – | Cancel a follow-up | D/follow-ups/page.tsx:59 |
| 86 | /api/follow-ups | GET → P:followups.read (L13) | – | List follow-ups | D/follow-ups/page.tsx:41 |
| 87 | /api/handoffs/[id] | POST → P:handoffs.claim (L14) for all actions (claim/release/resolve/route); `resolve` additionally `auth.require("handoffs.resolve")` (L31) | `route` action = assign to another agent with only `handoffs.claim` | Handoff actions | D/handoffs/page.tsx:68 (claim/release/resolve only; `route` unused) |
| 88 | /api/handoffs | GET → P:handoffs.read (L13) | – | List handoffs | D/handoffs/page.tsx:50 |
| 89 | /api/health | GET → PUBLIC (plain handler, no createHandler) | – | DB/queue/FB-config health (no secrets) | NONE (no hit in UI, Dockerfile or docker-compose.yml) |
| 90 | /api/intelligence/conversations/[id] | GET → P:conversations.read (L21); POST → P:conversations.reply (L38) | POST RL 10/min | Collection-lab detail; reanalyze / confirm field | D/intelligence/page.tsx:220 (GET), :281,:317 (POST) |
| 91 | /api/intelligence/conversations | GET → P:conversations.read (L14) | – | Collection-lab list | D/intelligence/page.tsx:205 |
| 92 | /api/intelligence/fields | GET → P:conversations.read (L24); PUT → **P:settings.manage** (L33) | PUT RL 20/min | Fields the AI must collect | D/intelligence/page.tsx:189 (GET), :370 (PUT) |
| 93 | /api/intelligence/overview | GET → P:conversations.read (L9) | – | Lab overview | D/intelligence/page.tsx:165 |
| 94 | /api/jobs/run | POST → PUBLIC (L15) | `x-jobs-secret` header vs `JOBS_SECRET`, timing-safe (L19-25); if secret unset: open outside production, 403 in production (L26-29); RL 30/min | Run background jobs (cron) | NONE |

## 1.9 /api/knowledge (5), /api/media (6), /api/order-drafts (4), /api/orders (2)

| # | Path | Methods → permission | Extra | Purpose | UI callers |
|---|---|---|---|---|---|
| 95 | /api/knowledge/[id]/reindex | POST → P:knowledge.write (L8) | – | Reindex doc | D/knowledge/page.tsx:140 |
| 96 | /api/knowledge/[id] | GET → P:knowledge.read (L9); PATCH → P:knowledge.write (L18); DELETE → P:knowledge.write (L30) | – | Doc CRUD | D/knowledge/page.tsx:190 (GET), :145,:174 (PATCH), :155 (DELETE) |
| 97 | /api/knowledge | GET → P:knowledge.read (L16); POST → P:knowledge.write (L47) | – | List / create doc | D/knowledge/page.tsx:84, :106 |
| 98 | /api/knowledge/search | POST → P:knowledge.read (L18) | RL 30/min | Semantic search | D/knowledge/page.tsx:201 |
| 99 | /api/knowledge/upload | POST → P:knowledge.write (L20) | runtime nodejs | Upload knowledge file | D/knowledge/page.tsx:130 |
| 100 | /api/media/[id] | GET → P:media.read (L9); PATCH → P:media.write (L18); DELETE → P:media.write (L27) | – | Asset detail/edit/delete | D/media/page.tsx:176 (DELETE). GET, PATCH: NONE |
| 101 | /api/media/feedback | GET → P:media.read (L21) | – | Feedback-media picker list | C/inbox/MediaPicker.tsx:47 |
| 102 | /api/media/file/[...key] | GET → P:media.read (L18) | storage key must start with `${workspaceId}/` | Authenticated file stream | indirect only: URL built by modules/media/storage.ts:165 and used as asset `url` (no literal in UI) |
| 103 | /api/media/public/[token] | GET → PUBLIC (plain handler) | HMAC-signed token `verifyPayload`, expiry, workspace-bound asset lookup | Signed public media link for customers | NONE (URL built by modules/media/service.ts:396 for outbound sends) |
| 104 | /api/media | GET → P:media.read (L20) | – | Library list | D/media/page.tsx:89; D/products/[id]/page.tsx:78; C/inbox/MediaPicker.tsx:44,:46 |
| 105 | /api/media/upload | POST → P:media.write (L16) | – | Upload asset | D/media/page.tsx:128 (XHR); D/products/[id]/page.tsx:152 |
| 106 | /api/order-drafts/[id]/cancel | POST → P:orders.write (L8) | – | Cancel AI order draft | NONE |
| 107 | /api/order-drafts/[id]/confirm | POST → P:orders.write (L11) | – | Confirm draft → order | NONE |
| 108 | /api/order-drafts/[id] | GET → P:orders.read (L8) | – | Draft detail | NONE |
| 109 | /api/order-drafts | GET → P:orders.read (L12) | – | Draft list (`conversationId` filter) | C/inbox/inbox.tsx:129 |
| 110 | /api/orders/[id] | GET → P:orders.read (L9) (returns `canWrite`, `canDelete: role==="OWNER"` L20); DELETE → P:orders.write (L26) | **OWNER-only inside service** `requireWorkspaceOwner` (order-service.ts:565, code OWNER_ONLY_ORDERS); PATCH → P:orders.write (L44) | Order detail / status+notes / hard delete | D/orders/[id]/page.tsx:60 (GET), :79,:267 (PATCH), :94 (DELETE); D/orders/page.tsx:66 (DELETE) |
| 111 | /api/orders | GET → P:orders.read (L16) | returns `canDelete: auth.role === "OWNER"` (L20) | Orders list | D/orders/page.tsx:47 |

## 1.10 /api/personas (5), /api/products (6), /api/setup (1)

| # | Path | Methods → permission | Extra | Purpose | UI callers |
|---|---|---|---|---|---|
| 112 | /api/personas/[id]/versions/[version]/publish | POST → P:personas.write (L12) | – | Publish persona draft | D/personas/page.tsx:169 |
| 113 | /api/personas/[id]/versions/[version]/rollback | POST → P:personas.write (L12) | – | Roll back persona | D/personas/page.tsx:184 |
| 114 | /api/personas/[id]/versions | GET → P:personas.read (L24); POST → P:personas.write (L34) | – | Version list / create draft | D/personas/page.tsx:150 (GET), :161 (POST) |
| 115 | /api/personas | GET → P:personas.read (L27); POST → P:personas.write (L38); PATCH → P:personas.write (L62) | – | Persona CRUD | D/personas/page.tsx:76,:124,:141; D/ai/playground/page.tsx:79; D/channels/page.tsx:50 |
| 116 | /api/products/[id]/compliance | GET → P:products.read (L22); POST → P:products.write (L28) | – | System-1 claim rules | C/products/compliance-card.tsx:39 (GET), :55 (POST) |
| 117 | /api/products/[id]/media | POST, PATCH, DELETE, PUT → P:products.write (L17, L29, L40, L53) | – | Product ↔ media links | D/media/page.tsx:189 (POST), :204 (DELETE); D/products/[id]/page.tsx:119 (POST), :128 (DELETE), :137 (PATCH), :177 (PUT) |
| 118 | /api/products/[id] | GET → P:products.read (L9); PATCH → P:products.write (L18) | GET returns full Product row (incl. `wholesalePrice`, `discountCap`) — see Part 2d | Product detail / edit | D/products/[id]/page.tsx:70 (GET), :101 (PATCH); D/products/page.tsx:126,:135 (PATCH) |
| 119 | /api/products/import | POST → P:products.write (L13) | RL 6/min | Bulk import | D/products/page.tsx:109 |
| 120 | /api/products | GET → P:products.read (L7); POST → P:products.write (L22) | – | List / create | D/products/page.tsx:58,:86; D/catalogs/[id]/page.tsx:138; D/feedback/page.tsx:42; D/media/page.tsx:99; D/ai/playground/page.tsx:81 |
| 121 | /api/setup/progress | GET → P:settings.manage (L15) | – | Onboarding checklist flags | C/setup-checklist.tsx:176 |

## 1.11 System-token (/api/v1) and webhook-signature routes (9)

| # | Path | Methods → auth | Extra | Purpose | UI callers |
|---|---|---|---|---|---|
| 122 | /api/v1/capacity | GET → system-token scope `capacity.read` (L11) | per-token RL (`token.rateLimitPerMin`, system-handler.ts:65); workspace from token | Capacity for media-buyer system | NONE (external system) |
| 123 | /api/v1/retarget-campaigns/[id] | GET → system-token scope `retarget.read` (L7) | same | Read a retarget campaign | NONE |
| 124 | /api/v1/retarget-campaigns | POST → system-token scope `retarget.create` (L12) | same | Clipper creates retarget campaign | NONE |
| 125 | /api/v1/webhooks/[system] | POST → webhook-signature (HMAC-SHA256 `x-zaki-signature` + `x-zaki-timestamp`, 5-min window, L54) | workspace from `?ws=` slug then connection secret; event_id dedupe; PII guard | Inbound events from sister systems | NONE |
| 126 | /api/webhooks/instagram | GET → Meta verify challenge (L14); POST → webhook-signature `x-hub-signature-256` vs META_APP_SECRET (L31-32) | – | IG webhook | NONE |
| 127 | /api/webhooks/meta | GET → challenge (L13); POST → webhook-signature `x-hub-signature-256` (L30-31) | – | Facebook webhook | NONE |
| 128 | /api/webhooks/telegram | POST → shared secret header `x-telegram-bot-api-secret-token` (checked in `ingestTelegramWebhook`) | – | Telegram webhook | NONE |
| 129 | /api/webhooks/tiktok | POST → shared secret header `x-tiktok-secret` + `?account=` (interim, checked in ingest) | – | TikTok webhook (local-only surface per comment) | NONE |
| 130 | /api/webhooks/whatsapp | GET → challenge (L14); POST → webhook-signature `x-hub-signature-256` (L31-32) | – | WhatsApp webhook | NONE |

Integration-token issuance: `src/modules/integrations/tokens.ts` only; no API route or UI to create tokens/connections found (UNVERIFIED whether a script does it).

## 1.12 /api/workspaces (4)

| # | Path | Methods → permission | Extra | Purpose | UI callers |
|---|---|---|---|---|---|
| 131 | /api/workspaces/members/[id] | PATCH → P:users.manage (L17); DELETE → P:users.manage (L31) | service rules (Part 4); no RL | Change role/permissions/status, or reset password (`newPassword`); remove member | D/settings/team/team-view.tsx:102 (reset), :116 (change), :128 (DELETE) |
| 132 | /api/workspaces/members/accept | POST → PUBLIC (L20; manual `readSession`) | self only; sets session cookie to target workspace | Accept own invitation | NONE |
| 133 | /api/workspaces/members | GET → P:users.manage (L27); POST → P:users.manage (L33) | OWNER-rank checked in service | Roster; invite OR create-with-password | D/settings/team/team-view.tsx:67 (GET), :85 (POST) |
| 134 | /api/workspaces | GET → AUTH (L10); POST → AUTH (L16, RL 10/min); PATCH → AUTH (L25, membership ACTIVE check L27-32); PUT → P:workspace.manage (L45) | – | List workspaces; create workspace (caller becomes OWNER); switch active workspace; rename/timezone | D/settings/team/team-view.tsx:143 (PUT). GET, POST, PATCH: NONE |

(Numbering note: section 1.4 rows are #30-#34, so all 134 route files appear exactly once, #1-#134.)

---------------------------------------------------------------------------

# PART 2 — Findings

## 2a. Routes (or methods) with no UI caller

Whole routes with NO caller (found by grepping src/app/(dashboard), (auth), components, lib):
- /api/ai/playground (retired 410 stub, #1)
- /api/analytics/queue, /sla, /usage, /rebuild (#20, #23, #25, #21)
- /api/campaigns/[id]/recipients (#38), /api/campaigns/ai-draft (#42)
- /api/channels/by-id/[id]/persona (#52) — the UI assigns a persona via PATCH /api/channels/facebook instead
- /api/comments/{facebook,instagram}/{hide,private,reply} (6 routes, #60-65) — no comment-moderation UI at all
- /api/conversations/[id]/assign (#66) — no assign UI; inbox only filters by assignee (inbox.tsx:148)
- /api/discount-approvals and /[id] (#79, #80) — whole P8 discount approval flow has no UI
- /api/order-drafts/[id], /[id]/confirm, /[id]/cancel (#106-108) — UI only lists drafts (inbox.tsx:129)
- /api/segments/[id] (#44) (no segment edit/preview UI)
- /api/workspaces/members/accept (#132), /api/workspaces GET/POST/PATCH (#134) — there is no invite-accept screen and no workspace switcher / create-workspace UI
- /api/auth/* are used. /api/health, /api/jobs/run, /api/v1/*, /api/webhooks/*, /api/channels/facebook/callback, /api/media/public/[token] are external/ops by design.
- /api/media/file/[...key] has only an indirect caller (storage.ts:165 URL).

Methods with no caller on otherwise-used routes:
- automations/[id]: GET, PATCH, POST `duplicate` · automations: GET
- workflows/[id]: PATCH, DELETE (UI deletes workflows through DELETE /api/automations/[id], D/automation/page.tsx:176 — works because both are the same `automation` table)
- campaigns/[id]: PATCH
- customers/[id]: PATCH (UI cannot edit customers although customers.write exists)
- feedback/[id]: GET
- media/[id]: GET, PATCH
- handoffs/[id]: action `route` (UI sends claim/release/resolve only, handoffs/page.tsx:124-129)
- conversations: the `coach` etc. are used. `notes`, `read` used.

## 2b. UI fetch calls to routes/methods that do not exist

I mapped every `fetch`/`api(`/`apiJson` call in D/, C/ (about 150 call sites) to the route files and the exported methods. Result: **no call to a non-existent route or method was found.** Points I checked specifically:
- `/api/automations/${id}` POST body `{action:"enable"|"disable"}` (automation/page.tsx:116) → route POST accepts enable/disable/duplicate (automations/[id]/route.ts:65-68). OK
- `/api/workflows/${id}` POST `{action: publish|pause|stop}` (:134) → exists. OK
- `/api/campaigns/${id}/${path}` with validate|approve|schedule (campaigns/page.tsx:139) → three routes exist. OK
- `/api/conversations/${id}/${path}` with takeover|release-ai (inbox.tsx:374) → exist. OK
- `/api/channels/${type}` (multi-channel-section.tsx:78,94) with INSTAGRAM/TELEGRAM/WHATSAPP/TIKTOK → `[type]` route; FACEBOOK handled by static facebook/* routes. OK
- `/api/order-drafts?conversationId=` → query schema accepts `conversationId` (order-drafts/route.ts:6). OK
- `/api/media/upload` is called by XHR (media/page.tsx:128) → exists.
- Cosmetic: comment in `D/campaigns/segment-dialog.tsx:11` and `components/analytics` mention routes only in comments.
- Behavioural oddity (not a missing route): `/api/channels/facebook/disconnect` PUT is used as "test connection" (channels/page.tsx:150) although the route comment says `POST /api/channels/facebook/test` (disconnect/route.ts:60) — the documented path does not exist but the UI uses the real one.

## 2c. Page permission (nav-model.ts) vs APIs the page calls on load

Role holders come from permissions.ts (see Part 5). Roles lacking `channels.read`: SALES_AGENT, SUPPORT_AGENT. Roles lacking `followups.read`: SUPPORT_AGENT, VIEWER.

Page → on-load calls that can 403 although the page opens:
1. **/conversations** (conversations.read, all 6 roles). `inbox.tsx:109` calls `/api/channels` (channels.read) → 403 for SALES_AGENT and SUPPORT_AGENT; error swallowed (inbox.tsx:119-122) → page/channel filter silently empty.
2. **/intelligence** (conversations.read). `intelligence/page.tsx:178` calls `/api/channels` → 403 for SALES_AGENT / SUPPORT_AGENT, swallowed (:179 `if (!res.ok) return`). Same page has write controls needing `settings.manage` (fields PUT), `channels.connect` (send-mode), `conversations.reply` (reanalyze/confirm) — visible to VIEWER, fail with 403.
3. **/catalogs/[id]** (catalogs.read). `catalogs/[id]/page.tsx:103` calls `/api/channels/facebook` (channels.read) → 403 for SALES/SUPPORT agents; swallowed (:105 `if (res.ok)`), so the "assign to page" list is empty.
4. **/analytics** (analytics.read, all roles). Tabs are NOT permission filtered (analytics/page.tsx:45-55). "agents" tab → 403 for non-managers is handled by design (`res.status === 403` → null, :243; empty state "مش متاحة لدورك" :487). Export for `type=agents` by a non-manager returns 403 and shows the generic toast "تعذّر إنشاء ملف التصدير" (:271).
5. **/dashboard** (analytics.read): embeds SetupChecklist → `/api/setup/progress` needs `settings.manage` → 403 for SALES/SUPPORT/VIEWER; handled (setup-checklist.tsx:178 hides card).
6. **/channels** (channels.read, includes VIEWER): reads are fine; persona assignment `PATCH /api/channels/facebook` requires `ai.configure` (not channels.connect), so a VIEWER sees connect/disconnect/assign controls that all 403. Note the same operation has two different permissions: PATCH /api/channels/facebook = ai.configure (L45) vs PATCH /api/channels/by-id/[id]/persona = channels.connect (L19).
7. **/follow-ups** (followups.read, SALES_AGENT yes / SUPPORT_AGENT no): SALES_AGENT can list but "cancel" needs followups.cancel (MANAGER+) → button visible, 403.
8. **/campaigns** (campaigns.read: SALES_AGENT, VIEWER, MANAGER+): create / validate / approve / schedule / pause / cancel buttons are shown to everyone; need campaigns.write / campaigns.approve → 403.
9. **/automation** (automation.read: VIEWER, MANAGER+): toggle / delete / publish / preview need automation.write.
10. **/products/[id]** (products.read): saves need products.write; FAQ create/edit needs `knowledge.write` (products/[id]/page.tsx:187,:197); media upload needs `media.write`; media link needs products.write. Buttons visible to read-only roles.
11. **/media** (media.read): "link to product" uses `/api/products/:id/media` = products.write (media/page.tsx:189) while upload/delete = media.write — two different permissions for one page's actions.
12. **/orders, /orders/[id]** (orders.read): `canWrite` is returned by the server (orders/[id]/route.ts:20) but the client type declares it and never uses it (orders/[id]/page.tsx:16, no other reference) → status-change/notes controls are shown to SUPPORT_AGENT/VIEWER (no orders.write) and 403 on use. Delete button hidden by `canDelete` (OWNER-only), see 2e.
13. **/ai/playground** (ai.configure) loads `/api/channels/facebook` (channels.read), `/api/personas` (personas.read), `/api/products` (products.read): consistent for the 4 roles holding ai.configure (all of them hold the others). Would 403 only for a member who got ai.configure through a `permissions` override.
14. **/settings/team** (users.manage) page: `enforcePermission("users.manage")` server-side (team/page.tsx:5); workspace rename `PUT /api/workspaces` needs workspace.manage; holders of both = SUPER_ADMIN/OWNER/ADMIN → consistent.
15. **/settings/account** (null): `/api/auth/me` GET public, PATCH AUTH → no mismatch (but a member with no active workspace gets 403 on PATCH, since PATCH uses getAuthContext).

"Reverse" mismatches (API more permissive than the page that owns the data):
- `analytics/campaigns|automations|handoffs|follow-ups|orders|products|conversations|ai|funnel|sla|trends|overview` require only `analytics.read` (held by all 6 roles) while their pages need campaigns.read / automation.read / handoffs.read / followups.read / orders.read / products.read. E.g. VIEWER (no handoffs.read) can read handoff analytics; SUPPORT_AGENT (no campaigns.read) can read campaign analytics; and `analytics/export` exports `campaigns` etc. the same way.
- `analytics/queue` (background-job queue metrics) = analytics.read (all roles), no UI.
- `/api/discount-approvals` GET = orders.read (VIEWER sees approval requests).
- `/api/products` list and `/api/products/[id]` return the full Product row to any `products.read` holder including `wholesalePrice` and `discountCap` (schema.prisma:471-472; service.ts:143-190 and :119-131 do no select/omit). The `products.wholesale` permission is only enforced inside the sales coach (sales/coach.ts:53), not on the product APIs (UNVERIFIED whether the UI renders the field; the API does send it).

Page-level enforcement: the dashboard layout redirects only when there is no session. Only `settings/team/page.tsx:5` enforces a permission server-side for page access. The layout comment (layout.tsx:21-22) says "pages themselves enforce permission on the server" — in code that is true only for the team page. A VIEWER typing /settings/ai gets the page shell; its API calls 403.

## 2d. Write actions guarded by weaker / odd permissions

Withheld from this copy: the repository is public, and this section lists route by route where the current guard is weaker than intended. The owner holds the full copy. The target permission for every affected route is in `04-PERMISSIONS.md` §5.3; read the route file itself for the current guard before changing it.

## 2e. Every place the server distinguishes roles directly (not via a permission)

| # | Location | What it does |
|---|---|---|
| 1 | src/modules/auth/context.ts:29,46-48 | `user.isSuperAdmin` → effective permissions = `effectivePermissions("SUPER_ADMIN", overrides)` (= ALL). Still requires an ACTIVE membership in the session workspace (L38-45); `ctx.role` stays the membership role. |
| 2 | src/modules/rbac/owner-guard.ts:14-45 | `getActorRole` (fresh DB read, ACTIVE only) and `requireWorkspaceOwner` → `role !== "OWNER"` → 403 `OWNER_ROLE_REQUIRED` (default). |
| 3 | src/modules/orders/order-service.ts:565 | `requireWorkspaceOwner(...)` with code `OWNER_ONLY_ORDERS` — hard delete order. |
| 4 | src/app/api/orders/[id]/route.ts:20 | `canDelete: auth!.role === "OWNER"` returned to client. |
| 5 | src/app/api/orders/route.ts:20 | `canDelete: auth!.role === "OWNER"` returned to client. (Uses membership role; a super admin whose membership is not OWNER gets `false`, and service-level `requireWorkspaceOwner` also rejects them although context grants them ALL.) |
| 6 | src/modules/members/service.ts:40-48 (`assertOwnerRank`), called at :146 (invite OWNER), :269 (update touching OWNER row or promoting to OWNER), :309 (remove OWNER), :363 (create OWNER), :419 (reset OWNER password) | Only role OWNER can create/promote/modify/suspend/remove/reset-password an OWNER. |
| 7 | src/modules/members/service.ts:84-103, :260-265, :302-306 | Last-active-OWNER floor (`ensureOwnerRemaining`, `verifyOwnerFloor`, code `OWNER_REQUIRED`). |
| 8 | src/modules/members/service.ts:23-32,:54-58 | `ASSIGNABLE_ROLES` excludes SUPER_ADMIN (`INVALID_ROLE`). |
| 9 | src/modules/members/service.ts:235,:299,:412 | Self-edit ban (`CANNOT_EDIT_SELF`). |
| 10 | src/app/api/analytics/rebuild/route.ts:22,29-32 | `isSuperAdmin` lookup (L29-30) + permission fallback. |
| 11 | src/app/(dashboard)/layout.tsx:32 | `hasPermission(auth.role, auth.memberPermissions, …)` — uses the raw membership role, ignores `isSuperAdmin`. |
| 12 | src/modules/handoffs/handoff-service.ts:171 | `routeHandoff` candidate agents = members whose role is in OWNER/ADMIN/MANAGER/SALES_AGENT/SUPPORT_AGENT (hard-coded role list; VIEWER and SUPER_ADMIN rows excluded; member permission overrides ignored). |
| 13 | src/modules/workspaces/service.ts:33 | New workspace creator gets role `"OWNER"` (assignment, not a check). |
| 14 | src/app/api/auth/me/route.ts:15 | returns `isSuperAdmin` to client. |
| 15 | src/components/role-matrix.tsx:103 | UI shows ✓/– by `hasPermission(role, [], perm)` (static, no overrides, no owner-only rules). |

No other `role ===` / `isSuperAdmin` / `"OWNER"` comparisons exist in src (grep over src/**/*.ts{,x}).

---------------------------------------------------------------------------

# PART 3 — Identity / permissions end to end

## 3.1 Session
- Cookie `acre_session`, HS256 JWT `{userId, workspaceId|null}`, 7-day expiry (src/modules/auth/session.ts:7-9, 31-60). Login picks first ACTIVE membership by createdAt (login/route.ts:20-35); register creates a workspace (user = OWNER, workspaces/service.ts:33); `members/accept` and `workspaces` POST/PATCH rewrite the cookie.
- No middleware.ts exists (`find` shows none). Auth happens only in `createHandler` and the dashboard layout redirect (layout.tsx:12-19).

## 3.2 `getAuthContext()` (src/modules/auth/context.ts:23-62)
Returns `AuthContext { userId, workspaceId, role, memberPermissions, can(p), require(p) }`.
- `role` = `membership.role` (never "SUPER_ADMIN" unless the row says so); `memberPermissions` = raw `WorkspaceMember.permissions`.
- `can(p)` = `effective.has(p)`; `require(p)` throws `AuthorizationError("Missing permission: p")` → HTTP 403 code AUTHORIZATION_ERROR (errors.ts:29-32).
- `effective` = `isSuperAdmin ? effectivePermissions("SUPER_ADMIN", overrides) : effectivePermissions(role, overrides)`; `effectivePermissions` = role matrix **union** overrides (only known permission strings, permissions.ts:~188-197). Overrides can only ADD; there is no deny/revoke.
- Failures: no session or user gone → 401 AUTHENTICATION_ERROR; no `workspaceId` in session → 403 "No active workspace in session"; membership missing or status != ACTIVE (INVITED/SUSPENDED) → 403 "You are not an active member of this workspace". Everything is read fresh from DB on each request (role changes/suspension take effect immediately; the JWT is not revoked).
- Super admin: elevated only inside workspaces where they hold an ACTIVE membership. There is no cross-workspace super-admin access in the API (except `analytics/rebuild` — see `04-PERMISSIONS.md` §5.3), and no UI or API sets `User.isSuperAdmin` (schema default false, schema.prisma:36; no route writes it).
- `createHandler` flow (handler.ts:89-92): not `public` → `getAuthContext()`, then `auth.require(options.permission)` if given. Routes with neither `permission` nor `public` = "any active member": `/api/auth/me` PATCH, `/api/workspaces` GET/POST/PATCH, `/api/analytics/rebuild`.

## 3.3 Member-level permission overrides (`WorkspaceMember.permissions`, schema.prisma:109-127)
- Stored as `String[]`. Read in `getAuthContext` (select at context.ts:38-41) and in the layout (layout.tsx:32).
- **Editable via API only**: `POST /api/workspaces/members` (`permissions` in body → invite/create; members/route.ts:16, service.ts:171,189,384) and `PATCH /api/workspaces/members/[id]` (`permissions`, [id]/route.ts:8 → service.ts:244-247). Validated: array ≤ 40, each must be a known permission string (service.ts:60-69). Requires `users.manage`.
- **No UI edits them**: `team-view.tsx` never sends `permissions`; `Member.permissions` is typed (team-view.tsx:19) and returned by GET but never rendered. Re-inviting a pending invite resets overrides to `[]` (service.ts:171 `permissions: input.permissions ?? []`).
- Grant rules, and what a role change does to overrides: target rules in `04-PERMISSIONS.md` §5.4.
- Tests exist that exercise them (tests/p3-members.test.ts:97-99).

## 3.4 `/api/auth/me` (src/app/api/auth/me/route.ts:9-22)
GET (public handler, manual session): `{ user: {id, email, name, isSuperAdmin}, activeWorkspace: {id,name,slug,plan,createdAt}|null, workspaces: [{id,name,slug,plan,createdAt}] }` (workspaces/service.ts:42-48). **No role, no permissions, no member status.** Only caller: `D/settings/account/page.tsx:53`; `isSuperAdmin` is declared in the `Me` type (account/page.tsx:13) and never used.

## 3.5 Does any client component receive or check the user's role/permissions?
Server → client leaks of role-derived data (no permission list is ever sent):
1. `D/orders/page.tsx:38` (`canDelete` state), `:54` (set from `data.canDelete` returned by `GET /api/orders`, which is `role === "OWNER"`), `:168`, `:188`, `:216` (hide delete column/button).
2. `D/orders/[id]/page.tsx:16` (`canWrite?: boolean` declared; **never read**), `:18` (`canDelete?`), `:149` (`order.canDelete && …` shows delete button).
3. `C/app-shell.tsx:128` — `canTeam = groups.some(... href === "/settings/team")` computed from the server-filtered `groups` prop; used at `:367` to show the "الفريق" link in the account menu. (Also true when auth failed — layout fallback shows all pages.)
4. `C/app-shell.tsx:9`, `C/app-sidebar.tsx:9`, `src/lib/page-guides.ts:1,131` — `app-shell.tsx` is `"use client"` and imports `page-guides.ts`, which imports `NAV_GROUPS` (nav-model.ts), so every page's required permission string is bundled into the client JS. The layout passes only `NavGroupView` (no permissions, nav-model.ts:131-138), so there is no per-user permission data on the client.
5. `C/role-matrix.tsx:3,103` — imports `hasPermission` and calls `hasPermission(r.value, [], c.perm)`; the component is imported by `"use client"` `team-view.tsx:13`, so the whole role→permission matrix (permissions.ts) ships to the browser. It displays static role defaults only (no per-member overrides, no owner-only rules).
6. `D/settings/team/team-view.tsx:19` (`permissions: string[]` typed, unused), `:207-218` (role dropdown with all 6 roles for every row), `:225` (status dropdown disabled only for INVITED).
7. `D/analytics/page.tsx:243,487` and `C/setup-checklist.tsx:178` — client reacts to 403 from APIs instead of checking a permission.
8. `D/settings/account/page.tsx:13` — `isSuperAdmin` typed, unused.
No client component compares the user's role or permission to a literal. All other write buttons are shown unconditionally and fail with 403.

## 3.6 Layout nav filtering (src/app/(dashboard)/layout.tsx)
- `readSession()` null → redirect /login (:12-13); user row missing → redirect /login (:19).
- Default `groups = visibleGroups(NAV_GROUPS, () => true)` (:23) — **all pages visible**.
- Inside `try`: `getAuthContext()`; `groups = visibleGroups(NAV_GROUPS, p => hasPermission(auth.role, auth.memberPermissions, p))` (:32). `catch {}` (:33-35) swallows everything.
- Oddities:
  a. **Auth failure = full nav**: a member with no active workspace, SUSPENDED, INVITED, removed, or a DB error sees every menu item (and `canTeam` true); the page APIs then 403. (Comment says this is intentional for "member without active workspace".)
  b. **Ignores `isSuperAdmin`**: uses `auth.role` + overrides, not the effective set from `auth.can`; a super admin whose membership role is e.g. VIEWER has ALL permissions on the API but a VIEWER nav.
  c. Nav permission is a single page-level permission; sub-pages (/orders/[id], /products/[id], /catalogs/[id]) inherit via longest-prefix match in `pageForPath`; there is no check at page level (only team page).
  d. `/settings/account` has `permission: null` (nav-model.ts:112); `/dashboard` requires `analytics.read`.
  e. `NAV_REDIRECTS` (nav-model.ts:122-129) merge old URLs; redirect targets are not re-checked for permission.

---------------------------------------------------------------------------

# PART 4 — Team / members as built

## 4.1 Roles (prisma/schema.prisma:15-23) and statuses (:25-29)
`UserRole`: SUPER_ADMIN, OWNER, ADMIN, MANAGER, SALES_AGENT, SUPPORT_AGENT, VIEWER (default on `WorkspaceMember.role` = VIEWER, :113). `MemberStatus`: ACTIVE, INVITED, SUSPENDED. `ASSIGNABLE_ROLES` (service.ts:23-30) = all except SUPER_ADMIN.

## 4.2 Operations (src/modules/members/service.ts) — all routes require `users.manage` (OWNER, ADMIN, SUPER_ADMIN)
| Operation | Service fn (line) | Route | Rules |
|---|---|---|---|
| List roster | `listMembers` (:105) | GET /api/workspaces/members | Workspace-scoped; returns role, status, permissions, expiresAt, user{id,email,name}. |
| Invite existing user | `inviteMember` (:134) | POST /api/workspaces/members (no password) | Email format; role must be assignable; permissions validated; OWNER role only by OWNER (:145); **user must already exist** else 404 "User" (:149-154); already ACTIVE → 409; SUSPENDED → 422 `MEMBER_SUSPENDED`; existing INVITED row is refreshed (new expiry + role + permissions reset); new row = status INVITED, expiry 7 days (`INVITE_TTL_MS` :20); audit `member.invited`. |
| Create account with password | `createMember` (:345) | POST /api/workspaces/members with `name`+`password` (route refine L20-23) | name 2-100, password 8-128, role assignable, OWNER only by OWNER; **email must be new** (409 `EMAIL_TAKEN`, :367-373); creates User + ACTIVE membership; password hashed, never audited; audit `member.created`. |
| Accept invitation | `acceptInvite` (:205) | POST /api/workspaces/members/accept (PUBLIC + manual session) | Self-only; only INVITED rows; expired → 410 `INVITE_EXPIRED`; sets ACTIVE, clears expiry; rewrites session cookie to that workspace; audit `member.accepted`. **No UI**. An invitee who has no ACTIVE membership logs in with `workspaceId = null` and every API fails (getAuthContext 403); an invitee who has their own workspace never sees the invite (listWorkspacesForUser returns only ACTIVE memberships, workspaces/service.ts:44) and there is no switcher/accept UI. |
| Change role / permissions / status | `updateMember` (:228) | PATCH /api/workspaces/members/[id] | Cannot edit self (`CANNOT_EDIT_SELF`); role assignable; status only ACTIVE/SUSPENDED; empty update → 422; **last active OWNER cannot be demoted/suspended** (409 `OWNER_REQUIRED`, :260-265) with post-write race compensation (:272-285); any change that touches an OWNER row or promotes to OWNER requires actor role OWNER (:267-270, `OWNER_ROLE_REQUIRED`); audit `member.updated` (field names only). ADMIN can therefore change MANAGER/agent/VIEWER/other ADMIN rows, but not OWNER rows. |
| Suspend / reactivate | same fn via `status` | PATCH `{status}` | As above; suspended → `getAuthContext` rejects next request (membership status check). UI warns "الجلسة الحالية تُبطل". |
| Reset password | `resetMemberPassword` (:405) | PATCH `{newPassword}` (route branches first, [id]/route.ts:20-23) | Rules as built and the required hardening: `docs/HANDOVER-2026-10-27.md` §4.1 and work package 0.1. Audit `member.password_reset`. |
| Remove member | `removeMember` (:297) | DELETE /api/workspaces/members/[id] | Not self; last-active-OWNER floor with race compensation (:302-328); OWNER target needs OWNER actor; deletes only the membership row (the User remains); audit `member.removed`. |
| Rename workspace | `updateWorkspaceDetails` (workspaces/service.ts:55) | PUT /api/workspaces | `workspace.manage` (OWNER/ADMIN/SUPER_ADMIN); name 2-100 / timezone; audit. |

Everything is audited via `recordAudit`. Rate limiting for these routes: `04-PERMISSIONS.md` §5.5.

## 4.3 Team page UI (src/app/(dashboard)/settings/team/)
- `page.tsx:5` — `enforcePermission("users.manage")` (only page in the app that enforces on the server); renders `TeamView`.
- `team-view.tsx` exposes:
  - Title "الفريق والصلاحيات" (:156), description (:157); header button **"دعوة عضو"** (:160); empty state title "لا أعضاء", button **"ادعُ عضو"** (:181).
  - Table columns: "العضو", "الدور", "الحالة", "إجراءات" (:188-191). INVITED members show "تنتهي الدعوة <date>" (:200-204).
  - Role dropdown per row (aria-label "الدور"), options = `ROLE_OPTIONS` labels (all 6 roles, even for OWNER rows and for the viewer's own row) → immediate PATCH `{role}` (:210). Errors from the server (e.g. owner-only) are shown in a red banner.
  - Status dropdown per row (aria-label "الحالة"): "نشط" / "موقوف" (:232-233), disabled for INVITED rows; choosing "موقوف" opens confirm dialog title `إيقاف وصول "<name>"؟`, confirm button **"إيقاف العضو"** (:372); re-activation is an immediate PATCH.
  - Row buttons: **"كلمة المرور"** (:240, tooltip "تعيين كلمة مرور جديدة لهذا العضو") → dialog "كلمة مرور جديدة" with field "كلمة المرور الجديدة" and submit **"تعيين كلمة المرور"** (:347); **"إزالة"** (:243) → confirm title `إزالة العضو "<name>"؟`, confirm **"إزالة العضو"** (:358).
  - Invite dialog: title "دعوة عضو" or "إنشاء حساب جديد" (:284) with mode switch **"دعوة حساب مسجل"** / **"إنشاء حساب جديد"** (:292,:299); fields "الاسم" (create only), "البريد الإلكتروني" (hint: must already be registered / account opens immediately), "كلمة المرور" (create only, plain-text input, 8+ chars), "الدور" (default SALES_AGENT, with a one-line blurb); submit **"إرسال الدعوة (صالحة 7 أيام)"** or **"إنشاء الحساب وتنشيط العضوية"** (:325).
  - Collapsible card **"شو بيقدر كل دور؟"** (:259) with `RoleBlurbs` and `RoleMatrix`, caption "للقراءة بس — مأخوذ من صلاحيات النظام الفعلية…".
  - Card **"اسم مساحة العمل"** with field "الاسم الجديد" and button **"حفظ"** (:270-278); success "تم حفظ اسم مساحة العمل".
  - **Not exposed**: editing permission overrides, resending an invite (other than re-inviting the same email), accept-invite, any display of the owner-only rule or of overrides. Role dropdown is not filtered by actor rank (an ADMIN sees "مالك" and gets a 403 on use).
- Role labels (`ROLE_OPTIONS`, role-matrix.tsx:16-23): OWNER "مالك", ADMIN "مدير عام", MANAGER "مشرف", SALES_AGENT "موظف مبيعات", SUPPORT_AGENT "موظف دعم", VIEWER "مشاهد". SUPER_ADMIN is internal (not listed).
- `ROLE_BLURB` (:26-33): ADMIN = "مدير عام — نفس صلاحيات المالك." — in code ADMIN has all 46 permissions but cannot manage OWNER rows and cannot delete orders (owner-only), so the blurb and the matrix (ADMIN column all ✓) overstate.

## 4.4 RoleMatrix `CAPABILITIES` rows (role-matrix.tsx:36-59) — 22 rows, perm each maps to
يشوف المحادثات والعملاء والطلبات (conversations.read) · يرد على العملاء (channels.send_human) · يعدّل بيانات العملاء (customers.write) · يكتب الطلبات ويعدّلها (orders.write) · يعطي خصم ضمن السقف (orders.discount) · يعتمد خصم فوق السقف (orders.discount.approve) · يشوف سعر الجملة (products.wholesale) · يستلم التحويلات من الذكاء ويحلّها (handoffs.claim) · يوزّع المحادثات على الموظفين (conversations.assign) · يحذف المحادثات (conversations.delete) · يعدّل المنتجات والكتالوجات (products.write) · يعدّل شخصية الوكيل والمعرفة (personas.write) · يضبط الذكاء الاصطناعي (ai.configure) · يربط القنوات (channels.connect) · يشغّل الأتمتة والمتابعات (automation.write) · ينشئ الحملات (campaigns.write) · يعتمد الحملات وينشرها (campaigns.approve) · يشوف استهلاك الذكاء (usage.read) · يشوف أداء الموظفين (analytics.agents) · يغيّر الإعدادات (settings.manage) · يدير الفريق وصلاحياتهم (users.manage) · يدير مساحة العمل (workspace.manage).
Not represented: catalogs.*, media.*, feedback.*, knowledge.write (folded into "personas.write" row), followups.*, handoffs.resolve, automation.execute, campaigns.read, analytics.read, channels.read, the owner-only order delete, and per-member overrides. The cells call `hasPermission(role, [], perm)`; tests/role-matrix.test.ts guards claims.

---------------------------------------------------------------------------

# PART 5 — Permission usage census (46 permissions, permissions.ts:1-56)

"Handlers" = `permission:` occurrences in route files (via createHandler/analyticsRoute); "inline" = extra `can()/require()` checks. Role sets computed by importing permissions.ts (SUPER_ADMIN/OWNER/ADMIN = ALL; "A3" below = those three).

Role sizes: SUPER_ADMIN/OWNER/ADMIN 46, MANAGER 44 unique (array of 45 contains a duplicate `catalogs.read`), SALES_AGENT 20, SUPPORT_AGENT 16, VIEWER 13.

| Permission | Route handlers requiring it (routes) | Inline checks | Roles holding it |
|---|---|---|---|
| customers.read | 3 | – | A3, MANAGER, SALES_AGENT, SUPPORT_AGENT, VIEWER |
| customers.write | 2 | – | A3, MANAGER, SALES, SUPPORT |
| conversations.read | 9 | – | A3, MANAGER, SALES, SUPPORT, VIEWER |
| conversations.reply | 5 | – | A3, MANAGER, SALES, SUPPORT |
| conversations.assign | 1 | – | A3, MANAGER |
| conversations.delete | 1 | – | A3, MANAGER |
| orders.read | 5 | – | A3, MANAGER, SALES, SUPPORT, VIEWER |
| orders.write | 4 handlers (3 routes) | – (+ owner-only delete) | A3, MANAGER, SALES |
| orders.discount | 1 | – | A3, MANAGER, SALES |
| orders.discount.approve | 1 | pricing-policy.ts:143 | A3, MANAGER |
| products.read | 3 | – | A3, MANAGER, SALES, SUPPORT, VIEWER |
| products.write | 8 handlers (5 routes) | – | A3, MANAGER |
| **products.wholesale** | **0** | sales/coach.ts:53 only | A3, MANAGER |
| catalogs.read | 3 | – | A3, MANAGER, SALES, SUPPORT, VIEWER |
| catalogs.write | 5 handlers (3 routes) | – | A3, MANAGER |
| catalogs.assign | 2 handlers (1 route) | – | A3, MANAGER |
| media.read | 4 | – | A3, MANAGER, SALES, SUPPORT, VIEWER |
| media.write | 3 handlers (2 routes) | – | A3, MANAGER |
| feedback.read | 2 | – | A3, MANAGER, SALES, SUPPORT, VIEWER |
| feedback.write | 2 | – | A3, MANAGER |
| knowledge.read | 4 | – | A3, MANAGER, SALES, SUPPORT, VIEWER |
| knowledge.write | 7 handlers (6 routes) | – | A3, MANAGER |
| personas.read | 2 | – | A3, MANAGER, SALES, SUPPORT, VIEWER |
| personas.write | 5 handlers (4 routes) | – | A3, MANAGER |
| ai.configure | 11 handlers (8 routes) | – | A3, MANAGER |
| channels.read | 2 | – | A3, MANAGER, VIEWER (NOT SALES/SUPPORT) |
| channels.connect | 8 handlers (6 routes) | – | A3, MANAGER |
| channels.send_human | 8 | – | A3, MANAGER, SALES, SUPPORT |
| automation.read | 4 | – | A3, MANAGER, VIEWER |
| automation.write | 9 handlers (5 routes) | – | A3, MANAGER |
| **automation.execute** | **0** | – | A3, MANAGER |
| followups.read | 1 | – | A3, MANAGER, SALES |
| **followups.write** | **0** | – | A3, MANAGER |
| followups.cancel | 1 | – | A3, MANAGER |
| handoffs.read | 1 | – | A3, MANAGER, SALES, SUPPORT |
| handoffs.claim | 1 | – | A3, MANAGER, SALES, SUPPORT |
| handoffs.resolve | 0 at route option level | handoffs/[id]/route.ts:31 | A3, MANAGER, SALES, SUPPORT |
| campaigns.read | 7 handlers (6 routes) | – | A3, MANAGER, SALES, VIEWER |
| campaigns.write | 8 | – | A3, MANAGER |
| campaigns.approve | 1 | campaign-service.ts:215 | A3, MANAGER |
| analytics.read | 15 routes (13 via analyticsRoute default + export + queue) | – | A3, MANAGER, SALES, SUPPORT, VIEWER |
| analytics.agents | 1 (analytics/agents via analyticsRoute arg) | analytics/export L162 (type=agents) | A3, MANAGER |
| usage.read | 1 | – | A3, MANAGER |
| users.manage | 4 handlers (2 routes) | – | **A3 only** |
| settings.manage | 2 | rebuild L22 | A3, MANAGER |
| workspace.manage | 1 | rebuild L22 | **A3 only** |

Flags
- Never required by any route handler option: `products.wholesale` (only coach inline), `automation.execute`, `followups.write`, `handoffs.resolve` (inline only), and `analytics.agents` is route-level only on one route plus inline. Effectively dead permissions (not required anywhere in code): **automation.execute**, **followups.write**. Enforced only inline/service-level: products.wholesale, handoffs.resolve, orders.discount.approve (also route), campaigns.approve (also route).
- Permissions no role except the ALL roles (SUPER_ADMIN/OWNER/ADMIN) holds: **users.manage**, **workspace.manage**. (Everything else is held by MANAGER too; MANAGER = ALL minus those two.)
- Held only by ALL + MANAGER (22 permissions, "manager and up"): conversations.assign, conversations.delete, orders.discount.approve, products.write, products.wholesale, catalogs.write, catalogs.assign, media.write, feedback.write, knowledge.write, personas.write, ai.configure, channels.connect, automation.write, automation.execute, followups.write, followups.cancel, campaigns.write, campaigns.approve, analytics.agents, usage.read, settings.manage.
- Roles with no distinction from ADMIN/OWNER in the permission matrix: SUPER_ADMIN, OWNER and ADMIN are identical (`ALL`), so OWNER vs ADMIN vs SUPER_ADMIN differences exist only in the code paths listed in 2e.
- Odd holder gaps: `channels.read` missing for SALES/SUPPORT (inbox & catalogs/[id] & intelligence call it); `followups.read` for SALES but not SUPPORT; `automation.read` and `channels.read` held by VIEWER but not by the agents; `campaigns.read` held by SALES and VIEWER but not SUPPORT.

---------------------------------------------------------------------------

# Appendix — Key things to double-check / UNVERIFIED
- Whether any script (scripts/) provisions integration tokens/connections for /api/v1 (only modules/integrations/tokens.ts found; no route).
- Whether the Dockerfile/compose/hosting uses /api/health or /api/jobs/run (not referenced in repo files grepped: Dockerfile, docker-compose.yml, package.json, src, scripts).
- Whether the products list/detail responses and the UI respect `products.wholesale` in the live tree (re-check; see `04-PERMISSIONS.md` §5.3).
- Webhook secret checks for Telegram/TikTok are inside `ingest*Webhook` modules (not read in detail).
- The purpose lines for routes whose internals I did not open (e.g. knowledge upload, media upload, personas internals) are taken from file names, comments and call sites; permissions/guards/methods in the matrix were all read from the route files themselves.

---------------------------------------------------------------------------

# PART 6 — LIVE DELTA (brief — verified by the planner on `MKory18/AIresponse@84f0258`)

Parts 1–5 above were produced on the older fork tree (`b8a4a1e`). `src/modules/rbac/*`, `src/modules/members/service.ts`, `src/modules/auth/*`, `src/app/api/auth/me/route.ts`, `src/app/(dashboard)/layout.tsx` and the team page are **not** among the files that differ between the two trees (`git diff --name-status b8a4a1e 84f0258`), so Parts 3–5 still hold. Where this part contradicts Part 1, this part wins. Always re-check with `grep` before building on a row.

## New routes in the live tree

| Route | Methods → permission | Notes |
|---|---|---|
| `/api/settings/oms` | GET, PUT → `settings.manage` | MANAGER holds `settings.manage`, so a MANAGER can change the OMS URL and token |
| `/api/settings/oms/sync` | POST → `settings.manage` | rate limit 6/min |
| `/api/settings/oms/test` | POST → `settings.manage` | rate limit 10/min |
| `/api/settings/telegram-orders` | GET, PUT → `settings.manage` | |
| `/api/settings/telegram-orders/test` | POST → `settings.manage` | rate limit 5/min |
| `/api/events` (SSE) | GET → `conversations.read` | realtime stream for inbox, orders and dashboard |
| `/api/channels/by-id/[id]/order-fields` | GET → `channels.read` · PUT → `settings.manage` | per-page required order fields |
| `/api/orders/[id]/oms` | GET → `orders.read` · POST → `orders.write` | manual "send to OMS"; rate limit 20/min |
| `/api/conversations/[id]/memory/reset` | POST → `conversations.delete` | |
| `/api/customers/[id]/refresh-profile` | POST → `customers.write` | rate limit 5/min |
| `/api/analytics/ai-stages` | GET → `analytics.read` | `analyticsRoute` default |

## Still true in the live tree

- `/api/orders/[id]`: GET `orders.read` · PATCH `orders.write` · DELETE `orders.write` plus the owner-only check in the service.
- There is no `POST /api/orders` (no manual order creation).
- Navigation is 6 groups / 18 pages. `/campaigns`, `/automation`, `/follow-ups` and `/workflows` are redirect stubs to `/dashboard`; their API routes (`/api/campaigns*`, `/api/segments*`, `/api/automations*`, `/api/workflows*`, `/api/follow-ups*`, `/api/discount-approvals*`) still exist and are unreachable from the UI.
