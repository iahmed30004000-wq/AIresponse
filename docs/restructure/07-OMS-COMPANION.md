# 07 — التعديلات المطلوبة بالـ OMS (ريبو تاني — حزمة مستقلة)

> هذا الملف **مش** للبنّاء اللي بيشتغل على الردود. هو مواصفة لجلسة منفصلة على ريبو الـ OMS، بقواعد ذاك الريبو (`AGENTS.md` تبعه — خصوصًا: `contracts/oms-app-events.v1` ما بينلمس بلا موافقة المالك).
> المصدر: قراءة `iahmed30004000-wq/zaki-ai-OSM@main` (`6bfe0cb`) — وهو **الأصل الوحيد** حسب `AGENTS.md` تبع الـ OMS نفسه (الريبوهين `MKory18/Zaki-ai-CRM` و`MKory18/zaki-ai-OSM-1` نسخ قديمة ما بينشتغل عليها). ⚠️ `AGENTS.md` بالردود لسا بيقول العكس — بيتصحح بالحزمة 0.2.
> قواعد ذاك الريبو اللي بتحكم هالشغل: فرع + PR وولا رفع على `main` · «تعريف الإنجاز» بأول كل PR · الدمج مش نشر (المالك بيكبس Redeploy) · ولا قبول لإعادة تهيئة قاعدة البيانات · ولا بيانات زبون لموديل خارجي.
> الردود **بتشتغل من أول يوم بلا هالتعديلات** (مسار «ملاحظة + موظف» بـ `02-ORDER-FLOW.md` §10.3-ج). هدول بيحوّلوا التعديل والإلغاء من يدوي لآلي.

## 1. الموجود اليوم بباب `/api/v1` (للعلم)

| المسار | الصلاحية | ملاحظة |
|---|---|---|
| `POST /api/v1/orders` | `orders.create` | الطلب بيبدأ `confirmationStatus: NEW` · `shippingStatus: NOT_READY` · الـ OMS بيعيد التسعير · `delivery_fee` **بينتجاهل** (`shippingCost = 0`) · `Idempotency-Key` إلزامي |
| `GET /api/v1/orders/{id}` | `orders.read` | `{order_id, order_number, status (= confirmationStatus), shipping_status, updated_at}` |
| `POST /api/v1/orders/{id}/notes` | `orders.note` | `{kind, note}` — الأنواع: `follow_up` · `return` · `settlement` · `internal` |
| `GET /api/v1/customers/lookup?phone=` | `customers.lookup` | `{exists, blacklisted, orders_count, delivered_count, total_value}` |
| `GET /api/v1/products` | `products.read` | الرمز · الاسم · السعر · العروض · `?fields=aliases` |

**الناقص:** ولا مسار تعديل أو إلغاء لطلبات الباب. الموجود (`/api/v1/landing/orders/{n}/{cancel|address|confirm}`) خاص بطلبات صفحات الهبوط وبتوكن محادثة 72 ساعة.

## 2. المطلوب — 4 إضافات صغيرة، كلها إعادة استعمال

### 2.1 `POST /api/v1/orders/{id}/amend` — صلاحية جديدة `orders.amend`

```json
{ "customer": { "name": "…", "phone": "…", "city": "…", "address": "…" },
  "items": [{ "product_code": "LEG-21", "quantity": 2 }],
  "notes": "…", "reason": "العميل غيّر الرقم بالمحادثة" }
```
كل الحقول اختيارية؛ الموجود بيستبدل، والغايب بيضل.

- **مين بيعدّل شو:** النظام بيعدّل بس طلب هو أنشأه — `order.source` = نظام التوكن (`caller.system`). غير هيك 404.
- **متى:** نفس قاعدة `chatEditable` الموجودة (`src/lib/v1/landing-chat.ts:129`): `confirmationStatus ∉ {CONFIRMED, REJECTED, CANCELLED}` ومش على شحنة (بلا دفعة · بلا رقم تتبع · بلا ملصق · `shippingStatus = NOT_READY`). غير هيك `409 NOT_EDITABLE`. **استخرج `chatEditable` و`onShipment` لملف مشترك واستعملهم من المكانين — ما تنسخهم.**
- **التسعير:** نفس إعادة التسعير بـ `createSystemOrder` (`src/lib/v1/order-intake.ts`) عند تغيير البنود؛ الرد فيه `totals` و`price_adjusted`.
- **الهاتف:** نفس فحص صيغة بلد المتجر والحظر (`422 INVALID_PHONE` · `422 CUSTOMER_BLACKLISTED`).
- **الأثر الظاهر:** `OrderActivity` بنوع واضح («تعديل من المحادثة») فيه الفرق — موظف التأكيد اللي ماسك الطلب لازم يشوفه فورًا.
- **عدم التكرار:** `Idempotency-Key` إلزامي (نفس `src/lib/v1/idempotency.ts`). الردود بتبعت `replies:{workspaceId}:{orderId}:rev{n}`.
- **الرد:** `200 {order_id, order_number, status, totals, price_adjusted}`.

