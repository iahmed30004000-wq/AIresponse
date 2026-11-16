# Zaki response — UI inventory as built (read-only audit)

Scope and method
- Parts A–D describe the tree at `…\scratch-2026-11-16-d96b4e\AIresponse` (called **"audited tree"** below). The coordinator later reported this tree is 73 commits behind production; **Part E** (end of file) describes the deployed tree `…\AIresponse-live` and states what changed. Read Part E before acting on anything in Parts A–D.
- Every `page.tsx`, `layout.tsx`, dialog/section file under `src/app/(auth)` and `src/app/(dashboard)`, every file under `src/components`, and `src/lib/{nav-model,nav-search,nav-redirect,page-guides,guided-tour,theme,provider-labels,i18n,ai-connection,timezones}.ts`, `globals.css`, `tailwind.config.ts` were read in full. API route files were only read where needed to verify an endpoint, its HTTP methods or its permission.
- Line counts are `wc -l` (±1 vs. editor numbering). Arabic labels are copied from code; where a label comes from the dictionary the key is given (`t.inbox.takeOver`).
- Nothing was run. Anything about runtime behaviour that cannot be read from code is marked **UNVERIFIED**.
- "Controls" = rough count of distinct interactive elements on the screen in its default state (not counting per-row repeats unless stated).

