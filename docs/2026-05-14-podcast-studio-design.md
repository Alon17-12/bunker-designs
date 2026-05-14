# מערכת ניהול חדר פודקאסט - מסמך אפיון (Design Spec)

**תאריך:** 2026-05-14
**גרסה:** 1.0
**בעלים:** אלון גבאי
**סטטוס:** Draft לאישור

---

## 1. חזון ומטרה

מערכת **פנימית** לניהול end-to-end של הפקות פודקאסט באולפן יחיד.
מטרת הליבה: **לאחסן, לערוך, לסמן ולמסור** קטעי פודקאסט ללקוח, עם תהליך פידבק חלק ושקוף.

### עקרונות מנחים
- **חוויית לקוח חלקה** - הלקוח מסמן Reels ומשאיר פידבק בלי חיכוך
- **תפעול חכם לצוות** - מה שאוטומציה יכולה לעשות, היא תעשה (Make)
- **ניצול כלים קיימים** - לא להמציא מחדש (Monday לKanban, Supabase ל-DB)
- **התרחבות הדרגתית** - MVP ממוקד, פאזות עתידיות מתוכננות

---

## 2. משתמשים ותפקידים

| תפקיד | MVP | פאזה 2+ |
|---|---|---|
| **Admin (אלון + שותף)** | גישה מלאה: ניהול לקוחות, פרקים, סטטוסים, ייצוא, ניהול ספרייה | + ניהול מסלולים, חיוב |
| **לקוח (Host)** | גישה לפורטל ייעודי: צפייה, סימון Reels, הערות | + ספרייה היסטורית (Retainer) |
| **עורך חיצוני** | אין | רואה מהמכל המוקצה לו, מסמן "סיים", מעלה גרסה |
| **לקוח קצה (סופי)** | אין | אם נפתח portal צרכני |

### היקף שימוש
- **משתמשים פעילים בו-זמנית:** עד 5 (צוות + לקוחות פעילים)
- **לקוחות פעילים:** ~10-20 במקביל
- **פרקים בחודש:** 16-40

---

## 3. מושגי ליבה (Domain Model)

```
Customer (לקוח)
├── Package (מסלול - צילום בלבד / צילום+עריכה / Retainer)
└── Episodes (פרקים)
    ├── Raw Recording (קובץ הקלטה גולמי)
    ├── Edited Version (גרסה ערוכה - v1, v2, v3...)
    ├── SRT (תמלול - פאזה 2)
    ├── Feedback (הערות הלקוח)
    │   ├── Timestamped Comments (הערות בנקודת זמן)
    │   └── General Notes (תיקונים כלליים)
    └── Reels[] (קטעי רילס - 5+ לפרק)
        ├── Time Range (start/end)
        ├── Title (שם שהלקוח נתן)
        ├── Order (מספור אוטומטי לפי טיימליין)
        └── Status (נבחר → ערוך → אושר)
```

### מחזור חיים של פרק (Status Lifecycle)
```
הוקלט → סקירה פנימית → גרסה 1 נשלחה ללקוח
  ↓
  ↓ ← אושר ✓ (סיום)
  ↓
תיקונים התקבלו → גרסה 2 נשלחה
  ↓
  ↓ ← אושר ✓
  ↓
תיקונים התקבלו → גרסה 3 נשלחה (סופי)
```
**מגבלה:** מקסימום 3 סבבים. אחרי v3 - "סופי" אוטומטית.

### מחזור חיים של ריל
```
ריל סומן (ע"י לקוח) → ממתין לעריכה → ערוך v1 → נשלח ללקוח
  ↓
  ↓ ← אושר ✓
  ↓
תיקונים → v2 → ... → v3 (סופי)
```

---

## 4. דרישות פונקציונליות - MVP

### 4.1 פורטל הלקוח (Client Portal) ⭐ הלב

**נגן וידאו מתקדם:**
- Play / Pause / Seek
- שליטה במהירות (1x, 1.25x, 1.5x, 2x) — להערכה מהירה של פרק
- תצוגת זמן נוכחי (HH:MM:SS)
- (פאזה 2) הצגת SRT/Captions אוטומטית

