---
{"dateCreated":"2024-03-24 22:02","tags":["concurrency"],"pageDirection":"rtl","dg-publish":true,"permalink":"/computer-science/computer-system/simd/","dgPassFrontmatter":true}
---

# SIMD
בסיכום זה נתמקד באחד מהכלים לבצע תכנות מקבילי העונה לשם SIMD - single instruction multiple data. בד״כ מדובר בחומרה ופקודות מעבד (משתנה לפי ISA) שמבצעות את אותה פקודה על מספר יחידות מידע בבת אחת.

הדיאגרמה הבאה מציגה את סוגי התכנות המקבילי הקיימים לפי [Flynn's taxonomy](https://en.wikipedia.org/wiki/Flynn%27s_taxonomy)

![Pasted image 20240324220832.png](/img/user/Assets/Pasted%20image%2020240324220832.png)

למשל, נוכל לקחת שני וקטורים A,B בגודל 4 ולבצע סכימה של $A[i]+B[i]$ לכל $i\in[0,3]$ בבת אחת. 

![Pasted image 20240324221151.png|250](/img/user/Assets/Pasted%20image%2020240324221151.png)

## AVX
אחד הרכיבי חומרה העיקריים במערכות x86 לביצוע פקודות SIMD הם רכיבי הAVX Advanced Vector Extensions.

נזכיר שב [[Computer Science/Computer System/x86-64 Assembly\|x86-64 Assembly]] ישנם general purpose registers לחישובים אריתמטיים של integers בייצוג signed ו unsigned ולאריתמטיקת כתובות. החומרה הזאת לא תמכה בחישובים אריתמטיים של [[Floating Point\|Floating Point]] ברמת החומרה. 

לאחר מספר עשורים הוכנסו רכיבי ה AVX שתומכים גם באריתמטיקה של מספרים עשרוניים וגם באריתמטיקה __וקטורית__ לוקטורים בגודל של עד 256 ביט. 

![Pasted image 20240324223350.png|200](/img/user/Assets/Pasted%20image%2020240324223350.png)


הregisters הללו הם מעין תחליף לכל מה שקשור לקוד שמבצע מניפולציה על float ו double. כמו שיש registers שיש להם תפקיד ב רגיסטרים ה״רגילים״ שאנחנו מכירים (rax כערך החזרה למשל). כך גם לregisters הנ״ל יש תפקידים

1) xmm0-תופס את הארגומנט הראשון וגם את ערך ההחזרה
2) xmm1-xmm7 - העברת שאר הארגומנטים לפונקציה
3) xmm9-xmm31- [[Computer Science/Computer System/Functions in assembly\|שמירת ה Caller saved]].

### scalar and packed modes
כפי שציינו הרגיסטרים האלה תומכים גם באריתמטיקה סקלרית של floats וגם באריתמטיקה וקטורית. לכן ה ISA צריך לתמוך בפקודות שמאפשרות לחומרה להבין בשביל מה אנחנו רוצים להשתמש בחומרה הזאת. 

מבנה ההוראות הכללי עבור הפקודות האלה נראה ככה

$$\text{<OP>}\textcolor{red}{\text{<s/p>}}\textcolor{cyan}{\text{<s/d>}}$$

* $<OP>$ -הפקודה עצמה, יש המון פקודות כמו הזזה של ערכים, חיבור, חיסור וכו׳. כל הפקודות של AVX יתחילו ב 'v' (יש מערכת ישנה יותר שנקראת SSE ששם הפקודות הן בלי 'v').
* $\textcolor{red}{\text{<s/p>}}$ - קובעים האם אנחנו רוצים לעבוד עם data מסוג scalar או עם packed 
* $\textcolor{cyan}{\text{<s/d>}}$ - single precision או double precision. לפי הבחירה הזאת אנחנו נקבע גם כמה ערכים יש לנו בוקטור (אם עובדים עם doubles אז ברגיסטר xmm יכולים להיות 2 מספרים double או 4 מספרי float).

_דוגמה:_ $\text{vaddpd xmm1, xmm2}$. פקודת AVS שמבקשת לחבר את xmm1 ו xmm2 בpacked mode ועם double precision.

__בניגוד לפקודות מהרגיסטרים של integers, כאן התוצאה יכולה גם להכנס לרגיסטר שלישי שנרצה (נראה דוגמה בהמשך)__

