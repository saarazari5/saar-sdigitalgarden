---
{"dateCreated":"2024-02-22 14:51","tags":["concurrency"],"pageDirection":"rtl","dg-publish":true,"permalink":"/computer-science/programming-concepts/concurrency-programming/open-mp/","dgPassFrontmatter":true}
---

# OpenMP
OpenMP API הוא אוסף כלים שמאפשרים תכנות מקבילי בשפת c ו cpp ושפות נוספות.
[[pthreads\|pthreads]] הוא עדיין הכלי העדיף מבחינת ביצועים וגמישות. עם זאת, מבחינת רמת האבסטרקצייה והנוחות למתכנת עם ביצועים טובים עבור הרבה תבניות עיצוב מקביליות. בכל מקרה, ניתן לכתוב קוד שמשלב בין השניים. 

OpenMP מתלבשת על כמה שכבות ב software stack, מרכיבים בסיסיים של OpenMP הם compiler directives, OpenMP library ו- environment variables.

![Pasted image 20240222150516.png|350](/img/user/Assets/Pasted%20image%2020240222150516.png)

נסתכל על קוד hello world לדוגמה ב OpenMP:

```C
#include <omp.h>
#include <stdio.h>

int main() {
	#pragma omp parallel
		{
			printf(" hello ");
			printf(" world\n ");
		}
	return 0;
}
```

השורה `pragma omp parallel` היא הנחיות לקומפיילר. 

הבלוק בפנים חייב להיות עם שורת כניסה אחת ושורת יציאה אחת. הבלוק שבפנים נקרא parallel region.

נבחן את פלט התוכנית:
![Pasted image 20240222151243.png|200](/img/user/Assets/Pasted%20image%2020240222151243.png)ראשית, נשים לב שקימפלנו עם הדגל fopenmp.
זה דגל שאומר לgcc להתייחס לdirectives שהגדרנו בקוד (במקרה הזה השורה pragma omp parallel). 

נתנו הוראה ל- gcc להריץ את הקוד שב- block במקביל עם מספר default של threads, ששווה (בדרך כלל) למספר cores שיש במחשב כפול מספר ה Hardware threads שיש בכל core. לכן הקוד במקרה הזה, מתבצע במקביל ע"י 4 threads ואנחנו רואים 4 הדפסות של hello world.

* נשים לב שבסוף ה pragma block הקומפיילר מוסיף wait אוטומטי, כלומר רק אחרי שכל הthreads מסיימים את הבלוק, הקוד ימשיך לרוץ לכל שאר הקוד שמחוץ לבלוק.


>[!info] CPU Info
>כדי לקבל מידע מפורט על ארכיטקטורת המחשב שברשותנו נוכל להשתמש בפקודת lscpu. אם נרצה רק את מספר הליבות נשתמש בפקודת nproc 
>![Pasted image 20240222172229.png](/img/user/Assets/Pasted%20image%2020240222172229.png)
>

__num of threads:__
כדי להגדיר כמה threads נרצה כדי להריץ קטע קוד מקבילי נשתמש בפקודה omp_set_num_threads(n) לפני פתיחת הבלוק.

כדי לקבל את המזהה של כל thread נריץ את הפקודה omp_get_thread_num .

באופן סכמתי אפשר לתאר תוכנית המשתמשת ב pragma באופן הבא-
![Pasted image 20240308141508.png](/img/user/Assets/Pasted%20image%2020240308141508.png)

* ניתן לראות שניתן להכניס parallel regions בתוך parallel regions בעת הצורך
* כל תהליך מקבילי מחכה שכל הthreads בתוכו יסיימו כדי להמשיך בחלק הסדרתי של ה main thread .
* כדאי לשים לב ש omp עלול לייצר פחות threads ממה שרצינו.
* הסכמה המתוארת נקראת [Fork-Join parallelism](https://en.wikipedia.org/wiki/Fork%E2%80%93join_model)

__Fork-Join parallelism:__
קטעים מסוימים מהקוד ממוקבלים, קטע קוד מקבילי הבא מתחיל רק כאשר קטע קוד מקבילי הקודם לו מסתיים. ההנחה של שיטה זו היא שהתוכנית מתחילה מthread יחיד שנקרא master thread או main וממנו יוצאים כל הקטעים המקביליים.

![Pasted image 20240308220651.png](/img/user/Assets/Pasted%20image%2020240308220651.png)

## חישוב Euler Mascheroni constant
נראה איך מחשבים באופן סדרתי ועם OMP את הנוסחה של [קבוע אוילר](https://en.wikipedia.org/wiki/Euler%27s_constant) 

$$\lim_{ n \to \infty } \log(n) - \sum^{n}_{i=1} \frac{1}{i} \approx -0.577216 $$

החישוב בצורה סדרתית הוא פשוט 

```C 
double seq_euler_mascheroni() {
	double sum = 0;

	for( int i = 1; i < N; i ++){
		sum+= 1.0/i;
	}

	return log(N) - sum;
}
```

כעת נשווה לגרסה המקבילית 

```C
double sum = 0;

#pragma omp_parallel 
{
	const int num_threads = omp_get_num_threads();

	double thread_sum = 0;
	int tid = omp_get_thread_num + 1;

	for(int i= tid; i < N; i+= num_threads) {
		threads_sum += 1.0/i;
	}

	#pragma omp critical
		sum += thread_sum
}
```

* sum הוא משתנה גלובלי
* כל thread מחזיק את מספר הthreads ו thread sum עם ערך התחלתי 0
* כל thread מחזיק את ה thread id שלו כאשר זה מספר טבעי שגדל ב1 עבור כל thread לפי סדר היצירה
* כל אחד מהthreads מחשב את תת הסכום שהוא מתחיל מהמזהה של הthreads וקופץ ב num of threads עד שמגיע למספר היעד N
* כעת מגדירים לקומפיילר קטע קוד קריטי שסוכם את המידע למשתנה הגלובלי.


אם נבצע benchmark באמצעות הפונקציה `omp_get_wtime` נקבל זמן ריצה שמשמעותית יותר מהיר באלגוריתם המקבילי 

![Pasted image 20240308150356.png](/img/user/Assets/Pasted%20image%2020240308150356.png)

## False sharing
נסתכל על קוד סדרתי ומקבילי לסכימת מערך 

![Pasted image 20240308151108.png|350](/img/user/Assets/Pasted%20image%2020240308151108.png)
![Pasted image 20240308151057.png|350](/img/user/Assets/Pasted%20image%2020240308151057.png)

האלגוריתם המקבילי מחשב את הסכומים החלקיים ולבסוף מחבר את הכל ביחד. נשים לב שהוא משתמש פה בפקודת omp single שזאת פקודה שמאפשרת רק לthread אחד להכנס אליו ולאחר מכן לא מכניסה יותר threads לקטע הקוד הזה (אם לא היינו משתמשים ב OpenMP לכתוב דבר כזה היה דורש מנגנון משלנו). השימוש בפקודה זאת היא טובה כאשר נרצה להזין פרמטר רק פעם אחת במקרה הזה thread_num הוא הפרמטר הזה.

אם נבדוק את תוצאות ההרצאה עבור מערכים גדולים במיוחד נקבל
![Pasted image 20240308151300.png](/img/user/Assets/Pasted%20image%2020240308151300.png)
כלומר אנחנו רואים שזמני ההרצה המקביליים גרועים יותר מהסדרתיים.

זה יכול לנבוע מכמה סיבות:
	1) overhead 
	2) [[Computer Science/Computer System/Computer System Cache Memory\|cache]]- לכל thread שרץ על מעבד אחר יש L1 cache משלו. נזכיר שכאשר קוראים מידע מהזכרון ושמים ב cache שמים גם את המידע מסביב באותו cache line כדי לשמור על לוקליות גבוהה. בתכנות מקבילי multicore כאשר thread משנה את הcache במעבד שלו, החומרה מודיעה לשאר המעבדים שהcache השתנה ומתרחש cache invalidation מה שגורם לכל L1 cache של הthreads לגשת שוב לRAM על כל פעולת חיבור מה שיוצר חוסר ניצול של הcache .

__מה הפתרון?__
נוכל לשנות את Partial sum לעבור עם מערך דו מימדי וכל threads יקבל שורה משלו וככה לא יהיה מצב שthreads דורסים את הlines אחד של השני.

![Pasted image 20240308152659.png](/img/user/Assets/Pasted%20image%2020240308152659.png)

ניתן לראות שזה אכן משפר משמעותית את הביצועים של תוכנית מקבילית
![Pasted image 20240308152712.png](/img/user/Assets/Pasted%20image%2020240308152712.png)