Top-level findings (details in Part D; "LIVE" = status in the deployed tree, Part E)
1. **The order draft cannot be acted on in the UI.** The inbox shows a read-only "مسودة طلب" card; `POST /api/order-drafts/{id}/confirm` and `/cancel` exist but no screen calls them, there is no draft-edit endpoint and no `POST /api/orders`. — LIVE: still no action. The labels are now Arabic, several open drafts are shown as separate cards and they refresh when a customer message arrives, but confirm/cancel/edit are still absent.
2. **Permissions are only applied to the navigation.** Pages are client components that never receive the user's permissions; write buttons (take over, resolve, delete, order status, toggles…) are rendered for everyone and fail with a generic toast. — LIVE: unchanged, except the new customer profile (`canWrite`/`canConfirm`), the OMS/Telegram cards (hidden on 403) and the OMS order card (`canSend`).
3. **Lists silently show only the first page**: orders/customers/products/feedback = 25 rows, media = 30, messages = last 50 — only the inbox list has "تحميل المزيد". Order stat cards are computed from those 25 rows. — LIVE: unchanged.
4. **Outbound AI replies and human replies look identical** in the thread (the "الذكاء الاصطناعي" label is only rendered for non-outbound AI messages). — LIVE: fixed (every outbound bubble is stamped "الذكاء الاصطناعي" or the employee's name).
5. Same concept, many names (handoff: التحويلات / التسليم للموظفين / بحاجة لموظف / تحتاج مراجعة / بانتظار موظف / استلام / تحويل إلى موظف; AI: الذكاء / الذكاء الاصطناعي / الوكيل / الوكيل الذكي / المساعد / AI), mixed Levantine and formal Arabic, raw English enums and env-var names shown to the owner. — LIVE: unchanged, plus new jargon ("الـ OMS", "chat_id", "topic_id") and three spellings of Telegram.
6. LIVE only: the nav is down to 6 groups / 18 pages (campaigns, automation, follow-ups, workflows now redirect to `/dashboard`); the inbox, orders list and dashboard update through SSE (`/api/events`) with a slow polling fallback; `/channels` gained three more configuration blocks (OMS link, Telegram order forwarding, per-page order fields); a customer profile page `/customers/[id]` exists.

---

# PART A — Shell

## A.1 Composition chain

```
src/app/layout.tsx (92, server)            <html lang="ar" dir="rtl"> + 3 Google fonts + inline theme script
 └─ src/app/(dashboard)/layout.tsx (47, server)   session check → nav filtered by permission → <ToastProvider><AppShell>
     └─ src/components/app-shell.tsx (407, client)
         ├─ src/components/app-sidebar.tsx (223, client)   one link per GROUP (desktop fixed pane / mobile drawer)
         ├─ <header> (inside app-shell)                    breadcrumb · search/jump · ? · tour · bell · theme · account menu
         ├─ src/components/group-tabs.tsx (54, client)     tabs for the pages of the current group (only if group has >1 page)
         ├─ src/components/page-guide.tsx (86, client)     "شو هاي الصفحة؟" panel
         └─ <main> {children}                              the page (p-4 lg:p-6, except /conversations = full-bleed)
```
`(auth)` pages have no layout of their own; they render `components/auth-form.tsx` directly under the root layout. There is **no `middleware.ts`**, and **no `loading.tsx`, `error.tsx` or `not-found.tsx`** anywhere under `src/app` (audited tree).

## A.2 Root layout — `src/app/layout.tsx`
- `<html lang="ar" dir="rtl" suppressHydrationWarning>`; body gets three font variables: IBM Plex Sans Arabic (`--font-arabic`, 400/500/600/700), Reem Kufi (`--font-reem`, not preloaded), Lalezar (`--font-lalezar`, not preloaded).
- Inline `<script>` = `THEME_INIT_SCRIPT` (sets `data-theme` and `.dark` on `<html>` before first paint).
- Metadata: title "Zaki response", description "Zaki response — منصة المحادثات والمبيعات بالذكاء الاصطناعي", `robots: noindex`, PWA manifest (`src/app/manifest.ts`, start_url `/dashboard`, standalone, portrait).
- `ToastProvider` is imported here but not used (line 3); the provider is mounted in the dashboard layout only.

## A.3 Dashboard layout — where visibility is decided (`src/app/(dashboard)/layout.tsx`)
1. `readSession()`; no session → `redirect("/login")`. Loads `user.name` (and email, unused).
2. Default: all groups visible (`visibleGroups(NAV_GROUPS, () => true)`).
3. `getAuthContext()` → workspace name, and `groups = visibleGroups(NAV_GROUPS, p => hasPermission(auth.role, auth.memberPermissions, p))`. If this throws (member without an active workspace) the user keeps the **full** nav (comment lines 21–22: "الصفحات نفسها بتفرض الصلاحية من الخادم، مش القائمة").
4. Renders `<ToastProvider><AppShell groups workspaceName userName>`.

**This is the only place the UI filters by permission.** What reaches the client is `NavGroupView` = `{key,label,icon,placement,pages:[{href,label}]}` — "بلا صلاحيات" (nav-model.ts:131). Consequences verified in code:
- A page hidden from the nav is still reachable by URL; nothing in the layout blocks it. The page then renders its normal chrome and its `fetch` calls return 403, which every page maps to a generic load error.
- No page or component receives the role or permission set. The only permission-aware UI in the whole tree:
  | Where | What |
  |---|---|
  | `app-shell.tsx:128-131` | `canTeam` (shows "الفريق" in the account menu) and `nextAllowed` (hides the guide's "next step" link) — derived from the filtered nav |
  | `lib/nav-search.ts` | quick-jump suggestions only from permitted groups |
  | `settings/team/page.tsx` | the only server-side page guard: `await enforcePermission("users.manage")` (throws; there is no `error.tsx`, so what the user sees is **UNVERIFIED**) |
  | `orders/page.tsx`, `orders/[id]/page.tsx` | `canDelete` flag returned by the API (owner only). `canWrite` is also returned by `GET /api/orders/{id}` and typed (line 16) but **never used** |
  | `setup-checklist.tsx:177` | card hides silently when `/api/setup/progress` is not OK (needs `settings.manage`) |
  | `analytics/page.tsx:243,487` | a 403 on a tab endpoint → "مش متاحة لدورك" (agents tab) |
  | `intelligence/page.tsx:263` | 403 on send-mode → "ما عندك صلاحية تغيير وضع الصفحة." |
  | `integrations/ui.tsx:188` (`apiJson`) | 403 → "ما عندك صلاحية لهالإجراء" (used only by MarketingCard and ComplianceCard) |

## A.4 Navigation model — `src/lib/nav-model.ts` (189)
> LIVE: the deployed nav has **6 groups / 18 pages** — group 5 (الأتمتة) and the page "الحملات" are gone and their routes redirect to `/dashboard` (E.1). The shell components themselves (`app-shell`, `app-sidebar`, `group-tabs`) are byte-identical in production, so A.1–A.3 and A.5–A.8 describe the live shell exactly.

Seven groups; the sidebar shows **groups**, the tab bar shows the **pages** of the active group.

| # | Group (key) | Icon | Placement | Pages (tab label → route → required permission) |
|---|---|---|---|---|
| 1 | الرئيسية (`home`) | LayoutDashboard | main | لوحة التحكم → `/dashboard` → `analytics.read` |
| 2 | المحادثات (`chats`) | MessageSquare | main | الوارد → `/conversations` → `conversations.read` · التحويلات → `/handoffs` → `handoffs.read` · العملاء → `/customers` → `customers.read` · مختبر التجميع → `/intelligence` → `conversations.read` |
| 3 | المبيعات (`sales`) | ShoppingBag | main | الطلبات → `/orders` → `orders.read` · المنتجات → `/products` → `products.read` · الكتالوجات → `/catalogs` → `catalogs.read` · الحملات → `/campaigns` → `campaigns.read` |
| 4 | الذكاء (`ai`) | Sparkles | main | الشخصية → `/personas` → `personas.read` · المعرفة → `/knowledge` → `knowledge.read` · الوسائط → `/media` → `media.read` · آراء العملاء → `/feedback` → `feedback.read` · التجربة → `/ai/playground` → `ai.configure` · التشغيل → `/settings/ai` → `ai.configure` |
| 5 | الأتمتة (`automation`) | Workflow | main | القواعد → `/automation` → `automation.read` · المتابعات → `/follow-ups` → `followups.read` |
| 6 | التقارير (`reports`) | ChartNoAxesCombined | main | التقارير → `/analytics` → `analytics.read` |
| 7 | الإعدادات (`settings`) | Settings | bottom | القنوات → `/channels` → `channels.read` · الفريق → `/settings/team` → `users.manage` · حسابي → `/settings/account` → `null` (everyone) |

- 21 real pages. `NAV_REDIRECTS` (exact-match): `/order-warehouse`→`/orders`, `/ai/studio`→`/settings/ai`, `/workflows`→`/automation`, `/analytics/ai`→`/analytics?tab=ai`, `/analytics/products`→`/analytics?tab=products`, `/settings`→`/settings/account` (each is a 6–9 line `page.tsx` calling `redirectToMerged`).
- `pageForPath` = longest-prefix match, so `/orders/abc` belongs to "الطلبات", `/products/x` to "المنتجات", `/catalogs/x` to "الكتالوجات".
- `groupLanding(group)` = first permitted page of the group (what a sidebar click opens).
- Routes with no nav entry of their own: `/orders/[id]`, `/products/[id]`, `/catalogs/[id]` (detail pages), the redirects, `/login`, `/register`, `/`.

## A.5 Sidebar — `src/components/app-sidebar.tsx`
Desktop (`lg+`): fixed pane on the start side (right in RTL), 280 px, collapsible to 68 px (state in `localStorage["sidebar-collapsed"]`, `sidebar-toggle` window event; main column padding follows).
1. Brand block: `BrandLogo` + "Zaki **response**" + workspace name + collapse button (aria "طي القائمة" / "توسيع القائمة").
2. Nav list: one `NavLink` per group → `groupLanding(g)`. Under the label a hint line lists the group's page labels joined by " · " (only if >1 page). Active group = group of the current path (primary-soft background + 3 px bar). Group with `placement:"bottom"` (الإعدادات) is pinned at the bottom under a divider. Collapsed: icons only with `title`.
3. User block: avatar, user name, workspace name, logout icon button (`<form action="/api/auth/logout" method="post">`, aria "تسجيل الخروج"). The logout form is not rendered when collapsed.

Mobile (`<lg`): hamburger in the header (aria "فتح القائمة") opens a drawer (w-72, max 86vw, `role="dialog"`): brand, close ✕ (aria "إغلاق القائمة"), the same group list (closes on navigate), a row with `ThemeToggle` + a decorative bell icon + "Zaki response" (the bell here is not a link), then the user block.

There is **no unread/pending count** on any nav item and no badge on groups.

## A.6 Top bar (inside `app-shell.tsx`, sticky `top-0 z-20`, height `h-14`)
Left to right in DOM order (RTL: right to left on screen):
| Element | Visible | Behaviour |
|---|---|---|
| Menu button | `<lg` | opens mobile drawer |
| Breadcrumb | `xl+` only | "لوحة التحكم › {group} › {page}" — e.g. "لوحة التحكم › التقارير › التقارير" (group and page share the label); on `/dashboard` just "لوحة التحكم" |
| Search / quick-jump | `md+` only | input placeholder "بحث أو انتقل لصفحة…"; typing lists up to 6 page matches (label + group) from `PAGE_KEYWORDS` (Arabic formal + colloquial + English synonyms), ↑/↓ + Enter navigates; **Enter with no highlighted suggestion → `/conversations?q=<text>`** (conversation search). Footer hint: "↑↓ للتنقل · Enter للانتقال · أو اكتب واضغط Enter لتبحث بالمحادثات". Shortcut `/` or Ctrl/Cmd+K. |
| "?" button | only when the route has a guide | toggles the page-guide panel (aria/title "شرح هذه الصفحة") |
| Compass button | always | starts the guided tour (aria/title "جولة سريعة بالموقع") |
| Bell link | always | plain link to `/handoffs` (aria/title = `t.nav.handoffs` "التسليم للموظفين"). No count, no dropdown — comment: "لا نقطة زائفة". It is rendered even if the user lacks `handoffs.read`. |
| Theme toggle | always | light/dark (`ThemeToggle`); disabled with tooltip "هذا الثيم ليلي دائمًا — بدّل الثيم من «حسابي»" under the night theme |
| Account button | `sm+` only | avatar + name + chevron → menu: "حسابي" (`/settings/account`), "الفريق" (`/settings/team`, only if in nav), "تسجيل الخروج" (form POST `/api/auth/logout`) |

There is no notification centre, no workspace switcher (API `GET/PATCH/POST /api/workspaces` exists, unused by UI), no global "new order/new product" action, no language switch.
On phones (`<sm`) the account menu is hidden and (`<md`) the search is hidden; logout is then only reachable through the drawer's user block.

## A.7 Group tabs — `src/components/group-tabs.tsx`
Rendered under the header when the current group has more than one permitted page (so never on الرئيسية or التقارير). Horizontal scroll on small screens; the active tab is scrolled to centre; height 2.75 rem, exported as `TABS_HEIGHT` and written to the CSS variable `--shell-extra` so the inbox can subtract it from its viewport height.

Result: up to three stacked navigation layers on a page — sidebar group → tab bar → in-page tabs/chips (e.g. `/analytics` has 9 pill tabs, `/products/[id]` 6 tabs, the inbox 5 filter chips + composer tabs).

## A.8 Page guide — `src/lib/page-guides.ts` (163) + `src/components/page-guide.tsx`
- `PAGE_GUIDES` has one entry for each of the 21 nav pages: `what` (1–2 sentences), optional `tip`, `next {label, href}`. Detail pages inherit the parent's guide via `pageForPath`.
- Panel content: "شو هاي الصفحة؟" · `what` · 💡 `tip` · "الخطوة الجاية: {label} ←" (only if that page is in the user's nav) · close ✕ (aria "إغلاق الشرح").
- Opens automatically **on the first visit to each page** (`localStorage["zr-guide-seen"]`, JSON array capped at 200), afterwards only via "?". Esc closes.
- Normal pages: rendered in the flow above `<main>` (pushes content down). `/conversations`: never auto-opens; when toggled it is an absolutely-positioned overlay under the header.
- "Next step" chain encoded in the guides: dashboard → conversations → handoffs → customers → orders → analytics → products → personas → knowledge → playground → settings/ai → conversations; channels → products; media/feedback → playground; catalogs → campaigns → analytics; automation ↔ follow-ups; intelligence → conversations; team → handoffs; account → dashboard.

## A.9 Guided tour — `src/lib/guided-tour.ts` (38) + `src/components/guided-tour.ts` (40, driver.js)
10 steps anchored on `data-tour` attributes: the 7 sidebar groups (`group-home` … `group-settings`), `search`, `help`, `tour`. Steps whose target is not visible are dropped (hidden group, collapsed mobile drawer, hidden search on mobile, "?" absent). Auto-starts once, 700 ms after the first visit to `/dashboard` (`localStorage["zr-tour-seen"]="1"`); restartable from the compass button. Buttons "التالي" / "السابق" / "يلا نبلّش", progress "{{current}} من {{total}}". Step titles: الرئيسية, المحادثات, المبيعات, الوكيل الذكي (the sidebar group itself is labelled "الذكاء"), الأتمتة, التقارير, الإعدادات, انتقال سريع, شو هاي الصفحة؟, خلصنا 👋.

## A.10 Setup checklist — `src/components/setup-checklist.tsx` (191) + `src/modules/setup/progress.ts`
Shown only on `/dashboard`, under the hero. `GET /api/setup/progress` (permission `settings.manage`); any non-OK response → renders nothing.
Card "ابدأ من هون" — "خطوتك الجاية: {label}" · progress bar · "n / 6" · six step tiles (link each):
| # | Label | Hint (shown on the "next" step only) | Link | Required | Done when |
|---|---|---|---|---|---|
| 1 | اربط قناة | فيسبوك أو واتساب أو تيليغرام — من هون بتوصل رسائل عملائك. | `/channels` | yes | ≥1 channel `CONNECTED` |
| 2 | أضف منتجاتك | الوكيل بيجاوب بأسعارك ومواصفاتك — بلا منتجات ما عنده شي يبيعه. | `/products` | yes | ≥1 active, non-archived product |
| 3 | اضبط شخصية الوكيل | اللهجة والنبرة وأسلوب البيع. اختياري — في شخصية عامة جاهزة. | `/personas` | no ("اختياري" chip) | ≥1 active persona |
| 4 | علّمه أجوبة أسئلتك | التوصيل والإرجاع والضمان… | `/knowledge` | no | ≥1 active FAQ or READY document |
| 5 | جرّب الوكيل | اكتب له متل ما بيكتب زبونك… | `/ai/playground` | no | ≥1 playground session |
| 6 | شغّل الرد الآلي | اختر مزوّد ذكاء حقيقي (بمفتاحه) وفعّل الرد الآلي… | `/settings/ai` | yes | `aiEnabled && autoRespond` and a real provider with credentials |
The "next" tile is highlighted and shows "ابدأ ←". When all required steps are done it collapses to a bar "النظام جاهز — الخطوات الأساسية كلها خلصت" with "عرض الخطوات" / "طي". The grid is `xl:grid-cols-5` for six tiles (sixth wraps to a second row).

## A.11 Other shell facts
- `<main>` padding `p-4 lg:p-6`; page content normally wrapped in `.page-wrap` (`max-w-7xl`, `space-y-5`). `/conversations` is "full-bleed" (no padding) and sizes itself with `h-[calc(100dvh-4rem-var(--shell-extra,0px))]`.
- Toasts: `components/ui/toast.tsx`, bottom-centre, auto-dismiss after 3.5 s.
- No global data layer: every page does its own `fetch` in `useEffect`; no SWR/React-Query; no realtime. **The only timer in the UI is the inbox's 20 000 ms poll** (`inbox.tsx:188-195`).
  > LIVE: a `useRealtime` hook (SSE on `/api/events`) now drives the inbox, the orders list and the dashboard; fallback timers are 60 s (5 s when the stream is down) in the inbox and 60 s on `/orders` (E.3.1, E.4, E.6). The tour has 9 steps and `/conversations` and `/orders` have `loading.tsx` skeletons.

---

# PART B — Route-by-route inventory (audited tree)

Conventions: "Server/Client" = component type of the page file. Endpoints are written `METHOD /path`. Permission in brackets is what the API route enforces (from the route file), which the UI does **not** check unless stated.

**Which Part B sections still match production** (details in Part E):
| Section | Status in the deployed tree |
|---|---|
| B.0 `/`, B.1 login/register, B.4 `/handoffs`, B.6 `/intelligence`, B.9, B.10 `/products`, B.15 `/personas`, B.17 `/media`, B.18 `/feedback`, B.19 playground, B.20, B.25 `/analytics`, B.26, B.28, B.29 team, B.30 account | identical files — description is current |
| B.2 `/dashboard` | + realtime refresh; "الأتمتة" section removed (E.6) |
| B.3 inbox | SSE instead of the 20 s poll, several draft cards with Arabic labels, sender stamp on outbound bubbles, panel lost consent card and AI summary, two new panel actions (E.3) |
| B.5 `/customers` | rows now open the new `/customers/[id]` profile (E.5) |
| B.7 `/orders`, B.8 `/orders/[id]` | + realtime/60 s refresh, "تلجرام" column, OMS card (E.4) |
| B.11 `/products/[id]` | save feedback next to the button (E.9) |
| B.12, B.13 catalogs | forced-delete confirm; null-safe rows (E.9) |
| B.14 `/campaigns`, B.22 `/automation`, B.23 `/workflows`, B.24 `/follow-ups` | **removed** — redirect to `/dashboard` (E.2) |
| B.16 `/knowledge` | + per-product scope selectors (E.9) |
| B.21 `/settings/ai` | + "ليش ما رد؟" card, second test button, token warning (E.8) |
| B.27 `/channels` | + OMS card, Telegram order-forwarding card, per-page order fields (E.7) |

## B.0 `/` — `src/app/page.tsx` (8 lines, server)
Redirect only: session → `/dashboard`, else `/login`.

## B.1 `/login` and `/register` — `src/app/(auth)/{login,register}/page.tsx` (5 lines each, server) → `src/components/auth-form.tsx` (172, client)
Purpose: sign in / create account + workspace.
Layout: two columns on `lg+`.
1. **Showcase panel** (`data-showcase`, hidden `<lg`): logo, "Zaki response", tagline `t.app.tagline` "محرك المحادثات والمبيعات بالذكاء الاصطناعي"; pill "مساعد المبيعات الذكي لمتجرك"; H1 "حوّل محادثات عملائك إلى طلبات مؤكدة — تلقائيًا"; three highlights — "رد آلي ذكي / الوكيل الذكي يرد على عملائك لحظيًا على مدار الساعة", "طلبات بلا أخطاء / استخراج البيانات وتأكيد الطلبات تلقائيًا", "نمو تقيسه / تحليلات المبيعات والأداء لحظة بلحظة"; footer "حماية الأسعار والطلبات وتسليم ذكي للموظفين — دائمًا".
2. **Form card**: pill "مرحبًا بعودتك" (login) / "ابدأ مجانًا" (register); H1 "تسجيل الدخول" / "إنشاء حساب جديد"; sub "ادخل إلى لوحة التحكم لإدارة المحادثات والطلبات" / "أنشئ مساحة عملك وابدأ خلال دقيقة".
   - Register-only fields: "الاسم الكامل", "اسم مساحة العمل / المتجر".
   - Both: "البريد الإلكتروني", "كلمة المرور" (`PasswordInput` with eye toggle, min 8).
   - Submit: "دخول" / "إنشاء الحساب" (while loading: "جارٍ التحميل…") → `POST /api/auth/login` or `POST /api/auth/register`; success → `router.replace("/dashboard")`.
   - Link: "ليس لديك حساب؟ إنشاء حساب" / "لديك حساب؟ تسجيل الدخول".
   - Footer text "Zaki response — محادثات أكثر، مبيعات أكثر".
Errors: inline red box (server message, or "تعذر تسجيل الدخول (status) — حاول مجددًا", or "تعذر الاتصال بالخادم — تحقق من الشبكة وحاول مجددًا").
Not present: forgot/reset password, remember-me, invite acceptance, social login, terms. Two `<h1>` on desktop.
Controls: login 5 (2 inputs, eye, submit, link); register 7.

## B.2 `/dashboard` — `src/app/(dashboard)/dashboard/page.tsx` (205, client)
Purpose: period overview of the store. Data: `GET /api/analytics/overview?range=&timezone=` [`analytics.read`]. No polling.
Sections in order:
1. **Hero card** (`data-hero`): chip = tagline; H1 `t.dashboard.greeting` "مرحبًا" (no user name); sub "نظرة سريعة على متجرك بالفترة الي بتختارها"; two selects — period (اليوم / أمس / آخر 7 أيام / آخر 30 يومًا; default 7 days) and timezone (التوقيت العالمي (UTC) / دمشق · عمّان / الرياض / دبي / القاهرة; **default UTC, not persisted**).
2. **SetupChecklist** (A.10).
3. Error card: "تعذّر تحميل البيانات" / "تعذّر تحميل التحليلات، حاول مجددًا." + button "إعادة المحاولة".
4. Loading: 6 + 1 skeleton blocks (only on first load — `loading` is never set back to true when the filters change).
5. **KPI row** — 6 `StatCard`s (not clickable): "المحادثات", "الطلبات", "العملاء (جدد)", "نسبة التحويل" (= confirmed orders ÷ conversations, "—" when no inbound), "ردود الوكيل" (= `ai.calls`), "تحتاج تدخلًا بشريًا" (= open handoffs).
6. Section "المبيعات" (link "التفاصيل ←" → `/orders`): طلبات مؤكدة · طلبات مُسلَّمة · إيرادات مؤكدة · إيرادات مُسلَّمة (no currency shown).
7. Section "المحادثات" (→ `/conversations`): نشطة · رسائل واردة · رسائل صادرة.
8. Section "الوكيل الذكي" (→ `/analytics?tab=ai`): استدعاءات فاشلة · توكنز مستهلكة · التكلفة التقديرية ("غير متاح" when null).
9. Two columns: "الدعم البشري" (→ `/handoffs`): إجمالي التحويلات · متوسط زمن الحل (دقيقة); "الأتمتة" (→ `/automation`): تنفيذات · ناجحة · فاشلة · متجاوزة.
Empty state: none (zeros). Permission checks: none in page (checklist hides on 403). The "التفاصيل" links are rendered even when the target is not in the user's nav.
Controls: 2 selects + 5 "التفاصيل" links + checklist (6 tiles + toggle) ≈ 14. There is no "what needs my attention now" list (no latest conversations/orders; the dictionary still has unused keys `recentConversations`, `recentOrders`, `topProducts`).

## B.3 `/conversations` — the inbox
Files: `conversations/page.tsx` (5, server, just `<Inbox />`) · `components/inbox/inbox.tsx` (909, client) · `customer-panel.tsx` (207) · `conversation-item.tsx` (149) · `message-bubble.tsx` (110) · `marketing-card.tsx` (128) · `MediaPicker.tsx` (142) · `types.ts` (140).
Purpose: read and answer every customer conversation, take over from / hand back to the AI.
Layout: three panes side by side, `gap-3`, height = viewport − header − tab bar.
- `<lg`: only one pane at a time (list, or thread once a conversation is selected; back arrow returns).
- `lg–xl`: list + thread; customer panel opens as a `Drawer` from the "ملف العميل" button.
- `xl+`: list (w-80) + thread + customer panel (w-72).

### B.3.1 Data flow and refresh
| Trigger | Calls |
|---|---|
| Mount | `GET /api/channels` [`channels.read`] for the page filter (failure ignored → filter just has "كل الصفحات") |
| Mount / filter change (search debounced 300 ms) | `GET /api/conversations?q&unreadOnly&status&channelId&assignee&channelType&cursor` [`conversations.read`], 30 per page |
| Select a conversation | parallel `GET /api/conversations/{id}/messages` (latest 50) + `GET /api/conversations/{id}`; then `GET /api/customers/{customerId}` and `GET /api/order-drafts?conversationId={id}`; then `POST /api/conversations/{id}/read {isRead:true}` (optimistic unread = 0) |
| **Every 20 000 ms** (`setInterval`, inbox.tsx:188) | `fetchList()` (first page, current filters) + `GET …/messages` of the open thread |
| URL params on mount | `?q=` prefills the search (from the top-bar search / customers page); `?id=` pre-selects a conversation (from handoffs / order page) |
Not refreshed by the poll: customer record, **order draft**, open-handoff banner, activity log, coach suggestions.
Code-level consequences:
- The poll calls `fetchList()` without `append`, so it **replaces the list with page 1** — rows loaded through "تحميل المزيد" disappear at the next tick.
- `selected` is looked up in the loaded list (`conversations.find`). If `selectedId` points at a conversation that is not in the currently loaded page (deep link `?id=` to an older or filtered-out conversation, or after the poll dropped it), the thread pane falls back to the "اختر محادثة لعرضها" placeholder while the right panel still shows that customer; on `<lg` the list is hidden (`selectedId && "max-lg:hidden"`) and the placeholder has no back button.
- The API returns `nextCursor` for older messages but the UI never requests them: only the latest 50 messages of a thread can be seen.

### B.3.2 Pane 1 — conversation list
1. Search `Input` — placeholder `t.inbox.searchPlaceholder` "بحث بالاسم أو الهاتف أو نص الرسالة…".
2. Filter chips (`role="tablist"`): "الكل" · "غير المقروءة" (+ total unread bubble) · "يديرها الذكاء الاصطناعي" · "بحاجة لموظف" · "تحتاج مراجعة".
   - "غير المقروءة" is server-side (`unreadOnly=true`).
   - The other three filter **client-side on the 30 loaded rows**: "يديرها الذكاء الاصطناعي" = `aiState !== "HUMAN_ACTIVE"`; "بحاجة لموظف" = `aiState === "HUMAN_ACTIVE"` (i.e. conversations a human already holds, not ones waiting for one); "تحتاج مراجعة" = `hasOpenHandoff`.
3. Four `Select`s in a 2×2 grid: الحالة (كل الحالات / مفتوحة / معلقة / محلولة / مغلقة) · الصفحة (كل الصفحات + every connected channel of any type) · الإسناد (كل الموظفين / مُسندة إليّ / غير مُسندة) · القناة (كل القنوات / فيسبوك ماسنجر / إنستغرام / واتساب / تيليغرام / الويب).
4. Rows (`ConversationItem`, a `<button role="option">`): avatar + small channel icon; name (fallback `t.inbox.unknownCustomer` "عميل"); relative time; channel display name with a **Facebook icon regardless of channel type** (`conversation-item.tsx:60-65`); last message text or phone; badges: conversation status (`StatusBadge`), "الذكاء الاصطناعي" or "موظف" (`t.roles`), "تحتاج مراجعة" (open handoff), "عاجلة"/"عالية" priority; assigned agent name (`sm+`); unread count; unread bar on the edge.
5. States: 7-row skeleton; error text `t.inbox.loadListFailed` "تعذّر تحميل المحادثات" + "إعادة المحاولة"; empty "لا توجد محادثات حتى الآن / ستظهر المحادثات الجديدة هنا عند وصول أول رسالة من عملائك." or, when filtered, "لا توجد نتائج مطابقة / جرّب كلمة بحث مختلفة أو أزل عوامل التصفية." (no link to connect a channel); "تحميل المزيد" when a cursor exists.

### B.3.3 Pane 2 — thread
Placeholder (nothing selected): "اختر محادثة لعرضها" / "ستظهر رسائل المحادثة هنا بمجرد اختيارها من القائمة."
**Header** — left: back (mobile), avatar, name, sub-line "{page name or channel label} · {customer status} · {assigned agent}", optional "شخصية الذكاء الاصطناعي: {persona}". Right cluster (wraps):
| # | Control | Label (exact) | Action |
|---|---|---|---|
| 1 | Badge (not a control) | "الرد بواسطة الذكاء الاصطناعي" / "الرد يدويًا" | shows `aiState` |
| 2a | Button (AI active) | `t.inbox.takeOver` "استلام من الذكاء الاصطناعي" | opens confirm: title "تحويل المحادثة إلى موظف؟", body "بعد التحويل سيتوقف الذكاء الاصطناعي عن الرد تلقائيًا على هذه المحادثة حتى تتم إعادتها يدويًا.", buttons "إلغاء" / "تحويل إلى موظف" → `POST /api/conversations/{id}/takeover` [`conversations.reply`] (server also assigns the conversation to the acting user) |
| 2b | Button (human active) | `t.inbox.backToAi` "إعادة إلى الذكاء الاصطناعي" | no confirm → `POST /api/conversations/{id}/release-ai` (server also cancels open handoffs) |
| 3 | Button | "مساعد البيع" | opens coach dialog → `POST /api/conversations/{id}/coach` (body `{}`) |
| 4 | Button (green) | `t.inbox.resolve` "تم الحل" | `PATCH /api/conversations/{id} {status:"RESOLVED"}`; toast "تم الحل" |
| 5 | Button (`<xl`) | "ملف العميل" | opens customer drawer |
| 6 | Button (red text) | `t.inbox.deleteConversation` "حذف المحادثة" | confirm: "حذف المحادثة نهائيًا؟" + long body, button "حذف المحادثة" → `DELETE /api/conversations/{id}` [`conversations.delete`]; toast "تم حذف المحادثة" / "تعذّر حذف المحادثة" |
Not available in the UI although the API supports it: assign to another agent (`POST …/assign`), change priority or tags, reopen/close (`PATCH` accepts `OPEN|PENDING|CLOSED`, UI only sends `RESOLVED`).
**Banner** (if an open handoff exists): "بانتظار موظف: {reason} · مُستلمة|مفتوحة" with reasons طلب العميل / شكوى / ثقة منخفضة / تعطل الذكاء / طلب محظور / مشكلة دفع / محاولة حقن / فشل متكرر / يدوي. Informational only (no claim button here).
**Activity log**: `<details>` "سجل المحادثة (n)" — إسناد · إلغاء إسناد · استلام الموظف · إعادة للذكاء · طلب مراجعة · تعطل الذكاء · ملاحظة داخلية + actor + date.
**Messages**: skeleton; empty "لا رسائل بعد / ستظهر رسائل هذه المحادثة هنا بمجرد وصولها."; day separators; bubbles (`message-bubble.tsx`):
- SYSTEM → centred grey pill.
- Internal note → centred amber block "ملاحظة داخلية · للفريق فقط".
- Inbound = start side (`bubble-in`), outbound = end side (`bubble-out`, primary colour).
- The "الذكاء الاصطناعي" tag is rendered only when `senderType === "AI" && !outbound` (lines 40-45) — AI replies are outbound, so **AI and human replies are visually identical**; no sender name on human replies either.
- Media: only a text link "مرفق (type)" opening a new tab — no inline image/audio preview.
- Outbound footer: time · "تم الإرسال ✓✓", or "جارٍ الإرسال"/"فشل الإرسال" + link-button "إعادة المحاولة" → `POST /api/conversations/{id}/messages/{messageId}/retry` [`channels.send_human`]. No delivered/read distinction (the exported `DeliveryTick` is unused).
**Composer**:
- Tabs "رد للعميل" / "ملاحظة داخلية" (amber styling in note mode).
- Paperclip button (aria `t.inbox.attach` "إرفاق ملف", title "أرسل من المكتبة") → MediaPicker; disabled in note mode.
- Emoji button — permanently `disabled`, title "رموز تعبيرية (قريبًا)" (dead control).
- Textarea — placeholder "اكتب ردك… (Enter للإرسال، Shift+Enter لسطر جديد)" / "ملاحظة للفريق فقط — لن يراها العميل…".
- Send icon button → reply: `POST /api/conversations/{id}/messages {text?, mediaId?, idempotencyKey}` [`channels.send_human`] (optimistic bubble; toast "جارٍ الإرسال" when queued, "تعذّر إرسال الرد" on failure); note: `POST /api/conversations/{id}/notes {text}` [`conversations.reply`].
- Hint line "Enter إرسال · Shift+Enter سطر جديد" (said twice: placeholder + hint).
- The composer is enabled while the AI is still active; `sendHumanReply` (`src/modules/channels/outbound.ts`) contains no reference to `aiState`, i.e. a manual reply does not itself switch the conversation to human mode (other handlers UNVERIFIED).
**MediaPicker dialog** "أرسل من المكتبة": tabs "صور المنتجات" (`GET /api/media?category=PRODUCT&type=IMAGE&limit=30`), "المكتبة" (`GET /api/media?type=IMAGE&limit=30`), "آراء الزبائن" (`GET /api/media/feedback?limit=20`, with warning "⚠️ تأكد إنه الصورة ما فيها اسم أو رقم الزبون…"). Clicking a thumbnail **sends immediately** together with whatever text is in the composer — no preview/confirm. Not filtered to the conversation's product. No upload from device.
**Coach dialog** "مساعد البيع": note "اقتراحات داخلية فقط — لا يُرسل شيء للعميل تلقائيًا."; "تحديث"; cards with kind (معالجة اعتراض / خصم / بيع مُكمِّل / عرض / تلميح رد) + reason + text + "إدراج في المحرر"; "لا يوجد عرض: {reason}"; "إغلاق"; error state "تعذّر جلب الاقتراحات". `coachData` is not cleared when another conversation is selected and is only fetched `if (!coachData)` — opening the coach on a second conversation shows the first one's suggestions until "تحديث" is pressed (inbox.tsx:623-626).

### B.3.4 Pane 3 — customer panel (`customer-panel.tsx`), top to bottom
1. Identity: avatar, name, phone, channel name (Facebook icon), status badge, "قائمة سوداء · {source}".
2. "بيانات العميل": البريد الإلكتروني · المحافظة · العنوان · "المحادثات: n · آخر نشاط: …" · "حسابات القنوات" badges · "الوسوم" badges. **Read-only** — no edit/tag/block/notes UI although `PATCH /api/customers/{id}` exists; `customer.notes` is fetched but not rendered.
3. `MarketingCard` "الرسائل التسويقية": pill موافق / ألغى / ما وافق لسا; date + source; buttons "وافق صراحة" and "ألغِ الاشتراك" → `POST /api/customers/{id}/consent {consent}` [`customers.write`] (`GET` same path on load); attribution line "المصدر: إعلان|رمز كليبر · {channel} · {refCode} · {adId} · {date}"; re-opt-in confirm "الزبون ألغى قبل — طلب صراحة يرجع؟" / "نعم، طلب صراحة".
4. "آخر الطلبات": rows (number · status · total) linking to `/orders/{id}`, or "لا توجد طلبات لهذا العميل بعد."; "مصدر الاكتساب: {adSource}".
5. **Order draft card** (below).
6. "ملخص الذكاء الاصطناعي" (or "لم يُولَّد ملخص لهذا العميل بعد.").
7. "النشاط الأخير" (only if the customer has >1 conversation): status + relative time of up to 5 conversations (not clickable).
Empty (no selection): "اختر محادثة لعرض بيانات العميل".

### B.3.5 The ORDER DRAFT ("مسودة طلب") — exactly as built in the audited tree
- **Fetch**: `loadDraft()` → `GET /api/order-drafts?conversationId={id}` [`orders.read`]; takes `items[0]` (the schema has `@@unique([conversationId])`, so one draft per conversation). Called once, after the conversation detail loaded (inbox.tsx:240). Reset to `null` on non-OK and when the conversation is deleted.
- **Refresh**: none. Not part of the 20 s poll, not re-read after sending a message, after takeover or after "تم الحل". The card is a snapshot from the moment the conversation was opened; to see what the AI collected since, the agent must click another conversation and come back.
- **Placement**: fifth block of the right-hand panel (under contact data, marketing consent and past orders); inside the drawer below `xl`.
- **Display** (`OrderDraftCard`, customer-panel.tsx:157-198): header `t.inbox.draftTitle` "مسودة طلب" + `StatusBadge` of `state`; progress bar = (4 − missing)/4; rows "المنتج: {productName}", "الكمية: {n}", "الهاتف: ✓ | —", "العنوان: ✓ | —"; footer either `t.inbox.draftMissing` "الحقول الناقصة: …" or `t.inbox.draftReady` "البيانات مكتملة وجاهزة للتأكيد". Green when complete, amber otherwise.
  - Phone and address **values are not shown**, only ✓/—. Not shown at all: customer name on the draft, notes, variant, price/total, per-field confidence.
  - `state` goes through `label()`: `READY`→"جاهزة", `CONFIRMED`→"مؤكد", `CANCELLED`→"ملغى"; **`EMPTY`, `COLLECTING`, `CONFIRMING`, `CONVERTED` have no dictionary entry and are printed in English**.
  - `missingFields` are the server's raw keys — the line reads "الحقول الناقصة: product، quantity، phone، address" (`draft-service.ts:164-167`; a server-side `fieldLabel()` exists but is not used by the UI).
  - Cancelled/converted drafts are displayed the same way (no state filter).
- **Actions**: **none**. No confirm, cancel or edit control exists anywhere in `src/app/(dashboard)` or `src/components`.
- **Endpoints that exist but have no caller in the UI** (verified by grepping every `fetch`/`api(` call): `GET /api/order-drafts/{id}` [`orders.read`]; `POST /api/order-drafts/{id}/confirm` [`orders.write`] — "Human agent confirms the draft → Rules Engine + OrderService create it" (`createOrderFromDraft(source:"HUMAN")`, audit `order_draft.confirmed_by_agent`, 422 `DRAFT_CLOSED` for cancelled/converted); `POST /api/order-drafts/{id}/cancel` [`orders.write`]. There is no PATCH/PUT for a draft and no `POST /api/orders`, so even via API a human cannot correct a draft field or create an order manually.
- Net effect in this tree: the agent can read that a draft is "جاهزة للتأكيد" but has nothing to press; orders only come into existence through the AI pipeline.

Loading/skeleton: list skeleton, thread skeleton, coach skeleton; customer panel has none (stays on the previous customer until the new one arrives).
Permission checks in UI: none — every button is rendered for every role (a VIEWER sees takeover/resolve/delete/composer; a sales/support agent sees "حذف المحادثة"); failures surface as `t.inbox.actionFailed` "تعذّر تنفيذ الإجراء، حاول مجددًا".
Controls (thread open, `xl`): search 1 + chips 5 + selects 4 + load-more 1 + header 5 + log toggle 1 + composer 5 + marketing 2 ≈ **24**, plus one per list row, per failed message and per order link.

## B.4 `/handoffs` — `src/app/(dashboard)/handoffs/page.tsx` (149, client)
Purpose: queue of conversations the AI escalated. Data: `GET /api/handoffs?status=` [`handoffs.read`], max 50, sorted priority desc. No polling.
1. `PageHeader`: `t.pages.handoffsTitle` "التسليم للموظفين" / "محادثات تحتاج تدخلًا بشريًا مع أولويات وأسباب واضحة." (tab label is "التحويلات").
2. Filter card: one `Select` — مفتوحة (default) / مستلَم / تم الحل. No "all", no reason or priority filter.
3. Cards, one per handoff: hand icon · reason (العميل طلب موظفًا / شكوى / ثقة AI منخفضة / فشل AI / طلب محجوب بالقواعد / مشكلة دفع / محاولة اختراق / فشل متكرر / يدوي) · priority badge · line "{date-time} · {assignedTo} · فتح المحادثة" (link → `/conversations?id={conversationId}`) · status badge · actions:
   | Status | Button | Call |
   |---|---|---|
   | OPEN | "استلام" | `POST /api/handoffs/{id} {action:"claim"}` [`handoffs.claim`] |
   | CLAIMED | "إرجاع للقائمة" | `{action:"release"}` |
   | CLAIMED | "حل" (green) | `{action:"resolve"}` [`handoffs.resolve`] — no confirmation |
   - `assignedTo` is printed as stored — a user id (`HumanHandoff.assignedTo String?`), not a name.
   - The card shows **no customer name, channel, last message or waiting time** — the agent must open each conversation to know who it is. (The API also returns `conversation.status`, unused.)
   - The API supports `action:"route"` (ROUND_ROBIN / LEAST_LOADED); no UI.
4. States: 4 skeletons; `ErrorState` "تعذّر تحميل البيانات" + "إعادة المحاولة"; empty "لا توجد تحويلات / ستظهر هنا المحادثات التي تحتاج تدخلًا بشريًا." + `EmptyNext` "افتح الوارد".
Failure toast: "تعذّر تنفيذ الإجراء، حاول مجددًا". Permission checks in UI: none.
Controls: 1 select + (link + 1–2 buttons) per row.
Overlap: the same queue is reachable as inbox chip "تحتاج مراجعة" and the same reasons appear in the inbox banner with different wording (D.5).

## B.5 `/customers` — `src/app/(dashboard)/customers/page.tsx` (176, client)
Purpose: customer directory. Data: `GET /api/customers?q=` [`customers.read`] — **first 25 only, no pagination control**. No polling.
1. `PageHeader` "العملاء" / "جميع عملائك ونشاطهم وطلباتهم في مكان واحد."; in the action slot a badge "{loaded count} العملاء" (max 25, not the real total).
2. Filter card: search "بحث بالاسم أو الهاتف…" (server, 350 ms debounce); status select "الكل — الحالات" + عميل جديد / مهتم / يضع القرار / جاهز للشراء / طلب سابق / عميل متكرر / عرضة للفقد / محظور (**client-side filter over the 25 loaded rows**).
3. `md+` table (`.data-table`): العميل (avatar, name link, first two `#tags`) · الهاتف · الحالة · حسابات القنوات · آخر نشاط. `<md`: cards.
   - Clicking a customer goes to `/conversations?q={phone or name}` — an inbox **search**, not a customer page. There is no customer detail route in this tree.
4. States: `SkeletonTable`; error card + "إعادة المحاولة"; empty "لا يوجد عملاء بعد / سيظهر العملاء هنا بمجرد وصول أول رسالة من قنواتك." + `EmptyNext` "اربط قناة" (the same empty state appears when the status filter simply matches nothing).
Not present: create/edit/merge/block customer, order count or total spent, export. The page description promises "نشاطهم وطلباتهم" but neither is shown.
Controls: 2 + one link per row.

## B.6 `/intelligence` — `src/app/(dashboard)/intelligence/page.tsx` (854, client) — "مختبر التجميع"
Purpose: "collection mode" — a Facebook page is switched so the AI does not reply and only extracts customer data from incoming messages. Root wrapper is `space-y-5 pb-10` (the only page not using `.page-wrap`, so it is not width-capped).
Data: `GET /api/intelligence/overview`, `GET /api/channels` (Facebook only), `GET /api/intelligence/fields`, `GET /api/intelligence/conversations[?channelId]`, `GET /api/intelligence/conversations/{id}` — all on mount/selection, **no polling** (manual "تحديث").
Sections in order:
1. `PageHeader` "مختبر التجميع" / "مراقبة رسائل العملاء الواردة من صفحات فيسبوك وجمع معلوماتهم — بدون أي رد آلي على العميل."; actions: button "تحديث"; link "إدارة القنوات ←" → `/channels`.
2. Red inline error strip.
3. Six `StatCard`s: صفحات في وضع التجميع (hint "الرد الآلي معطّل عليها") · محادثات مستلمة · عملاء جُمعت عنهم معلومات · رسائل تنتظر التحليل · حالات فشل التحليل · آخر رسالة واردة.
4. Page bar (card): chips "كل صفحات التجميع" + one per collection page (or text "ما في صفحات بوضع التجميع بعد."); at the end "تفعيل وضع التجميع لصفحة:" + `Select` "اختر صفحة فيسبوك…" — **choosing an option immediately** sends `PATCH /api/channels/by-id/{id}/send-mode {sendMode:"COLLECTION_ONLY"}` [`channels.connect`] (turns the AI's auto-reply off for that page, no confirmation); button "إرجاع للرد الآلي" (only while a specific page chip is selected) → `{sendMode:"AUTO_REPLY"}`. Notice line: "تم تفعيل وضع التجميع — لن يُرسل أي رد آلي لهذه الصفحة." / "رجعنا الصفحة للرد الآلي." / "ما عندك صلاحية تغيير وضع الصفحة."
5. Collapsible card "البيانات المطلوب تجميعها" (badges "{n} مطلوبة من {m} مفعّلة", "تغييرات غير محفوظة"): intro text; one row per field — enable checkbox, label, type chip (نص / رقم هاتف / رقم / قائمة), "خاص" chip, toggle button "مطلوب" ⇄ "اختياري", trash for custom fields; add row "حقل خاص جديد" (`Input` placeholder "مثال: المقاس · اللون · الشركة") + type `Select` + "إضافة" (max 30 fields); footer status text + "حفظ البيانات المطلوبة" → `PUT /api/intelligence/fields` [`settings.manage`].
6. Two columns:
   - **"المحادثات"** list (count): rows = name ("عميل" fallback), date, page, last inbound text, badges: بانتظار التحليل / جاهز / يحتاج مراجعة / فشل التحليل / "ما تحلّلت بعد", progress "x / y", up to two highlight chips, "مراجَعة". Empty: "لا محادثات بعد / أول ما توصل رسالة على صفحة بوضع التجميع، بتظهر هون." + `EmptyNext` "اربط صفحة".
   - **Detail**: placeholder "اختر محادثة / بتشوف الرسائل الأصلية، ومعلومات العميل المستخرجة مع مصدر كل معلومة."; card 1: name, page, status badge, button "إعادة تحليل" → `POST /api/intelligence/conversations/{id} {action:"reanalyze"}` [`conversations.reply`]; error strip "آخر فشل: …"; read-only message bubbles (sender العميل / الرد الآلي / موظف / النظام, date, raw `deliveryStatus`); card 2 "معلومات العميل المستخرجة" ("آخر تحديث: … · الثقة: n%", badge "مستخرجة بالذكاء"), progress "البيانات المطلوبة: x من y" (مكتمل ✅ / "n ناقص" / "ما في حقول مطلوبة"), one row per enabled field: label, "مطلوب", status badge (غير متوفر / مستخرج ويحتاج مراجعة / مؤكد من موظف / متعارض بين رسالتين), value, "قيم متعارضة: …", "المصدر: …abc123" (last 6 chars of message ids), button "تأكيد" → **`window.prompt("القيمة المؤكدة لـ «label»")`** → `POST … {action:"confirm", key, value}`.
7. `<details>` "أداء التحليل (تفاصيل تقنية)": P50/P95 of queue wait, AI extraction and total time in ms, failures/runs in 24 h, stuck jobs; legend "P50 = الوقت المعتاد · P95 = أبطأ ٥٪ من المرات."
Permission checks in UI: only the 403 text for the mode switch. No link from a row to the real conversation in the inbox; no reply possible here.
Controls: 2 header + chips + mode select + 1 + field panel (3 per field + 3 + save) + reanalyze + one "تأكيد" per field ≈ **25–60** depending on fields.
This is the largest page file and it sits in the daily "المحادثات" tab row next to the inbox although it is a configuration/monitoring tool for a special channel mode.

## B.7 `/orders` — `src/app/(dashboard)/orders/page.tsx` (252, client)
Purpose: order list. Data: `GET /api/orders?status&confirmationStatus` [`orders.read`] — **default limit 25, the UI never sends `cursor`/`limit`**. No polling (despite the description "لحظة بلحظة").
1. `PageHeader` `t.orders.title` "الطلبات" / "تابع حالات الطلبات وتأكيدها لحظة بلحظة."; action "طباعة" (`window.print()` — prints the whole shell; no print stylesheet in `globals.css`).
2. Five `StatCard`s **computed from the loaded rows**: إجمالي الطلبات (= rows, ≤25) · طلبات جديدة (`NEW`) · قيد التأكيد (`CONFIRMATION_PENDING` or confirmation `PENDING`) · مكتملة (`DELIVERED`) · تحتاج مراجعة (`CANCELLED|FAILED|RETURNED`). They change with the filters and are not clickable.
3. Filter card: search "بحث برقم الطلب أو العميل…" (client-side over loaded rows; matches number, name, phone, first product); `Select` "الكل — الحالات" + 9 statuses; `Select` "الكل — حالات التأكيد" + قيد الانتظار / مؤكد من العميل / مؤكد من الموظف / مرفوض.
4. Error card; green notice "تم حذف الطلب {n}"; `SkeletonTable`.
5. `md+` table: رقم الطلب (link → `/orders/{id}`) · العميل · الهاتف · المنتج (first item only) · الكمية (first item) · الإجمالي · الحالة · التاريخ · [trash button, only if `canDelete`]. `<md`: cards (number, status, customer, total, first product, "حذف").
6. Empty: illustration + "لا توجد طلبات بعد / ستظهر الطلبات هنا بمجرد تأكيد أول طلب من المحادثات." + `EmptyNext` "افتح الوارد".
7. Delete confirm: "حذف الطلب {n} نهائيًا؟" / "ينحذف الطلب مع كل بنوده من النظام، وما بيرجع. …الإجراء متاح للمالك فقط." / "حذف نهائي" → `DELETE /api/orders/{id}`.
Permission checks in UI: `canDelete` (API says `role === "OWNER"`).
Not present: create order, pending drafts, date range, assignee, confirmation-status column, bulk actions, export (exists only in `/analytics`), pagination.
Controls: 1 + 3 filters + one link (and trash) per row.

## B.8 `/orders/[id]` — `src/app/(dashboard)/orders/[id]/page.tsx` (299, client)
Purpose: one order — move it through its statuses. Data: `GET /api/orders/{id}` (order + `timeline` from the audit log + `canWrite` + `canDelete`).
1. Back link "كل الطلبات".
2. `PageHeader` "طلب {orderNumber}" + created date-time; action buttons = allowed next statuses, **labelled with the status name** (not a verb), `PATCH /api/orders/{id} {status}` [`orders.write`]:
   | Current | Buttons |
   |---|---|
   | NEW, CONFIRMATION_PENDING | "مؤكد" · "ملغى" |
   | CONFIRMED | "قيد المعالجة" · "ملغى" |
   | PROCESSING | "قيد التوصيل" · "ملغى" |
   | OUT_FOR_DELIVERY | "تم التوصيل" |
   | DELIVERED | "مُرجَع" |
   "ملغى" opens a confirm ("إلغاء الطلب {n}؟" / "سيُلغى الطلب نهائيًا وفق انتقالات الحالة المسموحة. لا يمكن التراجع من هذه الواجهة." / buttons "إلغاء" and "إلغاء الطلب"). Owner only: "حذف الطلب" → confirm → `DELETE` → back to `/orders`. No `FAILED` transition button although the API accepts it. All buttons spin together (shared `saving`).
3. Inline red error card (e.g. "انتقال غير مسموح").
4. Left column: "البنود" (name, SKU, unit × qty, line total; then المجموع الفرعي / الخصم / التوصيل / الإجمالي) · "السجل الزمني" (audit actions through `label()`, e.g. "انعمل الطلب", "تغيّرت الحالة"; or "لا يوجد سجل.").
5. Right column: "العميل" (name, phone, link "افتح المحادثة" → `/conversations?id=` or `?q=`) · "التوصيل" (العنوان, المحافظة, ملاحظات) · "الحالات" (الطلب / التأكيد / الدفع / التنفيذ badges).
States: two skeletons; not-found card "الطلب غير موجود" + "العودة للطلبات".
Permission checks in UI: `canDelete` only. `canWrite` is ignored — a support agent or viewer sees the status buttons and gets an error after clicking.
Not present: editing items, quantity, address, phone, notes (API accepts `notes`), discount, payment status; print/invoice; who changed what in the timeline (actor not shown).
Controls: 1–3 buttons + 2 links.

## B.9 `/order-warehouse` — `page.tsx` (6, server): redirect → `/orders`.

## B.10 `/products` — `src/app/(dashboard)/products/page.tsx` (379, client)
Purpose: product catalogue the AI sells from. Data: `GET /api/products?q&status&category` [`products.read`] — **default 25, no pagination control**. Search debounced 300 ms. No polling.
1. `PageHeader` "المنتجات" / "أدر الكتالوج والأسعار والمخزون والوسائط."; actions "استيراد اكسل" (secondary) · "منتج جديد" (primary).
2. Filter card: search "بحث بالاسم أو SKU أو الوسم…" (this input is `h-12`, taller than every other search box); status `Select` النشطة (default) / غير النشطة / المؤرشفة / الكل.
3. Category chips (`role="tablist"`): "الكل" + the distinct categories **of the products currently loaded** (so after picking a category the other chips disappear).
4. Card grid (1–4 columns): image or placeholder icon; name (link → `/products/{id}`); SKU; a `StatusBadge` whose value is faked from flags — archived → `CLOSED` ("مغلقة"), active → `IN_STOCK` ("متوفر"), inactive → `PENDING` ("قيد الانتظار") (line 234; unrelated to real stock); price + currency; up to two extra price tiers; "{stock label} · {category}"; buttons "تعطيل"/"تفعيل" → `PATCH /api/products/{id} {isActive}` and "أرشفة"/"استعادة" → `PATCH {isArchived}` [`products.write`] (no confirmation, response not checked); round arrow link to the detail page.
5. States: 6 skeletons; error card "حدث خطأ" + "إعادة المحاولة"; empty: illustration + "لا توجد منتجات / أضف أول منتج لبدء البيع عبر المحادثات." + "منتج جديد".
6. Dialog "منتج جديد": اسم المنتج · SKU (required) · السعر · العملة (default **"USD"**) · التصنيف · وصف قصير; "إلغاء" / "إنشاء المنتج" → `POST /api/products`.
7. Dialog "استيراد منتجات من ملف اكسل": explanation; native file input (`.xlsx,.csv`) → `POST /api/products/import`; "جارٍ التحليل والإدراج… ما تغلق الصفحة"; result "ادرجت n منتج — تحليل الأعمدة بالذكاء | تلقائي محلي", "ملاحظة الذكاء: …", created list, skipped rows "صف n · reason"; button "إلغاء" (only closes).
Permission checks in UI: none. No delete (API has no DELETE; archive only). No bulk actions. No stock editing anywhere (see B.11).
Controls: 2 + 2 + chips + 4 per card.

## B.11 `/products/[id]` — `src/app/(dashboard)/products/[id]/page.tsx` (556, client) + `src/components/products/compliance-card.tsx` (170)
Purpose: edit one product. Data: `GET /api/products/{id}`, `GET /api/media?limit=50`.
`PageHeader`: product name / "{sku} · {stock label}"; actions: link "المنتجات" (back) + badge "تم الحفظ" for 2 s after a save. Then the shared `Tabs` component (its only use in the app):
| Tab | Content | Actions |
|---|---|---|
| نظرة عامة | الاسم · SKU · وصف قصير · الوصف الكامل · التصنيف · الوسوم (سطر لكل وسم) | "حفظ" → `PATCH /api/products/{id}` |
| التسعير | note "السعر هنا هو المصدر الوحيد للحقيقة في النظام — الذكاء الاصطناعي لا يحدد الأسعار أبدًا."; السعر · السعر قبل الخصم · العملة; "أسعار إضافية (سعر الجملة · سعر الموزع …)" rows (label + value + "حذف"), "+ سعر إضافي" (max 10); "الخيارات" (variants, read-only) | "حفظ" → `PATCH` incl. `priceTiers` |
| المحتوى البيعي | الفوائد · المكونات · طريقة الاستخدام · التحذيرات · الاعتراضات الشائعة · ملاحظات البيع | "حفظ" |
| الوسائط | "صور المنتج" grid: badge "رئيسية", "تعيين رئيسية" (`PATCH /api/products/{id}/media`), ↑ "تقديم" / ↓ "تأخير" (`PUT …/media {orderedMediaIds}`), "إزالة من المنتج" (`DELETE …/media`); "رفع صورة جديدة" (`POST /api/media/upload`, category PRODUCT); `<details>` "إضافة صورة" → search + thumbnails from the library, click = `POST …/media` | — |
| الأسئلة الشائعة | "أسئلة شائعة (بيانات منظمة — يستخدمها AI مباشرة)": list with pill toggle "نشط"/"معطل" (`PATCH /api/faqs/{id}`); inputs السؤال / الإجابة + "+ إضافة سؤال" (`POST /api/faqs {productId}`) | no edit/delete of an FAQ |
| الامتثال | `ComplianceCard` "الامتثال — قواعد سيستم 1": field "مرجع المنتج بسيستم 1" (hint "مثلًا pf_TIN01 — فاضي = المنتج مش مربوط"), "حفظ وقراءة", "اقرأ الآن" → `GET/POST /api/products/{id}/compliance`; red/orange notes; lists "مسموح نقول", "ممنوع نقول (الرد بينرفض إذا فيه)", "قواعد الإعلان"; meta line "التصنيف · تحديث سيستم 1 · آخر قراءة · آخر خطأ" | refers to a page «الربط مع الأنظمة» that does not exist in this tree |
States: skeletons; not-found "المنتج غير موجود" + "العودة للمنتجات"; inline red error strip (upload errors include raw `[code=…]`).
Gaps verified in code: **no input for stock status or quantity** (`stockStatus`/`stockQty` are in the type and shown read-only), no active/archive switch here, variants not editable, no FAQ editing, each tab saves separately and switching tabs discards unsaved edits (uncontrolled `defaultValue` forms are unmounted). Media/FAQ actions do not check the response.
Controls: 6 tabs + 5–12 per tab.

## B.12 `/catalogs` — `src/app/(dashboard)/catalogs/page.tsx` (249, client)
Purpose: named product sets; each Facebook page answers only from its catalog. Data: `GET /api/catalogs?q&isActive&sort` [`catalogs.read`].
1. `PageHeader` "كتالوجات المنتجات" / "مجموعات المنتجات التي تجيب عنها كل صفحة — تُنشأ هنا وتُربط بالصفحات من هنا أو من صفحة القناة."; action "إنشاء كتالوج جديد".
2. Filter card: "بحث بالاسم…" · status (كل الحالات / نشط / متوقف) · sort (الأحدث تحديثًا / الاسم).
3. Cards: name (link → `/catalogs/{id}`), description, badge "نشط"/"متوقف" (`tone="success"|"muted"` — neither exists in `Badge.TONES`, so both render the same grey), "{n} منتج · {m} صفحات", text links "التفاصيل" · "إيقاف"/"تفعيل" (`PATCH /api/catalogs/{id} {isActive}`) · "حذف" (confirm "حذف «name»؟" / "سيُحذف الكتالوج وروابطه بالصفحات. المنتجات الأصلية والطلبات والعروض لا تُمس أبدًا." / "احذف الكتالوج" → `DELETE`).
4. States: 6 skeletons; `ErrorState` "تعذر التحميل"; empty "لا كتالوجات بعد / أنشئ أول كتالوج واربطه بصفحة ليجيب الذكاء منه." + "كتالوج جديد".
5. Dialog "إنشاء كتالوج جديد": اسم الكتالوج · الوصف (اختياري) · note "بعد الإنشاء: أضف المنتجات واربط الصفحات من صفحة التفاصيل." · "إلغاء" / "إنشاء" → `POST /api/catalogs` [`catalogs.write`].
Feedback via toasts ("أُنشئ الكتالوج", "أُوقف الكتالوج", "حُذف الكتالوج — المنتجات نفسها لم تُمس").
Controls: 1 + 3 + 3 per card.

## B.13 `/catalogs/[id]` — `src/app/(dashboard)/catalogs/[id]/page.tsx` (449, client)
Data: `GET /api/catalogs/{id}`, `GET /api/channels/facebook`.
1. `PageHeader` name / description or "بلا وصف"; action link "← الكتالوجات"; badge نشط/متوقف (same invalid tones).
2. Card: "اسم الكتالوج" · "الوصف" · "احفظ" → `PATCH /api/catalogs/{id}`.
3. Card "المنتجات (n)": "إضافة منتجات" (opens picker); rows "{name}" / "{sku} · {price} · مخزون n | غير متوفر · موقوف" + "إزالة" → confirm → `DELETE /api/catalogs/{id}/products {productId}`; empty "كتالوج فارغ / أضف منتجات موجودة — لن يُنشأ منتج جديد هنا."
4. Card "الصفحات المرتبطة (n)": rows page name / "{externalId} · {raw status e.g. CONNECTED}" + "فك الربط" → confirm → `DELETE /api/catalogs/{id}/pages {channelId}`; empty "بلا صفحات / اربط صفحة ليبدأ الذكاء بالإجابة من هذا الكتالوج."; `Select` "اختر صفحة ماسنجر…" (Facebook pages only, label "{name} · {raw status}") + "ربط الصفحة" → confirm → `POST /api/catalogs/{id}/pages {channelId}` [`catalogs.assign`].
5. Dialog "إضافة منتجات": search box "بحث بالاسم أو SKU…" + button "بحث" (`GET /api/products?q&limit=20` — nothing is listed until "بحث" is pressed; hint "ابحث ثم حدد منتجات متعددة دفعة واحدة."), checkbox rows (image, name, sku · price, "بالكتالوج"), "المحدد: n", "إلغاء" / "أضف المحدد" → `POST /api/catalogs/{id}/products {productIds}`.
6. One shared confirm dialog for all three actions: always red/danger styling and the button text "تأكيد", including for the non-destructive "ربط الصفحة".
States: 2 skeletons; error → `ErrorState` + link back.
Only Facebook pages can be linked (the selector reads `/api/channels/facebook`); WhatsApp/Instagram/Telegram channels cannot get a catalog from the UI.
Controls: ≈8 + one per product/page row.

## B.14 `/campaigns` — `src/app/(dashboard)/campaigns/page.tsx` (385, client) + `segment-dialog.tsx` (149)
Purpose: broadcast a marketing message to a customer segment (WhatsApp template or Telegram). Data: `GET /api/campaigns`, `GET /api/segments` [`campaigns.read`]. No polling (a running campaign's numbers do not move until the card is reopened).
1. `PageHeader` "الحملات التسويقية" / "خطوتين: اختار مين بدك توصله (شريحة)، وبعدها اكتب الرسالة (حملة). ولا شي بينبعت قبل الفحص والاعتماد."; action "شريحة جديدة".
2. Card "١. الشرائح — مين بدك توصله": list of segments (name + badge فعّالة / متوقف مؤقتًا), or "ما في ولا شريحة لسا — ابدأ هون، وبعدها بتقدر تعمل حملة." + "شريحة جديدة". Segments cannot be opened, edited, sized, paused or deleted (API `GET/PATCH/POST /api/segments/{id}` unused).
3. Card "٢. حملة جديدة" (always visible, inline form): "اسم الحملة" · segment `Select` ("— اختر الشريحة —" / "أنشئ شريحة أول ↑") · channel `Select` (واتساب / تيليغرام) · rate "30/دقيقة"; WhatsApp only: "اسم القالب المعتمد في Meta" + "ar"; raw `<textarea>` "نص الرسالة — سيُراجع ويُعتمد قبل الإرسال"; note "المسارات المسموحة رسميًا: واتساب (قالب معتمد) وتيليغرام (مشتركو البوت). فيسبوك/إنستغرام/تيكتوك لا تدعم بثًا تسويقيًا مباشرًا — ممنوعة."; button "إنشاء مسودة" → `POST /api/campaigns` [`campaigns.write`].
4. Campaign cards: name button "{name} · {channel}" (click → `GET /api/campaigns/{id}` and shows the run log inline), status badge (DRAFT "مسودة", VALIDATING "قيد الفحص", READY "جاهزة", SCHEDULED "مجدولة", RUNNING "جارٍ التنفيذ", PAUSED "متوقف مؤقتًا", CANCELLED "ملغى", COMPLETED "مكتمل", FAILED "فشل"), line "مجدولة: … | غير مجدولة · n/دقيقة · سبب الإلغاء: {raw} · سبب الفشل: {raw}", buttons by status:
   | Status | Button | Call |
   |---|---|---|
   | DRAFT | "تحقق" | `POST /api/campaigns/{id}/validate` |
   | READY | "اعتماد المحتوى" | `POST …/approve` [`campaigns.approve`] |
   | READY | "جدولة" | `POST …/schedule` with `{}` — there is no date/time picker |
   | RUNNING | "إيقاف مؤقت" | `POST …/lifecycle {action:"pause"}` |
   | PAUSED | "استئناف" | `{action:"resume"}` |
   | DRAFT…PAUSED | "إلغاء" | confirm "إلغاء الحملة "name"؟" / "سيتوقف الإرسال فورًا…" / "إلغاء الحملة" → `{action:"cancel", reason:"MANUAL"}` |
   | any | "معاينة بلا إرسال" | `POST …/dry-run` → inline "معاينة (بلا إرسال وبلا تغيير أي بيانات): مؤهلون: n · مستبعدون: m · {raw reason key}: k", "ملاحظات: …" or "لا عوائق — جاهزة للجدولة بعد الاعتماد." |
   Run log inline: "سجل التنفيذ (أرقام من قاعدة البيانات): أُرسلت · فشلت · مستبعدون · متبقون · ملغاة" + up to 8 events printed as **raw `type` and `reason`**.
   Errors of "تحقق/اعتماد/جدولة" are written to `formError`, which is rendered inside card 3 ("حملة جديدة"), not next to the campaign that failed.
5. Empty: "لا حملات بعد / أنشئ شريحة ثم حملة — لا يُرسل شيء قبل التحقق والاعتماد والجدولة."
6. `SegmentDialog` "شريحة جديدة — مين بدك توصله؟": "اسم الشريحة"; six preset checkboxes (all must match): "وافقوا يستقبلوا رسائل تسويقية" ✓ default · "طلبوا منك قبل" · "حكوا معك بس ما طلبوا" · "حكوا معك آخر ٧ أيام" · "ما حكوا معك من ٣٠ يوم أو أكتر" · "ما وصلتهم حملة آخر ٧ أيام" ✓ default; "إلغاء" / "حفظ الشريحة" → `POST /api/segments`.
Not present: editing a draft campaign (`PATCH /api/campaigns/{id}` unused), recipients list (`GET …/recipients` unused), AI copy help (`POST /api/campaigns/ai-draft` unused), audience size before creating.
Permission checks in UI: none (a sales agent with `campaigns.read` sees every write button).
Controls: 1 + 6–8 form fields + up to 5 buttons per campaign.

## B.15 `/personas` — `src/app/(dashboard)/personas/page.tsx` (372, client)
Purpose: the AI's voice (tone, dialect, forbidden phrases, system instructions). Data: `GET /api/personas` [`personas.read`].
1. `PageHeader` `t.pages.personasTitle` "شخصية الذكاء الاصطناعي" / "حدّد نبرة المساعد وأسلوبه وحدود ردوده."; action "شخصية جديدة".
2. One card per persona: name, badge "الافتراضية للمساحة" (when `isActive`), "v{version}", summary "{عربي|إنجليزي} — {dialect} · {ودودة|احترافية|مرحة} · {رد قصير|متوسط|طويل} · إيموجي n/5 · {salesStyle as typed, default raw "helpful"}". Buttons:
   | Button | Action |
   |---|---|
   | "تعيين كافتراضية" (green, non-default only) | `PATCH /api/personas {id, isActive:true}` — no confirm, response not checked |
   | "جرّبها" | link → `/ai/playground?persona={id}` |
   | "تعديل" | opens the form dialog |
   | "الإصدارات" | `GET /api/personas/{id}/versions` → shows a versions card under the persona (cannot be collapsed again) |
   | "مسودة من الحالية" | `POST /api/personas/{id}/versions` (body `{}`) — snapshot as a draft version |
3. Versions card: heading "إصدارات {name} — النشر صريح والرجوع متاح (جرّبها بـ«التجربة» قبل النشر)"; rows "v{n} · {مسودة|منشور|مسترجَعة} · {name}"; "نشر" (drafts) → `POST …/versions/{n}/publish`; "رجوع لهذا" (published/rolled-back) → confirm "الرجوع إلى الإصدار v{n} من "{name}"؟" / "سيتغير سلوك المساعد الحي في كل القنوات المرتبطة بهذه الشخصية فورًا…" / "الرجوع لهذا الإصدار" → `POST …/rollback`; empty "لا مسودات بعد — أنشئ مسودة من التعليمات الحالية."; errors "فشل تحميل الإصدارات" / "فشل إنشاء المسودة" / "فشل النشر" / "فشل الرجوع".
4. States: 3 skeletons; `ErrorState`; empty "لا توجد شخصيات بعد / أنشئ شخصية لتحديد نبرة المساعد وأسلوبه." + "شخصية جديدة".
5. Form dialog (title "تعديل" or "شخصية جديدة"): الاسم · اللغة (العربية / English) · اللهجة · النبرة (ودودة / احترافية / مرحة) · الرسمية (غير رسمية / رسمية) · طول الرد (قصير / متوسط / طويل) · مستوى الرموز (0-5) · أسلوب البيع · تعليمات خاصة (حتى 20000 حرف) · عبارات ممنوعة (سطر لكل عبارة) · قواعد إضافية (سطر لكل قاعدة); "إلغاء" / "حفظ" → `POST` or `PATCH /api/personas` [`personas.write`].
Two parallel ways to change behaviour — "تعديل" (immediate save) and the draft → publish → rollback version flow — with no explanation of how they relate. No delete (no API). Which persona a channel uses is set on `/channels` (Facebook pages only).
Controls: 1 + 5 per persona + 2 per version; dialog 11 fields.

## B.16 `/knowledge` — `src/app/(dashboard)/knowledge/page.tsx` (484, client)
Purpose: documents + FAQs the AI answers from. Data: `GET /api/knowledge?status&category&q` (25) + `GET /api/faqs?limit=200` [`knowledge.read`].
`PageHeader` `t.knowledge.title` "قاعدة المعرفة" / "مستندات وأسئلة شائعة يتعلّم منها الذكاء الاصطناعي الرد على عملائك." (tab label "المعرفة"). Then three cards:
1. **Card "إضافة مصدر"** — contains, in this order, the create form, the upload control, the list filters and the list itself:
   - Form: "عنوان المستند" · "التصنيف" (سياسة / توصيل / دفع / عن المتجر / منتج / أسئلة شائعة / مستند / مخصص) · submit button labelled **"من نص"** (`t.knowledge.fromText`) · "نص المستند" → `POST /api/knowledge {sourceType:"MANUAL"}` [`knowledge.write`].
   - "من ملف (PDF/TXT/MD/CSV/JSON):" + label-styled button "رفع ملف" → `POST /api/knowledge/upload`.
   - Filters: "بحث…" · status (كل الحالات / تمت الفهرسة / قيد المعالجة / قيد الرفع / فشل / مؤرشف) · category (كل التصنيفات + 8).
   - Rows: title button (opens chunk viewer) "{category} · إصدار n · m مقطع" + error text; status badge; "إعادة الفهرسة" (`POST /api/knowledge/{id}/reindex`, shown for READY/FAILED); ✎ (edit dialog); "أرشفة" (`PATCH {isActive:false}`, no confirm); 🗑 (confirm → `DELETE`).
   - Empty: "لا توجد مصادر بعد / أضف أول مصدر معرفة ليتعلّم منه الذكاء الاصطناعي." + "أسرع بداية: سؤال شائع" (scrolls to card 3).
2. **Card "اختبر البحث"**: input "مثال: ما سعر المنتج؟" + button "اختبر البحث" → `POST /api/knowledge/search {query, limit:5}`; results "أفضل النتائج" with "الملاءمة 0.123" scores; "لا توجد نتائج مطابقة."
3. **Card "الأسئلة الشائعة العامة"** (`id="faqs"`): inline inputs "السؤال" · "الإجابة" · "التصنيف" + "إضافة" → `POST /api/faqs`; list (question, answer, category badge) with a pill button "نشط"/"معطل" → `PATCH /api/faqs/{id} {isActive}`. No edit or delete of an FAQ.
Dialogs: edit ("تعديل": عنوان المستند · التصنيف · "نص جديد (اتركه فارغًا للإبقاء على النص الحالي — التعديل يعيد الفهرسة)" · "إلغاء"/"حفظ" → `PATCH /api/knowledge/{id}`); delete confirm (title "حذف", body "سيتم حذف المستند وجميع مقاطعه نهائيًا.", button "حذف"); detail (status + category badges, chunks "#1…").
States: 3 skeletons in the list; error card "حدث خطأ". Processing documents do not refresh by themselves (no polling) — a document stays "قيد المعالجة" until the page is reloaded.
Product-specific FAQs live on the product page (B.11), general ones here — two places for the same object.
Controls: ≈14 + 4–5 per document + 1 per FAQ.

## B.17 `/media` — `src/app/(dashboard)/media/page.tsx` (528, client)
Purpose: media library. Data: `GET /api/media?category&type&q` (30, no pagination) [`media.read`], `GET /api/products?limit=100`.
1. `PageHeader` `t.media.title` "الوسائط" / "صور المنتجات والفيديوهات والمرفقات في مكان واحد."
2. **Red + orange notice** (`RiskWithFix`) at the top of the page: "مؤتمت: وسائط الزبون الواردة بتتحذف نهائيًا بعد 180 يوم" + long explanation + fix text naming the server setting `WA_MEDIA_RETENTION_DAYS`.
3. Upload card: drag-and-drop zone "اسحب الملفات وأفلتها هنا أو اختر من جهازك" + "رفع ملفات" (multi-file; XHR `POST /api/media/upload` [`media.write`]); per-file progress, "تم رفع الملف", errors "فشل الرفع: {message} [code=…]" + "إعادة المحاولة". The category sent is the current filter value (default `PRODUCT`) — there is no category choice at upload.
4. Filter card: search · category (كل التصنيفات / منتج / رأي عميل / شهادة / هوية المتجر / تعليمي / أخرى) · type chips (الكل / الصور / الفيديو / الصوت / الملفات) · grid/list toggle (aria "شبكة"/"قائمة").
5. Grid cards: thumbnail (opens preview), title, "size · date", WhatsApp warning "⚠️ ما بتنبعت عالواتساب: الصيغة أو الحجم / 🔧 ارفعها JPG أو PNG أقل من 5MB", product badges or "غير مرتبط", text links "معاينة" · "ربط بمنتج" · "حذف". List rows: thumbnail, title, "mime · size · date", "المنتج المرتبط: …", links "تنزيل/فتح" · "ربط بمنتج" · "حذف". (Raw type names such as `VIDEO`/`DOCUMENT` are printed on non-image thumbnails.)
6. Empty: illustration + "المكتبة فارغة / ارفع أول ملف لاستخدامه في المنتجات والمحادثات." + "رفع صور منتجاتك".
7. Dialogs: preview (image/video/audio player or download block; buttons "تنزيل/فتح", "ربط بمنتج", "إزالة الربط: {product}"); link dialog "ربط بمنتج" (product `Select`, "إلغاء" / "ربط بمنتج" → `POST /api/products/{id}/media`); delete confirm "حذف الملف؟" / "سيتم حذف الملف نهائيًا من مكتبة الوسائط. لن يؤدي حذف الملف إلى حذف المنتج." / "حذف الملف" → `DELETE /api/media/{id}`.
Failures of delete/link/unlink all display `t.media.uploadFailed` "فشل الرفع" (lines 181, 199, 210). No rename or re-categorise (API `PATCH /api/media/{id}` unused).
Controls: 2 + ≈9 filters/toggles + 3 per item.

## B.18 `/feedback` — `src/app/(dashboard)/feedback/page.tsx` (260, client)
Purpose: customer reviews the AI may quote once approved. Data: `GET /api/feedback?productId&sentiment&approved&q` (25) [`feedback.read`], `GET /api/products?limit=100`.
1. `PageHeader` "آراء العملاء" / "قيّم رضا العملاء واعتمد الشهادات المميزة."; action "إضافة رأي".
2. Filter card: "بحث في النص…" · product ("كل المنتجات") · sentiment ("كل المشاعر" / إيجابي / سلبي / محايد) · approval (الكل / معتمد فقط / غير معتمد).
3. Cards: optional image; badges "معتمد" / "بانتظار الاعتماد", sentiment, ★ rating, product, source; text; buttons "اعتماد" (green) / "إلغاء الاعتماد" → `PATCH /api/feedback/{id} {isApproved}` [`feedback.write`]; "أرشفة" → confirm "أرشفة هذا الرأي؟" / "سيختفي من قوائم الآراء المعروضة. يمكن الرجوع عنه من قاعدة البيانات فقط." / "أرشفة" → `PATCH {isArchived:true}`.
4. States: 4 skeletons; error card; empty "لا توجد آراء / أضف أول رأي لعميل لعرضه هنا." + "إضافة رأي".
5. Dialog "إضافة رأي": "نص الرأي" (hint "يُحفظ كما ورد حرفيًا — لا يُنسب لشخص لم يقله.") · "المنتج" · "المصدر" (placeholder "مثال: محادثة رقم 12") · "التقييم (1-5)"; "إلغاء" / "إضافة" → `POST /api/feedback {kind:"TEXT"}`.
No image attach here (review images are media items of category FEEDBACK/TESTIMONIAL uploaded on `/media`) — two screens for one concept. No edit.
Controls: 1 + 4 + 2 per item.

## B.19 `/ai/playground` — `src/app/(dashboard)/ai/playground/page.tsx` (429, client)
Purpose: chat with the AI in a sandbox. Data on mount: `GET /api/personas`, `GET /api/channels/facebook`, `GET /api/products?limit=100`, `GET /api/ai/playground/sessions` [`ai.configure`]. No polling.
1. `PageHeader` `t.playground.title` "تجربة الذكاء الاصطناعي" / "اختبر ردود الذكاء الاصطناعي قبل استخدامها مع العملاء."; amber strip "جلسة داخلية — لا تصل إلى العملاء ولا إلى فيسبوك."
2. Left column, card 1 (settings — 9 controls): "الشخصية" `Select` ("الافتراضية (حسب الصفحة)") · "صفحة Facebook" `Select` ("بدون صفحة (الوضع الافتراضي)"; options "{page} · {raw status}") · "النموذج" `Input` (placeholder "الافتراضي حسب الإعدادات") · checkbox "استخدام قاعدة المعرفة" · "اختيار منتج" `Select` ("بدون منتج") · "سياق العميل" `Input` ("اسم تجريبي (اختياري)") · "المهمة (نموذجها الخاص)" `Select` (توليد الرد / استخراج البيانات / تلخيص المحادثة) · "تعليمات مسودة للاختبار (لا تُنشر)" `Input` · checkbox "اختبار استخراج الطلب"; when settings changed mid-chat: button "بدء محادثة بالإعدادات الجديدة".
3. Left column, card 2 "الجلسات": "محادثة جديدة" → `POST /api/ai/playground/sessions`; "مسح المحادثة" → `DELETE …/sessions/{id}` then a new session (no confirm); list of buttons "جلسة n · date" → `GET …/sessions/{id}`; empty "لا توجد جلسات بعد. ابدأ محادثة جديدة."
4. Chat card: header "اختبار الرد" + "الشخصية المستخدمة: {name}"; empty state repeats title + subtitle; bubbles — the tester's messages use the `bubble-in` style but are aligned to the end, the assistant's use `bubble-out` aligned to the start (the opposite arrangement to the inbox, B.3.3); under an answer: "المصادر المستخدمة" badges (title · score) and, if requested, "مقترح الطلب (تجريبي — لن يُنشأ طلب حقيقي)" as a **raw JSON `<pre>`**; error strip "تعذر تشغيل الذكاء الاصطناعي" + "إعادة المحاولة"; toggle "عرض تفاصيل الاختبار" / "إخفاء التفاصيل" (المزود · النموذج · الشخصية · زمن الاستجابة ms · الرموز a+b · المعرفة المستخدمة نعم/لا); composer textarea "اكتب رسالتك..." + send → `POST …/sessions/{id}/messages {text, withOrderProposal, customerName, task, draftInstructions}`.
Controls: 9 + 2–3 + sessions + 3 ≈ 16+.
Notes: the playground can only simulate Facebook pages (channel list is `/api/channels/facebook`); the model/task/"draft instructions" fields are developer-level options on a page the setup checklist sends every owner to.

## B.20 `/ai/studio` — `page.tsx` (9, server): redirect → `/settings/ai`.

## B.21 `/settings/ai` — `src/app/(dashboard)/settings/ai/page.tsx` (543, client) + `ai-status-card.tsx` (189)
Purpose: switch the AI on, choose the provider. (Tab label "التشغيل"; H1 `t.pages.aiSettingsTitle` "تشغيل الوكيل" / "اربط مزوّد الذكاء، جرّب، وبعدها شغّل الرد الآلي.") Data: `GET /api/ai/settings`, `GET /api/ai/status` [`ai.configure`].
The whole page is **one `<form>`** (`max-w-4xl`) saved by a single button at the bottom.
1. `AiStatusCard`: badge headline + hint, one of — "الذكاء متوقف / ما في ولا اقتراح ولا رد. فعّل «تشغيل الذكاء» تحت." · "ناقص مزوّد ذكاء / اختر مزوّد واكتب مفتاحه بقسم «الاتصال»، وبعدها «فحص الاتصال»." · "الذكاء شغّال — الرد الآلي مطفي / الوكيل بيقترح والموظف بيبعت. جرّبه، وبعدها فعّل «الرد الآلي»." · "الوكيل بيرد على العملاء لحاله / راقب أول المحادثات من «الوارد»."; icon button (aria "تحديث الحالة"); button-link "جرّب الوكيل" → `/ai/playground`; four tiles: "المزوّد" ({label} · يعمل / مهيأ / يعمل محليًا / يحتاج مفتاح / يحتاج ضبط على الخادم / فشل الاتصال / غير مهيأ / متوقف), "الشخصيات" (link `/personas`), "مستندات المعرفة" (link `/knowledge`; "n قيد التجهيز · m فشلت"), "أسئلة شائعة" (link `/knowledge`); red list "نموذج «task» ما بيناسب مهمته: … — عدّله من «متقدّم» تحت."; "قبل التشغيل: اضبط شخصية ← · أضف أسئلة شائعة ←". Error: "تعذّر قراءة حالة الوكيل." + "إعادة المحاولة".
   (The headline says "الوكيل بيقترح والموظف بيبعت" for AI-on/auto-reply-off, but no screen shows AI-suggested replies for approval — the only suggestions UI is the manual "مساعد البيع" dialog.)
2. Section "التشغيل" — "قرارين بس: الذكاء شغّال؟ وبيرد على العميل لحاله ولا بيستنّى موظف؟": two checkbox cards — "تشغيل الذكاء" ("بيقرأ الرسائل، بيقترح ردود، وبيجمع بيانات الطلب.") and "الرد الآلي" ("الوكيل بيبعت رده للعميل لحاله بلا ما يستنّى موظف. جرّبه بـ«التجربة» قبل."; disabled while AI is off). **They only change form state; nothing is saved until "حفظ الإعدادات" at the bottom** and there is no unsaved-changes warning.
3. Section "الاتصال" (action button "تعبئة GLM" = fills provider `custom`, URL `https://glm.erv.zakiai.io/v1` and five `auto/*` model names): "المزوّد" `Select` (مزوّد مخصص (أي API) / الوكيل الذكي (سيرفرك) / OpenAI / OpenRouter / داخلي / وهمي (تجريبي)); then by provider: custom → "عنوان الـAPI" + "المفتاح" (`PasswordInput kind="secret"`, placeholder "sk-…"); internal/hermes → "عنوان الاتصال (اختياري)"; openai/openrouter/mock → note "مفتاح هذا المزوّد بينضبط من بيئة الخادم — ما بينكتب هنا."; button "فحص الاتصال" → `POST /api/ai/providers/test {provider}` → badge "متصل · 123ms" or "ناقص مفتاح" / "ناقص عنوان أو مفتاح" / "راجع العنوان والمفتاح" / "تعذّر الوصول للخادم"; note "الفحص بيستخدم الإعدادات المحفوظة." (so a newly typed URL/key must be saved first, with the save button at the bottom of the page).
4. `<details>` "متقدّم — النماذج وأسلوب التوليد والبديل": six model inputs (فهم النية · استخراج البيانات · كتابة الرد · قراءة الصور · تلخيص المحادثة · فهرسة المعرفة; placeholders `auto/cheap`, `auto/fast`, `auto/best-chat`, `auto/best-vision`, "اتركه فارغاً — محلي"); "درجة الإبداع" (0–2) · "أقصى طول للرد" ("بالتوكنز — كل ١٠٠ ≈ ٧٠ كلمة"); "بديل عند الفشل": "مزوّد البديل" (`Select`, "بلا بديل") · "نموذج البديل".
5. Red validation strip ("عنوان الـAPI مطلوب للمزوّد المخصص", "مفتاح الـAPI مطلوب للمزوّد المخصص", "أقصى عدد توكنز لازم 50 أو أكثر", server message).
6. Save bar: badge "تم الحفظ" (2 s) + "حفظ الإعدادات" → `PATCH /api/ai/settings`.
States: skeletons; `ErrorState` on load failure.
Controls: 2 + 2 toggles + 5–7 connection + 10 advanced + save ≈ **22**.

## B.22 `/automation` — `src/app/(dashboard)/automation/page.tsx` (358, client) + `quick-rule-dialog.tsx` (103) + `workflow-builder-dialog.tsx` (229)
Purpose: "when X happens do Y" rules. Data: `GET /api/workflows` [`automation.read`] (one table holds both rule kinds). No polling.
1. `PageHeader` "القواعد التلقائية" / "لما يصير شي (طلب جديد، رسالة، تأكيد…) النظام بيعمل خطوة لحاله: يبعت رسالة، يجدول متابعة، يحط وسم أو يسند لموظف."; actions "قاعدة سريعة" (primary) · "قاعدة بخطوات" (secondary).
2. Rule cards. The clickable left part (title attr "سجل التنفيذ") opens the execution log (`GET /api/workflows/{id}`): icon (⚡ quick / branch = steps), name, state badge — "مسودة — ما بتشتغل" / "شغّالة" / "موقوفة" —, line "{trigger} ← {step ثم step} · n تنفيذ · آخر مرة: {status}", description. Buttons depend on the kind:
   | Kind / state | Buttons | Call |
   |---|---|---|
   | Steps rule, any | "محاكاة" | `POST /api/workflows/{id}/preview {sample:{}}` |
   | Steps rule, not running | "تشغيل" | `POST /api/workflows/{id} {action:"publish"}` |
   | Steps rule, running | "إيقاف مؤقت" · "إيقاف وإلغاء الجاري" | `{action:"pause"}` · `{action:"stop"}` |
   | Quick rule | "إيقاف" / "تشغيل" | `POST /api/automations/{id} {action:"disable"|"enable"}` |
   | Any, not running | "حذف" | confirm "حذف القاعدة "name"؟" / "ما رح تشتغل بعد هيك. سجل التنفيذ السابق بيضل محفوظ." / "حذف نهائي" → `DELETE /api/automations/{id}` |
   Footer text: "الرسائل المجدولة الي بتعملها القواعد بتلاقيها بـ المتابعات."
3. States: 3 skeletons; `ErrorState`; empty "ولا قاعدة لسا / ابدأ بقاعدة سريعة: مثلًا رسالة «كيف كان طلبك؟» بعد ٢٤ ساعة من التأكيد." + "قاعدة سريعة".
4. Dialog "قاعدة سريعة — رسالة متابعة تلقائية": "اسم القاعدة" · "لما يصير" (عند إنشاء طلب / عند تأكيد طلب / عند إلغاء طلب / عند تسليم الطلب / عند إنشاء محادثة / عند إنشاء عميل) · "بس إذا المحافظة" · "بس إذا مجموع الطلب على الأقل" · "نص رسالة المتابعة" (default "مرحبًا {{customer.name}}، نود التأكد من أن كل شيء على ما يرام مع طلبك.") · "ابعتها بعد (ساعات)" (24); "إلغاء" / "إنشاء وتشغيل" → `POST /api/automations` [`automation.write`] (enabled immediately).
5. Dialog "قاعدة بخطوات (بتنحفظ مسودة)": "الاسم" · "لما يصير" (10 triggers) · "وصف قصير (اختياري)"; box "الشروط (اختياري — بلا شروط = كل مرة)" with "+ شرط" rows: field `Select` (حالة المحادثة / حالة الذكاء بالمحادثة / القناة / وسوم العميل / العميل محظور / درجة اهتمام العميل / حالة الطلب / مجموع الطلب / دقائق من آخر رسالة للعميل), operator `Select` (12: يساوي … غير موجود), free-text value (no value picker — for status/AI-state/channel fields the stored codes such as `OPEN`, `HUMAN_ACTIVE`, `WHATSAPP` have to be typed by hand; server-side matching rules UNVERIFIED), trash; box "الخطوات (بالترتيب)" with "+ خطوة" rows: type `Select` (إرسال رسالة / ملاحظة داخلية / جدولة متابعة / إلغاء المتابعات / إضافة وسم / إزالة وسم / إسناد المحادثة لموظف / تشغيل/إيقاف الذكاء بالمحادثة / انتظار (دقائق) / إنهاء القاعدة) + type-specific fields ("نص الرسالة", "بعد (ساعات)", "الوسم", **"معرّف الموظف" — a raw user id, hint "من صفحة «الفريق»"; the team page never displays ids**, "نص الملاحظة", "مدة الانتظار (دقائق)", "الحالة الجديدة": الذكاء نشط / الذكاء موقوف); "إلغاء" / "حفظ كمسودة" → `POST /api/workflows`.
6. Dialog "سجل التنفيذ — {name}": status badge, "خطوة n", raw error text (LTR), date; empty "ما اشتغلت ولا مرة لسا."
7. Dialog "محاكاة — بدون ما يتبعت أو يتغيّر شي": "الشروط متحققة: نعم/لا" + numbered steps with badge سينفَّذ / سيدخل انتظارًا / سيتجاوز / فرع متّبع …
No edit of an existing rule (`PATCH /api/automations/{id}` and `PATCH /api/workflows/{id}` unused) — delete and recreate. Permission checks in UI: none.
Controls: 2 + 2–4 per rule; builder dialog 5 + 4 per condition + 3–4 per step.

## B.23 `/workflows` — `page.tsx` (9, server): redirect → `/automation`.

## B.24 `/follow-ups` — `src/app/(dashboard)/follow-ups/page.tsx` (139, client)
Purpose: scheduled messages created by rules. Data: `GET /api/follow-ups?status=` (50) [`followups.read`]. No polling.
1. `PageHeader` "المتابعات" / "رسائل مجدولة لإعادة إشراك العملاء وإتمام الطلبات."
2. Filter: `Select` "الكل — الحالات" / مجدولة / أُرسلت / فشل / ملغى / متجاوزة / جارٍ التنفيذ.
3. Cards: clock icon, message text (or the raw type, e.g. `CHECK_IN`, when empty), "مجدولة: {date-time} · محاولات: n · {lastError}", status badge, for SCHEDULED a red text button "إلغاء" → confirm "إلغاء المتابعة المجدولة؟" / "لن تُرسل هذه الرسالة المجدولة إلى العميل. لا يمكن التراجع." with **both dialog buttons labelled "إلغاء"** (`confirmLabel={t.common.cancel}`) → `POST /api/follow-ups/{id}/cancel` [`followups.cancel`].
4. States: 4 skeletons; `ErrorState`; empty "لا توجد متابعات / ستظهر هنا الرسائل المجدولة لعملائك." + `EmptyNext` "اضبط قاعدة أتمتة".
The row shows neither the customer nor the conversation nor the rule that created it (the page guide says "متى رح تنبعت ولمين"). Nothing can be created or edited here.
Controls: 1 + 1 per scheduled row.

## B.25 `/analytics` — `src/app/(dashboard)/analytics/page.tsx` (613, client) — "التقارير"
Purpose: all reports in tabs. No polling; manual "تحديث".
1. `PageHeader` "التقارير" / "أرقامك الفعلية حسب الفترة — والي ما انسجّل بيبيّن «غير متاح» مش صفر."
2. Filter card: period `Select` (same 4 options) · timezone `Select` (default UTC) · raw checkbox "مقارنة مع الفترة السابقة" (first tab only) · "تحديث" · "تصدير Excel (CSV)" → `GET /api/analytics/export?type&range&timezone` (file `analytics-{type}.csv`; hidden on tabs without a matching export: الأتمتة, الاتجاهات).
3. Tab pills (hand-rolled `role="tablist"`, aria "أقسام التحليلات"; tab kept in `?tab=`):
   | Tab | Endpoints (`GET /api/analytics/…`) | Content |
   |---|---|---|
   | نظرة عامة (`exec`) | overview, funnel, sales (+ overview for the previous window when comparing) | "مؤشرات تنفيذية": محادثات جديدة · عملاء جدد · طلبات مؤكدة · قيمة مؤكدة · رسائل واردة · ردود الذكاء · تحويلات للموظف · تنفيذات أتمتة (± %); funnel: محادثات → مسودات طلب → مسودات جاهزة → طلبات مؤكدة → طلبات مسلَّمة; note "المسودة مش بيع…" |
   | المحادثات | conversations, handoffs | محادثات جديدة · رسائل واردة (ليست محادثات) · رسائل صادرة · أول رد — متوسط (د) · أول رد — وسيط (د) · زمن الحل — وسيط (د); bars "التحويلات للموظف حسب السبب" (**raw reason keys**) |
   | المبيعات | orders, sales | طلبات منشأة (ليست مبيعات مؤكدة) · طلبات مؤكدة · مسلَّمة · ملغاة · قيمة مؤكدة · متوسط قيمة الطلب المؤكد; bars "العروض والمنتجات المكمّلة"; "نسبة قبول العروض (من الي ظهرت)"; link-button "أداء كل منتج ←" (switches tab) |
   | المنتجات | products | sort `Select` (الكميات / الإيرادات المؤكدة / الطلبات المؤكدة); table المنتج (link, 🏆 on first) · الكميات · طلبات مؤكدة · إيرادات مؤكدة · طلبات مُسلّمة · إيرادات مُسلّمة; empty "لا بيانات منتجات بهالفترة" + `EmptyNext` "شوف الطلبات" |
   | الموظفون | agents [`analytics.agents`] | one block of 8 numbers per agent (محادثات معينة · ردود صادرة · أول رد — وسيط (د) · طلبات مرتبطة · طلبات مؤكدة مرتبطة · قيمتها · تحويلات موظف · خصومات معتمدة) — **the block has no agent name** (the API row only carries `agentId` and the UI uses it as a React key); server-supplied note; 403 → "مش متاحة لدورك / أداء الموظفين بيشوفه المدير والمالك بس." |
   | الذكاء | ai | استدعاءات · ناجحة · فاشلة · متوسط سرعة الرد (ملّي ثانية) · توكنز داخلة · توكنز طالعة · التكلفة (معروفة فقط); bars "حسب المهمة" and "حسب النموذج" (raw keys) |
   | الحملات | campaigns | bars "حالة الحملات (الكل)", "المستلمون حسب الحالة", "أسباب الاستبعاد" (raw keys); مستلمون (بدون تكرار) · انبعتت للمنصّة · تسليم مؤكد من المزود |
   | الأتمتة | automations, follow-ups | per rule: مكتملة · فاشلة · نسبة النجاح; bars "المتابعات حسب النوع" |
   | الاتجاهات | trends | bars "محادثات كل يوم" or empty "بيانات الاتجاهات غير مبنية بعد"; "آخر بناء: {raw ISO timestamp} · أيام مفقودة: n" |
4. States: error card + "إعادة المحاولة"; 8 skeleton tiles on every load.
Everything is numbers in grey tiles plus horizontal bars drawn with `<span>`s — no chart library, no time-series line, no currency on money values.
Permission checks in UI: the per-endpoint 403 handling (A.3). Controls: 2 selects + checkbox + 2 buttons + 9 tabs ≈ 14.
Unused endpoints: `/api/analytics/queue`, `/sla`, `/usage`, `POST /rebuild`.

## B.26 `/analytics/ai`, `/analytics/products` — 6-line redirects → `/analytics?tab=ai` / `?tab=products`.

## B.27 `/channels` — `src/app/(dashboard)/channels/page.tsx` (310, client) + `multi-channel-section.tsx` (195) + `catalog-page-section.tsx` (142)
Purpose: connect messaging channels; per Facebook page choose persona and catalog. Data: `GET /api/channels/facebook` [`channels.read`], `GET /api/personas`, `GET /api/channels`, per page `GET /api/channels/by-id/{id}/catalog` + `GET /api/catalogs?sort=name`. No polling.
Order on screen:
1. `PageHeader` "القنوات" / "اربط صفحات فيسبوك وأدر حالة الاتصال."
2. Message strip (success = brand colour, error = red): "تم ربط الصفحة بنجاح." · "فشل ربط الصفحة — حاول مجددًا." · "رمز اختيار الصفحة غير صالح." · "تمت إزالة ربط الصفحة." · "تم حفظ شخصية الذكاء الاصطناعي للصفحة." · "الاتصال سليم." · "فشل اختبار الاتصال — قد يكون التوكن منتهيًا أو الصفحة غير متاحة." …
3. (After the OAuth round-trip, `?fb_pages=`) card "اختر الصفحة المرتبطة": one button per page "{name} ({id})" → `POST /api/channels/facebook/select-page {selectionToken, pageId}` [`channels.connect`].
4. Card "Facebook Messenger" / "ربط صفحة فيسبوك رسميًا عبر Meta Graph API." + link styled as a button "ربط صفحة Facebook" → `GET /api/channels/facebook/connect` (OAuth redirect). If OAuth is not configured the button is replaced by "META_APP_ID و META_APP_SECRET غير مضبوطين على الخادم — أضفهما لتفعيل الربط."
5. `MultiChannelSection` card "قنوات إضافية" / "ربط رسمي لكل قناة — بلا جلسات متصفح ولا تحايل.": four rows —
   | Row | Hint | Form fields ("ربط {name}" toggles the form) |
   |---|---|---|
   | إنستغرام | حساب احترافي + توكن صفحة بصلاحيات المراسلة. | "معرّف حساب إنستغرام (Instagram ID)" · "توكن الصفحة (Page Access Token)" |
   | تيليغرام | توكن بوت من BotFather. يُولَّد سرّ webhook تلقائيًا. | "توكن البوت (Bot Token من BotFather)" |
   | واتساب | رقم تجاري + System User Token. الرد الحر داخل 24 ساعة فقط. | "معرّف رقم واتساب (Phone Number ID)" · "توكن مستخدم النظام (System User Token)" |
   | تيك توك | يتطلب اعتماد Business Messaging — الربط مغلق حتى الاعتماد. | no button |
   Submit "تحقق واربط" → `POST /api/channels/{TYPE}` [`channels.connect`]; the result message may read "… Webhook: {url}" (the URL the owner must paste into Meta/Telegram is shown only as transient text). Connected rows: "{name} {externalId}" · "آخر إرسال: date" / "بلا إرسال بعد" · "فصل" → **`window.confirm("فصل هذه القناة؟ ستتوقف الرسائل الجديدة فورًا.")`** → `DELETE /api/channels/{TYPE}`.
6. One card per connected Facebook page (rendered **after** the "قنوات إضافية" card): page name, page id, status badge; "آخر Webhook: … · آخر إرسال: … · حالة التوكن: {raw VALID|EXPIRED|LONG_LIVED} · رُبطت: …"; row "شخصية الذكاء الاصطناعي" + raw `<select>` ("الافتراضية للمساحة" + personas) + "حفظ" → `PATCH /api/channels/facebook {pageId, aiPersonaId}` [`ai.configure`]; "اختبار الاتصال" → `PUT /api/channels/facebook/disconnect {pageId}`; "إزالة الربط" → confirm "إزالة ربط الصفحة؟" / "سيتم إيقاف اتصال هذه الصفحة بالنظام…" / "إزالة الربط" → `POST /api/channels/facebook/disconnect`; then `CatalogPageSection` "كتالوج المنتجات": "النشط: {name} · n منتج متاح للرد" + links "إدارة المنتجات" and "معاينة الكتالوج" (**both go to the same URL** `/catalogs/{id}`), or amber "«page» بلا كتالوج نشط — الذكاء لا يستخدم أي منتج لها حتى تُسند." / "لا كتالوجات بعد — أنشئ واحدًا من «كتالوجات المنتجات» ثم اربطه هنا."; raw `<select>` "تغيير الكتالوج…" + "تبديل" → confirm "تبديل كتالوج الصفحة؟" / … / "بدّل الكتالوج" → `POST /api/catalogs/{id}/pages {channelId}`.
7. Empty: "ما في صفحة فيسبوك مربوطة / اربطها من زر «ربط صفحة Facebook» فوق — أو اربط واتساب أو تيليغرام أو إنستغرام من «قنوات إضافية»." + the connect link.
States: one skeleton; `ErrorState`.
Asymmetry verified in code: persona, catalog, "test connection" and collection mode exist **only for Facebook pages**; Instagram/WhatsApp/Telegram rows have connect/disconnect only (API `PATCH /api/channels/by-id/{id}/persona` exists, unused). The page's send mode (auto-reply vs. collection) is not shown here at all — it is changed on `/intelligence`.
Controls: 1 + 3 + per Facebook page 6 + per other channel 1–4.

## B.28 `/settings` — `page.tsx` (9, server): redirect → `/settings/account`.

## B.29 `/settings/team` — `settings/team/page.tsx` (7, **server**, `enforcePermission("users.manage")`) + `team-view.tsx` (378, client)
Purpose: members, roles, workspace name. Data: `GET /api/workspaces/members` [`users.manage`].
1. `PageHeader` "الفريق والصلاحيات" / "الأعضاء، الدعوات، الأدوار، والإيقاف — كل عملية إدارية مسجلة بسجل التدقيق."; action "دعوة عضو".
2. Red error card (API message).
3. Table (`.data-table`): العضو (name, email, "تنتهي الدعوة {date}") · الدور (`Select`: مالك / مدير عام / مشرف / موظف مبيعات / موظف دعم / مشاهد — **changing it saves immediately**, `PATCH /api/workspaces/members/{id} {role}`, no confirm) · الحالة (badge نشط/مدعو/موقوف + `Select` نشط/موقوف; disabled for invited; "موقوف" asks for confirmation) · إجراءات: "كلمة المرور" · "إزالة". Empty "لا أعضاء / ادعُ أول عضو في فريقك." + "ادعُ عضو".
4. Collapsible "شو بيقدر كل دور؟": "للقراءة بس — مأخوذ من صلاحيات النظام الفعلية. اختيار الدور بيتم من القوائم فوق." + six one-line role descriptions + matrix table (22 capabilities × 6 roles, ✓ / –).
5. Card "اسم مساحة العمل": "الاسم الجديد" (empty; the current name is not shown) + "حفظ" → `PUT /api/workspaces {name}` [`workspace.manage`]; message "تم حفظ اسم مساحة العمل".
6. Dialog (title "دعوة عضو" / "إنشاء حساب جديد") with a two-button segmented switch "دعوة حساب مسجل" | "إنشاء حساب جديد": ("الاسم") · "البريد الإلكتروني" (hint "يجب أن يكون مسجلًا مسبقًا في المنصة." / "يُفتح الحساب فورًا بهذا البريد وكلمة المرور.") · ("كلمة المرور", plain text by design) · "الدور" with the role blurb as hint; submit "إرسال الدعوة (صالحة 7 أيام)" / "إنشاء الحساب وتنشيط العضوية" → `POST /api/workspaces/members`.
7. Dialog "كلمة مرور جديدة": "لكلمة مرور جديدة لـ {name} ({email}). أعلمها بها بصوتٍ آمن مباشرة — لا ترسلها برسالة." + field + "تعيين كلمة المرور" → `PATCH … {newPassword}`.
8. Confirms: "إزالة العضو "name"؟" / "إزالة العضو" → `DELETE /api/workspaces/members/{id}`; "إيقاف وصول "name"؟" / "إيقاف العضو" → `PATCH {status:"SUSPENDED"}`.
Dead end verified in code: an invitation ("دعوة حساب مسجل") creates an `INVITED` membership; the only way to accept is `POST /api/workspaces/members/accept {workspaceId}` and **no screen calls it** (login only loads `ACTIVE` memberships), so an invited user has no UI to join. Per-member permission overrides are accepted by the API (`permissions[]`) but have no UI.
Controls: 1 + 4 per member + 2.

## B.30 `/settings/account` — `src/app/(dashboard)/settings/account/page.tsx` (188, client)
Purpose: own profile and appearance. Data: `GET /api/auth/me`.
1. `PageHeader` "حسابي" / "اسمك، بريدك، وكلمة مرورك. تغيير البريد أو كلمة المرور يتطلب كلمة المرور الحالية."
2. Card "المظهر" (`ThemePicker`): three theme cards (radio) — "فلوينت", "إسلامي", "المدينة الليلية" (chip "ليلي دائمًا") with colour swatches and descriptions; segmented "فاتح" / "داكن" (light disabled under the night theme, with an explanation). Stored in `localStorage` per device.
3. Card "الاسم والبريد": form "الاسم" + "حفظ الاسم" → `PATCH /api/auth/me {name}`; form "البريد الإلكتروني" (hint "الدخول للمنصة يصبح بهذا البريد.") + "كلمة المرور الحالية" + "تغيير البريد" → `PATCH {email, currentPassword}`.
4. Card "كلمة المرور": "كلمة المرور الحالية" · "كلمة المرور الجديدة" (hint "8 أحرف على الأقل.") + "تغيير كلمة المرور" → `PATCH {newPassword, currentPassword}`.
Result text under each form ("تم حفظ الاسم" …) — the ✓ icon is shown for errors too (`Msg`, line 31-38). No toast. No logout button, no language, no notification preferences, no workspace switcher.
Controls: 5 (theme) + 3 forms (7 fields, 3 buttons) ≈ 15.

## B.31 Route summary (audited tree)
| Route | Page file lines (+ companions) | Type | In nav | Polling |
|---|---|---|---|---|
| `/` | 8 | server redirect | – | – |
| `/login`, `/register` | 5 + 172 | client form | – | – |
| `/dashboard` | 205 (+191 checklist) | client | الرئيسية | none |
| `/conversations` | 5 + 909 + 207 + 149 + 110 + 128 + 142 | client | المحادثات › الوارد | **20 s** |
| `/handoffs` | 149 | client | المحادثات › التحويلات | none |
| `/customers` | 176 | client | المحادثات › العملاء | none |
| `/intelligence` | 854 | client | المحادثات › مختبر التجميع | none |
| `/orders` | 252 | client | المبيعات › الطلبات | none |
| `/orders/[id]` | 299 | client | (child) | none |
| `/products` | 379 | client | المبيعات › المنتجات | none |
| `/products/[id]` | 556 + 170 | client | (child) | none |
| `/catalogs` | 249 | client | المبيعات › الكتالوجات | none |
| `/catalogs/[id]` | 449 | client | (child) | none |
| `/campaigns` | 385 + 149 | client | المبيعات › الحملات | none |
| `/personas` | 372 | client | الذكاء › الشخصية | none |
| `/knowledge` | 484 | client | الذكاء › المعرفة | none |
| `/media` | 528 | client | الذكاء › الوسائط | none |
| `/feedback` | 260 | client | الذكاء › آراء العملاء | none |
| `/ai/playground` | 429 | client | الذكاء › التجربة | none |
| `/settings/ai` | 543 + 189 | client | الذكاء › التشغيل | none |
| `/automation` | 358 + 103 + 229 | client | الأتمتة › القواعد | none |
| `/follow-ups` | 139 | client | الأتمتة › المتابعات | none |
| `/analytics` | 613 | client | التقارير | none |
| `/channels` | 310 + 195 + 142 | client | الإعدادات › القنوات | none |
| `/settings/team` | 7 + 378 | server guard + client | الإعدادات › الفريق | none |
| `/settings/account` | 188 | client | الإعدادات › حسابي | none |
| `/order-warehouse`, `/ai/studio`, `/workflows`, `/analytics/ai`, `/analytics/products`, `/settings` | 6–9 each | server redirect | – | – |

---

# PART C — Design system as built (audited tree)

> LIVE: `components/ui/*`, `globals.css`, `tailwind.config.ts`, `theme.ts` and the theme picker are byte-identical in production, so C.1 and C.2 are current. The usage counts (C.3) and the largest-files list (C.4) are re-run for the deployed tree in **E.10**.

## C.1 `src/components/ui/*` — the shared kit
| File (lines) | Exports | Props / variants / notes |
|---|---|---|
| `button.tsx` (56) | `Button` | `variant`: `primary` (filled primary) · `secondary` (outlined surface) · `ghost` (text) · `danger` (red outline) · `success` (filled green, **hard-coded `bg-[#107c10]`** — the only hex in any `.tsx`). `size`: `sm` h-8 text-xs · `md` h-9 (default) · `lg` h-10 · `icon` 9×9. `loading` shows a spinner and disables. No `asChild`/link variant → pages wrap `<Link><Button>` or hand-style `<a>` tags. |
| `input.tsx` (49) | `Input`, `Textarea`, `Select`, `Field` | One shared class string; `Input`/`Select` h-9. `Field {label, hint, error}` wraps children in a `<label>` (no required marker; no page passes `error`). |
| `password-input.tsx` (64) | `PasswordInput`, `passwordToggle` | `kind: "password" | "secret"`; eye button with aria "إظهار/إخفاء كلمة المرور|القيمة". |
| `badge.tsx` (88) | `Badge`, `StatusBadge` | `Badge {tone, dot}` — `tone` is a **string key of a 39-entry map of enum names** (`OPEN`, `PENDING`, `CONFIRMED`, `DELIVERED`, `FAILED`, `URGENT`, `CONNECTED`, `IN_STOCK`, `AI_ACTIVE`, …) rather than semantic tones; unknown keys silently fall back to grey. `StatusBadge {value}` = `Badge` + `label(value)` from `i18n.LABELS`. |
| `card.tsx` (87) | `Card`, `CardHeader`, `GlassCard`, `GlassPanel`, `PageHeader` | `GlassCard {level: "glass-subtle" | "glass" | "glass-strong"}` is the card used everywhere (119 uses); `Card`, `CardHeader`, `GlassPanel` have **zero** uses. `PageHeader {title, description, actions, children}` renders the page `<h1>` (24 uses) and carries `data-page-header`. |
| `dialog.tsx` (130) | `Dialog`, `Drawer` | `Dialog {open, onClose, title, className}` — centred modal, default `max-w-lg`, focus trap, Esc, overlay click, no footer slot (each page builds its own button row; alignment varies between `justify-start` and `justify-end`). `Drawer {side, title, until: "lg"|"xl"}` — used once (customer panel). |
| `confirm.tsx` (89) | `ConfirmDialog`, `ErrorState` | `ConfirmDialog {title, body, confirmLabel, busy, danger}`; secondary button is always `t.common.cancel` "إلغاء"; with `danger` shows a "تأكيد الإجراء" strip. `ErrorState {title="تعذّر تحميل البيانات", description, onRetry}`. |
| `empty-state.tsx` (78) | `EmptyState`, `EmptyNext` | `EmptyState {icon | illustration, title, description, action}`; `EmptyNext {href, secondary}` = link styled as a small button with an arrow ("next step"). |
| `skeleton.tsx` (35) | `Skeleton`, `SkeletonCard`, `SkeletonTable` | `SkeletonCard` unused. |
| `stat-card.tsx` (63) | `StatCard` | `{label, value, icon, tone: brand|emerald|amber|sky|red, trend, hint}` — `trend` is never passed by any page. |
| `tabs.tsx` (47) | `Tabs` | `{items, activeKey, onSelect}`; used **once** (product detail). |
| `theme-toggle.tsx` (61) | `ThemeToggle`, `useTheme` | sun/moon button; `useTheme` unused. |
| `toast.tsx` (73) | `ToastProvider`, `useToast` | `show(message, kind: success|error|info)`; bottom-centre, 3.5 s. |
| `avatar.tsx` (55) | `Avatar` | `{name, src, size: xs|sm|md|lg, ring}`; initial on one of six hard-coded gradient pairs. |

Other shared pieces outside `ui/`: `brand-logo.tsx` (`BrandLogo` sizes xs–xl), `page-guide.tsx`, `setup-checklist.tsx`, `theme-picker.tsx`, `role-matrix.tsx`, `integrations/ui.tsx` (`RiskNote` red, `FixNote` orange, `RiskWithFix`, `HealthPill {health: ok|warn|error|off}`, `SectionHead`, `apiJson`, `fmtDateTime`; `InfoNote` and `CopyField` unused), `empty-section.tsx` (default component unused; only its `enforcePermission` helper is used), `inbox/*`, `products/compliance-card.tsx`.
Locally re-declared helpers that duplicate each other: `Section` + `MiniStat` (dashboard), `Section` + `Toggle` + `LiveProbe` (settings/ai), `KV` + `Bar` + `Buckets` + `Cell` (analytics), `PerfCell` (intelligence), `Row` (customer panel), `Msg` (account), three identical `async function api()` fetch wrappers (`multi-channel-section.tsx:31`, `settings/account/page.tsx:15`, `team-view.tsx:32`) next to `apiJson`.

## C.2 Theme system
- **Keys** (`src/lib/theme.ts`): `ThemeKey = "default" | "islamic" | "night"`, labels "فلوينت", "إسلامي", "المدينة الليلية" (`forcesDark`). Theme → `data-theme` attribute on `<html>` (absent for default); scheme → `.dark` class. Storage: `localStorage["zr-theme"]` and `localStorage["theme"]` (per device, not per account). `THEME_INIT_SCRIPT` runs inline before paint; `applyThemeToDocument` / `applySchemeToDocument` dispatch the `zr-theme-change` window event that `ThemeToggle` and `ThemePicker` listen to.
- **CSS variable roles** (`globals.css`, RGB triplets, redefined in `:root`, `.dark`, `[data-theme="islamic"]:not(.dark)`, `[data-theme="islamic"].dark`, `[data-theme="night"]`):
  surfaces `--background --surface --surface-glass --surface-elevated --surface-muted --surface-1 --surface-2 --surface-3 --surface-hover`; lines `--border --border-glass --line`; text `--text --text-muted --text-subtle`; brand `--brand --brand-strong --primary --primary-hover --primary-soft --primary-fg --brand-50…950`; decoration `--accent --accent-fg --showcase --ornament --ornament-alpha --app-glow-1/2`; status `--success --warning --danger --info`; elevation `--shadow-card --shadow-pop --glass-shadow`; logo `--logo-from --logo-via --logo-to --logo-spark --logo-glow --logo-bubble`; type `--font-display` (set on `body` per theme: Reem Kufi for islamic, Lalezar for night; applied to `h1` and `.font-display`).
- **Tailwind mapping** (`tailwind.config.ts`): colours `primary{DEFAULT,hover,soft}`, `accent{DEFAULT,fg}`, `showcase`, `logo.*`, `brand.50–950`, `surface.{1,2,3,hover}`, `ink{DEFAULT,muted,subtle}`, `line`; `text-primary` reads `--primary-fg`. `darkMode: "class"`. Radii are **remapped**: `rounded-lg` = 4 px, `rounded-xl` = 6 px, `rounded-2xl` = 8 px, `rounded-3xl` = 12 px. Shadows `xs`, `card`, `pop`, `glass`, `glow`. Animations `fade-in`, `slide-in`, `scale-in`.
- **`--success/--warning/--danger/--info` are defined for every theme but are not exposed as Tailwind colours.** Components therefore use fixed palette utilities for every status colour: 497 occurrences of `bg|text|border|ring-(red|emerald|amber|sky|slate|orange|indigo|violet|…)-NNN` versus 986 of the semantic tokens (`bg-surface-*`, `text-ink*`, `border-line`, `*-primary*`, `*-brand-*`). Status/alert colours do not follow the theme.
- **Component classes** in `globals.css`: `.glass-subtle` / `.glass` / `.glass-strong` (legacy names — now flat bordered surfaces; 50 / 11 / 3 uses), `.page-wrap` (`max-w-7xl`, `space-y-5`), `.section-title`, `.section-sub`, `.data-table` (+ thead/tbody rules), `.bubble-in` / `.bubble-out` / `.bubble-sys`, `.skeleton-shimmer`, `.num` (LTR tabular numbers; 110 uses), driver.js skin `.zr-tour`.
- **Theme hooks** (`data-*` read by CSS): `data-page-header` (ornament line under every `PageHeader`; night theme also gradient-fills the `h1`), `data-pane="nav"` (sidebar gets its own darker variable scope in islamic/night), `data-hero` (dashboard hero background illustration), `data-showcase` / `data-showcase-dots` (login panel artwork), `data-illustration` (hue-rotates empty-state images), `data-brand-logo` / `data-logo-tile` / `data-logo-spark` (logo tile becomes an 8-point star / a striped sun). Body ornaments via `body::before/::after` (girih tile; neon glow + skyline).
  Non-theme data attributes used by tests/tour: `data-tour`, `data-tabs-bar`, `data-guide`, `data-setup`, `data-step`, `data-state`, `data-theme-card`, `data-scheme`, `data-empty-next`, `data-ai-status`, `data-role-matrix`, `data-can`.
- **Dark mode**: top-bar/drawer `ThemeToggle` + the picker on `/settings/account`; night theme forces dark and disables the toggle. `viewport.themeColor` and the manifest colours are fixed hex values unrelated to the chosen theme.
- Fonts: body IBM Plex Sans Arabic; no Latin companion font (falls back to Segoe UI).

## C.3 Consistency measurements (`grep` over `src/app` + `src/components`, `*.tsx`)
| Pattern | Shared component | Raw element | Where the raw ones are |
|---|---|---|---|
| Buttons | `<Button` **167** (164 outside `ui/`) | `<button` **69** (62 outside `ui/`) | media 11 · intelligence 7 · inbox 6 · app-shell 5 · sidebar 3 · playground 3 · 2 each in theme-picker, setup-checklist, MediaPicker, team-view, products, products/[id], knowledge, catalogs, catalogs/[id], analytics · 1 each in page-guide, integrations/ui, message-bubble, conversation-item, channels, campaigns, automation |
| Text inputs | `<Input` **84** (+ `<PasswordInput` 8) | `<input` **13** (12 outside `ui/`) | 7 checkboxes (playground ×2, settings/ai, intelligence, catalogs/[id], segment-dialog, analytics) · 4 file inputs (products, products/[id], media, knowledge) · the top-bar search |
| Selects | `<Select` **51** | `<select` **3** (2 outside `ui/`) | `channels/page.tsx:244`, `catalog-page-section.tsx:113` |
| Textareas | `<Textarea` **17** | `<textarea` **4** (3 outside `ui/`) | inbox composer, playground composer, campaign message |
| Field wrapper | `<Field` 97 | — | many inputs have only a placeholder + `aria-label` and no visible label (campaign form, FAQ add rows, catalog search…) |
| Tables | — (no component) | `<table className="data-table">` ×5 | orders, customers, analytics products, team, role matrix; every other list is a stack of cards |
| Modals | `<Dialog` 19 · `<ConfirmDialog` 19 · `<Drawer` 1 | `window.confirm` ×1 (`multi-channel-section.tsx:91`) · `window.prompt` ×1 (`intelligence/page.tsx:312`) | plus hand-rolled popovers in app-shell (account menu, search suggestions) |
| Disclosure | — | `<details>` ×5 | inbox activity log, AI settings "متقدّم", product media picker, intelligence performance, team role matrix; two more are hand-rolled buttons (intelligence fields panel, setup checklist) |
| Tabs / segmented | `<Tabs` 1 | 7 hand-rolled tab strips (6 with `role="tablist"` + the group-tabs `<nav>`) + 2 segmented switches | inbox filters, inbox composer, MediaPicker, analytics, product categories, media types, group tabs; team invite mode, theme scheme — five different "selected" styles (filled primary · primary-soft · `bg-brand-500/15` · `bg-brand-600` · amber) |
| Badges | `<Badge` 43 · `<StatusBadge` 20 · `<HealthPill` 2 | ≈23 inline pill class strings | FAQ active pills, "اختياري", "ليلي دائمًا", intelligence chips, unread bubble, dashboard/auth pills… |
| Toggles | — (no Switch component) | checkbox cards, pill buttons, text links, `Select`s | "on/off" is expressed five ways: checkbox card (AI settings), pill "نشط/معطل" (FAQ), button "تعطيل/تفعيل" (products), text link "إيقاف/تفعيل" (catalogs), `Select` نشط/موقوف (team) |
| Feedback | `toast.show` 48 calls in 12 files | inline message cards / paragraphs / badges in the other pages | orders, channels, team, account, AI settings, products, knowledge, feedback, media, personas use inline text; inbox, handoffs, follow-ups, automation, campaigns, catalogs use toasts; several mutations give no feedback at all (products toggle/archive, personas activate, product media/FAQ, knowledge reindex/archive/FAQ) |
| Page wrapper | `.page-wrap` on every page | — | except `/intelligence` (`space-y-5 pb-10`) and `/settings/ai` (`page-wrap max-w-4xl`) |

Typography and sizing actually in use: `text-xs` 196 · `text-[11px]` 104 · `text-sm` 101 · `text-[10px]` 40 · `text-[13px]` 31 · `text-[12px]` 7 · `text-[15px]` 2 · others ≤3 — i.e. most text is 10–13 px, with three near-identical arbitrary sizes next to the scale. Control heights: `h-9` 22 · `h-10` 17 · `h-8` 11 · `h-12` 5 · `h-11` 5. Radii: `rounded-xl` 93 · `rounded-lg` 74 · `rounded-2xl` 71 · `rounded-full` 45 · `rounded-md` 8 · 3 arbitrary.
Number/date formatting: `formatNumber()` (locale `ar`) alongside 12 bare `toLocaleString()` (browser locale), 9 `toLocaleString("ar")`, 6 `toLocaleDateString("ar")`, one `"en-US"`, one `"ar-JO"` with 24 h clock, plus `timeAgo()` / `formatTime()` / `fmtDateTime()` / two local `formatDate()` — so digits and date shapes can differ between pages and even inside one row (**rendering UNVERIFIED**, depends on the browser).
Copy: `src/lib/i18n.ts` says "All components must use these keys instead of scattered hardcoded strings", yet there are 331 `t.*` references (on 308 lines, in 27 files) against ≈1 277 source lines with inline Arabic text; analytics (118 lines), intelligence (112), team, catalogs, personas, AI settings, campaigns, automation are almost entirely hard-coded. 28 dictionary keys were checked and found unused (e.g. `nav.inbox`, `inbox.filterMine`, `inbox.filterOrders`, `common.assignToMe`, `common.export`, `dashboard.recentConversations/topProducts/salesOverview`, `orders.assignee`, `pages.mediaTitle/settingsTitle/analyticsTitle`).

## C.4 Largest `.tsx` files (audited tree, `wc -l`)
| # | File | Lines |
|---|---|---|
| 1 | `src/components/inbox/inbox.tsx` | 909 |
| 2 | `src/app/(dashboard)/intelligence/page.tsx` | 854 |
| 3 | `src/app/(dashboard)/analytics/page.tsx` | 613 |
| 4 | `src/app/(dashboard)/products/[id]/page.tsx` | 556 |
| 5 | `src/app/(dashboard)/settings/ai/page.tsx` | 543 |
| 6 | `src/app/(dashboard)/media/page.tsx` | 528 |
| 7 | `src/app/(dashboard)/knowledge/page.tsx` | 484 |
| 8 | `src/app/(dashboard)/catalogs/[id]/page.tsx` | 449 |
| 9 | `src/app/(dashboard)/ai/playground/page.tsx` | 429 |
| 10 | `src/components/app-shell.tsx` | 407 |
| 11 | `src/app/(dashboard)/campaigns/page.tsx` | 385 |
| 12 | `src/app/(dashboard)/products/page.tsx` | 379 |
| 13 | `src/app/(dashboard)/settings/team/team-view.tsx` | 378 |
| 14 | `src/app/(dashboard)/personas/page.tsx` | 372 |
| 15 | `src/app/(dashboard)/automation/page.tsx` | 358 |
All `.tsx` under `src`: 14 250 lines. Every page is a single client component holding its own state, fetches and dialogs (the inbox has 33 `useState` hooks in one function, `inbox.tsx:66-99`).

---

# PART D — Problems visible from code (audited tree)

Judgement calls, each tied to a file. Items changed by the deployed version are re-assessed in Part E.

## D.1 Duplicated / overlapping functionality
| # | Same thing in several places | Files |
|---|---|---|
| 1 | **Escalated conversations** shown as: page `/handoffs`; inbox chip "تحتاج مراجعة"; inbox banner "بانتظار موظف"; top-bar bell (plain link); dashboard KPI + section "الدعم البشري". Two different "take" actions with two endpoints — "استلام" on `/handoffs` (`POST /api/handoffs/{id}` claim) and "استلام من الذكاء الاصطناعي" in the inbox (`POST /api/conversations/{id}/takeover`) — and two "done" actions ("حل" vs "تم الحل"). Nothing explains the difference (whether claiming also pauses the AI is **UNVERIFIED** — the UI calls only the handoff endpoint). | `handoffs/page.tsx`, `inbox.tsx:611-632,653-659`, `app-shell.tsx:331-339`, `dashboard/page.tsx:163,189` |
| 2 | **AI on/off** at three scopes on three unlinked pages: workspace toggles (`/settings/ai`), per-page "وضع التجميع" (`/intelligence`), per-conversation takeover (inbox) — plus a workflow step "تشغيل/إيقاف الذكاء بالمحادثة". | `settings/ai/page.tsx:337-351`, `intelligence/page.tsx:475-498`, `inbox.tsx:611-619`, `workflow-builder-dialog.tsx:59` |
| 3 | **Two conversation viewers**: the inbox and `/intelligence` (own list + message view + extracted customer data, read-only, different layout). Three unrelated renderings of "customer data": inbox panel, intelligence fields, customers table. | `inbox.tsx`, `intelligence/page.tsx:609-815`, `customers/page.tsx` |
| 4 | **FAQs** managed on `/knowledge` (general) and on each product's tab (product-specific) with copy-pasted UI. | `knowledge/page.tsx:399-427`, `products/[id]/page.tsx:530-552` |
| 5 | **Reviews**: text reviews on `/feedback`; review images as media categories "رأي عميل/شهادة" on `/media`; third surface = MediaPicker tab "آراء الزبائن". | `feedback/page.tsx`, `media/page.tsx:43`, `MediaPicker.tsx:38` |
| 6 | **Product images** linked from `/media` (dialog) and from the product's "الوسائط" tab — both call `/api/products/{id}/media`. | `media/page.tsx:187-211`, `products/[id]/page.tsx:118-183` |
| 7 | **Catalog ↔ page link** editable on `/channels` (per page) and on `/catalogs/{id}` (per catalog) — same endpoint, different wording ("تبديل" vs "ربط الصفحة"). | `catalog-page-section.tsx:59-79`, `catalogs/[id]/page.tsx:190-209` |
| 8 | **Which persona speaks**: default on `/personas`, per page on `/channels`, per session in the playground; plus persona "edit" vs persona "versions". | `personas/page.tsx:140-195`, `channels/page.tsx:134-146` |
| 9 | **Overview numbers** rendered twice (dashboard and analytics "نظرة عامة") from the same endpoint with separate, non-persisted period/timezone selectors; `/orders` has a third set of counters computed differently. | `dashboard/page.tsx:79-90`, `analytics/page.tsx:354-403`, `orders/page.tsx:80-86` |
| 10 | **Two rule builders** ("قاعدة سريعة" / "قاعدة بخطوات") backed by two API families on one list; a quick rule is a one-step rule. | `quick-rule-dialog.tsx`, `workflow-builder-dialog.tsx`, `automation/page.tsx:52-54` |
| 11 | **Five onboarding mechanisms**: setup checklist, per-page guide panel with "next step", guided tour, empty-state "next" buttons, AI status card "قبل التشغيل" links. | `setup-checklist.tsx`, `page-guide.tsx`, `guided-tour.ts`, `empty-state.tsx:60`, `ai-status-card.tsx:172-186` |
| 12 | Logout (sidebar + account menu), theme switch (top bar + drawer + account page), search (top bar + inbox + each list). | `app-sidebar.tsx:145`, `app-shell.tsx:378` |

## D.2 Overloaded screens
| Screen | What is stacked on it | Evidence |
|---|---|---|
| Inbox | 10 filter controls above a 320 px list (search, 5 chips, 4 selects); thread header with 5 equally-weighted actions in one wrapping row, the destructive "حذف المحادثة" beside "تم الحل"; composer with 2 tabs + 2 icon buttons; right panel of 7 blocks; 2 dialogs + drawer + 2 confirms. ≈24 persistent controls. | `inbox.tsx:460-522, 606-650, 706-784` |
| `/intelligence` | 6 stat cards, page switch bar with a mode-changing dropdown, field configuration, conversation list, message viewer, extracted-data table, performance metrics — six jobs in one 854-line page. | whole file |
| `/settings/ai` | status card with 7 links/buttons, 2 toggles, provider form, connection test, 10 advanced inputs, single save at the very bottom. | whole file |
| `/analytics` | 9 tabs, ≈60 metrics, each tile carrying its own caveat sentence. | `analytics/page.tsx:45-55` |
| `/knowledge` | create form + file upload + filters + list in one card titled "إضافة مصدر", then a search tester, then FAQs. | `knowledge/page.tsx:241-349` |
| `/campaigns` | segment list, always-open create form (6–8 fields), campaign list with up to 5 actions per row and inline dry-run / run-log panels. | whole file |
| `/channels` | Facebook connect, 4 other channels with inline credential forms, then per Facebook page: stats, persona, test, disconnect, catalog switcher. | whole file |
| `/products/[id]` | 6 tabs, ≈40 fields, separate save per tab. | whole file |
| `/ai/playground` | 9 settings (model id, task, "draft instructions"…) before the first message. | `ai/playground/page.tsx:224-278` |
| `/dashboard` | 22 numbers (6 KPI + 16 mini) + checklist + 2 selects; nothing actionable, nothing recent. | `dashboard/page.tsx:156-200` |

## D.3 Controls that do nothing, dead ends, API without UI
No UI call targets a missing route: all ≈150 `fetch`/`api()` call sites were matched to an existing `route.ts` export. The problems are the other way round.

**Dead or misleading controls**
| Item | File |
|---|---|
| Emoji button permanently disabled ("قريبًا") | `inbox.tsx:748-755` |
| Order draft card says "جاهزة للتأكيد" but has no action; confirm/cancel endpoints unused | `customer-panel.tsx:157-198`, `api/order-drafts/[id]/{confirm,cancel}/route.ts` |
| Bell icon = plain link, no count | `app-shell.tsx:331-339` |
| "إدارة المنتجات" and "معاينة الكتالوج" are two links to the same URL | `catalog-page-section.tsx:97-102` |
| "جدولة" posts `{}` — no date/time can be chosen | `campaigns/page.tsx:135-143,307` |
| Workflow step asks for "معرّف الموظف … من صفحة «الفريق»"; the team page never shows ids | `workflow-builder-dialog.tsx:200-202`, `team-view.tsx` |
| "دعوة حساب مسجل" creates an invitation nobody can accept in the UI | `team-view.tsx:284-327`, `api/workspaces/members/accept/route.ts` |
| AI status text "الوكيل بيقترح والموظف بيبعت" — no screen lists AI suggestions for approval | `ai-status-card.tsx:69-71` |
| Compliance tab tells the user to check «الربط مع الأنظمة» — no such page in this tree (`AGENTS.md` also lists `settings/integrations` and `settings/agents`; neither exists) | `compliance-card.tsx:104` |
| Workspace-name form does not show the current name | `team-view.tsx:269-282` |
| Persona versions panel cannot be closed once opened | `personas/page.tsx:263-280` |
| "طباعة" prints the whole app chrome (no print CSS) | `orders/page.tsx:102-104` |
| Customers guide promises "ملاحظاتك عليه"; notes are never rendered | `page-guides.ts:44`, `customer-panel.tsx` |
| TikTok row with no action | `multi-channel-section.tsx:28,130` |
| Exported but unused UI: `Card`, `CardHeader`, `GlassPanel`, `SkeletonCard`, `CopyField`, `InfoNote`, `EmptySection`, `useTheme`, `OrderIndicatorBadge`, `AiIndicator`, `DeliveryTick`, `StatCard.trend` | `components/ui/*`, `inbox/*` |

**Server capabilities with no screen** (route exists, zero UI callers)
| Capability | Endpoint |
|---|---|
| Confirm / cancel / read an order draft | `POST /api/order-drafts/{id}/confirm`, `…/cancel`, `GET /api/order-drafts/{id}` |
| Assign a conversation to an agent | `POST /api/conversations/{id}/assign` |
| Change conversation priority / tags / reopen / close | `PATCH /api/conversations/{id}` (UI only sends `RESOLVED`) |
| Older messages; list filters by priority, date range, AI state | `GET …/messages?cursor`, `GET /api/conversations?priority&dateFrom&dateTo&aiState` |
| Edit a customer (name, phone, address, tags, notes, block) | `PATCH /api/customers/{id}` |
| Discount requests and approvals | `GET/POST /api/discount-approvals`, `POST …/{id}` |
| Facebook / Instagram comment reply, hide, private reply | `POST /api/comments/{facebook,instagram}/{reply,hide,private}` |
| Edit order notes; `FAILED` status | `PATCH /api/orders/{id}` |
| Edit / inspect / preview a segment | `GET/PATCH/POST /api/segments/{id}` |
| Edit a campaign, list recipients, AI copy draft | `PATCH /api/campaigns/{id}`, `GET …/recipients`, `POST /api/campaigns/ai-draft` |
| Edit a rule | `PATCH /api/automations/{id}`, `PATCH /api/workflows/{id}` |
| Rename / re-categorise media | `PATCH /api/media/{id}` |
| Persona for a non-Facebook channel | `PATCH /api/channels/by-id/{id}/persona` |
| Route a handoff automatically | `POST /api/handoffs/{id} {action:"route"}` |
| List / create / switch workspace; workspace timezone | `GET/POST/PATCH /api/workspaces`, `PUT {timezone}` |
| Accept an invitation | `POST /api/workspaces/members/accept` |
| Queue, SLA and usage reports; rebuild trends | `GET /api/analytics/{queue,sla,usage}`, `POST /api/analytics/rebuild` |

## D.4 Actions shown regardless of permission
Pages never learn the role (A.3), so every write control is rendered for everyone who can open the page.
| Role (from `ROLE_PERMISSIONS`) | Sees but cannot use (API returns 403) |
|---|---|
| مشاهد (VIEWER) — nav shows 16 pages | inbox composer, takeover, resolve, delete, consent buttons; order status buttons; product create/import/toggle/archive and all product tabs' "حفظ"; catalog create/delete/link; campaign create/validate/approve/schedule/cancel; persona create/edit/default/publish; knowledge add/upload/reindex/archive/delete; media upload/link/delete; feedback approve/archive; rule create/run/delete; channel connect/disconnect/persona/catalog; intelligence mode switch, field settings, re-analyse, confirm |
| موظف دعم (SUPPORT_AGENT) | "حذف المحادثة" (`conversations.delete`), **all order status buttons** (`orders.write`; the API even returns `canWrite:false`, ignored at `orders/[id]/page.tsx:136-148`), every product/catalog/media/feedback/knowledge/persona write action; inbox "الصفحة" filter silently empty (no `channels.read`) |
| موظف مبيعات (SALES_AGENT) | "حذف المحادثة", product/catalog/media/feedback/knowledge/persona writes, campaign create/approve; intelligence mode switch and field save |
| مشرف (MANAGER) | almost no mismatch (has every write permission used by the pages it can open). Order "حذف" is correctly hidden for it by the API's `canDelete` flag — the single place in the tree where a control is hidden by permission |
What the user gets instead: the generic toast `t.inbox.actionFailed` "تعذّر تنفيذ الإجراء، حاول مجددًا" (invites a retry), a page-level "حدث خطأ", or **nothing** where the response is not checked (`products/page.tsx:125-141`, `personas/page.tsx:140-147`, `knowledge/page.tsx:139-151,211-229`, `products/[id]/page.tsx:118-143,170-203`).
Links that ignore the nav filter: dashboard "التفاصيل" (5), the bell, every `EmptyNext`, AI status tiles, automation → "المتابعات", catalog section → `/catalogs/{id}`.

## D.5 Inconsistent wording for the same concept
**Escalation to a human**
| Term | Where |
|---|---|
| التحويلات | nav tab; dashboard "إجمالي التحويلات"; empty state "لا توجد تحويلات" |
| التسليم للموظفين | page H1, bell tooltip (`i18n.nav.handoffs`, `pages.handoffsTitle`); login "تسليم ذكي للموظفين" |
| تحتاج تدخلًا بشريًا / الدعم البشري | dashboard KPI and section |
| تحتاج مراجعة | inbox chip + list badge (= open handoff) — the same phrase is the orders counter for cancelled/failed/returned |
| بحاجة لموظف | inbox chip that actually filters conversations a human already holds |
| بانتظار موظف | inbox banner |
| استلام من الذكاء الاصطناعي → "تحويل المحادثة إلى موظف؟" → "تحويل إلى موظف" | one inbox button, its dialog title and its confirm button (three names for one action; "استلام" = receive, "تحويل" = transfer) |
| استلام / إرجاع للقائمة / حل | handoffs page buttons |
| استلام الموظف / طلب مراجعة | inbox activity log |
| تحويلات للموظف / تحويلات موظف | analytics tiles |
Reasons are translated twice, differently: inbox `HANDOFF_REASONS` "طلب العميل · ثقة منخفضة · تعطل الذكاء · طلب محظور · محاولة حقن" vs handoffs `REASONS_AR` "العميل طلب موظفًا · ثقة AI منخفضة · فشل AI · طلب محجوب بالقواعد · محاولة اختراق" (`inbox.tsx:43-53`, `handoffs/page.tsx:26-36`); analytics prints the raw enum. Status: "مستلَم" (`LABELS`) vs "مُستلمة" (banner).

**The AI**
| Term | Where |
|---|---|
| الذكاء | nav group, analytics tab, "تشغيل الذكاء", "الذكاء متوقف", "ردود الذكاء" |
| الذكاء الاصطناعي | inbox badges/buttons, persona page title, playground, knowledge description |
| الوكيل | page guides, H1 "تشغيل الوكيل", dashboard "ردود الوكيل", "جرّب الوكيل", checklist |
| الوكيل الذكي | dashboard section, tour, login — **and** the provider option "الوكيل الذكي (سيرفرك)" in the provider dropdown (`provider-labels.ts:17`) |
| المساعد | persona description, playground bubble name, rollback warning |
| AI | handoff reasons, product FAQ heading, intelligence metrics |
| الرد الآلي | the auto-send toggle, and the sender label of AI messages in `/intelligence` |
| مساعد البيع | the reply-suggestion dialog (a different feature) |
Per-conversation AI state alone has five phrasings: "الرد بواسطة الذكاء الاصطناعي / الرد يدويًا" (header badge), "الذكاء الاصطناعي / موظف" (list badge), "يديرها الذكاء الاصطناعي / بحاجة لموظف" (chips), "الذكاء نشط / الذكاء موقوف / موظف يدير المحادثة" (`LABELS`, workflow builder), "وضع التجميع / الرد الآلي" (page level).

**"مسودة" (draft) names five unrelated objects**: order draft ("مسودة طلب"; analytics "مسودات طلب", "مسودات جاهزة"), campaign ("إنشاء مسودة"), rule ("حفظ كمسودة", "مسودة — ما بتشتغل"), persona version ("مسودة من الحالية"), playground ("تعليمات مسودة للاختبار"). Order-draft states render as "جاهزة" / "مؤكد" / "ملغى" or raw `EMPTY` / `COLLECTING` / `CONFIRMING` / `CONVERTED`.

**Order status**
| Enum | Badge (`LABELS`) | Other wording |
|---|---|---|
| NEW | جديد | "طلبات جديدة" |
| CONFIRMATION_PENDING | قيد التأكيد | confirmation `PENDING` = "قيد الانتظار" (also the conversation status) |
| CONFIRMED | مؤكد | "طلبات مؤكدة"; action button also reads "مؤكد" |
| PROCESSING | قيد المعالجة | same words as a knowledge document being indexed |
| OUT_FOR_DELIVERY | قيد التوصيل | — |
| DELIVERED | تم التوصيل | "مُسلَّمة" / "طلبات مُسلَّمة" / "مسلَّمة" (dashboard, analytics), "مكتملة" (orders counter), trigger "عند تسليم الطلب" |
| CANCELLED | ملغى | "ملغاة" (analytics, `t.states`); the cancel **button** is labelled "ملغى" |
| FAILED / RETURNED | فشل / مُرجَع | together with cancelled counted as "تحتاج مراجعة" |
Four status dimensions on the order page (الطلب · التأكيد · الدفع · التنفيذ) with overlapping vocabularies ("تم التوصيل" exists in two of them).

**Conversation status**: badges "مفتوحة / قيد الانتظار / تم الحل / مغلقة" (`LABELS`) vs the inbox filter options "مفتوحة / معلقة / محلولة / مغلقة" (`inbox.tsx:494-498`); the resolve button is the state name "تم الحل".

**On/off, active/inactive**
| Object | States | Verbs |
|---|---|---|
| Product | النشطة / غير النشطة / المؤرشفة; badge متوفر / قيد الانتظار / مغلقة | تعطيل / تفعيل · أرشفة / استعادة |
| Catalog | نشط / متوقف | إيقاف / تفعيل |
| FAQ | نشط / معطل | (pill) |
| Rule | شغّالة / موقوفة / مسودة — ما بتشتغل | تشغيل / إيقاف / إيقاف مؤقت / إيقاف وإلغاء الجاري |
| Campaign, segment | متوقف مؤقتًا / فعّالة | إيقاف مؤقت / استئناف |
| Member | نشط / موقوف / مدعو | (select) · إيقاف العضو |
| AI | شغّال / متوقف / مطفي | تشغيل الذكاء |
| Channel | متصل / غير متصل | ربط / فصل (other channels) / إزالة الربط (Facebook) / فك الربط (catalog page) |

**Other pairs**: عميل/العملاء vs زبون/الزبائن (`MediaPicker.tsx:38`, `marketing-card.tsx:121`, media notice, guides); إسناد (inbox) vs تعيين (`i18n.common`, analytics "محادثات معينة"); القناة vs الصفحة vs صفحة فيسبوك vs صفحة ماسنجر vs صفحة Facebook vs Facebook Messenger vs فيسبوك ماسنجر (the inbox filter "الصفحة" lists WhatsApp numbers and bots too); التقارير vs التحليلات (aria label, error texts); الوسائط vs المكتبة vs مكتبة الوسائط vs مرفق; المعرفة vs قاعدة المعرفة vs مصدر vs مستند.
**Buttons for the same act**: save = "حفظ" · "احفظ" · "حفظ الإعدادات" · "حفظ الاسم" · "حفظ الشريحة" · "حفظ البيانات المطلوبة" · "حفظ كمسودة" · "حفظ وقراءة"; create = "إنشاء" · "إنشاء المنتج" · "منتج جديد" · "شخصية جديدة" · "إضافة" · "+ إضافة سؤال" · "إنشاء مسودة" · "إنشاء وتشغيل" · "من نص"; remove = "حذف" · "حذف نهائي" · "إزالة" · "إزالة من المنتج" · "فك الربط" · "أرشفة" · "احذف الكتالوج". "إلغاء" is at once dialog-dismiss, cancel-follow-up, cancel-campaign and (as "ملغى") cancel-order; two dialogs end up with two buttons both reading "إلغاء" or "إلغاء"/"إلغاء الطلب" (`follow-ups/page.tsx:127-136`, `orders/[id]/page.tsx:260-285`).
**Register**: formal Arabic and Levantine colloquial alternate inside the same screen — e.g. `/campaigns` header "خطوتين: اختار مين بدك توصله" above the empty state "أنشئ شريحة ثم حملة — لا يُرسل شيء قبل التحقق والاعتماد والجدولة."; `/catalogs` formal ("سيُحذف الكتالوج وروابطه بالصفحات") vs `/automation` colloquial ("ولا قاعدة لسا"); dictionary strings are formal, newer inline strings colloquial.
**Tab label vs page title**: التحويلات → "التسليم للموظفين"; الكتالوجات → "كتالوجات المنتجات"; الحملات → "الحملات التسويقية"; الشخصية → "شخصية الذكاء الاصطناعي"; المعرفة → "قاعدة المعرفة"; التجربة → "تجربة الذكاء الاصطناعي"; التشغيل → "تشغيل الوكيل" (which contains a section also titled "التشغيل"); القواعد → "القواعد التلقائية"; الفريق → "الفريق والصلاحيات"; الرئيسية / لوحة التحكم → "مرحبًا".

## D.6 Technical jargon and raw English shown to the owner
| Where | What is displayed |
|---|---|
| Inbox draft card (`customer-panel.tsx:171,191`) | `COLLECTING`, `EMPTY`, `CONFIRMING`, `CONVERTED`; "الحقول الناقصة: product، quantity، phone، address" |
| Channels (`channels/page.tsx:204-240`, `multi-channel-section.tsx:24-29,165-177`) | "Facebook Messenger", "Meta Graph API", "META_APP_ID و META_APP_SECRET غير مضبوطين على الخادم", "آخر Webhook", "حالة التوكن: LONG_LIVED", "Instagram ID", "Page Access Token", "Bot Token من BotFather", "Phone Number ID", "System User Token", "Business Messaging", "Webhook: https://…" |
| AI settings | "تعبئة GLM", "OpenRouter", "عنوان الـAPI — متوافق مع OpenAI — بينتهي بـ /v1", "sk-…", `auto/cheap`, `auto/best-chat`, "بالتوكنز", "بيئة الخادم", "متصل · 123ms" |
| Dashboard / analytics | "توكنز مستهلكة", "استدعاءات فاشلة", "توكنز داخلة/طالعة", "ملّي ثانية", raw bucket keys (handoff reasons, `RESPONSE`/`EXTRACTION`, model ids, campaign and recipient statuses, exclusion reasons, follow-up types), ISO timestamp after "آخر بناء:", "تصدير Excel (CSV)" |
| `/intelligence` | "P50 / P95", "ms", "انتظار في الطابور", "زمن استخراج AI", raw `deliveryStatus`, message-id fragments as "المصدر" |
| Playground | "النموذج", "المهمة (نموذجها الخاص)", order proposal as raw JSON, provider/model/latency/token counts, "{page} · CONNECTED" |
| Products | mandatory "SKU" ("HONEY-001"); default currency "USD"; upload errors "[code=STORAGE_WRITE_FAILED]", "HTTP 413", "(NO_RESPONSE)" |
| Product › الامتثال | "سيستم 1", "pf_TIN01", "compliance.updated", "رابط الاستقبال" |
| Media | server setting name `WA_MEDIA_RETENTION_DAYS`; MIME types; `VIDEO` / `DOCUMENT` on thumbnails |
| Campaigns | raw event types and reasons, "سبب الإلغاء: MANUAL", "اسم القالب المعتمد في Meta", "ar" |
| Automation | raw error strings (LTR) in the run log; condition values must be typed as stored codes; "معرّف الموظف" |
| Catalogs | "{id} · CONNECTED"; "التبديل ذري", "يُستبدل ذريًا" |
| Knowledge | "مقطع", "إصدار 3", "الملاءمة 0.412", "إعادة الفهرسة", "(PDF/TXT/MD/CSV/JSON)" |
| Personas | "v3"; raw `helpful` in the summary line |
| Handoffs / follow-ups | user id instead of a name; raw `CHECK_IN`; raw `lastError` |
| Shell | "Enter", "Ctrl+K" in hints |

## D.7 Illogical flows
1. **"Why is the AI not answering?" has no single place.** It depends on: channel connected (`/channels`), page not in collection mode (`/intelligence`), page has an active catalog (`/channels` or `/catalogs` — otherwise "الذكاء لا يستخدم أي منتج لها"), active products (`/products`), workspace toggles + provider (`/settings/ai`), conversation not taken over (inbox). The status card on `/settings/ai` checks only the toggles/provider and counts personas and knowledge.
2. **A page's reply mode is switched from a lab page** in the "المحادثات" tabs by choosing an option in a dropdown, with no confirmation; neither `/channels` nor the inbox shows that a page is in collection mode.
3. **Catalog is a hidden prerequisite.** A connected Facebook page without a catalog gives the AI no products, but the setup checklist has no catalog step and marks "أضف منتجاتك" done with one product. Catalogs live under "المبيعات", the page link under "الإعدادات › القنوات". Non-Facebook channels cannot be given a catalog or persona at all.
4. **Human takeover leaves the order half-done.** After "استلام" the agent sees the draft but cannot confirm, edit or cancel it, and cannot create an order by hand; the order page's own actions are named after states ("مؤكد", "ملغى").
5. **Campaign prerequisites are scattered**: marketing consent is recorded one customer at a time inside the inbox side panel → segment (create-only) → WhatsApp template name typed from Meta → "تحقق" → "اعتماد المحتوى" + "جدولة" shown together → no send time.
6. **Customers → inbox search.** Clicking a customer runs a text search in the inbox instead of opening a profile; `?id=` deep links (handoffs, order page) only work when that conversation is within the first 30 loaded rows (B.3.1).
7. **Time zone**: the workspace has a time zone on the server, yet dashboard and analytics each start in UTC on every visit.
8. **Team**: invited users cannot accept; "إنشاء حساب جديد" makes the manager invent and hand over a password; role changes apply on select without confirmation.
9. **AI settings**: toggles look instant but need the bottom "حفظ الإعدادات"; "فحص الاتصال" tests the saved configuration, not what was just typed.
10. **Stock**: products show a stock label and catalogs print "مخزون n", but no screen edits stock status or quantity (import only).
11. **Rules**: cannot be edited; the assign step needs an id the product never shows; condition values have no pickers.
12. **Media upload** files land in whichever category the filter happens to show.
13. **Knowledge**: documents stay "قيد المعالجة" until a manual reload; the primary button is "من نص".
14. **Guides send users in circles** (team → handoffs → customers → orders → analytics → products …) regardless of what is already configured.
15. **Mobile**: no search (`<md`), no account menu (`<sm`), logout only in the drawer; the inbox's customer data and draft are two taps away in a drawer.

## D.8 Features in the navigation that are empty or dormant
| Nav item | Why it is dormant for a typical store |
|---|---|
| المحادثات › مختبر التجميع | only meaningful after a page is put in "collection mode"; otherwise six zero counters and an empty list — yet it is in the daily tab row |
| المبيعات › الحملات | needs opted-in customers, a segment, and (WhatsApp) an approved Meta template |
| المبيعات › الكتالوجات | plumbing for multi-page shops, but mandatory for any Facebook page |
| الأتمتة (القواعد, المتابعات) | empty until a rule is written; follow-ups is a read-only log |
| الذكاء › آراء العملاء, الوسائط | content libraries placed between persona/knowledge and the on-switch |
| التقارير | a group with one page of the same name |
| القنوات › تيك توك | announced, not connectable |
| Six redirect stubs | `/order-warehouse`, `/ai/studio`, `/workflows`, `/analytics/ai`, `/analytics/products`, `/settings` |

## D.9 Code-level defects that surface in the UI
| # | Defect | File |
|---|---|---|
| 1 | Lists truncated without notice: orders 25, customers 25, products 25, feedback 25, knowledge 25, media 30, handoffs/follow-ups 50, messages 50; order counters and the customers badge count the loaded rows | `orders/page.tsx:43-56,80-86`, `customers/page.tsx:35-45,63-67`, services' `limit ?? 25` |
| 2 | 20 s poll replaces the inbox list with page 1; deep-linked/older conversation leaves a blank thread (no back button on phones) | `inbox.tsx:138,163-168,188-195,441-449,454-459,561-571` |
| 3 | AI and human outbound bubbles indistinguishable | `message-bubble.tsx:40-45` |
| 4 | Facebook icon shown for every channel | `conversation-item.tsx:60-65`, `inbox.tsx:588-592`, `customer-panel.tsx:43-48` |
| 5 | Sales-coach suggestions carried over to the next conversation | `inbox.tsx:97,623-626` |
| 6 | Order draft never refreshed; raw labels | `inbox.tsx:128-136,240`, `customer-panel.tsx:171,191` |
| 7 | `Badge` tones that do not exist → everything grey: catalogs `success`/`muted`; member `ACTIVE`/`INVITED`/`SUSPENDED`; all eight customer statuses; handoff `CLAIMED`; campaign `DRAFT`/`READY`/`PAUSED`/`COMPLETED`; rule execution `COMPLETED`/`WAITING` | `badge.tsx:4-44`, `catalogs/page.tsx:191`, `team-view.tsx:222`, `customers/page.tsx:125` |
| 8 | Product badge borrows unrelated labels (archived = "مغلقة", active = "متوفر", inactive = "قيد الانتظار") | `products/page.tsx:234` |
| 9 | Delete/link/unlink failures on `/media` say "فشل الرفع" | `media/page.tsx:181,199,210` |
| 10 | Account messages show a ✓ icon for errors | `settings/account/page.tsx:31-38` |
| 11 | Product category chips vanish after one is chosen | `products/page.tsx:174-196` |
| 12 | Campaign action errors appear inside the "حملة جديدة" card | `campaigns/page.tsx:144-147,277` |
| 13 | "لا يوجد عملاء بعد — اربط قناة" shown when the status filter merely matches nothing | `customers/page.tsx:164-173` |
| 14 | Dashboard keeps old numbers with no loading state while a new period loads | `dashboard/page.tsx:79-94` |
| 15 | Unchecked mutations fail silently (see D.4) | several |
| 16 | Switching product tabs discards unsaved edits (uncontrolled forms unmount) | `products/[id]/page.tsx:253-439` |
| 17 | Playground bubbles mirror the inbox's sides | `ai/playground/page.tsx:336-337` |
| 18 | Setup checklist: 6 tiles in a 5-column grid | `setup-checklist.tsx:109` |
| 19 | `/intelligence` not width-capped (no `.page-wrap`) | `intelligence/page.tsx:410` |
| 20 | No `error.tsx` / `loading.tsx` / `not-found.tsx`; the team page guard throws into the default error screen (**UNVERIFIED** rendering) | `src/app` |
| 21 | Shell name/workspace name come from the server layout; account and team forms do not refresh it (**UNVERIFIED** at runtime) | `layout.tsx:39-45`, `settings/account/page.tsx:65-80` |

---

# PART E — LIVE DELTA (deployed tree `…\scratch-2026-11-16-d96b4e\AIresponse-live`)

Parts A–D describe a tree that is 73 commits behind production. This part describes the deployed code and says exactly what differs.

## E.0 Method and scope of change
- `diff -rq` between the two trees over `src/app` (excluding `api`), `src/components`, `src/lib`. Changed files were read as unified diffs against the versions already read in full; new files were read in full.
- **Byte-identical in both trees** (so Parts A–D apply unchanged): `app-shell.tsx`, `app-sidebar.tsx`, `group-tabs.tsx`, `nav-icons.ts`, `page-guide.tsx`, `setup-checklist.tsx`, `guided-tour.ts` (component), `theme-picker.tsx`, `role-matrix.tsx`, `auth-form.tsx`, everything in `components/ui/`, `integrations/ui.tsx`, `compliance-card.tsx`, `conversation-item.tsx`, `MediaPicker.tsx`, `marketing-card.tsx`, `globals.css`, `tailwind.config.ts`, root and dashboard `layout.tsx`, and the pages `/handoffs`, `/intelligence`, `/products` (list), `/personas`, `/media`, `/feedback`, `/ai/playground`, `/analytics`, `/settings/team`, `/settings/account`, `channels/multi-channel-section.tsx`, `channels/catalog-page-section.tsx`, `settings/ai/ai-status-card.tsx`.
- **Changed**: `lib/nav-model.ts`, `nav-search.ts`, `page-guides.ts`, `guided-tour.ts`, `i18n.ts`; `inbox/inbox.tsx`, `customer-panel.tsx`, `message-bubble.tsx`, `types.ts`; pages `dashboard`, `orders`, `orders/[id]`, `customers`, `channels`, `settings/ai`, `knowledge`, `products/[id]`, `catalogs`, `catalogs/[id]`.
- **New**: `components/realtime/use-realtime.ts` (71), `components/orders/oms-order-card.tsx` (113), `channels/oms-section.tsx` (224), `channels/telegram-order-section.tsx` (187), `channels/order-fields-section.tsx` (95), `settings/ai/why-no-reply-card.tsx` (154), `customers/[id]/page.tsx` (441), `conversations/loading.tsx` (18), `orders/loading.tsx` (16), `lib/product-save-error.ts` (55).
- **Reduced to redirect stubs** (6 lines each, `redirectToMerged(...)` → `/dashboard`): `automation/page.tsx`, `campaigns/page.tsx`, `follow-ups/page.tsx`, `workflows/page.tsx`. Their companions `quick-rule-dialog.tsx`, `workflow-builder-dialog.tsx`, `segment-dialog.tsx` are deleted.
- New API routes in live (none removed): `GET /api/events`, `POST /api/conversations/{id}/memory/reset`, `POST /api/customers/{id}/refresh-profile`, `GET/PUT /api/channels/by-id/{id}/order-fields`, `GET/POST /api/orders/{id}/oms`, `GET/PUT /api/settings/oms` (+ `/test`, `/sync`), `GET/PUT /api/settings/telegram-orders` (+ `/test`), `GET /api/analytics/ai-stages` (no UI caller).

## E.1 Navigation and guidance (live)
`src/lib/nav-model.ts` (182 lines) — **6 groups, 18 pages**:
| # | Group | Pages |
|---|---|---|
| 1 | الرئيسية | لوحة التحكم `/dashboard` |
| 2 | المحادثات | الوارد `/conversations` · التحويلات `/handoffs` · العملاء `/customers` · مختبر التجميع `/intelligence` |
| 3 | المبيعات | الطلبات `/orders` · المنتجات `/products` · الكتالوجات `/catalogs` (**الحملات removed**) |
| 4 | الذكاء | الشخصية · المعرفة · الوسائط · آراء العملاء · التجربة · التشغيل (unchanged) |
| 5 | التقارير | التقارير `/analytics` |
| 6 | الإعدادات (bottom) | القنوات `/channels` · الفريق `/settings/team` · حسابي `/settings/account` |
- The whole group **الأتمتة** (القواعد, المتابعات) is gone. `NAV_REDIRECTS` now also maps `/campaigns`, `/automation`, `/follow-ups`, `/workflows` → `/dashboard` (comment: "صفحات انشالت بقرار المالك (تركيز على الردود والطلبات فقط)"). `/workflows` previously went to `/automation`.
- New route `/customers/[id]` has no nav entry; it belongs to "العملاء" by prefix (tab stays highlighted, inherits the `/customers` guide).
- `nav-search.ts` (98): keywords for the three removed pages deleted; nothing added.
- `page-guides.ts` (150): entries for `/campaigns`, `/automation`, `/follow-ups` deleted; `/catalogs` next step is now "شوف الطلبات" → `/orders`. Stale texts left in live: `/analytics` still lists "…الحملات" among its tabs; `/customers` still says "اضغط على عميل لتفتح محادثته، ومنها بتشوف طلباته وملاحظاتك عليه." although a click now opens the profile page.
- `guided-tour.ts`: 9 steps (the "الأتمتة" step removed); sales text is now "الطلبات والمنتجات والكتالوجات — كل شي عن البيع."
- `i18n.ts` (533): new labels `EMPTY` "فارغة", `COLLECTING` "قيد الجمع", `CONFIRMING` "قيد التأكيد", `CONVERTED` "تحوّلت لطلب" (draft states — note "قيد التأكيد" is now both the draft state `CONFIRMING` and the order status `CONFIRMATION_PENDING`); audit actions `oms.order_handed_off` "نزل بالـ OMS", `oms.auto_handoff_refused` "ما نزل بالـ OMS".
- Shell (`app-shell`, `app-sidebar`, `group-tabs`): **no change at all**. The sidebar therefore shows 5 main groups + الإعدادات; the top bar, search, bell, "?" and tour button are as in Part A. `nav-icons.ts` still registers the now unused `Workflow` icon.
- Route-level skeletons now exist for two routes: `conversations/loading.tsx` (title bar + 6 list rows + thread block) and `orders/loading.tsx` (title + 5 tiles + table block). Still no `error.tsx` / `not-found.tsx`.

## E.2 Screens removed from the product
| Route | Live behaviour | What was there (Part B) |
|---|---|---|
| `/automation` | redirect → `/dashboard` | rule list, quick rule, step builder, simulation, run log (B.22) |
| `/workflows` | redirect → `/dashboard` | (was redirect → `/automation`) |
| `/follow-ups` | redirect → `/dashboard` | scheduled-message log with cancel (B.24) |
| `/campaigns` | redirect → `/dashboard` | segments, campaign builder, validate/approve/schedule (B.14) |
Left behind in live (verified in the unchanged files):
- `/analytics` still has the tabs "الحملات" and "الأتمتة", the tile "تنفيذات أتمتة" and the follow-up buckets (`analytics/page.tsx` unchanged) — reports for features that no longer have screens.
- The role matrix on `/settings/team` still lists "يشغّل الأتمتة والمتابعات", "ينشئ الحملات", "يعتمد الحملات وينشرها".
- `components/inbox/marketing-card.tsx` still exists but **nothing renders it** — marketing consent can no longer be viewed or recorded anywhere in the UI.
- All API routes for automations, workflows, follow-ups, campaigns and segments still exist. Rules or campaigns created before the removal can no longer be seen, paused or deleted from the UI (whether any exist or still execute in production is **UNVERIFIED**).

## E.3 `/conversations` — inbox (live): `inbox.tsx` 963 lines (+54), `customer-panel.tsx` 296 (+89), `message-bubble.tsx` 118, `types.ts` 146, `use-realtime.ts` 71
Everything in B.3 that is not listed here is unchanged (layout, 10 filters, list rows, header actions, banner, activity log, composer, MediaPicker, coach dialog, permission blindness).

### E.3.1 Realtime replaces the 20 s poll
- `useRealtime(onChange, {onStatus})` opens `new EventSource("/api/events")`. Server: `GET /api/events` [`conversations.read`], `text/event-stream`, heartbeat every 15 s, workspace-scoped, payload ids only: `{type, conversationId, entityId, at}`. Event types published in code: `message.inbound`, `message.outbound`, `conversation.updated`, `conversation.coded`, `order.created`, `order.telegram`.
- Reconnect: on `error` the hook reports "closed", closes the source and retries after `min(30 s, 1 s × 2^n)` (2 s, 4 s, 8 s, 16 s, 30 s …). No `EventSource` support → status stays "closed".
- Inbox handler (`inbox.tsx:196-208`): on **every** change event → `fetchList()` (page 1 with the current filters); if a thread is open and the event is for it (or carries no conversation id) → `GET …/messages`; and only when `type === "message.inbound"` → `loadDraft()`.
- Fallback timer (`inbox.tsx:210-218`): `setInterval` of **60 000 ms while the stream is open, 5 000 ms while it is closed** → `fetchList()` + messages of the open thread (not the drafts, not the customer).
- New header badge, first item of the thread's action cluster: "مباشر" (green) / "غير متصل" (amber). It only exists while a conversation is open, and "غير متصل" does not say what is disconnected.
- Consequences that follow from the unchanged `fetchList()`: the list is now reset to page 1 on every workspace event (not just every 20 s), so rows loaded through "تحميل المزيد" disappear sooner; the deep-link/blank-thread case of B.3.1 is unchanged.

### E.3.2 Order drafts in live — every label and action
- State is now a list: `orderDrafts: DraftInfo[]`. `loadDraft()` still calls `GET /api/order-drafts?conversationId={id}` [`orders.read`] and keeps the drafts whose `state` is `EMPTY`, `COLLECTING`, `READY` or `CONFIRMING` (cancelled/converted ones are no longer shown). The schema now allows several drafts per conversation (`@@unique([conversationId, sequence])`) and each draft has `items Json` (multi-product basket) — so "several drafts" and "several items in one draft" both exist.
- Refresh: on selecting the conversation, and on an SSE `message.inbound` for that conversation. Not on `order.created`, `message.outbound`, the fallback timer, takeover or resolve — a draft that the customer just confirmed (now `CONVERTED`) stays on screen until the next customer message or re-selection.
- Display (`customer-panel.tsx:174-181, 233-287`): the cards are stacked under "آخر الطلبات" with no group heading; every card has the same title `t.inbox.draftTitle` "مسودة طلب" + `StatusBadge` (now Arabic: "فارغة", "قيد الجمع", "جاهزة", "قيد التأكيد"). Rows: one per item "المنتج: {name} × {qty}"; "الهاتف: ✓ | —"; "العنوان: ✓ | —". Progress = filled ÷ `max(requiredFields.length || 4, missing)`. Footer: "الحقول الناقصة: …" with keys translated by `draftMissingLabel` (المنتج · الكمية · الهاتف · العنوان · اسم العميل), or `t.inbox.draftReady` "البيانات مكتملة وجاهزة للتأكيد".
- Still not shown: the phone number and address themselves, the customer name on the draft (it only appears in the missing list when the page requires it), notes, price/total.
- **Actions: still none.** The only UI reference to the draft API in the live tree is the list `GET` (`inbox.tsx:130`); `POST /api/order-drafts/{id}/confirm` and `/cancel` [`orders.write`] remain without a caller, and there is still no draft-edit endpoint or `POST /api/orders`.
- How a draft becomes an order in live (server code, for context): when the draft becomes ready the AI asks "هل أؤكد الطلب؟"; a customer "yes" runs `createOrderFromDraft` (`orders/intelligence.ts:187`), a "no" cancels the draft. The order is then pushed to the Telegram group and (if enabled) to the OMS. A human still has no button for any of these steps. Note a copy/code mismatch: the Telegram card says "كل مسودة طلب تكتمل بياناتها … تُرسل تلقائيًا" while `order-service.ts:419` states the forward happens once the official order is created ("المسودة ما بتترحّل").

### E.3.3 Message bubbles
- The inbound-only "الذكاء الاصطناعي" tag is removed. Every **outbound** bubble now ends with an internal stamp line: 🤖 "الذكاء الاصطناعي" when `senderType === "AI"`, otherwise 👤 `{senderName ?? "الموظف"}` (`message-bubble.tsx:83-96`; `MessageItem.senderName` is new). This fixes D.9 #3.
- Unchanged: media still a text link, delivery shown as "تم الإرسال" only, retry link.

### E.3.4 Customer panel (live order of blocks)
1. Identity card + new text button "تحديث الاسم والصورة" ("جارٍ التحديث…") → `POST /api/customers/{id}/refresh-profile` [`customers.write`, 5/min] then `GET /api/customers/{id}`; the server's message (or "تم التحديث." / "تعذّر التحديث." / "تعذّر الاتصال بالخادم.") is printed under it. Rendered only in the `xl` side panel, not in the drawer.
2. "بيانات العميل" — unchanged, read-only.
3. "آخر الطلبات" — unchanged.
4. Draft cards (E.3.2).
5. Ghost button "حذف ذاكرة المحادثة" → confirm: "حذف ذاكرة المحادثة؟" / "بينمسح اللي تعلّمه الذكاء من هالمحادثة: الذاكرة المستخرجة وملخّص المحادثة. الرسائل والطلبات وملاحظات الفريق بتضل. ما بيرجع." / "حذف الذاكرة" → `POST /api/conversations/{id}/memory/reset` [`conversations.delete`]; toasts "انحذفت ذاكرة المحادثة" / "تعذّر حذف ذاكرة المحادثة". The source comment says "المدير فما فوق" but the button is rendered for every role.
6. "النشاط الأخير" — unchanged.
**Removed** from the panel: the `MarketingCard` ("الرسائل التسويقية", consent buttons, ad/clipper attribution) and the "ملخص الذكاء الاصطناعي" block. There is no link from the panel to the new profile page `/customers/{id}`.
Controls in the open inbox (xl): ≈24 as before (−2 consent buttons, +2 new).

## E.4 Orders (live)
### `/orders` — `orders/page.tsx` (283, client) + `orders/loading.tsx` (16)
As B.7, plus:
- Realtime: `useRealtime` reloads the list on events whose type starts with `order` (`order.created`, `order.telegram`), at most once every 4 s; plus a `setInterval` reload every **60 000 ms**.
- New table column "تلجرام" (between الحالة and التاريخ): badge "تم ترحيله للتلجرام" (green) or "لم يُرحّل" (amber); on mobile cards "تم ترحيله للتلجرام" / "لم يُرحّل للتلجرام". Data: `telegramNotified` per order from `GET /api/orders`. Read-only — no "send again" action.
- Unchanged: 25-row limit with no pagination, counters computed from the loaded rows, client-side search, "طباعة", owner-only delete, no create.
### `/orders/[id]` — `orders/[id]/page.tsx` (301) + `components/orders/oms-order-card.tsx` (113)
As B.8, plus a card at the bottom of the right column — `OmsOrderCard` ("الـ OMS"), rendered only if the OMS is linked or the order was already sent:
- `GET /api/orders/{id}/oms` [`orders.read`] on mount (no polling).
- Sent: "رقم الطلب بالـ OMS" + number; "الحالة هناك: {raw status}"; "انبعت: {date}"; if the OMS priced it differently: red "الـ OMS سعّر الطلب غير سعرنا — المجموع المعتمد بالـ OMS: n CUR" + orange "بلّغ الزبون بالسعر الصحيح قبل الشحن، وتأكد إن سعر المنتج بالـ OMS هو المقصود."
- Not sent: "بالطريق للـ OMS…" / "الـ OMS ما رد — بنعيد المحاولة (n)" / "لسا ما انبعت للـ OMS."; on failure red "الطلب ما نزل بالـ OMS" + raw `lastError` + orange "صلّح السبب (رقم الزبون · المنتج موجود بالـ OMS) وبعدين اضغط «ابعته للـ OMS»."
- Cancelled/failed order: "الطلب ملغي — ما بينبعت للـ OMS."
- Button "ابعته للـ OMS" → `POST /api/orders/{id}/oms` [`orders.write`, 20/min], shown only when not yet sent, not cancelled, OMS connected and the API's `canSend` is true (permission-aware).
- Timeline can now show "نزل بالـ OMS" / "ما نزل بالـ OMS".
- Unchanged: status buttons named after states, `canWrite` still ignored, nothing editable.

## E.5 Customers (live)
### `/customers` — `customers/page.tsx` (176)
Identical to B.5 except the name link and the mobile card now go to **`/customers/{id}`** instead of the inbox search. Still 25 rows, client-side status filter, misleading empty state.
### `/customers/[id]` — NEW — `customers/[id]/page.tsx` (441, client)
Purpose: customer profile — edit the basics, review what the AI extracted. Data: `GET /api/customers/{id}` [`customers.read`] (returns `canWrite`, `canConfirm`, `intel[]`, `fieldHistory[]`, conversations, orders). No polling/realtime.
1. `PageHeader`: name or "بدون اسم" / "{status} · آخر نشاط: {relative time}"; action link "← العملاء".
2. Amber error strip ("تعذّر الحفظ (تحقق من الصلاحية أو الرقم المكرر)", "تعذّر التأكيد (تحقق من الصلاحية)").
3. Left card: avatar, name, status badge; "بيانات العميل" — if `canWrite`: five labelled inputs "الاسم", "الهاتف", "المدينة", "المحافظة", "العنوان" + button "حفظ" → `PATCH /api/customers/{id}` [`customers.write`]; otherwise the same data read-only; channel identity badges.
4. Right column, in order:
   - "معلومات مؤكدة (n)" (empty: "ما في معلومة مؤكدة بعد.")
   - "معلومات بانتظار المراجعة (n)" (empty: "ما في معلومات مستخرجة بانتظار المراجعة.")
   - "تعارضات (تحتاج حسمًا يدويًا) (n)" (only when conflicts exist)
     Each fact row: field label (الاسم, رقم الهاتف, المدينة, العنوان, الطلبية, المنتج, الكمية, الميزانية / السعر المذكور, نية الشراء, الاحتياجات, ملاحظات مستخرجة), value, "المصدر: {message ids or conversation-id prefix}", "قيم متعارضة: …"; if `canConfirm` a link-button "تأكيد/تصحيح" → inline `Input` + "حفظ" → `POST /api/intelligence/conversations/{conversationId} {action:"confirm", key, value}` [`conversations.reply`].
   - "سجل التعديلات (n)": field · "بواسطة {first 8 chars of the user id}" or "تلقائي" · relative time · "من: … · إلى: …".
   - "المحادثات (n)": rows (status, summary or AI-state label, time) → `/conversations?id={id}`; empty "ما في محادثات مرتبطة."
   - "آخر الطلبات": rows → `/orders/{id}`; empty "لا توجد طلبات لهذا العميل بعد."
States: skeleton card; not-found "العميل غير موجود" + "إعادة المحاولة".
Permission checks: `canWrite` and `canConfirm` from the API (computed from the role only).
Loaded but not rendered: `email`, `notes`, `tags`, `aiSummary`. Not available: block/unblock, tags, notes, consent, delete, "new conversation". Raw ids are shown ("بواسطة clx12abc", source ids).
Controls: 5 inputs + save + one "تأكيد/تصحيح" per fact + row links.
This page resolves "customers → inbox search" (D.7 #6) and gives `PATCH /api/customers/{id}` a UI, but it overlaps `/intelligence` (same confirm endpoint; there via `window.prompt`).

## E.6 `/dashboard` (live) — `dashboard/page.tsx` (208)
As B.2, with two changes: (1) `useRealtime` → on any event the overview is re-fetched, at most once every 15 s (no interval timer); (2) the "الأتمتة" section is removed — "الدعم البشري" now sits alone in the two-column row. Still UTC by default, 6 KPI + 12 mini numbers, nothing actionable.

## E.7 `/channels` (live) — `channels/page.tsx` (316) + three new sections
Order on screen now: header → message strip → (page picker after OAuth) → "Facebook Messenger" connect card → **"ربط الـ OMS"** → **"إرسال الطلبات الجاهزة إلى تيليجرام"** → "قنوات إضافية" → one card per Facebook page (stats, persona, test, disconnect, catalog, **"بيانات إكمال الطلب لهذه الصفحة"**) → empty state. So two integration cards that are not customer channels now sit between the Facebook connect button and the connected pages.

### `OmsSection` — "ربط الـ OMS" (`oms-section.tsx`, 224)
`GET /api/settings/oms` [`settings.manage`]; the card is **not rendered** on 401/403.
- Text: "كل طلب مؤكد بينزل بالـ OMS، والمنتجات والأسعار والعروض بتيجي منه كل ساعة. التوكن من الـ OMS: الإعدادات ← التطبيقات ← «توكنات الأنظمة»."; checkbox "مفعّل".
- Fields: "رابط الـ OMS" (placeholder `https://oms.example.com`); "توكن الـ OMS" (`PasswordInput kind="secret"`, placeholder "zk1_..." or "محفوظ — اتركه فاضي للإبقاء عليه").
- Checkbox "ابعت كل طلب مؤكد للـ OMS تلقائيًا"; when ticked: red "مؤتمت: الطلب بينزل بالـ OMS أول ما يتأكد مع الزبون — بلا مراجعة منك / اسم الزبون ورقمه وعنوانه بيروحوا للـ OMS." + orange "إذا بدك تراجع كل طلب قبل الشحن، طفّي هالخيار وابعت كل طلب من صفحته بزر «ابعته للـ OMS»."
- Always-visible red "منتجات الردود صارت منتجات الـ OMS — المزامنة (كل ساعة، أو بالزر تحت) بتوقّف أي منتج هون مش موجود بالـ OMS — ما بينحذف، بيرجع من صفحته. …الجديد بينضاف لكل الصفحات." + orange "أي منتج جديد ضيفه بالـ OMS أول. منتج جديد ما بدك ياه بصفحة معيّنة: شيله من كتالوجها."
- Buttons: "حفظ" → `PUT /api/settings/oms {baseUrl, isActive, autoHandoff, token?}` (message "انحفظ — اضغط «اختبر الاتصال» تتأكد."); "اختبر الاتصال" → `POST /api/settings/oms/test`; "استبدل المنتجات بمنتجات الـ OMS هلأ" → confirm "استبدل المنتجات بمنتجات الـ OMS؟" / "المنتجات الموجودة بالـ OMS بتتحدّث أو بتنضاف، وأي منتج هون مش بالـ OMS بيتوقّف (ما بينحذف — بيرجع من صفحته)." / "استبدل هلأ" (not styled as dangerous) → `POST /api/settings/oms/sync`; result "وصل n منتج من الـ OMS · تحدّث n · انضاف n · توقّف n · فشل n — {reason}".
- Footer "آخر مزامنة منتجات: {date} · آخر خطأ: {raw}". The two checkboxes only take effect after "حفظ".

### `TelegramOrderSection` — "إرسال الطلبات الجاهزة إلى تيليجرام" (`telegram-order-section.tsx`, 187)
`GET /api/settings/telegram-orders` [`settings.manage`]; hidden on 401/403.
- Text: "كل مسودة طلب تكتمل بياناتها (حسب إعداد الصفحة) تُرسل تلقائيًا لمجموعة تيليجرام — مجموعة وتوبيك تحدّدهم هنا."; checkbox "مفعّل".
- Fields: "توكن بوت تيليجرام" (secret; "123456789:AA..." / "محفوظ — اتركه فارغًا للإبقاء عليه"); "معرّف المجموعة (chat_id)" ("-1001234567890"); "معرّف التوبيك (topic_id — اختياري)" ("مثال: 42").
- Amber consent checkbox: "أوافق على إرسال بيانات الطلب (اسم الزبون · هاتفه · عنوانه) إلى مجموعة تيليجرام المحددة. تأكد أن المجموعة خاصة وبيد فريقك فقط." — local state only, so it has to be ticked again on every save while enabled ("لتفعيل الإرسال لازم توافق على إرسال بيانات الطلب إلى تيليجرام.").
- Buttons: "حفظ إعدادات تيليجرام" → `PUT /api/settings/telegram-orders {enabled, chatId, topicId, botToken?}` ("تم التفعيل — الطلبات الجاهزة رح توصل لتيليجرام." / "تم الحفظ (الإرسال متوقف)."); "إرسال رسالة اختبار" → `POST …/test` ("تم الإرسال ✅ (البوت: @name) — شوف المجموعة/التوبيك.").
- This is a second Telegram bot form on the same page (the customer-chat bot is the "تيليغرام" row of "قنوات إضافية").

### `OrderFieldsSection` — "بيانات إكمال الطلب لهذه الصفحة" (`order-fields-section.tsx`, 95) — inside every Facebook page card
`GET /api/channels/by-id/{id}/order-fields` [`channels.read`]. Text "المنتج والكمية مطلوبان دائمًا. اختَر شو كمان لازم يتعبّى قبل ما تصير المسودة جاهزة:"; checkboxes "رقم الهاتف", "العنوان", "اسم الزبون"; "حفظ" → `PUT … {fields:["product","quantity",…]}` [`settings.manage`]; "تم الحفظ ✓". Facebook pages only — other channels cannot be configured.
Controls on `/channels` now: FB connect 1 + OMS 7 + Telegram orders 7 + other channels 3–10 + per Facebook page 10 (persona select + save, test, disconnect, catalog select + switch, 3 checkboxes + save) ≈ **30+** for a single-page shop.

## E.8 `/settings/ai` (live) — `page.tsx` (571) + `why-no-reply-card.tsx` (154)
As B.21, with:
1. New card between the status card and "التشغيل": **"ليش ما رد؟"** — `GET /api/analytics/queue` [`analytics.read`]; button "تحديث"; counters "بالانتظار: n · عم تشتغل: n · فشلت: n"; list of the most frequent failed jobs: job name (توليد رد على رسالة زبون · استقبال رسالة من القناة · استقبال حدث من القناة · ترحيل طلب لتيليغرام · ترحيل طلب للـ OMS · مزامنة منتجات الـ OMS · استخراج بيانات الزبون · تنزيل وسائط واتساب, or the raw job type) + "n مرة" + a red explanation and orange fix for seven recognised causes (guardrail rejection, timeout, rate limit, bad API key, 24-hour window, expired page token, outbound delivery); unrecognised causes print the **raw reason string** in red. Empty: "ما في فشل مسجّل. إذا في رسالة بلا رد، افتح المحادثة وشوف آخر رسالة وصلت." Error: "ما قدرنا نقرأ حالة الطابور هلأ."
   It lists workspace-wide job failures only; it does not check the configuration causes of D.7 #1 (page in collection mode, no catalog, conversation taken over, AI toggles). One fix text refers to «اختبر الاتصال» while the button on this page is "فحص الاتصال".
2. Connection test now has two buttons: "فحص الاتصال" and "فحص التوليد" (`POST /api/ai/providers/test {provider, mode:"link"|"generate"}`), results "متصل · 120ms" / "التوليد شغّال · 1.4s", caption "زمن الشبكة والمفتاح" / "زمن توليد ٥ توكنات — مش زمن الرد الحقيقي".
3. "أقصى طول للرد": hint now "بالتوكنز — بالعربي كل ١٠٠ ≈ ٥ كلمات فقط"; below 300 a red note "الحد قصير عالعربي — الرد بينقطع بنص الكلمة، والنظام بيرفضه ويعيد المحاولة: بطء بالرد أو ما بيوصل رد أبدًا." + orange "خليه ٨٠٠ أو أكتر. رد مبيعات عربي عادي بياخد ٢٠٠ وحدة."
Unchanged: one big form, toggles saved only by the bottom button, provider jargon, advanced block.

## E.9 Smaller page changes (live)
- **`/knowledge`** (518): new "خاص بمنتج (اختياري)" `Select` in the create form and the edit dialog, and a scope `Select` next to the file upload (options "عام لكل الصفحات" + active products from `GET /api/products?limit=100&status=active`); `productId` sent with create/upload/patch. Everything else as B.16 (including the "من نص" button and no auto-refresh).
- **`/products/[id]`** (582): save result is shown beside each "حفظ" button ("تم الحفظ ✓" or an Arabic reason from `lib/product-save-error.ts`, e.g. "ما عندك صلاحية تعديل المنتجات. اطلب من المدير منحك صلاحية كتالوج المنتجات.", "تعذّر الحفظ — {field} (السطر n): …", "تعذّر الاتصال بالخادم — ما انحفظ شي. جرّب مرة ثانية."); product and media load in parallel. The top error strip and the header "تم الحفظ" badge are still there, so an error appears twice, and the one `error` state shows next to the save buttons of the other tabs too. Still no stock inputs, tabs still drop unsaved edits.
- **`/products`** (379): unchanged — "منتج جديد" and "استيراد اكسل" are still offered even though the OMS card states that the OMS is now the source of products.
- **`/catalogs`** (254): deleting the last catalog that still has pages returns 409 → a native `window.confirm(server message || "هذا آخر كتالوج وفيه صفحات مربوطة.")` → `DELETE /api/catalogs/{id}?force=1`.
- **`/catalogs/[id]`** (452): null-safe rows — "منتج غير موجود" / "المنتج ما عاد موجود — أزل الارتباط."; "صفحة غير موجودة" / "الصفحة ما عادت موجودة — فك الربط."
- **`/automation`, `/campaigns`, `/workflows`, `/follow-ups`**: redirect stubs (E.2).

## E.10 Part C re-run on the live tree
Same `grep` patterns on both trees (`<Name` followed by space, `>`, `/` or end of line; `src/app` + `src/components`, `*.tsx`; "outside `ui/`" in brackets):
| Pattern | Audited | Live |
|---|---|---|
| `<Button` | 167 (164) | **149 (146)** |
| raw `<button` | 69 (62) | **69 (62)** |
| `<Input` / `<PasswordInput` | 84 (82) / 8 | **73 (71) / 10** |
| raw `<input` | 13 (12) | **17 (16)** — 11 checkboxes, 4 file inputs, top-bar search |
| `<Select` / raw `<select` | 51 / 3 (2) | **45 / 3 (2)** |
| `<Textarea` / raw `<textarea` | 17 / 4 (3) | **14 / 3 (2)** |
| `<Field` | 97 | **87** |
| `<table>` (`.data-table`) | 5 | **5** |
| `<Dialog` / `<ConfirmDialog` / `<Drawer` | 20 (19) / 19 / 1 | **15 (14) / 18 / 1** |
| `window.confirm` / `window.prompt` | 1 / 1 | **2 / 1** (new: `catalogs/page.tsx:123`) |
| `<details>` | 5 | **5** |
| `<Badge` / `<StatusBadge` | 44 (43) / 20 | **47 (46) / 20** |
| `<RiskNote` / `<FixNote` (red/orange notes) | 5 / 5 | **12 / 11** |
| `toast.show` calls / files using toasts | 48 / 12 | **38 / 9** |
| `<GlassCard` / `<PageHeader` / `<StatCard` | 119 / 24 / 17 | **119 / 22 / 17** |
| `<EmptyState` / `<EmptyNext` / `<ErrorState` | 40 / 6 / 11 | **38 / 5 / 8** |
| hand-rolled tab strips / `<Tabs` | 7 / 1 | **7 / 1** |
| fixed palette classes / semantic token classes | 497 / 986 | **497 / 978** |
| `text-xs` · `text-[11px]` · `text-[10px]` · `text-[13px]` · `text-[12px]` | 196 · 104 · 40 · 31 · 7 | **207 · 112 · 44 · 30 · 2** |
| `t.*` dictionary references | 331 | **329** |
Reading: the share of raw buttons rose (62 raw vs 146 shared, was 62 vs 164) because the deleted pages mostly used the shared `Button`; raw checkboxes grew from 7 to 11 (all the new settings cards use bare `<input type="checkbox">`); red/orange notice blocks more than doubled; inline text feedback replaced toasts in every new card.
New inconsistencies introduced by the live-only files (verified: no definition in `globals.css` or `tailwind.config.ts`, which are identical in both trees):
- `text-muted-foreground` (3 uses, `why-no-reply-card.tsx:108,124,135`) and `text-base font-semibold` headings — class names from another design system; the colour utility does not exist here.
- `accent-[var(--color-primary)]` on 5 checkboxes (`oms-section.tsx:153,175`, `telegram-order-section.tsx:127,157`, `order-fields-section.tsx:83`) — the variable `--color-primary` is not defined (the project's variable is `--primary`).
- Three spellings of Telegram in the UI: "تيليغرام" (channel list, labels), "تيليجرام" (order-forwarding card, job names), "تلجرام" (orders table).
- Three labels for the same action: "فحص الاتصال" (AI), "اختبر الاتصال" (OMS), "اختبار الاتصال" (Facebook page) — plus "فحص التوليد" and "إرسال رسالة اختبار".

Largest `.tsx` files in the live tree (`wc -l`; all `.tsx` = 14 439 lines):
| # | File | Lines |
|---|---|---|
| 1 | `src/components/inbox/inbox.tsx` | 963 |
| 2 | `src/app/(dashboard)/intelligence/page.tsx` | 854 |
| 3 | `src/app/(dashboard)/analytics/page.tsx` | 613 |
| 4 | `src/app/(dashboard)/products/[id]/page.tsx` | 582 |
| 5 | `src/app/(dashboard)/settings/ai/page.tsx` | 571 |
| 6 | `src/app/(dashboard)/media/page.tsx` | 528 |
| 7 | `src/app/(dashboard)/knowledge/page.tsx` | 518 |
| 8 | `src/app/(dashboard)/catalogs/[id]/page.tsx` | 452 |
| 9 | `src/app/(dashboard)/customers/[id]/page.tsx` | 441 |
| 10 | `src/app/(dashboard)/ai/playground/page.tsx` | 429 |
| 11 | `src/components/app-shell.tsx` | 407 |
| 12 | `src/app/(dashboard)/products/page.tsx` | 379 |
| 13 | `src/app/(dashboard)/settings/team/team-view.tsx` | 378 |
| 14 | `src/app/(dashboard)/personas/page.tsx` | 372 |
| 15 | `src/app/(dashboard)/channels/page.tsx` | 316 (+ 224 + 187 + 195 + 142 + 95 in its section files = 1 159 for the screen) |

## E.11 Part D re-assessed for the live tree
**Fixed or no longer reachable in live**
| Part D item | Live status |
|---|---|
| Raw draft enums and English missing-field keys (D.6, D.9 #6) | translated |
| AI vs human bubbles identical (D.9 #3) | fixed — sender stamp on every outbound bubble |
| Only refresh = 20 s poll; draft never refreshed | SSE + 60 s/5 s fallback; drafts reload on inbound message (not on other events) |
| Customers click → inbox search; no customer editing (D.3, D.7 #6) | profile page with editable basics and inline confirm |
| Campaign/rule/follow-up problems: two rule builders, "جدولة" without a time, "معرّف الموظف", errors in the wrong card, two "إلغاء" buttons, dormant nav tabs (D.1 #10, D.3, D.7 #5/#11, D.8) | screens removed (APIs remain) |
| Product save feedback invisible | message next to the button, Arabic reasons |
| No explanation when the AI stays silent (D.7 #1) | partly — "ليش ما رد؟" lists failed jobs; configuration causes still spread over four pages |
| Catalog detail crash on missing product/page | null-safe rows |
| No route skeletons | `loading.tsx` for `/conversations` and `/orders` only |

**Still true in live** (files unchanged or logic untouched): navigation-only permissions and write buttons for everyone (D.4); 25/30/50-row truncation and counters from loaded rows (D.9 #1); inbox list reset to page 1 and the blank thread on deep links (D.9 #2 — now triggered by every realtime event); Facebook icon on every channel (#4); stale sales-coach suggestions (#5); disabled emoji button; no assignment UI; order status buttons named after states and `canWrite` ignored; invalid `Badge` tones (#7); product badge wording (#8); `/media` error texts (#9); account ✓ icon (#10); `/handoffs` without customer names and with user ids; `/intelligence` as a second inbox with `window.prompt`; `/analytics` without agent names and with raw keys; persona edit vs versions; playground developer options; invite that cannot be accepted; AI-settings save model; persona/catalog/order-fields for Facebook pages only; UTC default on dashboard and analytics; all wording inconsistencies of D.5; **the order draft still has no confirm/cancel/edit action**.

**New in live**
| # | Observation | File |
|---|---|---|
| 1 | `/channels` now mixes four kinds of things — customer channels, OMS integration, Telegram order forwarding, per-page order rules — ≈30 controls and up to five red/orange notices on one screen | `channels/page.tsx:229-285` |
| 2 | Two sources of truth for products: the OMS card says products come from the OMS hourly and local-only products get deactivated, while `/products` still offers "منتج جديد", "استيراد اكسل" and price editing with no hint that the OMS is linked (what exactly a sync overwrites is **UNVERIFIED**) | `oms-section.tsx:188-192`, `products/page.tsx` |
| 3 | The order pipeline is automated end to end (draft → customer "yes" → order → Telegram → OMS) but the agent in the inbox can only watch: several look-alike "مسودة طلب" cards, no action, converted drafts linger until the next inbound message | `customer-panel.tsx:174-181`, `inbox.tsx:128-142,196-208` |
| 4 | Telegram card copy ("كل مسودة … تُرسل تلقائيًا") contradicts the server comment (orders, not drafts, are forwarded) | `telegram-order-section.tsx:118-120`, `modules/orders/order-service.ts:419` |
| 5 | Jargon added: "الـ OMS" (never expanded), "توكنات الأنظمة", "zk1_...", "chat_id", "topic_id", "التوبيك", raw `omsStatus`/`lastError`, raw queue failure reasons | `oms-section.tsx`, `telegram-order-section.tsx`, `oms-order-card.tsx`, `why-no-reply-card.tsx:143` |
| 6 | "حذف ذاكرة المحادثة" shown to every role (needs `conversations.delete`); "تحديث الاسم والصورة" missing in the drawer | `customer-panel.tsx:93-108,183-190`, `inbox.tsx:853-868` |
| 7 | Marketing consent and the AI summary disappeared from the inbox panel; the profile page loads but does not show email, notes, tags or the AI summary; no link between inbox panel and profile | `customer-panel.tsx`, `customers/[id]/page.tsx:32-53` |
| 8 | Reports, role matrix and one guide still describe campaigns/automation; existing rules have no UI | `analytics/page.tsx:45-55`, `role-matrix.tsx:51-53`, `page-guides.ts` |
| 9 | Telegram consent checkbox must be re-ticked on each save; the OMS/Telegram "مفعّل" checkboxes look instant but need their own save button — the page now has five independent save actions (OMS "حفظ", "حفظ إعدادات تيليجرام", and per Facebook page: persona "حفظ", catalog "تبديل", order fields "حفظ") | `telegram-order-section.tsx:30,58-62` |
| 10 | Realtime badge "غير متصل" is ambiguous and only visible inside an open thread | `inbox.tsx:654-657` |
| 11 | A third native dialog (`window.confirm` for forced catalog delete) | `catalogs/page.tsx:123` |
| 12 | Undefined utilities/variables and three Telegram spellings (E.10) | new section files |
| 13 | Dashboard row left with a single card in a two-column grid | `dashboard/page.tsx:198-203` |
| 14 | `inbox.tsx` grew to 963 lines / 34 `useState` calls; realtime, profile refresh and memory reset were added to the same component | `inbox.tsx` |