### 2.2 `POST /api/v1/orders/{id}/cancel` — صلاحية جديدة `orders.cancel`

```json
{ "reason": "العميل تراجع بالمحادثة" }
```
- نفس شرطي الملكية والقابلية للتعديل.
- بينهي الطلب بـ `endOrder` كـ `REJECTED` بسبب «العميل غيّر رأيه» — نفس مسار إلغاء محادثة الهبوط (`…/landing/orders/{n}/cancel`).
- كان منتهي أصلًا ← `200 {already: true}`.

### 2.3 `GET /api/v1/orders/{id}` — حقل زيادة

`"editable": true|false` من نفس دالة 2.1. الردود بتقراه بمزامنة الحالة فبتعرف تقول للعميل الحقيقة («بقدر عدّل» / «رح يتواصل معك الفريق») بلا محاولة.

### 2.4 الصلاحيات

`V1_SCOPES` (`src/lib/v1/system-token.ts:32`) += `orders.amend` · `orders.cancel`، و`REPLIES_SCOPES` بياخدهم. التوكنات القديمة بتضل بلا الصلاحيتين (بتاخد `403 SCOPE_MISSING`) — المالك بيصدر توكن جديد للردود من «التطبيقات ← توكنات الأنظمة».

## 3. قرارات للمالك (مش تنفيذ)

| # | القرار | ليش مهم للردود |
|---|---|---|
| ق1 | هل الـ OMS يبعث `order.created` لطلبات الباب (RV55 بالباكلوغ) ويعلن `REJECTED`؟ وتسجيل الردود كتطبيق يستقبل الأحداث الستة | بلاه الردود بتستطلع `GET /orders/{id}` كل 10 دقايق/ساعة — شغّال بس أبطأ |
| ق2 | رسم التوصيل (RV59): الـ OMS بيتجاهل `delivery_fee` | الوكيل بيقول للعميل «التوصيل 2$» والـ OMS بيسجّل 0. إما الـ OMS يقبل الرسم من الردود، أو يرجّع جدول مناطقه ورسومه بمسار قراءة والردود بتعرضه |
| ق3 | نوع ملاحظة جديد `change_request` بـ `NOTE_KINDS` | لحد ما ينبنى 2.1: طلبات التعديل من المحادثة بتوصل اليوم كملاحظة `follow_up` — نوع خاص بيخلّي فريق التأكيد يميّزها |
| ق4 | عميل أعطى رقمه وما كمّل الطلب ← يروح للـ OMS كـ«سلة متروكة»؟ (`POST /api/v1/landing/abandoned-carts` موجود بس مصمّم لصفحات الهبوط) | فرصة مبيعات: فريق الاتصال بيتابع العملاء اللي وصلوا لنص الطريق. بدها مسار خاص بالردود أو قرار باستعمال الموجود |

## 4. الاختبارات (Vitest بالـ OMS — نفس نمط اختبارات الباب)

- تعديل طلب `NEW` أنشأه نفس النظام ← 200 والقيم تغيّرت و`OrderActivity` انكتب.
- تعديل طلب نظام تاني أو شركة تانية ← 404.
- تعديل طلب `CONFIRMED` أو على شحنة ← `409 NOT_EDITABLE` والطلب ما تغيّر.
- نفس المفتاح مرتين ← نفس الرد، تعديل واحد. نفس المفتاح بجسم مختلف ← `409 IDEMPOTENCY_CONFLICT`.
- رقم محظور بالتعديل ← 422.
- تغيير البنود ← إعادة تسعير من كتالوج الـ OMS.
- إلغاء ← `REJECTED`؛ إلغاء تاني ← `already: true`.
- توكن بلا `orders.amend` ← `403 SCOPE_MISSING`.
- `GET` بيرجّع `editable` صح قبل وبعد التأكيد.

## 5. أثرها على الردود لما تجهز

`src/modules/oms/client.ts` بياخد: `amendOrder` · `cancelOrder` · قراءة `editable`. `oms/connection.ts` (فحص الاتصال) بيحفظ `capabilities: { amend: boolean, cancel: boolean }` من تجربة الصلاحيات (أو من رد فحص جديد) — و`orders/amend.ts` بيختار المسار الآلي لحاله. **ولا تغيير بالواجهة.**