**Timeline לסימון Reels:**
- שני Markers על ה-Timeline: Start + End לכל ריל
- כפתור "הוסף ריל" מהזמן הנוכחי
- שדה שם לכל ריל
- **מספור אוטומטי** לפי סדר על ציר הזמן (לא לפי סדר ההוספה)
- רשימת Reels בצד עם אפשרות עריכה/מחיקה

**הערות על הפרק:**
- **הערות ממוקדות-זמן:** לחיצה במקום ב-timeline → פותח פופאפ הערה → ההערה תוקבל עם timestamp
- **תיקונים כלליים:** שדה טקסט חופשי בתחתית

**גישה:**
- לינק חד-פעמי במייל + WhatsApp (לא דורש סיסמה ב-MVP)
- Token עם תאריך פג תוקף

### 4.2 דאשבורד הצוות (Admin Dashboard)

**מסך ראשי:**
- רשימת לקוחות פעילים + סטטוס הפרקים שלהם
- לוח Kanban לפי סטטוס פרקים (סנכרון עם Monday.com)
- התראות: לקוחות שהגיבו, תיקונים חדשים

**ניהול פרק:**
- העלאת קובץ גולמי (קישור ל-Bunny / Drive)
- שיוך ללקוח
- מעבר סטטוס ידני (עד שיש אוטומציה מוחלטת)
- צפייה בהערות + סימוני Reels של הלקוח
- ייצוא רשימת Reels כ-CSV/JSON לעבודת עורך

**ניהול לקוחות:**
- CRUD בסיסי (שם, מייל, ווצאפ, מסלול, הערות פנימיות)
- היסטוריית פרקים לכל לקוח

### 4.3 אחסון וידאו

**Bunny Stream:**
- העלאה: Admin מעלה קובץ → Bunny ממיר ל-HLS streaming
- נגן: Bunny Player Embed או iframe.io הטמעה עם custom UI
- בטחון: Signed URLs / Token-based access

### 4.4 התראות

**אוטומטיות (דרך Make):**

| אירוע | יעד | ערוץ |
|---|---|---|
| פרק חדש מוכן לצפייה | לקוח | Email + WhatsApp (Green API) |
| לקוח שלח תיקונים | צוות | WhatsApp + Monday update |
| לקוח סימן Reels | צוות | WhatsApp + Monday update |
| 3 ימים ללא תגובת לקוח | לקוח | תזכורת אוטומטית WhatsApp |
| סיכום שבועי | צוות | Email — מה על המדף, deadlines |

---

## 5. ארכיטקטורה טכנית (גישה A)

```
┌─────────────────────────────────────────────────────────────┐
│              CLIENT PORTAL (Next.js + Tailwind RTL)         │
│              Vercel Hosting · podcast.alon-gabay.co.il      │
└────────────┬──────────────────────────────────┬─────────────┘
             │                                  │
             ▼                                  ▼
┌────────────────────────┐         ┌────────────────────────┐
│      Supabase          │         │    Bunny Stream        │
│  • Postgres DB         │         │  • Video hosting       │
│  • Auth (Magic Link)   │         │  • HLS streaming       │
│  • RLS per customer    │         │  • Signed URLs         │
│  • Storage (small files)│        │                        │
└────────┬───────────────┘         └────────────────────────┘
         │
         │ (Webhook)
         ▼
┌────────────────────────┐         ┌────────────────────────┐
│       Make.com         │ ──────► │     Monday.com         │
│  (Automation Engine)   │         │  (Kanban + Statuses)   │
└────────┬───────────────┘         └────────────────────────┘
         │
         ├──► Green API (WhatsApp)
         ├──► Gmail SMTP (Email)
         └──► (Phase 2) YouTube/IG/TikTok APIs
```

### בחירות טכנולוגיות