![Pasted image 20240324225300.png|250](/img/user/Assets/Pasted%20image%2020240324225300.png)

אם נחליף ל single precision נקבל:

![Pasted image 20240324225152.png|250](/img/user/Assets/Pasted%20image%2020240324225152.png)

החישובים המצויינים בתמונה נעשים באופן מקבילי לחלוטין.

אם היינו משתמשים ב scalar mode היינו מבצעים את פעולת החיבור רק על 32/64 הביטים הראשונים.
האופציה של scalar קיימת מכיוון שהרגיסטרים האילו מיועדים לכל הסוגים של חישובים float-point ולא בהכרח חישובים וקטוריים.

![Pasted image 20240324225641.png|250](/img/user/Assets/Pasted%20image%2020240324225641.png)

### פקודות אריתמטיקה
![Pasted image 20240324230044.png](/img/user/Assets/Pasted%20image%2020240324230044.png)

* $S_{1}$ (הרגיסטר האמצעי מבין השלושה) - יכול להיות גם כתובת זכרון, נזכיר שהסינתקס לגשת לכתובת זכרון היא '(reg)'.
* שאר הארגומנטים לא יכולים להיות כתובת אלא רגיסטרים של AVX בלבד.
* אם נביא רגיסטר שלישי בארגומנטים אז התוצא תכנס אליו:
	* הפקודה vmulsd %xmm0, %xmm1, %xmm4 תעשה - xmm4 ← xmm1 $\cdot$ xmm0
	* הפקודה vmulps %ymm0, (%rcx), %ymm1 תעשה - $ymm1 ← [b_{7}\cdot𝑎_7,…,b_{1}\cdot a_{1},b_{0}\cdot a_{0}]$ כאשר rcx מצביע ל128 הביטים שבהם נמצא הוקטור $[b]$ . 
	* _אם לא ניתן ארגומנט שלישי התוצאה תכנס לארגומנט הראשון_.
* נשים לב שלפקודה sqrt אין גרסה עם 'v' כי זאת פקודה של AVX בלבד.

