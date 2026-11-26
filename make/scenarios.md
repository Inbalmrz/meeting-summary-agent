<div dir="rtl">

# תרחישי Make

ארבעה תרחישים (Scenarios). Make הוא הצנרת והטריגרים, והסוכן ב-Claude Platform עושה את העבודה. אין ב-Make שום שיפוט, חוץ מסיווג קצר אחד של Claude Haiku בשלב 4 של שיוך הפרויקט.

**חיבורים ב-Make (Keychain):** Claude API (`x-api-key`), Fireflies (`Authorization: Bearer`), Google Drive, Gmail, Airtable. אף מפתח לא נכתב בגוף תרחיש.

**כותרות לכל קריאה ל-Claude API:**

| כותרת | ערך |
|---|---|
| `x-api-key` | מ-Keychain |
| `anthropic-version` | `2026-10-21` |
| `anthropic-beta` | `managed-agents-2026-04-01` (לזיכרון: `agent-memory-2026-07-22`, **במקום** הקודמת) |
| `content-type` | `application/json` |

**בגופי JSON** משתמשים במודול **JSON ← Create JSON**, כדי שעברית, מרכאות ושורות חדשות ייעטפו נכון.

---

## S1: תמלול חדש ← פתיחת ריצה

**טריגר:** Webhooks ← Custom webhook, עם "Get request headers". מופעל מ-Fireflies V2: `{"event":"meeting.transcribed","meeting_id":"...","timestamp":...}`.

</div>

```
1  [Webhooks] Custom webhook
2  [Filter] event = "meeting.transcribed" AND יש meeting_id
        (מומלץ: אימות X-Hub-Signature = "sha256=" + sha256(גוף; hex; סוד החתימה))
3  [Airtable] Search Records: Transcripts where Source ID = meeting_id
4  [Filter] לא נמצא   ← כפילות או עיבוד מחדש (reprocessed_at) נעצרים כאן, ללא עלות
5  [HTTP] POST https://api.fireflies.ai/graphql
        query($id:String!){ transcript(id:$id){ title dateString organizer_email participants
                                                sentences{ speaker_name text start_time } } }
6  [Tools] דחיסת התמלול: איחוד משפטים רצופים של אותו דובר לשורה אחת,
        חותמת זמן רק בתחילת קטע. פורמט:  [mm:ss] דובר: טקסט
7  חילוץ שם הפרויקט מ-title (ארבעה שלבים, עוצרים בראשון שמצליח):
     1. יש " - "?  החלק שלפניו מול Projects.Name / Aliases
     2. חיפוש שם או כינוי של פרויקט פעיל בתוך הנושא, כמילה שלמה (מותרת אות שימוש: ב, ל, ה, מ, ו, ש)
     3. דומיין המייל של המשתתפים מול Projects.Client Domains,
        רק אם ללקוח פרויקט פעיל אחד
     4. [HTTP] Claude Haiku 5.5: סיווג הנושא מול רשימת הפרויקטים,
        מתקבל רק אם project_id ברשימה AND confidence = high
8  [Router]
   ├─ לא זוהה פרויקט:
   │    [Drive] Upload a File ← /פרויקטים/ממתין לשיוך/
   │    [Airtable] Create Record: Transcripts, Status = "ממתין לשיוך"
   │    [Gmail] Send an Email ← למנהל הפרויקט בלבד: "לא זוהה פרויקט: <title>"
   └─ זוהה פרויקט:
        9   [Drive] Upload a File ← תיקיית תמלולים של הפרויקט, <date>_<topic>.txt  (מקבלים file id)
        10  [Airtable] Create Record: Transcripts
                Source ID, Project, Meeting Date, Topic, Meeting Type, Status = "בעיבוד"
        11  [Airtable] Get Record: Projects  (Client Contacts, Memory Store ID, Summaries Folder ID)
        12  אם Memory Store ID ריק:
              [HTTP] POST /v1/memory_stores  ← יצירת זיכרון ללקוח  (פירוט למטה)
              [Airtable] Update Record: Projects.Memory Store ID
        13  [HTTP] POST https://api.anthropic.com/v1/sessions  (פירוט למטה)
        14  [Airtable] Update Record: Transcripts.Session ID

   Error handler על שלב 13: Break, 3 ניסיונות כל 5 דקות.
   אחרי הכישלון: Status = "שגיאה" + הודעה למנהל.
```

<div dir="rtl">

**חילוץ מה-title:** הפורמט המועדף הוא `<פרויקט> - <נושא>`, אבל שלבים 2–4 מטפלים גם בנושא בלי מקף. אם אף שלב לא הצליח, אין ניחוש.

### שלב 12: יצירת זיכרון ללקוח (פעם אחת לכל לקוח)

`POST https://api.anthropic.com/v1/memory_stores`, כותרת `anthropic-beta: agent-memory-2026-07-22`:

</div>

```json
{
  "name": "client-<client_slug>",
  "description": "Working notes about one client - participants and roles, open items per project, lessons from the manager's edits. Read client-profile.md and open-items first. Notes only, no secrets, no raw transcript."
}
```

<div dir="rtl">

התשובה מחזירה `id` (`memstore_...`).

### שלב 13: פתיחת ריצה

`POST https://api.anthropic.com/v1/sessions`:

</div>