| שכבה | בחירה | חלופה אם נכשל |
|---|---|---|
| Frontend | Next.js 15 + React 19 + Tailwind | Remix |
| Component Lib | shadcn/ui (RTL adapted) | Mantine |
| Video Player | Bunny Stream Player + custom Timeline overlay | Plyr.io + Bunny HLS |
| Database | Supabase Postgres | Neon + Prisma |
| Auth | Supabase Auth (Magic Link) | Clerk |
| Storage (video) | Bunny Stream | Cloudflare Stream |
| Storage (assets) | Supabase Storage | Bunny Storage |
| Hosting | Vercel | Cloudflare Pages |
| Automation | Make.com | n8n (self-hosted) |
| WhatsApp | Green API | Wati |
| Kanban | Monday.com (sync via Make) | Notion |
| Marketing site | WordPress קיים (alon-gabay) | — |

---

## 6. מודל נתונים (Data Model - גרסה ראשונה)

```sql
-- לקוחות
customers (
  id uuid PK,
  name text,
  email text,
  whatsapp text,
  package text,  -- 'photo_only' | 'photo_edit' | 'retainer'
  retainer_active boolean,
  notes text,
  created_at timestamp
)

-- פרקים
episodes (
  id uuid PK,
  customer_id uuid FK,
  title text,
  recorded_at date,
  status text,  -- 'recorded' | 'reviewing' | 'sent_v1' | 'revisions_v1' | ...
  current_version int,  -- 1, 2, or 3
  bunny_video_id text,
  share_token text,
  share_expires_at timestamp,
  created_at timestamp
)

-- גרסאות וידאו
episode_versions (
  id uuid PK,
  episode_id uuid FK,
  version_number int,
  bunny_video_id text,
  sent_at timestamp,
  approved_at timestamp
)

-- רילס
reels (
  id uuid PK,
  episode_id uuid FK,
  order_num int,  -- מספור אוטומטי לפי start_time
  title text,
  start_time_seconds float,
  end_time_seconds float,
  status text,  -- 'selected' | 'editing' | 'sent_v1' | ...
  bunny_video_id text,  -- אחרי עריכה
  created_by text,  -- 'client' | 'admin'
  created_at timestamp
)

-- הערות
feedback_items (
  id uuid PK,
  episode_id uuid FK,
  version_id uuid FK,  -- לאיזו גרסה ההערה
  type text,  -- 'timestamped' | 'general'
  timestamp_seconds float NULL,
  comment text,
  resolved boolean,
  created_at timestamp
)
```

---

## 7. מפת UI/UX (High-Level)

### צד הלקוח
```
/portal/[token]
  ├── דף נחיתה: "שלום [שם], הפרק שלך מוכן"
  ├── נגן וידאו (מרכז המסך)
  │     └── overlay: Timeline עם markers
  ├── Sidebar ימני: רשימת Reels שסומנו
  ├── Sidebar שמאלי: הערות (timestamped + general)
  ├── Footer: כפתור "שלח פידבק" / "אשר פרק"
```

### צד הצוות
```
/admin
  ├── /dashboard       — מסך ראשי, Kanban, התראות
  ├── /customers       — ניהול לקוחות
  ├── /customers/[id]  — פרופיל לקוח + היסטוריית פרקים
  ├── /episodes        — רשימת פרקים מסוננת
  ├── /episodes/[id]   — ניהול פרק (העלאה, סטטוס, צפייה בפידבק)
  ├── /reels           — תור Reels להפקה
```

---

## 8. מפת אינטגרציות (Make Scenarios)

| תרחיש | טריגר | פעולות |
|---|---|---|
| **Episode Ready → Client** | DB: episode.status = 'sent_v1' | 1. צור לינק → 2. Email ללקוח → 3. WhatsApp ללקוח → 4. עדכון Monday |
| **Client Feedback → Team** | DB: feedback_items insert | 1. WhatsApp לאלון → 2. עדכון Monday → 3. עדכון סטטוס episode |
| **Reels Selected → Team** | DB: reels bulk insert | 1. WhatsApp לעורך → 2. יצירת items ב-Monday לכל ריל |
| **No Response Reminder** | Cron יומי: episodes שנשלחו לפני 3 ימים ללא תגובה | 1. WhatsApp תזכורת ללקוח |
| **Weekly Digest** | Cron יום ראשון 09:00 | 1. שאילתה: פרקים פעילים → 2. Email סיכום לאלון |