### פקודות move
נזכיר ש[[Computer Science/Computer System/basic commands in assembly#move to\|move]] אלא פקודות שמאפשרות להזיז תוכן בין רגיסטרים או בין memory לרגיסטר או בין רגיסטר לmemory. 

![Pasted image 20240324231800.png](/img/user/Assets/Pasted%20image%2020240324231800.png)
* סינתקס של פקודה:
	* v-vector instruction
	* ss-scalar single
	* sd-scalar double
	* ps-packed single
	* pd-packed double
	* $M_{32}$ ו $M_{64}$ - לוקח 64 או 32 ביט מהכתובת הנתונה.
	* X מייצג רגיסטר של AVX.

* addressing mode זהה לפקודת mov הרגיל
	![Pasted image 20221222185500.png|450](/img/user/Assets/Pasted%20image%2020221222185500.png)

* מידע בזכרון חייב להיות [aligned](https://www.youtube.com/watch?v=OKjOZBaKlOc)לפי הגודל של הרגיסטר בביטים. הסיבה לכך היא ביצועים שקשורים ל[[Computer Science/Computer System/Computer System Cache Memory\|Cache]], השמירה ב cache היא ברמת הבלוקים שנקבע לפי כמות הביטים בכתובת שמיועדת לblock offset. לכן מידע שהוא לא aligned עלול לגרום לכך שdata struct אחד יתפוס שתי lines של cache. דבר שפוגע מאוד בביצועים. 
	* לכן האות a בפקודה מחייב שהdata שקוראים מהזכרון יהיה aligned של 16 בתים (128 ביטים), אחרת תזכר שגיאה.
	* זה רלוונטי רק ל mov בין רגיסטר וזכרון כי בין שני רגיסטרים אין אפשרות שזה יקרה.

### פקודות conversion
__המרה לintegers__:
![Pasted image 20240324235921.png](/img/user/Assets/Pasted%20image%2020240324235921.png)
* v-vector instruction
* cvtt - convert with truncation המרה עם חיתוך (עיגול כלפי מטה), המקרה הזה הפיכת החלק שמייצג את השבר במספר העשרוני ל 0.
* ss2si - מספר float ל int
* sd2si- מספר double ל int 
* ss2siq, sd2siq - מספר double ל quad word integer.
* ה source יכול להיות register X כלומר מסוג AVX או כתובת זכרון שמפנה ל float או double 
* היעד הוא תמיד register של 32 או 64 ביט רגיל.

![Pasted image 20240325000839.png](/img/user/Assets/Pasted%20image%2020240325000839.png)
1) הקוד מזיז מהכתובת שבה val נמצא את התוכל שלו ל xmm0.
2) מתבצע המרה מ xmm1 למספר quad word שאותו שמים ב rdi.

__המרה מinteger לfloat/double__
![Pasted image 20240325001046.png](/img/user/Assets/Pasted%20image%2020240325001046.png)
* v - vector
* cvt - convert
* si2ss/sd - המרה ל single או double
* q - מסמל שגודל ה int הוא quad word.
* נתעלם מ source 2 לרוב, פשוט נשים גם ב source 2 וגם ב destination את אותו רגיסטר. 
![Pasted image 20240325002556.png](/img/user/Assets/Pasted%20image%2020240325002556.png)

__המרה מ float ל double__
אפשר גם להמיר מספרים עשרוניים ולשנות את רמת הדיוק שלהם באופן הבא 

![Pasted image 20240325002944.png](/img/user/Assets/Pasted%20image%2020240325002944.png)
* vcvtpd2ps - לוקחת double וממירה לfloat 
* vcvtss2sd - לוקת float וממירה לdouble.

==_הסיבה שבאפור נראה כאילו עברנו ממערך בגודל 4 למערך בגודל 2 כאשר המרנו ל double והפוך כאשר המרנו ל float היא שבפועל זה באמת מה שקורה, בגלל שמדובר גם בוקטורים אמרנו שיכול להיות לי בdouble שתי מספרים וב float ארבעה, אבל אם אני ממיר אני או מרפס באפסים או מוותר על שתי המספרים בבתים הגבוהים_==

כמובן שהפעולה הזאת עלולה לגרום לנו או לאבד דיוק של המספר או להרחיב את הדיוק שלו במחיר של עוד 4 בתים. 

__דוגמה__
נסתכל על קטע הקוד הבא
```C
double func(double w, int x, float y, long z) {
	return y*x - w/z;
}
```

נשים לב לכמה דגשים:
* x ו z נשמרים ב rdi ו rsi בהתאמה.
* w ו y נשמרים ב xmm0 ו xmm1 בהתאמה.
* $y \cdot x$ זה חישוב שמערב int ו float ולכן צריך להמיר את x ל float.
* תחת אותו הסבר צריך להמיר את z ל double.
* פעולת החיסור היא בין float ל double ולכן צריך להמיר את ה float ל double

```assembly
func: 
	vcvtsi2ss edi, xmm2, xmm2 
	vmulss xmm1, xmm2, xmm1
	vcvtss2sd xmm1,xmm2
	vcvtsi2sdq rsi xmm1, xmm1
	vdivsd xmm1, xmm0, xmm0
	vsubsd xmm0 xmm2 xmm0
	ret
```

1) השורה הראשונה ממירה את x לfloat ושמה אותו ב xmm2
2) השורה השנייה מכפילה בין y ל x ושמה את התוצאה ב xmm1 
3) השורה השלישית ממירה ל double את התוצאה ושמה ב xmm1. 
4) השורה הרביעית ממירה את z ל double 
5) השורה החמישית והשישית מבצעות את החילוק והכפל. 

### constants
ב AVX לא ניתן לרשום immediate כמו בregisters הרגילים בפעולות אריתמטיות. לכן יש צורך לטעון אותם ישירות מהזכרון. 
השמירה בזכרון תעשה או על ידי שמירת הייצוג הבינארי $.byte$ או על ידי שמירת המספר השלם כפי שאותם ביטים היו מייצגים אותו ב IEEE. 

למשל עבור הפונקציה 
```C
double f(double temp) {
	return 1.8 * temp + 32.0;
}
```

