<div dir="rtl">

# מדריך בנייה: הסוכן ב-Claude Platform וב-Make

המדריך מוביל מאפס לסוכן שרץ: התקנת ה-CLI, בניית הסוכן ב-Claude Platform, הרשאות, זיכרון, חיבור Fireflies ו-Make, וריצת ניסיון. ההגדרות עצמן ב-[Agent-Config.md](Agent-Config.md), התרחישים ב-[make/scenarios.md](make/scenarios.md), והתכנון ב-[SPEC.md](SPEC.md).

> **סימונים:** `<MANAGER_GMAIL>` הוא כתובת ה-Gmail של מנהל הפרויקט שהסוכן יוצר בה טיוטות. מחליפים אותה בכתובת האמיתית, והיא לא נשמרת בריפו.

---

## 0. סקירה: מה בונים ובאיזה סדר

| # | שלב | איפה | זמן משוער |
|---|---|---|---|
| 1 | חשבונות ודרישות | Claude Console, Google, Airtable, Fireflies, Make | 30 דק' |
| 2 | **התקנת ה-CLI** (`ant`) | המחשב שלך | 10 דק' |
| 3 | התחברות ל-Claude Platform | המחשב שלך + דפדפן | 5 דק' |
| 4 | **בניית הסוכן:** Agent, Environment, Vault, Memory | Claude Platform | 10 דק' |
| 5 | הרשאות לסוכן (Google, Airtable) בכספת | Google Cloud, Airtable | 45 דק' |
| 6 | זריעת הזיכרון | Claude Platform | 5 דק' |
| 7 | **ריצת ניסיון ידנית** | Claude Platform | 15 דק' |
| 8 | בסיס הנתונים ב-Airtable | Airtable | 30 דק' |
| 9 | Fireflies: webhook ו-API | Fireflies | 15 דק' |
| 10 | **Make:** חיבורים ותרחישים S1–S4 | Make | 2–3 שעות |
| 11 | Webhook מ-Claude Platform ל-Make | Claude Console | 5 דק' |
| 12 | בדיקות קבלה | הכול | שבוע |

**סדר חשוב:** קודם הסוכן וריצת ניסיון ידנית (שלבים 2–7). רק כשהיא מצליחה מחברים את Make. כך כל תקלה מאותרת בשכבה אחת בלבד.

---

## 1. דרישות מקדימות