## PI Approximation 
אנחנו יודעים שמתקיים

$$\int_{0}^{1} \, \frac{4.0}{1+x^{2}}dx = \pi $$

כלומר נוכל לקבל קירוב של $\pi$ באמצעות חישוב של [[Computer Science/Calculus/The Integral\|האינטגרל]] כסכום של N שטחי מלבנים 

$$\sum_{i=0}^{n} F(x_{i})\cdot \Delta x\approx \pi$$

![Screenshot 2024-03-08 at 17.55.15.png|300](/img/user/Assets/Screenshot%202024-03-08%20at%2017.55.15.png)

נכתוב קוד מקבילי שמאפשר לנו לחשב את זה ונבחן את הביצועים שלו לעומת קוד סדרתי

החישוב המקבילי יהיה דומה למה שעשינו בדוגמאות הקודמות, כל thread יבצע סכום חלקי של שטחי המלבנים ולבסוף בקטע קוד סדרתי יסכום את כל הסכומים החלקיים. את הסכומים ניתן לשמור באמצעות מערך שגודלו כמספר הthreads וככה כל thread מקבל משבצת אחת במערך מבלי לדאוג לrace condition (כפי שהראנו, זה יכול לפגוע בביצועים אם לא לוקחים בחשבון את המטמון).

![Pasted image 20240308180620.png|300](/img/user/Assets/Pasted%20image%2020240308180620.png)

בתכנות מקבילי זה ייראה ככה
```C
#include <omp.h>
#include<stdio.h>

	#define N 2 // required number of threadsint main ()
	
    int N_actual; // actual number of threads
    long num_steps = 100000;
    double sum[N] = {0};
    double step = 1.0/(double) num_steps;

    omp_set_num_threads(N);
    #pragma omp parallel
    {
        int id = omp_get_thread_num();
        double x;
        #pragma omp single
           N_actual = omp_get_num_threads();

        for (int i=id; i< num_steps; i+= N_actual) {
            x = (i+0.5)*step;
            sum[id] += 4.0/(1.0+x*x);
        }
    }
    double pi = 0;
    for (int i=0; i < N_actual; i++)
        pi += step * sum[i];
    printf("%f\n", pi);
    return 0;
}
```

אם נמדוד ביצועים נקבל 
![Pasted image 20240308181117.png|300](/img/user/Assets/Pasted%20image%2020240308181117.png)

ניתן לראות שהחל מ N=3 יש ירידת ביצועים. הסיבה לכך היא מכיוון שבמעבד אחד יש כמה ליבות ובכל ליבה ניתן להריץ מספר HW threads (בד״כ 2) ואחד הפתרונות החומרתיים שיש למעבדים __מרובי ליבות__ הוא שברגע שיש שני cores על אותה תוכנית וה L1 cache של אחד מהם משתנה , הוא מודיע לL1 cache של האחרים שהם invalid ועליהם לעדכן את המטמון שלהם.

ניקח ביצוע של ארבעה threads . מכיוון שכל core מכיל שני HR (Hardware threads), הוא יכול להריץ שני threads. ארבעה threads ירוצו על שני cores שונים. לכל core יש L1 cache פרטי משלו. כאשר thread0 רוצה לעדכן $sum[0]$, חלק של RAM בגודל cache line (בגודל 64 bites עבור 64-bit architecture) מובא ל- L1 cache של core אשר מריץ את thread1. 

![Pasted image 20240308182134.png|300](/img/user/Assets/Pasted%20image%2020240308182134.png)

כאשר thread2 רוצה לעדכן $sum[2]$, חלק של RAM בגודל cache line מובא ל- L1 cache של core אשר מריץ את thread2. נשים לב ששני cache lines מכילות את המערך שלו. ברגע ש- thread2 מבצע עדכון של $sum[2]$, ה- cache line הזה ב- L1 cache of Core1 מסומן כ- invalid, כלומר, לא לשימוש. זה מכיוון שהוא מכיל data לא עדכני. כאשר thread0 ירצה לעדכן שוב פעם את $sum[0]$, הוא יצטרך לבצע גישה ל- RAM כדי להביא את ה- cache line מחדש.

![Pasted image 20240308182644.png|300](/img/user/Assets/Pasted%20image%2020240308182644.png)

הפתרון הוא הקצאה של מערך דו מימדי שתמנע חפיפה בין cache lines של threads שונים. אומנם, זה בזבוז זכרון RAM, אבל זכרון הוא מדד משני לעומת זמן ריצה.

```C
#include <omp.h>
#include<stdio.h>

#define N 2 // required number of threads
#define PAD 8 // assume 64-byte L1 cache line size
int main () { 
    int N_actual; // actual number of threads
    long num_steps = 100000000;
    double sum[N][PAD] = {{0}};
    double step = 1.0/(double) num_steps;

    double tdata = omp_get_wtime();
    omp_set_num_threads(N);
    #pragma omp parallel
    {
        int id = omp_get_thread_num();
        double x;
        #pragma omp single
           N_actual = omp_get_num_threads();

        for (int i=id; i< num_steps; i+= N_actual) {
            x = (i+0.5)*step;
            sum[id][0] += 4.0/(1.0+x*x);
        }
    }
    double pi = 0;
    for (int i=0; i < N_actual; i++)
        pi += step * sum[i][PAD];
    tdata = omp_get_wtime() - tdata;
    printf("pi = %f in %f secs\n", pi, tdata);
    return 0;
}
```

כיוון שכל thread עובד על שורה שלו במערך בהתאם ל id, לכן אין שורות cache משותפות.

![Pasted image 20240308183013.png|300](/img/user/Assets/Pasted%20image%2020240308183013.png)

הזמן ריצה אכן השתפר גם עבור N=3,4. המחיר שקיבלנו הוא קוד מאוד מסורבל. ננסה לבחון דרכים נוספות.

כלי נוסף הוא הגדרת critical section בתוך הקוד המקבילי , במקום להחזיק מערך של סכומים חלקיים נוכל לדאוג שכל thread יחשב את הסכום שלו בצורה מקומית ובקטע הקוד הקריטי נסכום את התוצאה הסופית לערך של $\pi$ .

```C
#include <omp.h>
#include<stdio.h>

#define N 2 // required number of threadsint main () { 
    long num_steps = 100000000;
    double pi = 0;
    double step = 1.0/(double) num_steps;

    double tdata = omp_get_wtime();
    omp_set_num_threads(N);
    #pragma omp parallel
    {
        int id = omp_get_thread_num();
        double x, sum = 0;
        #pragma omp single
           N_actual = omp_get_num_threads();

        for (int i=id; i< num_steps; i+= N_actual) {
            x = (i+0.5)*step;
            sum += 4.0/(1.0+x*x);
        }
        #pragma omp critical
            pi += sum * step;
    }
    tdata = omp_get_wtime() - tdata;
    printf("pi = %f in %f secs\n", pi, tdata);
    return 0;
}
```

בשיטה זו נקבל ביצועים דומים לשיטה הקודמת בקוד יותר מסודר. 
## Loops 
כמה כללי לולאות חשובים ב OpenMP:
1) אם for loop מופיע בקטע קוד מקבילי, אז כל אחד מה threads יבצע את הלולאה במלואה.
```C
#include <omp.h>
#include<stdio.h>

int main () { 
    #pragma omp parallel
    {
        for(int i=0; i<2; i++)
            printf("i=%d, ID=%d\n", i, omp_get_thread_num());
    }
    return 0;
}

```
2) אם רוצים לבצע לולאה פעם אחת בלבד, אבל לחלק אותה לחלקים כך שכל חלק יתבצע על ידי thread אחר אז ניתן להשתמש בpragma omp for בתוך parallel block אוpragma omp parallel for. 

```C
#include <omp.h>
#include<stdio.h>

int main () { 
    #pragma omp parallel
    {
          #pragma omp for
            for(int i=0; i<2; i++)
                printf("i=%d, ID=%d\n", i, omp_get_thread_num());
    }
    return 0;
}

//or use:
int main () { 
    #pragma omp parallel for
     for(int i=0; i<2; i++)
        printf("i=%d, ID=%d\n", i, omp_get_thread_num());
  
    return 0;
}
```

נשים לב שבדוגמת הקוד האחרון נקבל 2 threads ולא 4 כמו ברוב המקרים הדיפולטים בגלל שהקומפיילר בשיתוף עם מערכת ההפעלה החליטו להקצות פחות ממה שהוא יכול כי מספר האיטרציות הוא 2.
### Nested loops
כאשר יש nested loop שכתוב בתוך בלוק מקבילי, רק הלולאה החיצונית תתמקבל. 