```json
{
  "agent": { "type": "agent", "id": "agent_...", "version": 1 },
  "environment_id": "env_...",
  "vault_ids": ["vlt_<כספת של מנהל הפרויקט>"],
  "resources": [
    { "type": "memory_store", "memory_store_id": "memstore_<house-standard>", "access": "read_only",
      "instructions": "Writing standard. Read style.md and glossary.md first." },
    { "type": "memory_store", "memory_store_id": "memstore_<לקוח>", "access": "read_write",
      "instructions": "Notes about this client. Read client-profile.md and open-items first." }
  ],
  "budget": { "type": "limit", "max_list_cost": { "amount": "50", "currency": "USD" } },
  "title": "<project> | <date> | <topic>",
  "metadata": { "transcript_record": "rec..." },
  "initial_events": [
    { "type": "user.message",
      "content": [ { "type": "text", "text": "<ההודעה הראשונה: הבלוק למטה>" } ] }
  ]
}
```

<div dir="rtl">

**ההודעה הראשונה** (שורות `מפתח: ערך`, כפי שההנחיות של הסוכן מצפות):

</div>

```
airtable_base_id: app...
transcripts_table_id: tbl...
tasks_table_id: tbl...
transcript_record_id: rec...
transcript_file_id: <Drive file id מהשלב 9>
summaries_folder_id: <Projects.Summaries Folder ID>
client_slug: <Projects.Client Slug>
project_name: <שם הפרויקט>
meeting_topic: <נושא>
meeting_type: <לקוח / ועדת היגוי / ספק>
meeting_date: <YYYY-MM-DD>
attendees: <שמות המשתתפים>
client_contacts: <שם <מייל>; ...>
manager_name: <Users.Name>
```

<div dir="rtl">

**הגדרות חשובות בבקשה:**

| שדה | למה |
|---|---|
| `version` נעוץ | גרסה חדשה של הסוכן לא משפיעה על ייצור עד שמעדכנים את המספר כאן |
| `vault_ids` של המנהל | הסוכן מקבל רק את ההרשאות של מנהל הפרויקט הנוכחי |
| `budget` = `"50"` | 50 סנט, הגבלה קשיחה. בתקרה הסשן נעצר (`budget_reached`) ואין ניסיון חוזר אוטומטי |
| `initial_events` | הסשן מתחיל לעבוד באותה קריאה |

**הערכת פעולות Make:** כ-15–20 לפגישה (יותר כשהחילוץ מגיע לשלב 2 או 4).

---

## S2: הסוכן סיים ← הודעה למנהל

**טריגר:** Custom webhook ש-Claude Platform קורא לו (Console ← Manage ← Webhooks, אירוע `session.status_idled`). ה-payload דל: `{"type":"event","data":{"type":"session.status_idled","id":"session_..."}}`.

</div>

```
1  [Webhooks] Custom webhook
2  [Filter] data.type = "session.status_idled"
3  [HTTP] GET https://api.anthropic.com/v1/sessions/{{data.id}}   ← מצב נוכחי ועלות (usage.list_cost)
4  [Airtable] Search Records: Transcripts where Session ID = data.id
5  [Airtable] Update Record: Cost = usage.list_cost
6  [Router]
   ├─ Status = "ממתין לאישור" / "נדרש בירור":
   │    [Gmail] Send an Email ← למנהל הפרויקט בלבד
   │       נושא: "סיכום מוכן: <פרויקט> – <נושא>"  (או "⚠️ נדרש בירור")
   │       גוף: קישור לטיוטה, Standard Check, Internal Notes, קישור לסשן
   └─ Status עדיין "בעיבוד" (הסוכן נעצר בתקרה או בשגיאה):
        [Airtable] Update Record: Status = "שגיאה", Error = סיבת העצירה
        [Gmail] Send an Email ← למנהל: "הסוכן נעצר: <סיבה>" + קישור לסשן
```

<div dir="rtl">

**הערכת פעולות:** כ-6 לפגישה.

---

## S3: המייל נשלח ← סטטוס "הופץ"

**טריגר:** Gmail ← Watch Emails, תיבת "נשלח". מתוזמן: כל 15 דקות, ימים א'–ה', 08:00–19:00 (כל בדיקה צורכת פעולה גם בלי מייל חדש).

</div>

```
1  [Gmail] Watch Emails: תיבת "נשלח"
2  [Airtable] Search Records: Transcripts where Status = "ממתין לאישור"
        AND נושא המייל = "סיכום <נושא> – <פרויקט> – <תאריך>"
3  [Filter] נמצאה התאמה
4  [Airtable] Update Record: Status = "הופץ", Sent Date, Recipients
5  [Tools] השוואה בין גוף הטיוטה המקורי (נשמר בקובץ הסיכום) לנשלח
6  [Filter] שינוי מהותי
7  [HTTP] POST /v1/sessions: סשן קצר וזול:
        "השווה בין הטיוטה לנשלח, וכתוב לקח ל-lessons.md אם יש"  (effort נמוך, תקציב $0.10)
```

<div dir="rtl">

---

## S4: רשת ביטחון יומית

</div>

```
1  [Schedule] כל יום ב-18:00
2  [Airtable] Search Records: Status ∈ {בעיבוד, ממתין לשיוך} AND Processing Date < עכשיו − 2 שעות
3  [Filter] נמצאו רשומות
4  [Gmail] Send an Email ← למנהל: רשימת תמלולים תקועים
```

<div dir="rtl">

**לא מפעילה את הסוכן.** רק מתריעה.

---

## רשימת בדיקה לפני הפעלת תרחיש

- [ ] כל התרחישים עם **Error handler**.
- [ ] **Filter** מוקדם בכל תרחיש (כפילות, סוג אירוע).
- [ ] ה-Custom webhooks עם כתובות שלא פורסמו.
- [ ] כותרת `anthropic-beta` נכונה בכל קריאה (זיכרון הוא היחיד עם כותרת אחרת).
- [ ] בריצת ניסיון, `client_contacts` מכיל **רק** את כתובת המנהל.

</div>