| מה | פרטים |
|---|---|
| **חשבון Claude Console** | ב-[platform.claude.com](https://platform.claude.com), עם billing פעיל ו-Workspace. Managed Agents נמצא בבטא (ללא תוספת הרשמה במדריך זה: לבדוק בקונסולה שהוא זמין בחשבון) |
| **Google** | חשבון Gmail של מנהל הפרויקט (`<MANAGER_GMAIL>`), ופרויקט ב-Google Cloud |
| **Airtable** | חשבון, ובסיס אחד לפרויקט |
| **Fireflies.ai** | תוכנית שכוללת **API ו-webhooks** (לבדוק לפני רכישה) |
| **Make** | חשבון, עם גישה ל-Custom webhooks ול-HTTP |
| **במחשב** | Windows 10/11 עם PowerShell, Git ו-`gh` (מומלץ) |

---

## 2. התקנת ה-CLI (`ant`)

ה-CLI של Anthropic (`ant`) מגדיר ומעדכן את הסוכן מקבצים בריפו, ומפעיל ריצות ניסיון. הוא נשלף מהריפו הרשמי: [anthropics/anthropic-cli](https://github.com/anthropics/anthropic-cli/releases). הגרסה האחרונה בעת כתיבת המסמך: 2.14.0. נדרשת גרסה 1.34.0 ומעלה.

### 2.1 Windows (PowerShell), עם `gh`

</div>

```powershell
# תיקיית ההתקנה ותיקייה זמנית
$dir = "$env:LOCALAPPDATA\ant"
$tmp = Join-Path $env:TEMP "ant-install"
New-Item -ItemType Directory -Force $dir, $tmp | Out-Null

# הורדת הגרסה האחרונה (Windows 64-bit) וקובץ סכומי הביקורת
gh release download --repo anthropics/anthropic-cli --dir $tmp --clobber `
  --pattern "ant_*_windows_amd64.zip" --pattern "ant_*_checksums.txt"
```

```powershell
# אימות סכום ביקורת: לא ממשיכים אם הוא לא תואם
$zip = Get-ChildItem "$tmp\ant_*_windows_amd64.zip" | Select-Object -First 1
$line = Select-String -Path "$tmp\ant_*_checksums.txt" -Pattern ([regex]::Escape($zip.Name)) | Select-Object -First 1
$expected = ($line.Line -split '\s+')[0]
$actual = (Get-FileHash $zip.FullName -Algorithm SHA256).Hash
if ($actual -ine $expected) { throw "Checksum mismatch - do not continue" } else { "Checksum OK" }
```

```powershell
# חילוץ והוספה ל-PATH של המשתמש
Expand-Archive $zip.FullName -DestinationPath $dir -Force
$userPath = [Environment]::GetEnvironmentVariable("Path", "User")
if ($userPath -notlike "*$dir*") { [Environment]::SetEnvironmentVariable("Path", "$userPath;$dir", "User") }
```

<div dir="rtl">

**פותחים חלון PowerShell חדש** ובודקים:

</div>

```powershell
ant --version
```

<div dir="rtl">

אם הפקודה לא נמצאת: בודקים ש-`ant.exe` נמצא ישירות ב-`%LOCALAPPDATA%\ant` (אם החילוץ יצר תת-תיקייה, מעבירים את הקובץ החוצה).

### 2.2 Windows בלי `gh`

</div>

```powershell
$v = "2.14.0"   # הגרסה האחרונה: https://github.com/anthropics/anthropic-cli/releases
$base = "https://github.com/anthropics/anthropic-cli/releases/download/v$v"
$tmp = Join-Path $env:TEMP "ant-install"; New-Item -ItemType Directory -Force $tmp | Out-Null
Invoke-WebRequest "$base/ant_${v}_windows_amd64.zip" -OutFile "$tmp\ant_${v}_windows_amd64.zip"
Invoke-WebRequest "$base/ant_${v}_checksums.txt"     -OutFile "$tmp\ant_${v}_checksums.txt"
```

<div dir="rtl">

ממשיכים מאימות סכום הביקורת בסעיף 2.1.

### 2.3 מערכות אחרות

| מערכת | פקודה |
|---|---|
| macOS | `brew install anthropics/tap/ant` |
| Linux / WSL | הורדת `ant_<גרסה>_linux_<ארכיטקטורה>.tar.gz` מדף ה-releases וחילוץ ל-`/usr/local/bin` |
| עם Go 1.25+ | `go install github.com/anthropics/anthropic-cli/v2/cmd/ant@latest` |

---

## 3. התחברות ל-Claude Platform

</div>

```powershell
ant auth login
```

<div dir="rtl">

נפתח דפדפן: בוחרים את הארגון ואת ה-Workspace שבהם הסוכן ייבנה, ומאשרים. אחר כך בודקים:

</div>

```powershell
ant auth status
```

<div dir="rtl">

בודקים בפלט שהארגון וה-Workspace הם אלה שתכננת.

> **מלכודת נפוצה:** משתנה סביבה ישן בשם `ANTHROPIC_API_KEY` גובר על ההתחברות. אם `ant auth status` מראה מקור אחר מהצפוי, מסירים אותו: `Remove-Item Env:ANTHROPIC_API_KEY`.

---

## 4. בניית הסוכן

הסוכן מוגדר בקבצים שבתיקייה [`agents/meeting-summarizer/`](agents/meeting-summarizer/). `ant apply` הופך אותם למשאבים ב-Claude Platform ורושם את המזהים ב-`claude-lock.json`.

| קובץ | מה יוצר בפלטפורמה |
|---|---|
| `agent.md` | **Agent:** מודל, רמת מאמץ, הנחיות, כלים, מחברי MCP וסקיל |
| `environment.yaml` | **Environment:** סביבת עבודה בענן עם רשת מוגבלת |
| `vault.yaml` | **Vault:** כספת ריקה להרשאות (הסודות נוספים בשלב 5) |
| `memory_store-house-standard.yaml` | **Memory store** משותף לסטנדרט הכתיבה |
| `skills/meeting-summary/` | **Skill:** הסקיל, מועלה ומצורף לסוכן |

מריצים מתיקיית הריפו.

</div>

```powershell
# 1. בדיקת "הליכה": חייב להציג בדיוק את חמשת המשאבים, ושום דבר אחר
ant apply --dry-run agents/meeting-summarizer

# 2. תוכנית מפורטת
ant apply --dry-run -v `
  agents/meeting-summarizer/agent.md `
  agents/meeting-summarizer/environment.yaml `
  agents/meeting-summarizer/vault.yaml `
  agents/meeting-summarizer/memory_store-house-standard.yaml
```

<div dir="rtl">

קוראים את התוכנית: **הארגון וה-Workspace בכותרת**, שם המודל `claude-sonnet-5-5` ורמת המאמץ `medium`. אם הכול נכון, מבצעים (הפקודה תבקש אישור):

</div>

```powershell
ant apply `
  agents/meeting-summarizer/agent.md `
  agents/meeting-summarizer/environment.yaml `
  agents/meeting-summarizer/vault.yaml `
  agents/meeting-summarizer/memory_store-house-standard.yaml
```

<div dir="rtl">

נוצר `claude-lock.json` בשורש הריפו. **מעלים אותו ל-Git יחד עם התיקייה `agents/`**: בלעדיו הרצה נוספת תיצור כפילויות.

לא מריצים `ant apply .` ולא `--prune` או `--force`. ממשיכים תמיד עם שמות קבצים מפורשים.

---

## 5. הרשאות לסוכן (Vault)

הסוכן מתחבר ל-Drive, Gmail ו-Airtable דרך שרתי MCP של הספקים עצמם. ההרשאות נשמרות בכספת. **הסוכן לא רואה אותן**, ו-Anthropic מצרפת אותן לבקשות בדרך החוצה מהסביבה.

| שירות | כתובת ה-MCP | סוג הרשאה | הרשאות מינימליות |
|---|---|---|---|
| Google Drive | `https://drivemcp.googleapis.com/mcp/v1` | OAuth (`mcp_oauth`) | `drive.readonly`, `drive.file` |
| Gmail | `https://gmailmcp.googleapis.com/mcp/v1` | OAuth (`mcp_oauth`) | `gmail.readonly`, `gmail.compose`, `gmail.modify` |
| Airtable | `https://mcp.airtable.com/mcp` | טוקן אישי (`static_bearer`) | `data.records:read`, `data.records:write`, `schema.bases:read`, לבסיס אחד בלבד |

הכתובות והרשאות נלקחו מדפי התיעוד של Google ושל Airtable: [Gmail MCP](https://developers.google.com/workspace/gmail/api/guides/configure-mcp-server), [Drive MCP](https://developers.google.com/workspace/drive/api/guides/configure-mcp-server), [Airtable MCP](https://support.airtable.com/docs/using-the-airtable-mcp-server).

> **חשוב: ל-Gmail MCP יש גם כלי שליחה** (`send_message`, `reply`, `forward`) וכלי מחיקה. הסוכן מוגדר ב-`agent.md` לרשימה מותרת בלבד: יצירת טיוטה, רשימת טיוטות ותוויות. **כלי השליחה כבויים ברמת הסוכן** (`default_config: {enabled: false}`), גם אם הרשאת ה-OAuth כוללת אותם.

### 5.1 Google: פרויקט והפעלת השרתים

1. ב-[Google Cloud Console](https://console.cloud.google.com/) יוצרים פרויקט (או משתמשים בקיים).
2. מפעילים את ה-APIs ואת שירותי ה-MCP:

</div>

```powershell
gcloud services enable gmail.googleapis.com gmailmcp.googleapis.com drive.googleapis.com drivemcp.googleapis.com --project=PROJECT_ID
```

<div dir="rtl">

3. **OAuth consent screen:** בוחרים Audience **External** (או Internal בחשבון Workspace), ומוסיפים תחת **Test users** את `<MANAGER_GMAIL>`.
4. תחת **Data Access** מוסיפים ידנית את ההרשאות:
   `https://www.googleapis.com/auth/gmail.readonly`, `.../gmail.compose`, `.../gmail.modify`, `.../drive.readonly`, `.../drive.file`.
5. יוצרים **OAuth client** לפי המדריכים של Google למעלה, ושומרים את ה-Client ID וה-Client Secret.
6. מקבלים **Access token ו-Refresh token** עבור `<MANAGER_GMAIL>` (נכנסים עם החשבון הזה ומאשרים). **חייבים** לבקש `access_type=offline` ו-`prompt=consent`, אחרת לא יתקבל Refresh token. אפשר לעשות זאת עם [OAuth 2.0 Playground](https://developers.google.com/oauthplayground) עם ה-Client שיצרת (בהגדרות: Use your own OAuth credentials).

### 5.2 Airtable: טוקן אישי

ב-[airtable.com/create/tokens](https://airtable.com/create/tokens) יוצרים טוקן **ייעודי לסוכן**, עם שלוש ההרשאות שבטבלה, ועם גישה לבסיס אחד בלבד.

### 5.3 הוספת ההרשאות לכספת

**מריצים בחלון PowerShell משלך, ולא דרך הצ'אט.** הסודות נקראים בהקלדה שקטה ולא נשמרים בקובץ או בהיסטוריה. מריצים בלוק אחד בכל פעם.

</div>

```powershell
# הגדרות עזר (פעם אחת בחלון)
$lock  = Get-Content claude-lock.json -Raw | ConvertFrom-Json
$vault = $lock.resources.'./agents/meeting-summarizer/vault.yaml'.id
function Read-Secret($p) { [Net.NetworkCredential]::new('', (Read-Host $p -AsSecureString)).Password }
```

```powershell
# Airtable -> נשלח רק ל-mcp.airtable.com
$tok = Read-Secret "Airtable personal access token"
@{ display_name = "Airtable"
   auth = @{ type = "static_bearer"; mcp_server_url = "https://mcp.airtable.com/mcp"; token = $tok } } |
  ConvertTo-Json -Depth 5 | ant beta:vaults:credentials create --vault-id $vault
Remove-Variable tok
```

```powershell
# Google Drive ו-Gmail -> אותם אסימונים, שני חיבורים (כל חיבור מקושר לכתובת ה-MCP שלו)
$cid  = Read-Host "Google OAuth client ID"
$csec = Read-Secret "Google OAuth client secret"
$acc  = Read-Secret "Google access token"
$ref  = Read-Secret "Google refresh token"
$exp  = (Get-Date).ToUniversalTime().AddMinutes(55).ToString("yyyy-MM-ddTHH:mm:ssZ")
foreach ($svc in @(
  @{ name = "Google Drive"; url = "https://drivemcp.googleapis.com/mcp/v1" },
  @{ name = "Gmail";        url = "https://gmailmcp.googleapis.com/mcp/v1" })) {
  @{ display_name = $svc.name
     auth = @{ type = "mcp_oauth"; mcp_server_url = $svc.url; access_token = $acc; expires_at = $exp
               refresh = @{ refresh_token = $ref; client_id = $cid; token_endpoint = "https://oauth2.googleapis.com/token"
                            token_endpoint_auth = @{ type = "client_secret_post"; client_secret = $csec } } } } |
    ConvertTo-Json -Depth 8 | ant beta:vaults:credentials create --vault-id $vault
}
Remove-Variable cid, csec, acc, ref
```

<div dir="rtl">

בודקים שנוצרו **3 חיבורים** (הרשימה מציגה שמות וכתובות, אף פעם לא סודות):

</div>

```powershell
ant beta:vaults:credentials list --vault-id $vault
```

<div dir="rtl">

> **למנהלי פרויקטים נוספים:** כספת נוספת לכל מנהל (`ant beta:vaults create`), עם החיבורים שלו. הסוכן עצמו לא משתנה.

---

## 6. זיכרון

`ant apply` יוצר את ה-Memory store **ריק**. זורעים בו את סטנדרט הכתיבה:

</div>

```powershell
$store = $lock.resources.'./agents/meeting-summarizer/memory_store-house-standard.yaml'.id
foreach ($f in "style", "glossary") {
@"
content: "@./agents/meeting-summarizer/data/house-standard/$f.md"
"@ | ant beta:memory-stores:memories create --memory-store-id $store --path "/$f.md"
}
ant beta:memory-stores:memories list --memory-store-id $store --view full
```

<div dir="rtl">

קוראים את הרשימה ובודקים שיש בקבצים **תוכן אמיתי** ולא את הנתיב עצמו. (אם מופיע נתיב, מתקנים עם `ant beta:memory-stores:memories update --memory-id <mem_...>`.)

**זיכרון לכל לקוח:** נוצר פעם אחת כשמופיע לקוח חדש (ב-Make, תרחיש S1, או ידנית):

</div>

```powershell
@'
name: client-demo-client
description: Working notes about one client - participants and roles, open items per project, lessons from the manager's edits. Read client-profile.md and open-items first. Notes only, no secrets, no raw transcript.
'@ | ant beta:memory-stores create --transform id -r
```

<div dir="rtl">

---

## 7. ריצת ניסיון ידנית

**לפני Make.** בריצה הזו אתה מעלה תמלול לדוגמה ל-Drive ויוצר רשומה ב-Airtable (שלב 8), ופותח סשן ידנית. **ממענים את הטיוטה לכתובת שלך** (`<MANAGER_GMAIL>`) ולא ללקוח.

</div>

```powershell
# מזהים מהקובץ ומה-lock
$agent = $lock.resources.'./agents/meeting-summarizer/agent.md'.id
$envId = $lock.resources.'./agents/meeting-summarizer/environment.yaml'.id
$clientStore = "memstore_..."   # המזהה שהודפס ביצירת זיכרון הלקוח

# פתיחת סשן: סוכן, סביבה, כספת, זיכרון ותקציב קשיח של 0.50$
$sid = @"
agent: $agent
environment_id: $envId
vault_ids: [$vault]
resources:
  - {type: memory_store, memory_store_id: $store, access: read_only, instructions: "Writing standard. Read style.md and glossary.md first."}
  - {type: memory_store, memory_store_id: $clientStore, access: read_write, instructions: "Notes about this client. Read client-profile.md and open-items first."}
budget: {type: limit, max_list_cost: {amount: "50", currency: USD}}
title: rehearsal 1
"@ | ant beta:sessions create --transform id -r

# שליחת ההודעה הראשונה (ערכו קודם את agents/meeting-summarizer/kickoff.yaml עם המזהים האמיתיים)
Get-Content agents/meeting-summarizer/kickoff.yaml -Raw | ant beta:sessions:events send --session-id $sid

# מעקב חי
ant beta:sessions connect $sid
```

<div dir="rtl">

**מה בודקים ב-[Session viewer](https://platform.claude.com) (Console ← Sessions) ובתוצאות:**

| בדיקה | עובר אם |
|---|---|
| הסוכן קרא את הזיכרון והתמלול | בלשונית ה-Transcript יש קריאות `read_file_content` וקריאות לקבצי הזיכרון |
| נוצר קובץ סיכום ב-Drive | קובץ בתיקיית הסיכומים |
| נוצרה **טיוטה אחת** ב-Gmail | בתיקיית הטיוטות של `<MANAGER_GMAIL>`, ממוענת כפי שהוגדר. **שום מייל לא נשלח** |
| Airtable עודכן | סטטוס "ממתין לאישור", משימות, בדיקת איכות |
| הזיכרון עודכן | `open-items/<פרויקט>.md` מכיל את הנושאים הפתוחים |
| הטון מכבד | אין האשמות, ובטיוטה אין הערות פנימיות |
| עלות | פחות מ-$0.50 (מופיע בסשן ב-`usage`) |

**אם כלי נכשל:** שמות הכלים של Gmail, Drive ו-Airtable ב-`agent.md` נלקחו מהתיעוד של הספקים, ושמות בודדים עלולים להיות שונים בשרת בפועל. כלי עם שם שגוי פשוט "לא קיים" לסוכן. בלשונית Tools בודקים אילו קריאות נכשלו, מתקנים את השם ב-`agent.md` ומריצים `ant apply agents/meeting-summarizer/agent.md`. הסוכן יוצר גרסה חדשה.

**מקבלים 3 ריצות נקיות לפני שמחברים את Make.**

---

## 8. בסיס הנתונים ב-Airtable

יוצרים בסיס עם ארבע טבלאות. השדות והקשרים בתרשים ב-[Architecture.md](Architecture.md#4-בסיס-הנתונים-airtable).

| טבלה | שדות עיקריים |
|---|---|
| **Users** | Name, Email, Fireflies Email, Vault (שם הכספת) |
| **Projects** | Project Name, Aliases, Client, Client Slug, Client Domains, Drive Folder ID, Summaries Folder ID, Client Contacts, Memory Store ID, Project Manager (קישור ל-Users), Status |
| **Transcripts** | Source ID (מזהה Fireflies, ייחודי), Project, Meeting Date, Topic, Meeting Type, Status, Summary, Gmail Draft ID, Standard Check, Internal Notes, Session ID, Cost |
| **Tasks** | Task, Transcript, Project, Owner Side, Owner Name, Due Date, Flag, Status |

**סטטוסים בטבלת Transcripts:** ממתין לשיוך, בעיבוד, ממתין לאישור, נדרש בירור, שגיאה, הופץ.

הדשבורד נבנה ב-Airtable Interface (חמישה עמודים: ממתין לאישור, ממתין לשיוך, משימות פתוחות, סיכונים, עלות ותפעול), עם הרשאות לפי משתמש.

רושמים את מזהי הבסיס והטבלאות (`app...`, `tbl...`): Make יעביר אותם לסוכן בכל ריצה.

---

## 9. Fireflies

| הגדרה | איפה | ערך |
|---|---|---|
| **Webhook** | Fireflies ← Developer Settings ← **Webhooks V2** (V1 הוצא משימוש) | כתובת ה-Custom webhook של תרחיש S1, אירוע **`meeting.transcribed`**, ו-Signing Secret |
| **API key** | Developer Settings | נשמר ב-Keychain של Make |
| **שפת תמלול** | הגדרות החשבון | עברית |
| **הצטרפות אוטומטית** | חיבור היומן | פגישות מקוונות של מנהל הפרויקט |
| **הודעה למשתתפים** | הגדרות הבוט | פעילה |

פרטי ה-API (מתוך [התיעוד של Fireflies](https://docs.fireflies.ai/graphql-api/webhooks-v2)): ה-webhook שולח `event`, `meeting_id` ו-`timestamp` עם כותרת חתימה `X-Hub-Signature`. את התמלול שולפים ב-GraphQL מ-`https://api.fireflies.ai/graphql` עם `Authorization: Bearer <API key>`, בשדות `title`, `dateString`, `organizer_email`, `participants` ו-`sentences { speaker_name text start_time }`.

**לאחר עיבוד מחדש של פגישה** Fireflies שולח שוב `meeting.transcribed` עם `reprocessed_at`. S1 לא מעבד אותה שוב: הוא מסמן "נדרש בירור".

---

## 10. Make

חיבורים (Connections): Google Drive, Gmail (לשליחת הודעות למנהל בלבד), Airtable, ו-**Keychain** לשני מפתחות: Claude API (header `x-api-key`) ו-Fireflies (`Authorization: Bearer`).

התרחישים, מודול אחר מודול, עם גופי הבקשות ל-Claude Platform: [make/scenarios.md](make/scenarios.md).

| תרחיש | טריגר |
|---|---|
| **S1** תמלול חדש ← פתיחת ריצה | Custom webhook מ-Fireflies (מיידי) |
| **S2** הסוכן סיים ← הודעה | Custom webhook מ-Claude Platform (מיידי) |
| **S3** המייל נשלח ← סטטוס "הופץ" | Gmail, כל 15 דקות בשעות העבודה |
| **S4** רשת ביטחון | כל יום ב-18:00 |

---

## 11. Webhook מ-Claude Platform ל-Make

ב-Claude Console ← **Manage ← Webhooks** יוצרים נקודת קצה:

| שדה | ערך |
|---|---|
| URL | כתובת ה-Custom webhook של תרחיש S2 (HTTPS) |
| Event types | `session.status_idled` |
| Signing secret | מוצג **פעם אחת**. שומרים אותו ב-Make |

ה-payload דל במכוון: סוג אירוע ומזהים בלבד. S2 שולף את הסשן מה-API לפי המזהה, כך שפנייה מזויפת לא יכולה לגרום לדבר מלבד קריאה לסשן אמיתי.

---

## 12. בדיקות קבלה

| # | בדיקה | עובר אם |
|---|---|---|
| A1 | מקצה לקצה: פגישה אמיתית ← טיוטה | טיוטה אחת בתיקיית הטיוטות, הודעה למנהל, Airtable מעודכן |
| A2 | כפילות: אותו webhook פעמיים | סשן אחד בלבד |
| A3 | זימון בלי שם פרויקט מוכר | "ממתין לשיוך" והסוכן לא מופעל |
| A4 | פגישה שנייה באותו פרויקט | סעיף המעקב מציג את הנושאים הפתוחים מהראשונה |
| A5 | שליחה ידנית מ-Gmail | Airtable מראה "הופץ" תוך 15 דקות |
| A6 | **אין כלי שליחה** | בלשונית Tools של כל ריצה אין קריאה ל-`send_message`, `reply` או `forward` |
| A7 | עלות | ממוצע של 10 ריצות עד $0.35, ואף ריצה לא מעל $0.50 |
| A8 | מנהל פרויקט שני | לא רואה טיוטות, תמלולים או זיכרון של הראשון |

---

## 13. עדכון ותחזוקה

| רוצים לשנות | עושים |
|---|---|
| הנחיות, מודל, רמת מאמץ או כלים | עורכים את `agent.md` ומריצים `ant apply agents/meeting-summarizer/agent.md`. נוצרת גרסה חדשה |
| הסקיל או התבנית | עורכים את `skills/meeting-summary/` ומריצים `ant apply` על `agent.md` |
| סביבת העבודה | עורכים את `environment.yaml` (חל על סשנים חדשים בלבד) |
| גרסה שמריצים בייצור | Make מצמיד גרסה (`{"type":"agent","id":"...","version":N}`). גרסה חדשה נבדקת קודם בסשן ידני, ורק אז מעדכנים את המספר ב-Make |
| הוספת מנהל פרויקט | כספת חדשה (סעיף 5.3), זיכרון ללקוחות שלו, רשומה ב-Users, וחיבור Fireflies ליומן שלו |

**לא מוחקים ולא מארכבים משאבים בלי צורך:** ארכוב הוא סופי.

---

## 14. תקלות נפוצות

| סימפטום | סיבה סבירה | פתרון |
|---|---|---|
| `ant: command not found` | חלון ישן | פותחים PowerShell חדש (סעיף 2) |
| המשאבים לא מופיעים ב-Console | Workspace אחר | `ant auth status` ובודקים מול ה-Workspace בקונסולה |
| הסשן נעצר בלי תוצאה | תקרת $0.50 | בלשונית Session: סיבת העצירה `budget_reached`. בודקים כמה סבבים ומה גדל |
| שגיאת MCP ב-`session.error` | הרשאה שפגה או שגויה | `ant beta:vaults:credentials list`. מחליפים את החיבור (ארכוב ויצירה מחדש) |
| `ant apply` מסרב לרוץ | נערך משאב מחוץ לקבצים | עורכים רק בקבצים |
| אין Refresh token מ-Google | לא נשלח `access_type=offline` | מאשרים מחדש עם `prompt=consent` |
| הטיוטה נוצרה בלי נמענים | `client_contacts` ריק | משלימים את אנשי הקשר ב-Airtable |

</div>