![Pasted image 20240308213755.png|350](/img/user/Assets/Pasted%20image%2020240308213755.png)

זאת __ההתנהגות הדיפולטיבית__ אם נרצה לשנות אותה נוכל להשתמש במילה השמורה collaps(#loops). זה אומר לקומפיילר שאנחנו רוצים למקבל את כל הלולאות הפנימיות עד הרמה שהיא המספר שהעברנו 

![Pasted image 20240308214014.png|350](/img/user/Assets/Pasted%20image%2020240308214014.png)

במקרה זה, כל thread ביצע איטרציה אחת. נשים לב שבקוד זה בניגוד לקודם לא פגענו ברמת המקביליות וגם לא בנכונות של הקוד. 

__End of loop barrier:__
נשים לב שיש barrier לא מפורש בסוף לולאת for מקבילית. כלומר threads יחכו בסוף הלולאה לשאר הthreads שרצים שיסיימו את הלולאה גם כן. 

![Pasted image 20240308214542.png|350](/img/user/Assets/Pasted%20image%2020240308214542.png)

ניתן לראות לפי ההדפסה שכל הthreads חיכו עד שכולם יסיימו את הלולאה ורק אז התחילו בהדפסה.

אם נרצה להוריד את הלוגיקה הזאת, נוכל להוסיף את מילת המפתח ליד הצהרת הparallel for שנקרא nowait. ברגע שמוסיפים את המילה הזו אז הקומפילר לא יציב מחסום סמוי היכן שהצהרנו (אם יש עוד הצהרות בהמשך אז שם לא יבוטל המחסום).

![Pasted image 20240308214759.png|350](/img/user/Assets/Pasted%20image%2020240308214759.png)

### Reduction 
נסתכל על הקוד הבא
```C
#include <omp.h>
#include<stdio.h>

int main () { 
    int sum = 0;
    #pragma omp parallel 
    {
          #pragma omp for reduction (+:sum)
               for(int i=0; i<10; i++) {
                    sum += i;
                    printf("thread%d sum = %d\n", 
                              omp_get_thread_num(), sum);
               }
    }
    printf("sum = %d\n", sum);
    return 0;
}
```

ננסה להבין מה קורה כאן על ידי התבוננות בהדפסות 

![Pasted image 20240308215107.png|250](/img/user/Assets/Pasted%20image%2020240308215107.png)

* ניתן לראות שחלק מהthreads קיבלו יותר איטרציות מאחרים.
* החלוקה של מספר האיטרציות לכל threads היא חלוקה _סטטית_ כלומר כל הthreads מקבלים מראש בזמן קימפול את מספר האיטרציות שכל אחד מהם יריץ. זה מתבצע על ידי חלוקת האיטרציות ל chunks וכל thread מקבל chunk מתאים.
* נשים לב שנראה מההדפסות שלמרות שהעברנו ל pragma את sum כפרמטר הוא שכפל אותו לכל אחד מהthreads והפך אותו למשתנה מקומי עבור כל thread. כלומר, כל אחד מהם חישב את הסכום החלקי לפי האיטרציות שהוא קיבל.
* רק לבסוף כאשר יוצאים מהבלוק sum החיצוני קיבל את הערך המתקבל מחיבור כל הסכומים החלקיים.

_אם כן, מהו הכלי reduction?_
הכלי הזה מקבל זוג סדור מהצורה $\text{(op:var)}$, כל אחד מהthreads מקבל עותק פרטי של משתנה var שמאותחלים לפי סוג פעולה op. לאחר מכן הפעולה מתבצעת בכל איטרציה מקומית עבור הthread. בסוף כל המשתנים הפרטיים var מסתמכים על ידי הפעולה op לתוך המשתנה המשותף.

אלה הן הפעולות האפשריות וערך האתחול עבורן-

![Pasted image 20240308220232.png|200](/img/user/Assets/Pasted%20image%2020240308220232.png)

נראה כעת כיצד למקבל את קוד ה Pi Approximation ממקודם באמצעות reduction 

![Pasted image 20240309003624.png](/img/user/Assets/Pasted%20image%2020240309003624.png)
כעת ניתן לראות איך במימוש מאוד נקי חישבנו את הקירוב של $\pi$ בניגוד לקוד הקודם שהיה מאוד מסורבל ולא נקי.

__Reduction with Arrays:__
![Pasted image 20240309003806.png](/img/user/Assets/Pasted%20image%2020240309003806.png)
ניתן לבצע reductio גם על מערכים ולייצר שכפול שלהם לכל thread. 
## Schedule 
נסתכל על קטע הקוד הבא
![Pasted image 20240308222126.png](/img/user/Assets/Pasted%20image%2020240308222126.png)
הלולאה החיצונית מתחלקת שווה (ביחס למספר האיטרציות) בין ארבעת הthreads שנוצרו. 

הלולאה הפנימית היא מקומית לכל threads כפי שאמרנו.
נשים לב שאומנם החלוקה של מספר האיטרציות שווה, אבל כל אחד מהthreads מריץ מריץ עם ערכי i שונים. לכן תהיה פה בעיה שthreads שמקבלים את 3 האיטרציות הגבוהות יותר יבצעו יותר פעולות חישוביות מאופי הגדרת הלולאה הפנימית.

![Pasted image 20240308222745.png](/img/user/Assets/Pasted%20image%2020240308222745.png)
ההיגיון של הלולאה הזאת פגע ב load balance שכן העבודה לא מתחלקת שווה בשווה. 
### Static Scheduling
כדי לטפל בזה נשתמש בkey work שנקרא schedule. המילה השמורה הזו מקבל כפרמטר את אופי ההקצאה (static או dynamic ויש גם נוספים) ופרמטר נוסף שהוא מספר שקובע ״כמה איטרציות נרצה להקצות לכל thread עבור ריצה אחת לפני שהוא ״מחליף״ לthread הבא. 
![Pasted image 20240308223157.png](/img/user/Assets/Pasted%20image%2020240308223157.png)
כאשר נשים 1 בקוד הנ״ל נקבל התנהגות שמזכירה [[Computer Science/Operating Systems/CPU Scheduling\|Round Robin]], כל thread ירוץ איטרציה אחת ואז הthread הבא יריץ את האיטרציה הבאה בתור וכן הלאה. 

![Pasted image 20240308223446.png](/img/user/Assets/Pasted%20image%2020240308223446.png)

אם היינו מגדילים את הפרמטר מ 1 ל 2 היינו מקבלים load balance פחות טוב:

![Pasted image 20240308224048.png](/img/user/Assets/Pasted%20image%2020240308224048.png)

אין באמת תשובה נכונה לקביעת גודל chunk מתאים אך חשוב לציין ששימוש ב static scheduling הוא מאוד יעיל כי אין צורך בכלל לתקשורת בין הthreads שכן בזמן הריצה כל אחד יודע בידיוק מה עליו להריץ וכמה איטרציות.

>[!caution] 
>חלוקה סטטית יכולה להיות לא מוצלחת כאשר לא ניתן לדעת מראש כמה חישובים יש בכל איטרציה למשל אם בקוד למעלה היינו מגדירים במקום לרוץ עד i לרוץ עד rand היינו יכולים לקבל חלוקה לא שיוויונית שפוגעת ב balance.
>![Pasted image 20240308224623.png](/img/user/Assets/Pasted%20image%2020240308224623.png)
### Dynamic Scheduling
למקרים כמו הנ״ל ניתן להגדיר גם שהחלוקה תהיה dynamic.
במצב זה כל thread מקבל chunk של איטרציות לפי הפרמטר שהגדרנו ומספר האיטרציות בלולאה החיצונית ומבצע אותן אבל הוא מחלק את העבודה בצורה דינמית בלי בהכרח סדר ספציפי אלא עם אלגוריתם שמזכיר [[Computer Science/Programming Concepts/Concurrency Programming/Thread Pool\|Thread Pool]]. 

```C
int main()
{
#pragma omp parallel for schedule(dynamic, 1) 
for (int i = 0; i < 20; i++)
	{
		printf("Thread %d is running number %d\n",omp_get_thread_num(),i);
	}
	return 0;
}
```

הפלט יהיה :

```CLI
Thread 1 is running number 2
Thread 1 is running number 7
Thread 1 is running number 9
Thread 1 is running number 10
Thread 1 is running number 11
Thread 1 is running number 13
Thread 1 is running number 14
Thread 1 is running number 15
Thread 1 is running number 17
Thread 1 is running number 19
Thread 3 is running number 0
Thread 0 is running number 4
Thread 8 is running number 12
Thread 4 is running number 3
Thread 6 is running number 6
Thread 9 is running number 16
Thread 5 is running number 1
Thread 7 is running number 8
Thread 10 is running number 18
Thread 2 is running number 5
```

חשוב לציין שההקצאה היא [[Computer Science/Algorithms/greedy algorithms\|חמדנית]] בשיטה שלפיה עובד התזמון הדינמי הוא First Come First Serve כלומר, אם ישנן איטרציות שצריך לבצע ויש thread פנוי שיכול לקחת אותן, הוא ייקח אותן.

כמו כן, התזמון הדינמי באופן די אינטואיטיבי, הרבה יותר יקר מהסטטי כי נדרש תקשורת בין ה threads לאחר כל איטרציה, כמו כן לעיתים צריך להגדיר את הchunk size של thread כלשהו וגם בגלל שהתזמון הוא מעין קופסא שחורה של הספרייה לא בהכרח שנקבל תזמון שמתאים לנו.
## Shared/Private variables
כאמור משתנים המוגדרים מחוץ לקטע המקבילי הם shared.
משתנים המוגדרים בתוך קטע מקבילי הם private, כלומר כל thread מקבל עותק של משתנים אלו. 

![Pasted image 20240308230822.png|250](/img/user/Assets/Pasted%20image%2020240308230822.png)

נשים לב שגם משתנים סטטיים בתוך בלוק מקבילי, הם shared.

ניתן לשנות את נראות ברירת מחדל של המשתנים:
![Pasted image 20240308231021.png](/img/user/Assets/Pasted%20image%2020240308231021.png)
i הופך אוטומטי למשתנה פרטי ברגע ששמים אותו בתוך parallel for ו j הפך לפרטי בגלל שהשתמשנו בהגדרה private(j). 

הסיבה שהגדרנו את המערכים כstatic כדי שהם ישבו ב global data segment ולא בstack וכך נמנע מ seg fault (מערכת ההפעלה מגדירה ל stack גודל קבוע ולפעמים היא לא תקצה עוד עבור הstack , ניתן לבדוק את גודל הstack הקבוע __לכל תוכנית__ על ידי הרצת התוכנית ulimit -s).

![Screenshot 2024-03-08 at 23.17.22.png](/img/user/Assets/Screenshot%202024-03-08%20at%2023.17.22.png)

__נשים לב:__ ההצהרה private מייצרת עותק של המשתנה ולכן נוצר עותק של המשתנה בתוך הthreads, הערך של העותקים לא בא לידי ביטוי במשתנה המקורי כלומר כל שינוי בתוך הקוד המקבילי לא משפיע על המשתנה המקורי. כמו כן אין הבטחה שגם הערך של המשתנה המקורי יעבור לעותק למשל:

![Pasted image 20240309005307.png](/img/user/Assets/Pasted%20image%2020240309005307.png)
אומנם הערך היה 1, אבל כל אחד מהthreads קיבל ערך התחלתי של 0 עבור העותק של sum.

אפשר להגדיר משתנה להיות private רק לחלק מסויים מתוך הקוד המקבילי. 
![Pasted image 20240309005450.png](/img/user/Assets/Pasted%20image%2020240309005450.png)

במקרה הזה ניתן לראות איך לאחר היציאה מהלולאה sum נשאר עם ערך 1.

* אם אכן חשוב לנו לשמור על ערך האתחול המקורי של המשתנה, במקום להשתמש בהצהרת private , נשתמש בהצהרה firstprivate

![Pasted image 20240309005812.png](/img/user/Assets/Pasted%20image%2020240309005812.png)

* אם נרצה שהsum החיצוני יקבל את הערך הסופי שקיבל משתנה באיטרציה המקבילית __האחרונה__ נשתמש במילת המפתח lastprivate.

![Pasted image 20240309005924.png](/img/user/Assets/Pasted%20image%2020240309005924.png)

__מה הערך שיודפס כאן?__
![Pasted image 20240309005949.png](/img/user/Assets/Pasted%20image%2020240309005949.png)
הfirstprivate אומר שכל הthreads יקבלו את הערך ההתחלתי בעותק שלהם, שהוא 0. הlastprivate אומר שנשים בsum החיצוני את הערך שקיבל הסכום החלקי של הthread האחרון. כיוון שבדיפולט מדובר בניהול סטטי אבל החלוקה כאן לא שווה (יש 9 איטרציות על 4 threads) לא ניתן לחזות את התוצאה מראש. נניח ש thread0 יקבל את 3 האיטרציות הראשונות ונבחן את ההדפסות

![Pasted image 20240309011145.png|200](/img/user/Assets/Pasted%20image%2020240309011145.png)

ניתן לראות שכיוון ש ID3 קיבל רק את 2 האיטרציות האחרונות אז הוא יסכום את הסכום החלקי ש i=8,9 ולכן הפלט יהיה 17.
### default 
הרגל מומלץ הוא להשתמש במילת המפתח default(none) כדי לקבל בדיקת קומפילציה שמכריחה אותנו לרשום עבור כל משתנה, מהי רמת השיתוף שלו. גורם לקוד להיות מובן יותר. 

![Pasted image 20240309011436.png](/img/user/Assets/Pasted%20image%2020240309011436.png)

## Algorithms with OpenMP
### חישוב מספרים ראשוניים
קלט: מספר טבעי n
פלט: כל המספרים הראשוניים מ 1 עד ל n.
```C
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

void calcPrimes(int n) {
    bool *prime = (bool *)malloc((n + 1) * sizeof(bool));
    if (prime == NULL) {
        printf("Memory allocation failed.\n");
        return;
    }

    // Initialize all elements of prime[] as true
    #pragma omp parallel for 
    for (int i = 0; i <= n; i++)
        prime[i] = true;

    // Mark non-prime numbers
    #pragma omp parallel for schedule(dynamic, 1)
    for (int p = 2; p * p <= n; p++) { 
            for(int d = 2; d<p && prime[p]; d++)
                 if(p % d == 0)
                     prime[p] = false;
    }

    // Print prime numbers
    printf("Prime numbers in the range 2 to %d are:\n", n);
    for (int p = 2; p <= n; p++) {
        if (prime[p])
            printf("%d ", p);
    }
    printf("\n");

    free(prime);
}
```

1) הלולאה שמבצעים אתחול למערך ה prime תהיה מקבילית
2) הלולאה הכפולה שרצה עד השורש של n בלולאה החיצונית ובלולאה הפנימית רצים על כל המספרים עד ל p כאשר המערך במיקום ה p הוא true ובודקים האם p מתחלק ב d ואם כן משנים את הערך של p במערך הראשוניים להיות false. את הלולאה הזאת נעשה בצורה מקבילית עם תזמון דינמי. הסיבה שלא השתמשנו ב collapse היא בגלל שהמקביליות של האיטרציה הפנימית עלול ליצור ריצות כפילות כיוון שכל thread בודק את המערך הגלובלי של המספרים הראשוניים ולכן שני threads יכולים לקבל מספרים שכל מהם משנה את המערך prime באותו המיקום אבל שניהם יגשו לערך הזה בגלל שהם עובדים במקביל ולא מסונכרנים.
3) הסיבה לשימוש ב dynamic היא שהאיטרציות הפנימיות הן לא בגודל שווה, לכן עדיף לנו לתזמן לפי FCFS ולתת לthreads שמסיימים קודם לכן את האיטרציות הקצרות יותר לקבל עבודות חדשות.
4) אין באמת סיבה chunk size = 1 או לשים מספר אחר, כדי לקבוע בידיוק צריך לעשות ניסוי ותהייה.
### BFS 
ראינו כבר דרך לחשב [[Computer Science/Algorithms/Parallel Algorithms/Parallel Graphs Algorithms#BFS\|bfs מקבילי]] עם pthreads. כעת נראה כיצד ניתן לחשב את זה בצורה מקבילית עם OpenMP.

```C
SEQUENTIAL_BFS(V, E, s)

   #pragma openmp parallel for
   for each 𝑣∈𝑉
         𝑐𝑜𝑙𝑜𝑟_𝑣←𝑤ℎ𝑖𝑡𝑒
         𝑑_𝑣←∞
         𝜋_𝑣←∅
   𝑐𝑜𝑙𝑜𝑟_𝑠←𝑔𝑟𝑒𝑦
   𝑑_𝑠←0
   define queue 𝑄←{𝑠}

   while 𝑄≠∅
          𝑣←𝑄.𝑑𝑒𝑞𝑢𝑒𝑢𝑒( )
          𝑐𝑜𝑙𝑜𝑟_𝑣←𝑏𝑙𝑎𝑐𝑘
          #pragma openmp parallel for
          for each 𝑢 adjacent to 𝑣
                 if 𝑐𝑜𝑙𝑜𝑟_𝑢 is 𝑤ℎ𝑖𝑡𝑒
                         #pragma omp critical
                                𝑄.𝑒𝑛𝑞𝑢𝑒𝑢𝑒(𝑢)
                         𝑐𝑜𝑙𝑜𝑟_𝑢←𝑔𝑟𝑒𝑦
                         𝜋_𝑢←𝑣
                         𝑑_𝑢←𝑑_𝑣+1

```

* מקבלנו את האתחול
* מקבלנו את האיטרציה על כל לולאה שרצה על השכנים של קודקוד v שיצא מהתור
* הגדרנו את ההשמה לתוך התור כקטע קוד קריטי.

השיטה הזאת היא בעייתית כפי שאמרנו יש צורך להגדיר תור לכל שכבה על מנת להמנע מrace condition שעלול לגרום לאלגוריתם למצוא מסלול ארוך יותר מהמסלול הקצר ביותר ״בטעות״.

ב OpenMP נוכל ליצור תור לכל רמה עם שני תורים בלבד. 
```C
SEQUENTIAL_BFS(V, E, s)

   #pragma openmp parallel for
   for each 𝑣∈𝑉
         𝑐𝑜𝑙𝑜𝑟_𝑣←𝑤ℎ𝑖𝑡𝑒
         𝑑_𝑣←∞
         𝜋_𝑣←∅
   𝑐𝑜𝑙𝑜𝑟_𝑠←𝑔𝑟𝑒𝑦
   𝑑_𝑠←0
   define queue 𝑄1←{𝑠}
   define queue 𝑄2←{ }

   while 𝑄1≠∅
      #pragma openmp parallel for
      for each v in Q1    
          𝑐𝑜𝑙𝑜𝑟_𝑣←𝑏𝑙𝑎𝑐𝑘
          #pragma openmp parallel for
          for each 𝑢 adjacent to 𝑣
                 if 𝑐𝑜𝑙𝑜𝑟_𝑢 is 𝑤ℎ𝑖𝑡𝑒
                         #pragma omp critical
                                𝑄2.𝑒𝑛𝑞𝑢𝑒𝑢𝑒(𝑢)
                         𝑐𝑜𝑙𝑜𝑟_𝑢←𝑔𝑟𝑒𝑦
                         𝜋_𝑢←𝑣
                         𝑑_𝑢←𝑑_𝑣+1
	    empty Q1
        swap Q1 and Q2
```

* הגדרנו שני תורים כאשר Q2 טוען כל פעם את הקודקודים של הרמה הבאה ו״מזין אותם״ ל Q1. 
* יש גם שני ״רמות״ של מיקבול, הראשונה היא בריצה על כל הקודקודים ברמה הנוכחית ב Q1 ושינוי הצבע לשחור, הרמה השנייה היא בתור הרמה הקודמת שבה רצים במקביל על כל u שכן של v ומזינים אותו לQ2. 
* נשים לב שלא שינינו את הend of loop barrier שכן נרצה שכל הthreads יזינו את השכנים לתור Q2 לפני ביצוע ה swap.
* כל המקביליות פה היא סטטית שכן, אין כאן צורך בתזמון דינמי כלל.
#### Bottom-up BFS algorithm
גישה שהוצג ב[Direction Optimizing BFS](https://parlab.eecs.berkeley.edu/sites/all/parlab/files/main.pdf) הציגה גישה שמתאימה יותר לתכנות מקבילי. הגישה הזאת אומרת שנחזיק מערך של frontier ובכל איטרציה bottom-up במקום לרוץ על כל השכנים של קודקוד $v\in frontier$ , נרוץ __על כל הקודקודים__ שלא הוגדר להם parent ברגע שמצאנו כזה רצים על כל השכנים שלו שנמצאים בfrontier ומבצעים עליהם BFS. במילים אחרות, נבדוק עבור כל הקודקודים האם יש להם שכן שהוא ברמה הנוכחית שצריך לסרוק. 

הסיבה שבצורה המקבילית השיטה הזאת יותר יעילה היא בגלל שאנחנו מונעים Race conditions רבים לאור העובדה שכל thread ניגש למערך parent באינדקס שמזוהה עם הקודקוד שעליו הוא עומד. נוכל גם להמנע מסנכון של קבוצת ה next נוכל לממש את זה באמצעות מערך עומקים וככה כל thread ניגש רק לכתובת המתאימה לקדוקוד המתאים עבורו.

האלגוריתם bottom-up בצורה סדרתית הרבה פחות טוב מהגישה הרגילה שאנחנו מכירים. 
המימוש הסדרתי נראה כך:
![Pasted image 20240309184052.png|300](/img/user/Assets/Pasted%20image%2020240309184052.png)
![Pasted image 20240309184021.png|300](/img/user/Assets/Pasted%20image%2020240309184021.png)

בצורה מקבילית , נוכל לאתחל מערך הparents בצורה מקבילית וגם בצורה דינמית לרוץ על כל הקודקודים (הסיבה שהצורה היא דינמית היא בגלל שאנחנו לא יודעים איזה קודקודים הם עם יותר שכנים ואיזה עם פחות). הלולאה השנייה כבר יכולה להיות סטטית שכן כמות העבודה היא שווה בכל איטרציה פנימית. כמו כן נשים לב שאין כאן איזה סנכרון פרט לסנכרון הדיפולטי, במקום שנעבוד עם שני תורים עם counter שהופך להיות 0 רק כאשר ה bottom-up לא מוצא שום קודקוד שההורה שלו ״לא קיים״ במערך הparent.

![Pasted image 20240309183913.png](/img/user/Assets/Pasted%20image%2020240309183913.png)

### DFS 
נראה קוד c שממש [[Computer Science/Algorithms/DFS\|DFS]] מקבילי באמצעות OMP:

```C
#include <stdio.h>
#include <stdlib.h>
#include <omp.h>

#define MAX_NODES 100

int graph[MAX_NODES][MAX_NODES];
int num_nodes;

enum colors {NOT_VISITED, VISIT_IS_IN_TASK_QUEUE, ALREADY_VISITED};
enum colors visited[MAX_NODES];

void parallel_dfs_visit(int node) {
    visited[node] = ALREADY_VISITED;
    printf("Node %d is visited by thread %d\n", node, omp_get_thread_num());

    for (int i = 0; i < num_nodes; i++) {
        if (graph[node][i]) {
            #pragma omp critical 
            {
                if (visited[i] == NOT_VISITED) {
                    printf("thread %d: create task for node %d\n", omp_get_thread_num(),i);
                    visited[i] = VISIT_IS_IN_TASK_QUEUE;
                    #pragma omp task
                        parallel_dfs_visit(i);
                }
            }
        }
    }
}

void dfs() {
    for (int i = 0; i < num_nodes; i++) {
        printf("thread %d: examine node %d\n", omp_get_thread_num(), i);
        if (visited[i] == NOT_VISITED) {
            printf("\nstarting DFS visit from %d\n", i);
            
            #pragma omp parallel
            {
                #pragma omp single
                {
                    parallel_dfs_visit(i);
                }
            }
            
        } else if (visited[i] == VISIT_IS_IN_TASK_QUEUE) {
            perror("error: VISIT_IS_IN_TASK_QUEUE shouldn't be possible here\n");
        }
    }
}

```

הקוד מכניס thread אחד לתוך ה dfs_visit והוא כל פעם מוסיף task לפונקציית הvisit המקבילית. ככה בעצם אנחנו מבצעים סריקה עומק אבל כל השכנים של קודקוד כלשהו עושים את זה מקבילי (קצת עיוות של DFS שמשלב בתוכו BFS). צריך לשים לב שגם עם העיוות הזה מזכיר BFS , כלומר היינו יכולים אולי לבנות מנגנון מבוסס priorities שנותן ל task את העדיפות של הרמה של אותו קודקוד בגרף. אבל עדיין זה לא היה BFS מושלם שכן יכלו להיות מצבים שהtask queue יקצה משימות עם עומק נמוך יותר כי יש threads באוויר, לפני שכל הרמה הסתיימה. 

>[!note] task priority
>אפשר להביא ל task ערך של priority בעת היצירה שלו עם task priority(#p) ב pragma block.

>[!warning] 
>המימוש הנ״ל אינו מושלם שכן הוא נועל אתDFS visit עם critical block , יש לכך השפעות של פגיעה במקביליות עליהם נדבר בהמשך
## Sections
בעיית producer/consumer מוגדרת באופן הבא. יש לנו bounded buffer בגודל N תאים. producer מייצר items ושם אותם ב- buffer במקום פנוי. consumer צורך items ומפנה מקום ב- buffer. על consumer לחכות כדי ש- item יכנס ל- buffer לפני שיוכל לצרוך אותו, ועל producer לחכות עד שיהיה מקום פנוי ב- buffer כדי שיוכל לשים שם item חדש.

קוד סדרתי יראה מהצורה שבה בכל איטרציה נוצר item אחד, והוא מיד נצרך.
![Pasted image 20240309012234.png](/img/user/Assets/Pasted%20image%2020240309012234.png)

איך להפוך אותה לתוכנית מקבילית כדי ש- producer ו- consumer יוכלו לעבוד כל אחד בנפרד ובמקביל ? עם הכלים שלמדנו ב OpenMP עד כה, נראה שלא קיים כלי שיעזור לנו שכן אנחנו לא רוצים לחלק משימה לחלקים אלא להגדיר שתי משימות שונות עבור כל thread. זה בעיה שלא נתקלנו בה עד כה כי אנחנו רוצים לחלק __תפקידים__ ולא __איטרציות__.

![Pasted image 20240309125402.png|300](/img/user/Assets/Pasted%20image%2020240309125402.png)

הכלי המדובר נקרא __sections__:
![Pasted image 20240309013048.png](/img/user/Assets/Pasted%20image%2020240309013048.png)
1) שני keyword חדשים - section ו sections. 
2) כל ה sections רצים במקביל בthread נפרד לכל section
3) התוכן של כל section הוא סדרתי אלא אם כן מגדירים אחרת
4) הגדרנו section מקבילי עצמאי עבור producer thread ו- consumer thread. עבור כל תא $A[i]$ יצרנו משתנה $flags[i]$ עבור כל תא במערך A כדי ש- producer יוכל ליידע את consumer שהוא מילא $A[i]$ ותא זה מוכן לצריכה, ו- consumer יוכל ליידע את producer שהוא צרך $A[i]$ ותא זה מוכן למילוי חוזר.

