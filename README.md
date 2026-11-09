# פלטפורמת הזמנות בשר B2B — Firebase Backend

מערכת רב־חברתית (Multi-Tenant) בעברית RTL: React + Vite + Tailwind בצד הלקוח, Firebase (Auth, Firestore, Storage, Functions, Hosting) בצד השרת.

## התקנה והרצה

```bash
npm install                 # צד לקוח
cp .env.example .env        # מלאו את ערכי VITE_FIREBASE_* מה־Firebase Console
npm run dev                 # http://localhost:5173
```

ב־Firebase Console: הפעתם **Authentication** (Google לצוות, Phone ללקוחות), **Firestore**, **Storage**, **Functions** ( Blaze / pay-as-you-go נדרש ל־Functions), ו־**App Check** (reCAPTCHA v3) עם המפתח ב־`VITE_APPCHECK_SITE_KEY`.

צד שרת:

```bash
cd functions && npm install && cd ..
cp .env.example functions/.env   # מלאו רק את הסודות השרתיים (WHATSAPP_*, SMS_*, ICOUNT/GREEN_INVOICE)
```

> סודות לעולם אינם נחשפים ללקוח: רק משתני `VITE_*` נכנסים ל־bundle. שאר המשתנים נטענים ב־Functions בלבד (`functions/.env`).

## מצב הדגמה / אורח (Demo & Guest)

- ללא הגדרת Firebase — האפליקציה עולה אוטומטית עם נתוני הדגמה מקומיים (`src/demoData.js`) ונשמרים ב־localStorage.
- **מצב אורח** (`signInGuest`) — ניהול מלא של חברת הדגמה אחת, ללא הרשאות אמיתיות.
- **פורטל הדגמה** (`?portal=true`) — לקוח הדגמה יחיד של חברת ההדגמה.
- מצב ההדגמה **מנותק לחלוטין** מנתוני אמת: שום קריאה/כתיבה ל־Firestore, שום הודעה אמיתית נשלחת, ולא מופקות חשבוניות מס.

## פריסה (Deployment)

```bash
npm run build
firebase login
firebase use <project-id>
firebase deploy            # hosting + firestore rules + storage rules + functions
```

לאחר הפריסה:
1. העניקו ל־Service Account של ה־Functions את התפקיד **Service Account Token Creator** (נדרש לחתימת קישורי הורדה V4).
2. הגדירו `APP_CHECK_ENFORCE=true` ב־`functions/.env` בסביבת ייצור.
3. משתמש ראשון נרשם דרך המסך "הקמת חברה" (`createCompany`) ומקבל אוטומטית claims של `admin`. שאר אנשי הצוות נוספים דרך מסך הצוות — ה־claims מסתנכרנים אוטומטית לפי האימייל.

## ברירות מחדל

| נושא | ברירת מחדל |
|---|---|
| מע״מ | **מתווסף מעל המחירים** (vatRate = 0.17, ניתן לשינוי בפרופיל). מחירי מוצר ומחירונים הם לפני מע״מ. |
| ביטול / שינוי הזמנה | חלון של **שעתיים** מרגע ההזמנה, ורק כל עוד הסטטוס `received`. שינוי בפועל = ביטול + הזמנה מחדש (השרת הוא סמכות המחירים). |
| התראת מלאי נמוך | סף **20 ק״ג** (`lowStockThreshold`). |
| התראת תעודת כשרות | **30 יום** לפני תפוגה (`certAlertDays`). מוצר שתעודותיו פגו חסום להזמנה. |
| מינימום הזמנה | 500 ₪ לפני מע״מ (`minOrder`). |
| מלאי | **נבדק בעת יצירת ההזמנה בלבד** (כולל חסימת אזל/לא פעיל/כשרות פגה). ההפחתה בפועל מתבצעת **ידנית על ידי המחסן בשלב הליקוט** (עדכון `actualKg`/מלאי). |
| מצב הדגמה | נפרד לחלוטין מנתוני אמת (localStorage בלבד). |
| מצב אורח | חברת הדגמה **אחת** בלבד. |
| שעת קיטון להזמנות | 14:00 (`orderCutoffTime`), לפי אזור אם הוגדר. |

## מסמכים וחשבוניות — הערה משפטית

המסמכים שמפיקה האפליקציה בצד הלקוח הם **הצעת מחיר** ו־**תעודת משלוח** בלבד. חשבונית מס כדין (עם מספר הקצאה) מופקת **אך ורק** דרך ה־callable `createInvoice`, בחיבור ל־iCount או חשבונית ירוקה (Green Invoice). ללא הגדרת מפתח API תוחזר שגיאה: "טרם הוגדר חיבור לתוכנת חשבוניות".

## ארכיטקטורה בקצרה

- כל המסמכים תחת `companies/{cid}/...` ונושאים `companyId == cid`; בידוד חברות מלא ב־Rules.
- צוות: Google Auth + Custom Claims `{role, companyId, companyIds[]}`. לקוחות פורטל: Phone Auth, בעלות לפי `uid` על מסמך הלקוח.
- יצירת חברות/הזמנות/תפוצות/חשבוניות — **callables בלבד** (Admin SDK); הכתיבה הישירה חסומה ב־Rules.
- נהגים: `driverIndex/{uid}` ממפה uid→staffId ומשמש את ה־Rules להרשאות על הזמנות וחתימות.