הקוד יראה ככה 
![Pasted image 20240325011330.png|350](/img/user/Assets/Pasted%20image%2020240325011330.png)
הסיבה שכאן שמרנו ה4 בתים בנפרד היא בגלל [[Computer Science/Computer System/Little and Big Endian\|endianness]]. היה אפשר גם לשמור את כל ה8 בתים בבת אחת אבל היה צריך לשים שהייצוג צריך להיות לפי האינדיאניות של המערכת.
### bitwise
![Pasted image 20240325102933.png|350](/img/user/Assets/Pasted%20image%2020240325102933.png)
צריך לשים לב שהפקודות הנ״ל עובדות על packed data ולכן הן מעדכנות את כל הבתים ב xmm registers גם אם השתמשנו רק בחלק מהם.

![Pasted image 20240325104535.png|300](/img/user/Assets/Pasted%20image%2020240325104535.png)
קטע הקוד הבא הופך מספרים בוקטור לתצורה השלילית שלהם באמצעות xor. המספרים שרשומים שם פשוט מרכיבים מספר בינארי שה MSB שלו הוא 1 וכל השאר אפסים. לכן הוא יהפוך את הביט סימן עבור כל float בוקטור וכל השאר יחזיר את אותו הערך של הביט שהיה מקודם בעת ביצוע xor. 

### Comparison
![Pasted image 20240325104901.png|300](/img/user/Assets/Pasted%20image%2020240325104901.png)

*  S2 חייב להיות xmm.
* S1 יכול להיות גם כתובת זכרון.
* הפעולה הזאת מדליקה את ה EFlags registers המוכרים ומדליקה גם כמה flags פחות מוכרים 
	* unordered אם אחד המספרים הוא NAN 
	* parity אם התוצאה היא NAN

![Pasted image 20240325105229.png|200](/img/user/Assets/Pasted%20image%2020240325105229.png)

נראה קוד c שבודק האם מספר הוא NAN ואיך זה נעשה באסמבלי

```C
f(float x) {
	if(x<0) return -1;
	if(x==0) return 0;
	if(x>0) return 1;
	if(isnan(x)) return 2;
}
```

![Pasted image 20240325111650.png|350](/img/user/Assets/Pasted%20image%2020240325111650.png)

הקוד בודק אם jp דולק (parity flag) ואם כן הוא קופץ לקוד NaN ומחזירים 2.

__Printf על float numbers__
נסתכל בקוד האסמבלי הבא 

![Pasted image 20240327000906.png|300](/img/user/Assets/Pasted%20image%2020240327000906.png)

1) הקוד מעתיק מהזכרון את הערכים ל xmm0 ו xmm1 בתצורת packed double.
2) מחברים בצורה מקבילית בין שני הוקטורים.
3) מעתיקים את התוצאה שנמצאת ב xmm0 ל xmm1
4) _מזיזים את xmm1, שמונה בתים ימינה_- את זה עושים כדי לחלץ את הערך השני בוקטור.
	1) fmt הוא הפורמט שprintf מקבל והוא מדפיס שני floats, במצב זה printf מחפש אותם ב low ordered bytes של xmm0 ו xmm1 כי printf מתייחס לוקטורים האלה כסקלר.
5) מעבירים את המידע שצריך כמו מספר הארגומנטים והפורמט ל printf ומפעילים אותו.
6) נשים לב שהשתמשנו ב align כדי למנוע התנהגות לא צפויה כפי שציינו כבר שיכול לקרות. 