בקוד הנ״ל ישנה בעיה, הפלט עלול להיות שונה בכל ריצה מכיוון ששני threads עלולים לעבוד על __שני מעבדים שונים__ ולכן העדכון של $A[i]$ על ידי producer threads עלול להיות לא נגיש עבור consumer thread וכנ״ל לגבי $FLAGS[i]$ . זה כמובן תלוי במדיניות הכתיבה של הcache אבל זה עדיין יוצר אי דטרמיניזם.

כדי לטפל בבעיה הזאת ניתן להשתמש בפונקציה flush 
![Pasted image 20240309130114.png](/img/user/Assets/Pasted%20image%2020240309130114.png)

הפונקציה הזאת בעצם מכריחה את המידע להתעדכן גם בram ולא רק ב cache. נשים לב שזה לא המצב של cache invalidation שדיברנו עליו קודם, מצב של cache invalidation גורם לטעינה של המידע מהRAM ולא לעדכון המידע בRAM. 

ככה נבטיח את אמינות המידע ב cache, אבל עדיין הפלט עלול להיות שונה בכל הרצה בגלל שיכול להיות מצב שלפני שנגיע ל flush הזכרון עבור flags יתעדכן אבל עבור המערך לא, לכן הconsumer יצא מהbusy wait שלו (לולאת ה while) ויסכום ערך לא עדכני.