---

## 9. מפת דרכים (Roadmap)

### 🎯 פאזה 1 — MVP (4-6 שבועות)
**מטרה:** להחליף את ההתנהלות הידנית הנוכחית.

- [ ] Setup: Supabase, Bunny, Vercel, Next.js skeleton + RTL
- [ ] Auth admin (סופאבייס)
- [ ] CRUD לקוחות + פרקים (admin)
- [ ] העלאת וידאו ל-Bunny (admin)
- [ ] יצירת share token + לינק חד-פעמי
- [ ] פורטל לקוח: נגן בסיסי + speed control
- [ ] Timeline UI לסימון Reels (start/end + שם + מספור אוטומטי)
- [ ] הערות timestamped + general
- [ ] שליחת הערות + Reels → DB
- [ ] Make scenario 1: Episode Ready (Email + WhatsApp via Green API)
- [ ] Make scenario 2: Client Feedback → צוות
- [ ] Dashboard admin: רשימת לקוחות + סטטוסים + Kanban Embed מ-Monday
- [ ] ייצוא רשימת Reels (CSV/JSON)

### 🚀 פאזה 2 — אוטומציה ו-AI (4-8 שבועות)
- [ ] תמלול אוטומטי SRT (Whisper API / AssemblyAI)
- [ ] הצגת captions בנגן
- [ ] AI Highlights: זיהוי קטעים אטרקטיביים
- [ ] AI Trailer: יצירת טריילר אוטומטי מהפרק
- [ ] Retainer Portal: ספרייה היסטורית ללקוחות קבועים

### 📡 פאזה 3 — הפצה ומונטיזציה (8-12 שבועות)
- [ ] מערכת הזמנות אונליין (לוח שנה + תשלום)
- [ ] העלאה אוטומטית: YouTube + Shorts
- [ ] העלאה אוטומטית: Instagram (Reels + Posts)
- [ ] העלאה אוטומטית: TikTok
- [ ] הפצה לפודקאסט: Spotify, Apple Podcasts (RSS feed)
- [ ] Analytics: ביצועי פרקים + רילס לפי פלטפורמה

---

## 10. דרישות לא-פונקציונליות

| תחום | דרישה |
|---|---|
| **שפה** | עברית בלבד, RTL מלא |
| **ביצועים** | טעינת פורטל לקוח < 2 שניות, וידאו מתחיל לשחק < 3 שניות |
| **בטחון** | Signed URLs ל-Bunny, RLS ב-Supabase, Tokens עם expiry |
| **גיבוי** | Backup יומי של Supabase, נכסים ב-Bunny עם redundancy |
| **נגישות** | Keyboard navigation, contrast WCAG AA |
| **מובייל** | Responsive design — לקוחות יצפו מהפלאפון |
| **עלות הפעלה (MVP)** | < $50/חודש (Supabase free + Bunny pay-per-use + Vercel free + Make free tier) |

---

## 11. שאלות פתוחות / החלטות עתידיות

1. **מסלולי תמחור** — איך בדיוק תבנה את החבילות? (ישפיע על Phase 3)
2. **רישום עורך חיצוני** — מתי להוסיף role of editor? (Phase 2)
3. **AI Highlights** — בעצמנו (OpenAI) או שירות מוכן (Opus Clip / Submagic API)?
4. **דומיין** — `podcast.alon-gabay.co.il` (subdomain) או דומיין נפרד?
5. **Monday board** — חדש או על board קיים? באיזה template?

---

## 12. הצעד הבא

לאחר אישור המסמך:
1. הפעלת סקיל **front-end-cto** ליצירת מסמך CTO מפורט לכל שכבה (data model מלא, API contracts, screen designs)
2. יצירת Monday board לניהול הפיתוח
3. תחילת פיתוח MVP בסדר התלות:
   - שבוע 1: Setup + Auth + DB schema
   - שבוע 2: Admin CRUD + Bunny integration
   - שבוע 3: Client portal player + Timeline marking
   - שבוע 4: Feedback flow + Make automations
   - שבוע 5: Testing + Polish
   - שבוע 6: השקה לפרק ראשון אמיתי