### חישובים אריתמטיים 
נחשב את הנוסחה של [[Computer Science/Probability/Further Topics on Random Variables#קונבולוצייה, שונות משותפת , פונקציות על מ״מ ועוד\|שונות משותפת]] .

![Pasted image 20240327001401.png|300](/img/user/Assets/Pasted%20image%2020240327001401.png)

בקוד c סדרתי הפתרון די פשוט 
```C
#include <math.h>
double corr ( double x[ ], double y[ ], long n ) { 
   double sum_x, sum_y, sum_xx, sum_yy, sum_xy; 
   sum_x = sum_y = sum_xx = sum_yy = sum_xy = 0.0; 

   for (long i = 0; i < n; i++ ) { 
        sum_x += x[i]; 
        sum_y += y[i]; 
        sum_xx += x[i]*x[i]; 
        sum_yy += y[i]*y[i];      
        sum_xy += x[i]*y[i]; 
   } 

   return (n*sum_xy-sum_x*sum_y) / 
                sqrt((n*sum_xx-sum_x*sum_x) * 
                (n*sum_yy-sum_y*sum_y)); 
}
```

נשים לב לרמז שמאפשר לנו להבין שניתן להשתמש ברכיבי AVX לייעול האלגוריתם -
יש כאן מספר סכומים שמצטברים בכל איטרציה בנפרד ולבסוף מתחברים ביחס באיזשהו אופן.

הרעיון:
1) xmm0 יכיל בכל פעם זוג מהסכום של $x$ 
2) xmm1 יכיל בכל פעם זוג מהסכום של y
3) xmm2 יכיל בכל פעם זוג מהסכום של $x^{2}$ 
4) xmm3 יכיל בכל פעם זוג מהסכום של $y^{2}$ 
5) xmm4 יכיל בכל פעם זוג מהסכום של xy 
6) נוכל לעצור כאן ולבנות לולאה שכל פעם זזה בקפיצות של 8 בתים או לעשות loop unroll ולהשתמש בעוד רגיסטרים כדי לצבור את האיברים הבאים ולזוז בקפיצות של 16 בתים ואף יותר. 
7) בסופו של דבר, כל איבר בוקטורים שנשתמש בהם יכיל את הסכימה של האיברים הזוגיים במיקום הראשון ובמיקום השני את הסכימה של האיברים האי זוגיים (אם נבצע loop unroll אז הקפיצות בכל איבר בוקטור יהיו גדולות יותר מזוגיים ואי זוגיים).
8) מבצעים את הloop unroll באופן כזה שתהיה חוקיות בין הרגיסטרים, למשל xmm0 ו xmm8 יכילו את הסכומים של x  וגם xmm1 ו xmm9 יכילו את הסכומים של y וכן הלאה (בקפיצות של 8).

![Pasted image 20240327010629.png|300](/img/user/Assets/Pasted%20image%2020240327010629.png)
![Pasted image 20240327010710.png|300](/img/user/Assets/Pasted%20image%2020240327010710.png)

בסוף האיטרציות נקבל בערך משהו כזה

![Pasted image 20240327010755.png|300](/img/user/Assets/Pasted%20image%2020240327010755.png)

וכעת נוכל לסכום את כולם ביחד

![Pasted image 20240327011252.png|300](/img/user/Assets/Pasted%20image%2020240327011252.png)

__horizontal add__ 
כעת לחלק האחרון שנשאר, לחבר את האיבר הראשון בכל וקטור עם האיבר השני שלו, לשים את התוצאה ב8 הבתים הראשונים ולקבל מספר double שהוא התוצאה בכל הרצויה לכל חלק (מעין מעבר בין packed ל scalar). 

את זה נעשה עם הפקודה haddpd. הקלט שלה הוא שני רגיסטרים/ רגיסטר ווקטור והיא מבצעת בידיוק את הנ״ל עבורם ושמה את התוצאה בכל איבר בוקטור הקלט הראשון.

![Pasted image 20240327011700.png|250](/img/user/Assets/Pasted%20image%2020240327011700.png)
![Pasted image 20240327011640.png|250](/img/user/Assets/Pasted%20image%2020240327011640.png)

כעת כל שנשאר לעשות הוא
1) להמיר את n לdouble באמצעות cvtsi2sd (convert scalar int to scalar double).
2) לבצע את פעולות הכפל והחילוק על התוצאות שהתקבלו כתוצאה מhadd ולהחזיר את התוצאה ב xmm0

![Pasted image 20240327110808.png|300](/img/user/Assets/Pasted%20image%2020240327110808.png)
![Pasted image 20240327110831.png|300](/img/user/Assets/Pasted%20image%2020240327110831.png)


הקוד הנ״ל משתמש ברגיסטרים הבסיסיים ביותר בגודל 128 ביט. יכלנו להשתמש ברגיסטרים של AVX (ymm) כדי לשפר את התוצאה אף יותר 
1) להחליף את הרגיסטרים xmmi ב ymmi.
2) להוסיף v לפני כל פקודות האריתמטיקה
3) להשתמש ב vhaddpd - כעת יש 4 ערכים בוקטור אז הפקודה סוכמת את השניים הראשונים ושמה אותם במקום הראשון של הוקטור ואת השניים האחרונים ושמה אותם במקום האחרון של הוקטור.
4) להשתמש ב vextractf128 כדי שתי ערכים עליונים מוקטור ymm לרגיסטר xmm . 

## C Intrinsics 
המקבילה של פקודות SSE אבל בשפת c. 