__הפתרון הוא להוסיף עוד flush__ גם אחרי עדכון של A וגם אחרי עדכון של flags 

![Pasted image 20240309132141.png](/img/user/Assets/Pasted%20image%2020240309132141.png)

צריך לשים לב שבמילים אחרות הפעולה הזאת אומרת לליבות לא לעבוד עם הcache בכלל אלא רק עם RAM ולכן זאת פעולה __מאוד יקרה__ ולוקחת זמן רב.

### Nested Sections 
נוכל להצהיר על nested section בתוך section קיים. כמו nested for , ה nested section באופן דיפולטי רצים בצורה סדרתית על ידי הthread שמריץ את הsection אב. 

![Pasted image 20240309170140.png](/img/user/Assets/Pasted%20image%2020240309170140.png)

אם נרצה לשנות את ההתנהגות הזאת נשתמש ב omp_set_nested(1) מחוץ ל pragma block כדי לאפשר מקביליות מקוננת.

## Tasks
### Linked List 
נרצה להראות כיצד להשתמש ב OpenMP כדי למקבל את הקוד הבא

![Pasted image 20240309170403.png|300](/img/user/Assets/Pasted%20image%2020240309170403.png)

ההצעה האינטואיטיבית כדי למקבל לולאת while , דבר שלא מתאפשר ב default היא להמיר את הרשימה המקושרת למערך ואז על זה לרוץ בצורה מקבילית.

