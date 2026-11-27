<div dir="rtl">

# מדיניות ניסיונות חוזרים: עד 2 ניסיונות

הכלל: **כשל לא מנסים שוב ושוב.** לכל שלב מותרים **2 ניסיונות בסך הכול** (הניסיון הראשון וניסיון חוזר אחד), ואם הם נכשלים, הריצה נעצרת, מסומנת כשגיאה, ומנהל הפרויקט מקבל הודעה.

הכלל חל בשני מקומות: **בסוכן** (Claude Platform) וב-**Make**. כל אחד מהם עוצר לבד, כך שכשל באחד לא גורר לולאה באחר.

| רכיב | מה נעצר | אחרי העצירה |
|---|---|---|
| **הסוכן** | כלי (Drive, Gmail, Airtable) שנכשל פעמיים באותו שלב | הסטטוס `שגיאה`, פירוט ב-Internal Notes, שורת דיווח אחת |
| **Make** | קריאה חיצונית שנכשלה פעמיים, או תרחיש שנכשל פעמיים ברצף | התרחיש נעצר, הרשומה מסומנת `שגיאה`, והמנהל מקבל הודעה |
| **התמלול** | יותר משתי ריצות של הסוכן על אותו תמלול | נדרש טיפול ידני |

**אין ניסיון חוזר אוטומטי אחרי שהסוכן עצר.** הרצה מחדש היא החלטה של מנהל הפרויקט.

---

## 1. הסוכן

### מה מוגדר

הסעיף הבא נוסף להנחיות של הסוכן ב-Claude Console (**גרסה 3**), לפני רשימת שרתי ה-MCP:

</div>

```text
# Failures and retries

- If a tool call fails, retry the same call once. That is at most 2 attempts in total for any single step.
- If the second attempt also fails, do not try a third time and do not look for a different workaround. Set the transcript's Status to "שגיאה" (when Airtable is reachable), write the step, the tool and the error message in Internal Notes, and stop.
- When you stop this way, finish with exactly one line: "<project_name> | <meeting_date> | stopped after 2 failed attempts: <tool>".
- Never repeat the same failing call in a loop.
- Never retry a step that already succeeded: do not create a second draft, a second Drive file or duplicate tasks.
```

<div dir="rtl">

### איך זה עובד

| מצב | מה הסוכן עושה |
|---|---|
| כלי נכשל פעם אחת | מנסה שוב את אותה קריאה, פעם אחת |
| הניסיון השני נכשל | עוצר. הסטטוס `שגיאה`, ובשדה Internal Notes: השלב, הכלי והשגיאה |
| Airtable עצמו לא זמין | עוצר ומדווח בשורה האחרונה בלבד |
| שלב שכבר הצליח | לא חוזר עליו. לא נוצרת טיוטה שנייה, קובץ שני או משימות כפולות |

**שכבת הגנה נוספת, בלי קשר להנחיות:** לכל ריצה יש תקציב קשיח של $0.50 ב-Claude Platform. אם הסוכן ממשיך לנסות, הריצה נעצרת בתקרה (`budget_reached`).

### עדכון הסוכן

עורכים בתצוגת Raw של הסוכן ומוסיפים את הסעיף, ואז **Save new version**. ריצות קיימות ממשיכות עם הגרסה שלהן, וריצות חדשות מקבלות את הגרסה החדשה (או את הגרסה שהוצמדה ב-Make).

---

## 2. Make

### 2.1 טיפול בשגיאות בכל מודול חיצוני

לכל מודול שקורא למערכת חיצונית (HTTP ל-Claude Platform, HTTP ל-Fireflies, Drive, Gmail, Airtable) מוסיפים **Error handler** מסוג **Break**:

| הגדרה | ערך |
|---|---|
| Error handler | **Break** |
| Number of attempts | **1** (ניסיון חוזר אחד, כלומר 2 ניסיונות בסך הכול) |
| Interval between attempts | 5 דקות |
| Allow storing of incomplete executions | **מופעל**, כדי שאפשר יהיה להריץ ידנית מחדש |

> **לאמת בעת ההגדרה:** ב-Make "Number of attempts" ב-Break סופר ניסיונות חוזרים. הסכום הכולל צריך להיות 2. אם בגרסה שלך הערך מוגדר כסך כל הניסיונות, מגדירים 2.

**כשל אחרי הניסיון השני:** מסלול השגיאה כולל, לפני ה-Break, מודול Airtable (Update Record) שמעדכן `Status = שגיאה` ו-`Error` (שם המודול והודעת השגיאה), ומודול Gmail (Send an Email) שמודיע למנהל.

### 2.2 עצירת התרחיש אחרי 2 שגיאות ברצף

בהגדרות כל תרחיש (Scenario settings): **Number of consecutive errors = 2**.

- תרחיש שנכשל פעמיים ברצף **נעצר** ו-Make שולח הודעה לבעלים.
- אחרי שמתקנים את הבעיה מפעילים אותו מחדש ידנית.
- שימו לב: בתרחישי webhook (S1 ו-S2) עצירה אומרת שהאירועים החדשים לא יתקבלו עד להפעלה מחדש. ההגנה היא ש-S4 (בדיקה יומית ב-18:00) מתריע על תמלולים שנתקעו.

### 2.3 לכל היותר 2 ריצות של הסוכן לכל תמלול

מונע מצב שבו אותו תמלול מפעיל שוב ושוב סשנים יקרים.

| שלב | מה עושים |
|---|---|
| שדה חדש ב-Airtable, טבלת Transcripts | `Attempts` (מספר), ברירת מחדל 0 |
| ב-S1, **לפני** פתיחת הסשן | Filter: `Attempts < 2`. אם לא, `Status = שגיאה` ושליחת הודעה למנהל |
| ב-S1, **אחרי** פתיחת הסשן | Update Record: `Attempts = Attempts + 1` |
| פתיחת הסשן עצמה | לפני ניסיון חוזר של ה-HTTP, בודקים ש-`Session ID` ריק ב-Airtable. כך כשל ברשת לא יוצר שני סשנים |

כך מתקבלות לכל היותר שתי ריצות לתמלול: הריצה הראשונה, והרצה חוזרת אחת שמנהל הפרויקט מאשר ידנית.

### 2.4 S2: כשהסוכן עצר בשגיאה

ב-S2 (הסוכן סיים) מסלול ה-Router "הסוכן נעצר" כבר קיים ב-[make/scenarios.md](make/scenarios.md). כשהסטטוס `שגיאה`:

1. מוסיפים להודעה למנהל את סיבת העצירה מ-Internal Notes (שלב, כלי והודעת שגיאה).
2. **לא מפעילים סשן חדש אוטומטית.**

---

## 3. הקבצים בריפו

הכלל נוסף לסוכן ב-Console (גרסה 4), ומאז **הקבצים בריפו עודכנו להתאים**: [agents/meeting-summarizer/agent.md](agents/meeting-summarizer/agent.md), [console/agent-raw.yaml](console/agent-raw.yaml) ו-[Agent-Config.md](Agent-Config.md). הגרסה הקודמת של המסמך הזה, שציינה שהקבצים עדיין מתארים גרסה 2, נשמרה ב-[versions/Retry-Policy-v1.md](versions/Retry-Policy-v1.md).

המקור המחייב להנחיות נשאר הסוכן ב-Console.

</div>
