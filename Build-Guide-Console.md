<div dir="rtl">

# מדריך בנייה: הסוכן ב-Claude Console (בדפדפן)

המדריך מתאר איך בונים את הסוכן **דרך הממשק של Claude Console** ([platform.claude.com](https://platform.claude.com/)), בלי שורת פקודה. הוא נכתב אחרי שהסוכן נבנה בפועל בממשק, ולכן המסכים והשדות מתוארים כפי שהם מופיעים. ההגדרות עצמן ב-[Agent-Config.md](Agent-Config.md), והחיבור ל-Fireflies ול-Make ב-[Build-Guide.md](Build-Guide.md) וב-[make/scenarios.md](make/scenarios.md).

> `<MANAGER_GMAIL>` הוא כתובת ה-Gmail של מנהל הפרויקט שהסוכן יוצר בה טיוטות. מחליפים בכתובת האמיתית.

---

## 0. סקירה: מה בונים ובאיזה סדר

| # | שלב | איפה ב-Console | מי עושה |
|---|---|---|---|
| 1 | סביבת עבודה (Environment) | Agents ← Environments | אפשר לבצע לפי המדריך |
| 2 | זיכרון הסטנדרט (Memory store) | Agents ← Memory | אפשר לבצע לפי המדריך |
| 3 | **הסוכן (Agent)** | Agents ← Agents | אפשר לבצע לפי המדריך |
| 4 | כספת (Credential vault) | Agents ← Credential vaults | הכספת ריקה: **ההרשאות מוקלדות על ידך** |
| 5 | הסקיל `meeting-summary` | Agent ← Skills | **העלאת קובץ על ידך** |
| 6 | ריצת ניסיון | Agents ← Sessions | אפשר לבצע לפי המדריך |

**תפריט הניווט** (הכפתור בפינה, ובו Agents): Overview, Create, Agents, Sessions, Deployments, Environments, Memory, Credential vaults. כל המשאבים שייכים ל-Workspace שנבחר למעלה (Default).

> **עלות:** יצירת סביבה, זיכרון, סוכן וכספת אינה עולה כסף. עלות נוצרת רק כשמריצים סשן.

---

## 1. סביבת עבודה (Environment)

Agents ← **Environments** ← **Create environment**.

| שדה | ערך |
|---|---|
| Hosting | **Cloud** (לא ניתן לשינוי אחרי היצירה) |
| Name | `meetings-limited` |
| Description | Cloud sandbox for the meeting-summarizer agent. Limited network: only the agent's MCP servers are reachable. |
| Networking | **Limited** (ברירת המחדל בטופס) |
| Allow package managers | **לא מסומן** |
| Allow MCP servers | **מסומן** (מאפשר לסוכן להגיע לשרתי Drive, Gmail ו-Airtable בלבד) |
| Allowed hosts | ריק |
| Packages, Metadata | ריק |

לוחצים **Create environment**. בעמוד הסביבה מופיע המזהה (`env_...`) והתיאור "Limited, allows MCP servers".

---

## 2. זיכרון הסטנדרט (Memory store)

Agents ← **Memory** ← **Create memory store**.

| שדה | ערך |
|---|---|
| Name | `house-standard` |
| Description | Writing standard for client meeting summaries: tone, preferred and forbidden wording, glossary. Shared, read-only reference. Read style.md and glossary.md before drafting. |

השם והתיאור מוצגים לסוכן בתוך ההנחיות שלו, ולכן התיאור כתוב עבור המודל.

בתוך ה-store לוחצים **Add memory** פעמיים:

| Path | Content (מעתיקים מהקובץ בריפו) |
|---|---|
| `/style.md` | [`agents/meeting-summarizer/data/house-standard/style.md`](agents/meeting-summarizer/data/house-standard/style.md) |
| `/glossary.md` | [`agents/meeting-summarizer/data/house-standard/glossary.md`](agents/meeting-summarizer/data/house-standard/glossary.md) |

בסוף ה-store מציג את שני הקבצים (כ-1.5KB וכ-0.75KB).

**זיכרון לכל לקוח** נוצר בנפרד, פעם אחת לכל לקוח (`client-<לקוח>`, ראו [Agent-Config.md](Agent-Config.md) סעיף 6.3). ב-Make הוא נוצר אוטומטית.

---

## 3. הסוכן (Agent)

### 3.1 יצירה

Agents ← **Agents** ← **Create agent**. הטופס מציע שלוש דרכים להתחיל: תיאור חופשי (Generate), תבנית, או סוכן ריק. בוחרים **סוכן ריק** ולוחצים **Create agent**. נוצר סוכן בשם `Untitled agent`, עם Opus 5.5 ורמת מאמץ נמוכה. את כל ההגדרות מחליפים בשלב הבא.

### 3.2 הדבקת ההגדרות בתצוגת Raw

בעמוד הסוכן: לשונית **Configuration** ← בצד ימין **Raw** (תצוגת YAML).

1. לוחצים בתוך העורך, **Ctrl+A** ואחריו **Delete**. **חשוב:** אם מדביקים בלי למחוק קודם, התוכן החדש נכנס באמצע התוכן הישן והעורך מציג שגיאה "Map keys must be unique".
2. מדביקים את כל התוכן של הקובץ [`console/agent-raw.yaml`](console/agent-raw.yaml).
3. לוחצים **Save new version** (מופיע בתחתית העמוד אחרי שינוי).

התוצאה: גרסה 2 של הסוכן, בשם `meeting-summarizer`.

### 3.3 מה מוגדר

| הגדרה | ערך |
|---|---|
| **מודל** | `claude-sonnet-5-5` |
| **רמת מאמץ** | `medium` (ב-YAML: `effort: {type: medium}`) |
| **ההנחיות (System prompt)** | הטקסט המלא מ-[Agent-Config.md](Agent-Config.md) סעיף 5 |
| **כלים מובנים** | `agent_toolset_20260401`: קבצים פעילים, `web_search` ו-`web_fetch` כבויים, הרשאות `auto` |
| **שרתי MCP** | Google Drive, Gmail, Airtable (כתובות בקובץ) |

### 3.4 אימות

בתצוגה **Rendered** בודקים:

- ב-Model מופיע `claude-sonnet-5-5` ו-**Medium effort**.
- ב-Tools מופיע "Built-in tools and 3 MCP servers", ולכל שרת הרשאות **Custom**.

**בדיקה חשובה: שליחת מייל כבויה.** ל-Gmail MCP יש **29 כלים**, וביניהם `send_message`, `reply`, `forward`, `create_filter` ומחיקה. לוחצים על **Tool permissions** של Gmail ומוודאים:

| כלי | מצב נדרש |
|---|---|
| `create_draft`, `list_drafts`, `list_labels`, `label_message`, `label_thread` | ⚡ **Auto** |
| **כל השאר**, ובהם `send_message`, `reply`, `forward`, `search_threads`, `mark_*_spam` | ⊗ **Always deny** |

לכל כלי ארבע אפשרויות: Auto, Always allow, Always ask, Always deny. כלי חדש שהשרת מוסיף בעתיד מקבל ברירת מחדל כבויה, כי ההגדרה בקובץ היא רשימה מותרת.

באותו אופן בודקים ב-Drive (4 כלים מופעלים: `search_files`, `get_file_metadata`, `read_file_content`, `create_file`) וב-Airtable (5 כלים: `get_table_schema`, `list_records_for_table`, `search_records`, `create_records_for_table`, `update_records_for_table`).

---

## 4. כספת (Credential vault)

Agents ← **Credential vaults** ← **Create vault**, בשם `meeting-summarizer - manager 1`. אחרי היצירה נפתח חלון **Add a credential**: אפשר לדלג (Skip for now) ולהוסיף אחר כך.

**כספת אחת לכל מנהל פרויקט.** הסוכן לא רואה את ההרשאות: Anthropic מצרפת אותן לבקשות בדרך החוצה.

### 4.1 הוספת ההרשאות (עושה מי שמחזיק בחשבונות)

בכספת לוחצים להוספת Credential:

| שדה | ערך |
|---|---|
| Type | **MCP OAuth** (ל-Google) |
| MCP server | `https://drivemcp.googleapis.com/mcp/v1`, ואחר כך `https://gmailmcp.googleapis.com/mcp/v1` |
| Connect | פותח חלון התחברות. נכנסים עם `<MANAGER_GMAIL>` ומאשרים |

ל-Airtable: אותו חלון, עם השרת `https://mcp.airtable.com/mcp`. אם סוג OAuth לא מתאים, בוחרים טוקן אישי (Bearer) עם הרשאות `data.records:read`, `data.records:write` ו-`schema.bases:read`, מוגבל לבסיס אחד.

> **ל-Google יש דרישה מקדימה:** פרויקט ב-Google Cloud עם Gmail ו-Drive MCP מופעלים, מסך הסכמה עם `<MANAGER_GMAIL>` כ-Test user, והרשאות `gmail.readonly`, `gmail.compose`, `gmail.modify`, `drive.readonly`, `drive.file`. הצעדים ב-[Build-Guide.md](Build-Guide.md) סעיף 5.1. אם כפתור Connect לא עובד מול Google, ההרשאה מוזנת עם Client ID ו-Refresh token לפי אותו סעיף.

**את הסודות מקלידים בעצמך ולא בצ'אט.** אל תדביקו מפתחות בהנחיות או בקבצי הזיכרון.

---

## 5. הסקיל `meeting-summary`

ההנחיות של הסוכן מורות להריץ את הסקיל, ולכן צריך לצרף אותו. ב-Console, בעמוד הסוכן, בקטע **Skills** ← **Add skill** מוצגים רק סקילים של Anthropic (`xlsx`, `pptx`, `pdf`, `docx`) ואפשרות **Create skill**.

1. יוצרים קובץ ZIP של התיקייה (בתיקיית הריפו):

</div>

```powershell
Compress-Archive -Path agents/meeting-summarizer/skills/meeting-summary -DestinationPath meeting-summary.zip -Force
```

<div dir="rtl">

2. ב-Add skill ← **Create skill** מעלים את `meeting-summary.zip`. התיקייה בתוך ה-ZIP חייבת להיות `meeting-summary`, והקובץ `SKILL.md` ישירות בתוכה.
3. חוזרים לסוכן, **Add skill**, בוחרים `meeting-summary`, ושומרים גרסה חדשה.

---

## 6. ריצת ניסיון

Agents ← **Sessions** ← **Create session**. הטופס:

| שדה | ערך לריצת ניסיון |
|---|---|
| Title | `rehearsal 1` |
| Agent | `meeting-summarizer` |
| Environment | `meetings-limited` |
| Credential vaults | הכספת של המנהל |
| MCP servers | נשאר כפי שהוגדר בסוכן (Custom) |
| **Budget** | **0.50** (דולר). בתקרה הריצה נעצרת |
| Resources | **Add resource** ← Memory store: `house-standard` (קריאה בלבד), ואחר כך זיכרון הלקוח (קריאה וכתיבה) |

אחרי יצירת הסשן שולחים את **ההודעה הראשונה**, לפי הדוגמה ב-[`agents/meeting-summarizer/kickoff.yaml`](agents/meeting-summarizer/kickoff.yaml) (אותן שורות `מפתח: ערך`, עם מזהי הבסיס, הטבלאות והקבצים האמיתיים).

**לריצת ניסיון ממענים את הטיוטה לכתובת שלך** (`<MANAGER_GMAIL>`) ולא ללקוח אמיתי.

### מה בודקים

| בדיקה | עובר אם |
|---|---|
| הסוכן קרא זיכרון ותמלול | בלשונית ה-Transcript של הסשן יש קריאות `read_file_content` וקריאות לקבצי הזיכרון |
| נוצרה **טיוטה אחת** ב-Gmail | בתיקיית הטיוטות. **שום מייל לא נשלח** |
| הכלים החסומים לא נקראו | אין קריאה ל-`send_message`, `reply` או `forward` |
| נוצר קובץ סיכום ב-Drive, ו-Airtable עודכן | כן |
| עלות | פחות מ-$0.50 |
| הטון מכבד | אין האשמות, ובטיוטה אין הערות פנימיות |

אם כלי נכשל: בלשונית Tools של הסשן רואים איזה, ומתקנים את ההרשאה או את השם ב-Agent ← Raw (גרסה חדשה). **מקבלים 3 ריצות נקיות לפני שמחברים את Make.**

---

## 7. מה הלאה

| שלב | איפה |
|---|---|
| בסיס הנתונים ב-Airtable | [Build-Guide.md](Build-Guide.md) סעיף 8 |
| Fireflies: webhook ו-API | [Build-Guide.md](Build-Guide.md) סעיף 9 |
| Make: תרחישים S1–S4 וחיבור ל-Claude Platform | [make/scenarios.md](make/scenarios.md) |
| Webhook מ-Claude Platform ל-Make | Console ← Manage ← Webhooks, אירוע `session.status_idled` |
| עדכון הסוכן | עורכים ב-Raw ושומרים גרסה חדשה. ריצות קיימות ממשיכות עם הגרסה שלהן |

</div>