![Pasted image 20240309170532.png](/img/user/Assets/Pasted%20image%2020240309170532.png)
יש פה overhead של העתקה. להשתמש ב sections הוא גם בעייתי כי אנחנו לא יכולים לדעת מהו מספר הsections שנרצה לייצר. נרצה כלי שמתאים למקבול של מצב כזה שבו לא יודע כמה tasks נרצה לייצר, הכלי הזה נקרא Tasks.

נסתכל על הסינתקס:
![Pasted image 20240309170726.png|300](/img/user/Assets/Pasted%20image%2020240309170726.png)
ההבדל המהותי בין tasks ל sections הוא שכאן יש task queue שמנהל את המשימות. בתוך הקוד המקבילי עטפנו את התוכנית בקוד single כדי שרק thread אחד ייצר את הtasks ויכניס אותם ל task pool. 

כאן יש פה מימוש דומה מאוד ל[[Computer Science/Programming Concepts/Concurrency Programming/Thread Pool\|Thread Pool]] , ה threads שולפים משימות מהtask queue כל עוד הם יכולים.

![Pasted image 20240309171008.png](/img/user/Assets/Pasted%20image%2020240309171008.png)
עבור התוכנית הנ״ל הפלט יהיה

![Pasted image 20240309171020.png](/img/user/Assets/Pasted%20image%2020240309171020.png)

בהרצה הזו רואים ש- thread2 הספיק לבצע 3 tasks, וכל שאר ה- threads רק task אחד. זה תלוי בתזמון של threads ובמרוכבות של tasks אם הם שונים.

ניתן גם להגדיר explicit barriers ל tasks כלומר, ניתן לחכות למספר tasks שיסתיימו לפני שמכניסים tasks חדשים:

![Pasted image 20240309171244.png|300](/img/user/Assets/Pasted%20image%2020240309171244.png)


אם נרצה לממש את זה על הlinked list שלנו נרצה להבין מה המגבלות שלא ניתן להמנע מהם ומה אנחנו יכולים לעשות.

1) חייבים לרוץ על כל הרשימה באופן מסונכרן כלומר thread 1 או עם כלי סנכרון.
2) צריך להשתמש בלולאת while כי גודל הרשימה לא ידוע.
3) כל process יכול להיות מקבילי.

אם כן המודל ייראה מהצורה

![Pasted image 20240309171705.png|300](/img/user/Assets/Pasted%20image%2020240309171705.png)

הקוד מאוד פשוט ויראה כך:
![Pasted image 20240309171223.png|300](/img/user/Assets/Pasted%20image%2020240309171223.png)

במילים- thread אחד מכניס את הtasks , כל task הוא הפעולה process על איבר כלשהו ברשימה. שאר ה threads שולפים את הtasks ומבצעים אותם מקבילי.

### Fibonacci
[[Computer Science/Algorithms/Fibonacci algorithm\|פיבונאצ׳י הפרד ומשול]] הוא אלגוריתם שפותר את פיב׳ בזמן $O(\log n)$.
אנחנו מכירים גם אלגוריתם פשוט יותר שפותר בצורה ריקורסיבית בזמן מאוד איטי

![Pasted image 20240309172333.png|300](/img/user/Assets/Pasted%20image%2020240309172333.png)

אם נרצה למקבל את זה נוכל לחשוב על להכניס כל קריאה רקורסיבית ל task. 

![Pasted image 20240309172420.png|300](/img/user/Assets/Pasted%20image%2020240309172420.png)

התוכנית הזאת מאוד לא יעילה שכן אנחנו משקיעים המון זמן ומקום ביצירת $2^{n}$ tasks ב queue. 

![Pasted image 20240309172525.png](/img/user/Assets/Pasted%20image%2020240309172525.png)

נוכל להגדיר קוד שמבקר את כמות הtasks ומונע מהtask queue להכיל כמות גדולה יותר של משימות מ capacity שנגדיר מראש

![Pasted image 20240309172814.png|300](/img/user/Assets/Pasted%20image%2020240309172814.png)

![Pasted image 20240309172825.png](/img/user/Assets/Pasted%20image%2020240309172825.png)

הקוד הזה בעצם מגדיר שאחרי עומק עץ רקורסיה של 5 הקוד יעבור לחשב את שאר העומקים בצורה סדרתית, כל עומק עץ נמוך מזה יעשה בצורה מקבילית.
## Synchronization 
ככלל אצבע נרצה להמנע מ[[Synchronization\|סנכרון]] כדי להעלות רמת המקבילות. 

![Pasted image 20240310162652.png](/img/user/Assets/Pasted%20image%2020240310162652.png)

__יש שני סוגי סנכרון ב openMP:__

__High level synchronization__ 
1) critical 
2) barriers

__Advanced synchronization operations__
1) ordered
2) locks
3) atomic
4) flush

### Ordered 
הקוד שכתוב בתוך ordered block מבטיח שגם אם הריצה תהיה מקבילית הסדר שבו הקוד יבוצע יהיה לפי הסדר שבוא הקוד מופיע ב parallel region למשל בדוגמה עם לולאה 

```C

do_some_work(i) {
     printf("work% in progress…\n", i)
     return i
}

main() {
   #pragma omp parallel
   {
           #pragma omp for ordered
           for i = 0 to 4
               result = do_some_work(i)
               #pragma omp ordered
                     printf("work%d: %d\n", result)
   }
}
```

הפלט יהיה
![Pasted image 20240310163128.png|200](/img/user/Assets/Pasted%20image%2020240310163128.png)

ניתן לראות שכל חישוב נעשה בצורה מקבילית על ידי הthreads אבל החלק של ה ordered נעשה לפי סידור סדרתי של האיטרציות.

דוגמה נוספת יכולה להיות על sections 

```C
#pragma omp parallel
{
    #pragma omp sections
    {
        #pragma omp section
        {
            #pragma omp ordered
            printf("Thread %d is executing section 1.\n", omp_get_thread_num());
        }
        #pragma omp section
        {
            #pragma omp ordered
            printf("Thread %d is executing section 2.\n", omp_get_thread_num());
        }
        #pragma omp section
        {
            #pragma omp ordered
            printf("Thread %d is executing section 3.\n", omp_get_thread_num());
        }
    }
}

```

כאן הsections רצים במקביל אבל ההדפסות יהיו לפי סדר הופעתן בתוך ה pragma block.

![Pasted image 20240310163445.png](/img/user/Assets/Pasted%20image%2020240310163445.png)

### Critical 
דיברנו על critical כבלוק המאפשר רק ל thread אחד לגשת לקטע קוד קריטי. עם זאת, יש כמה דגשים שלא דיברנו עליהם, למשל, אם אני לא נותן שם לcritical sections שלי בתוך הבלוק אז __יש רק מנעול אחד שאחראי עליהם__ כלומר אם נעלתי שתי אזורי קוד קריטים עם הקוד pragma omp critical אז רק thread אחד יוכל לגשת בכל פעם לאחד מקטע הבלוק הללו.

![Pasted image 20240310163820.png|250](/img/user/Assets/Pasted%20image%2020240310163820.png)

אפשר לתקן זאת על ידי מתן שמות ל critical sections כדי לתת לכל אחד מנעול שונה 

![Pasted image 20240310163901.png|250](/img/user/Assets/Pasted%20image%2020240310163901.png)

נשים לב ש Critical הוא מנגנון נעילה חזק מדי בעיקר מכיוון שהוא compiler directive ולכן הרבה פעמים אין לנו את היכולת לבצע נעילות דינמיות בהתאם למצב התוכנית. היכולת היחידה שלי היא לתת שמות hardcoded למנעולים מה שעלול ליצור מצבים של נעילה שלא לצורך.

![Pasted image 20240310164126.png|250](/img/user/Assets/Pasted%20image%2020240310164126.png)

למשל אם נסתכל על קטע הקוד הזה שנועל dfs_visit היא שקטע הקוד הקריטי נועל את כל הthreads בין אם סורקים ״עומקים״ שונים לגמרי בעץ. אין כאן הבחנה בין האם יש צורך להצהרת קטע קוד קריטי כי אנחנו נגשים לאותו קודקוד או לא. נרצה לבנות מנגנון שנותן מנעול לכל קודקוד.

### LOCKS
ישנם כמה פונקציות שניתן להשתמש בהם בעבודה עם מנעולים ב OpenMP

![Pasted image 20240310164907.png](/img/user/Assets/Pasted%20image%2020240310164907.png)

נבחן איך ניתן לבצע שינוי לקוד ה DFS עם מנעולים ולקבל רמת מקביליות גבוהה יותר על ידי נעילה רק בעת הצורך 

```C

global variables: omp_lock_t locks[|V|]
DFS(G=(V,E)) {
   #pragma omp parallel for
   for 𝑣∈ V
        visited[v] ← WHITE
        omp_init_lock(&locks[v])

   for 𝑣∈ V
        if visited[v] = WHITE
           #pragma omp parallel
               #pragma omp single
                    dfs_visit (v)
   for 𝑣∈ V
       omp_destroy_lock(&locks[v])
}

dfs_visit (v) {
   visited[v] ← BLACK

   for 𝑢∈ neighbors[v]
        omp_set_lock(&locks[u])
        if visited[u] = WHITE
            visited[u] ← GRAY
            #pragma omp task
                  dfs_visit(u)
            omp_unset_lock(&locks[u])
}

```

נשים לב שאנחנו נועלים רק את המנעול של קודקוד ספציפי שנכנס לתנאי ה if visited(u) , באופן הזה, אנחנו מונעים משני threads להכנס לקוד הרקורסיה כי בהכרח הראשון שייכנס ישנה את ה flag מ WHITE ל GRAY. תוך כדי אנחנו מאפשרים לשני threads שסורקים קודקודים שונים, להמשיך בצורה מקבילית.

### Atomic 
דרך נוספת להשיג את אותה תוצאה רצויה היא באמצעות Atomic instruction. נזכיר ש פעולות אטומיות הן פתרון חומרתי לבעית ה mutual exclusion שמבטיחה שרק thread אחד יכול לבצע את הפעולה האטומית בכל רגע נתון ובכך להמנע מ race conditions.

ההבדל המהותי בין פעולה אטומית לבין מנעולים היא שבאמצעות שימוש בפעולות אטומיות , אין overhead של הקפאת הפעולה של threads ושמירת המצב שלהם עד לרגע שהם יכולים לרוץ. 
ה mutual exclusion ב atomic הוא ברמת החומרה, לעומת Locks אבל הוא גם מחזיק את ה threads ב busy wait (מצב שבו ה thread מבזבז זמן מעבד על המתנה).

![Pasted image 20240310202839.png](/img/user/Assets/Pasted%20image%2020240310202839.png)

האטומיות הזו מסופקת על ידי חומרה, אשר משביתה את ההפרעות במעבד המתאים ומשביתה את הגישה למעבד -m (על ידי נעילת data bus), עד לביצוע הוראה אטומית במלואה.

![Pasted image 20240310203017.png](/img/user/Assets/Pasted%20image%2020240310203017.png)
מה שאטומי בתוך הבלוק atomic הוא הפעולה + בקוד הנ״ל. הפונקציה do some work היא אינה אטומית ומספר threads רצים במקביל.

![Pasted image 20240310203222.png](/img/user/Assets/Pasted%20image%2020240310203222.png)
ניתן לראות שכל ה- threads עובדים במקביל, ורק הגישה ל- x הינה מסונכרנת.

הסינתקס הוא מהצורה

$$\text{pragma omp atomic [read | write | update | capture]}$$

* ההתנהגות הדיפולטית היא update. ובמצב זה האטומיות מגן על עדכון למיקום בזכרון כלומר הופך את הפעולות הבאות לאטומיות x++ / ++x / x-- / --x / x = x op exp.
	![Pasted image 20240310204126.png](/img/user/Assets/Pasted%20image%2020240310204126.png)
* במצב של read הפעולה האטומיות שתבחר תהיה מתאימה לread, וזה מבצע את הפעולה v=x לאטומית. האטומיות היא על x כלומר קריאת הערך של x היא אטומית, ההשמה למשתנה v אינה אטומית.
![Pasted image 20240310204235.png|250](/img/user/Assets/Pasted%20image%2020240310204235.png)
* write הופך את פעולת השמה לאטומית x=expr
![Pasted image 20240310204302.png|250](/img/user/Assets/Pasted%20image%2020240310204302.png)
* compare - פעולת compare and swap אטומיים.
 ![Pasted image 20240310204325.png|250](/img/user/Assets/Pasted%20image%2020240310204325.png)
* capture - מאפשר לתפוס old values של משתנה לפני שמבצעים השמה עליו לערך חדש (מבטיח שלא יהיה מצב שנדרוס את העותק של המשתנה שמייצרים)

![Pasted image 20240310204407.png|250](/img/user/Assets/Pasted%20image%2020240310204407.png)

* compare capture - 
![Pasted image 20240310230931.png|250](/img/user/Assets/Pasted%20image%2020240310230931.png)

>[!info] למה קריאה אטומית זה חשוב?
>על פניו נראה שהקריאה האטומית היא מיותרת שכן מה הבעיה שמספר threads יקראו מאותו ערך ויבצעו השמה שלו למשתנה? הבעיה מתחילה כאשר תחילת הערך של המשתנה נמצא בכתובת שהיא לא [[Memory Alignment\|align]]. במצב זה עלול לקרות מצב שבו אנחנו מבצעים מספר גישות לRAM רק כדי להביא ערך של משתנה ובמצב כזה אם נקרא את הערך עלול להיות מצב שתוך כדי קריאה של בית אחד מהמשתנה הבית השני ישתנה תוך כדי אם לא נגדיר את הפעולה כאטומית (או נעבוד עםaligned addresses).
>לדוגמה - שקריאות מזכרון מתבצעות ב- chunks של 64 bits, החל מכתובת שמתחלקת ב- 8 . אם משתנה short x נמצא על התפר, נצטרך שתי גישות לזכרון כדי להביא את שני ה- bytes של x. בין קריאה ראשונה לקריאה שנייה עלול להכנס thread אחר שעלול להרוס את הקריאה של x.
>![Pasted image 20240310231106.png|300](/img/user/Assets/Pasted%20image%2020240310231106.png)
>![Pasted image 20240310231205.png|300](/img/user/Assets/Pasted%20image%2020240310231205.png)
>דוגמא לתזמון לא מוצלח של שני threads, כאשר thread אחד מנסה לקראו את הערך של x, ו- thread שני מנסה לכתוב ערך חדש ל- x. בסוף הקריאה v מקבל junk value.


נראה שימוש בפעולות אטומיות למימוש dfs 
![Pasted image 20240310232518.png](/img/user/Assets/Pasted%20image%2020240310232518.png)
כאן השתמשנו ב compare and capture פעם אחת כל פעם שנכנסים ל dfs visit. מצב זה גורם לכך שכל קודקוד בצורה אטומית שומר את הold value של קודקוד שהוא מבקר בוא וטוען בתוכו ערך חדש. אם הערך הישן היה white אז מבקרים בו. בגלל שפעולה הcapture היא אטומית, אז אין חשש שהתנאי יתקיים עבור שני threads שמבקרים באותו קודקוד.
### Sorted insertion to a linkes list
נרצה מבנה נתונים של רשימה מקושרת שתומכת במיקבול. 
נרצה במקום פונקציית הinsert הרגילה לתמוך בsorted_insert שמכניס איברים בצורה ממויינת לרשימה. 

בקוד סדרתי זה יראה כך 

```C
void sorted_insert(node_t *head, int val) {
    /* Find the right place to insert */
    node_t *ptr = head->next;
    node_t *prev = head;

    while (ptr != NULL && ptr->value < val) {
        prev = ptr;
        ptr = ptr->next;
    }

    /* Insert the new node */
    node_t *new_node = malloc(sizeof(node_t));
    new_node->value = val;
    new_node->next = ptr;
    prev->next = new_node;
}
```

אנחנו נקדם את המצביע כל עוד הערך הנוכחי של המצביע לאיבר ברשימה מקושרת קטן מהערך שנרצה להכניס. ברגע שהגענו נכניס את ה node החדש.

ננסה לכתוב את הקוד הזה כדי לתמוך במקביליות __כמה שיותר מהירה__ באמצעות כלי הסנכרון השונים.

### V1 
המימוש הכי נאיבי יהיה להשתמש במנעול אחד בתוך הפונקציה של sorted_insert. הרבה פעמים בפתרון של אלגוריתמים מקביליים נרצה להתחיל מלהבין בכלל מהו הפתרון הנאיבי ביותר ולפתח אותו.

```C
void sorted_insert(node_t *head, int val) {
    /* Find the right place to insert */
    omp_set_lock(&lock);
    node_t *ptr = head->next;
    node_t *prev = head;

    while (ptr != NULL && ptr->value < val) {
        prev = ptr;
        ptr = ptr->next;
    }

    /* Insert the new node */
    node_t *new_node = malloc(sizeof(node_t));
    new_node->value = val;
    new_node->next = ptr;
    prev->next = new_node;
    omp_unset_lock(&lock);
}
```

המימוש הזה הופך את הבלוק הזה לסדרתי ולכן רמת המקביליות זהה לסדרתי ובתוספת ה overhead של יצירת threads , הקוד שלנו יהיה גרוע בהרבה מהקוד הסדרתי.

### V2
הגרסה השנייה שלנו תציע רעיון עם מקביליות יותר גבוהה.
אנחנו נרצה שכל thread שקורא לפונקציה sorted_insert מאפשר ל threads שיבוא אחריו לעבוד רק על רישא של הרשימה. מעין מנגנון pipeline כך שבמקום שכל thread יצטרך למצוא היכן למקם את הערך שלו ברשימה כדי לאפשר ל thread הבא לרוץ, הוא חוסם לthreads אחרים רק תת רשימה ימנית ומאפשר להם לעבוד על השמאלית עד שהוא מסיים.

```C
void sorted_insert(node_t *head, int val) {
    /* Signal start of search */
    omp_set_lock(head->lock);

    /* Find the right place to insert */
    node_t *ptr = head->next;
    node_t *prev = head;

    while (ptr != NULL && ptr->value < val) {
        omp_set_lock(ptr->lock);
        omp_unset_lock(prev->lock);
        prev = ptr;
        ptr = ptr->next;
    }

    /* Insert the new node */
    node_t *new_node = malloc(sizeof(node_t));
    new_node->value = val;
    new_node->next = ptr;
    new_node->lock = malloc(sizeof(omp_lock_t));
    omp_init_lock(new_node->lock);
    prev->next = new_node;
    omp_unset_lock(prev->lock);
}
```

כעת לכל איבר ברשימה יש מנעול משלו. כל thread שנכנס נועל את הptr שעליו הוא כרגע נמצא ברשימה ומשחרר את הקודם וככה הוא ממשיך.  

### V3 
הגרסה השלישית של תמיכה במקביליות של sorted_insert תעבוד באופן הבא-

נאפשר לthreads למצוא את מיקום הptr שאחריו אמור להכנס הקודקוד החדש בצורה מקבילית בלי מנגנון סנכרון.
ברגע שthread כלשהו מצא את הptr הזה, הוא מנסה לנעול את המצביע prev , זה שמצביע לptr. 

הסיבה שהוא מנסה לנעול אותו היא בגלל שיכול להיות שתוך כדי החיפוש של הptr קרה מצב ש threads אחרים הוסיפו ערכים בין prev ל ptr וכעת המצב כבר לא עדכני. 

את זה אפשר לעשות על ידי השוואה בין הptr לבין prev.next . אם thread אחר הוסיף איבר בין prev ל ptr ההשוואה הזאת תגלה לנו את זה. במצב זה אנחנו מבצעים בידיוק את אותו הקוד בגרסה v2 רק שהפעם זה כנראה יהיה על תת רשימה מצומצמת בהרבה. 

לבסוף מבצעים הכנסה של הערך החדש לרשימה.

```C
void sorted_insert(node_t *head, int val) {
    /* Find the right place to insert */
    node_t *ptr = head->next;
    node_t *prev = head;

    while (ptr != NULL && ptr->value < val) {
        prev = ptr;
        ptr = ptr->next;
    }

    /* Lock the elment that may be modified */
    omp_set_lock(prev->lock);

    /* Confirm that the element is still in the right place */
    if (prev->next != ptr) {
        ptr = prev->next;
        while (ptr != NULL && ptr->value < val) {
            omp_set_lock(ptr->lock);
            omp_unset_lock(prev->lock);
            prev = ptr;
            ptr = ptr->next;
        }
    }

    /* Insert the new node */
    node_t *new_node = malloc(sizeof(node_t));
    new_node->value = val;
    new_node->next = ptr;
    new_node->lock = malloc(sizeof(omp_lock_t));
    omp_init_lock(new_node->lock);
    prev->next = new_node;
    omp_unset_lock(prev->lock);
}
```

