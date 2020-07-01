---
layout: post
title: Как стать Хакером 
category: [translations]
tag: [translation]
---

Оригинал: [How To Become Hacker](http://www.catb.org/~esr/faqs/hacker-howto.html)

![](/assets/images/ru/how_to_become_hacker/hacker.png)

# Для чего этот документ?

Как редактор [Jargon File](http://www.catb.org/jargon/) и автор нескольких других известных документов аналогичного характера, я часто получаю электронные письма от новичков в восторженных сетях, спрашивающих, "как я могу научиться быть волшебником-хакером?". Еще в 1996 году я заметил, что, похоже, не было никаких других часто задаваемых вопросов или веб-документов, посвященных этому жизненно важному вопросу, поэтому я начал этот. Многие хакеры теперь считают это фундаментальным трудом, и я полагаю, что это так. Тем не менее, я не претендую на то, чтобы быть исключительным авторитетом в этой теме и если вам не нравится то, что вы читаете здесь, напишите свою.

Примечание. В конце этого документа приведен список [FAQ](http://www.catb.org/~esr/faqs/hacker-howto.html#FAQ). Пожалуйста, прочитайте их дважды, прежде чем отправлять мне вопросы по поводу этого документа.

Доступны многочисленные переводы этого документа: [болагрский](http://weknowyourdreams.com/questions.html), [китайский](http://www.0x08.org/docs/hacker-howto.html#hacker-howto), [чешский](http://scientificachievements.com/jak-se-stat-hackerem/), [датский](https://tkkrlab.nl/wiki/Hoe_word_ik_een_hacker), [эстонский](http://www.kakupesa.net/hacker/) , [французский](https://thomasgil.com/hacker.html), [немецкий](http://www.linuxtaskforce.de/hacker-howto-ger.html), [греческий](https://sophron.latthi.com/hacker-howto-gr.html), [иврит](https://he.wikisource.org/wiki/%D7%90%D7%99%D7%9A_%D7%9C%D7%94%D7%99%D7%95%D7%AA_%D7%94%D7%90%D7%A7%D7%A8), [японский](https://cruel.org/freeware/hacker.html), [литовский](https://rtfb.lt/hacker-howto-lt.html), [польский](https://michalp.net/blog/posts/hacker-howto/), [португальский (бразильский)](http://jvdm.sdf.org/pt/raquer-howto/) и [турецкий](http://www.belgeler.org/howto/hacker-howto/hacker-howto.html). Обратите внимание, что, поскольку этот документ иногда изменяется, ссылки могут быть устаревшими (устаревшие удалены, только актуальные на момент перевода - прим. пер.)

Диаграмма «пять точек в девяти квадратах», которая украшает этот документ, называется _glider_. Это простой паттерн с некоторыми удивительными свойствами в математическом симуляторе под названием _Жизнь_, который очаровывал хакеров на протяжении многих лет. Я думаю, что это хорошая визуальная эмблема для того, на что похожи хакеры - абстрактные, поначалу немного загадочные, но являющиеся проводниками в целый мир со своей сложной логикой. Узнайте больше о эмблеме _glider_ [здесь](http://www.catb.org/~esr/hacker-emblem/).

Если вы находите этот документ ценным, пожалуйста, поддержите меня на [Patreon](https://www.patreon.com/esr) или [SubscribeStar](https://www.subscribestar.com/esr). И подумайте также о поддержке других хакеров, которые создали код, который вы используете и поддержите их через [Loadsharers](http://www.catb.org/esr/loadsharers/). Множество небольших, но постоянных пожертвований очень помогают добрым людям, которые подарили вам свой труд, чтобы привнести ценность в сообщество.

# Что такое хакер?

Файл [Jargon](http://www.catb.org/jargon/) содержит кучу определений термина «хакер», большинство из которых связано с технической компетентностью и удовольствием в решении проблем и преодолении ограничений. Если вы хотите знать, как стать хакером, только два из них действительно актуальны.

Существует сообщество, общая культура, опытных программистов и сетевых мастеров, которая ведет свою историю с ранних лет к первым time-sharing миникомпьютерам, и к самым первым экспериментам ARPAnet. Члены этой культуры создали термин «хакер». Хакеры построили интернет. Хакеры сделали операционную систему Unix такой, какая она есть сегодня. Хакеры заставляют Всемирную паутину работать. Если вы являетесь частью этой культуры, если вы внесли свой вклад в нее, и другие люди в ней знают, кто вы и называют вас хакером - _вы хакер_.

Хакерское мышление не ограничивается культурой хакерского программного обеспечения. Есть люди, которые применяют хакерское отношение к другим вещам, таким как электроника или музыка - на самом деле, вы можете обнаружить данный феномен в любой науке или искусстве. Software хакеры распознают родственные души и ​​могут также называть таких людей «хакерами» - и некоторые утверждают, что природа хакера не зависит от конкретной среды, в которой работает хакер. Но в остальной части этого документа мы сосредоточимся на навыках и отношении software хакеров, а также традиции общей культуры, которая породила термин «хакер».

Есть другая группа людей, которые громко называют себя хакерами, но это не так. Это люди (в основном подростки), которые получают удовольствие от взлома компьютеров и взлома телефонных систем. Настоящие хакеры называют этих людей «взломщиками» и не хотят иметь с ними ничего общего. Настоящие хакеры считают, что взломщики ленивы, безответственны и не очень умны, а способности взломщика, не делают вас хакером так же, как способность управлять автомобилями не делает вас инженером автомобилестроения. К сожалению, многие журналисты и писатели были введены в заблуждения и использовали слово «хакер» для описания взломщиков. Это постоянно раздражает настоящих хакеров.

Основное отличие заключается в следующем: хакеры строят вещи, взломщики ломают их.

Если вы хотите быть хакером, продолжайте читать. Если вы хотите быть взломщиком, то прочитайте новостную группу alt.26400 (ссылки больше нет - прим.пер.) и приготовьтесь принять факт, что вы не так умны, как думаете. И это все, что я собираюсь сказать о взломщиках (crackers - прим.пер.).

# Отношение Хакера

Хакеры решают проблемы и созидают вещи, и они верят в свободу и добровольную взаимопомощь. Чтобы быть принятым в качестве хакера, вы должны вести себя так, как если бы у вас было такое отношение. И чтобы вести себя так, как будто у вас есть хакерское отношение к вещам, вы должны действительно верить в это.

Но если вы думаете, что взращивать хакерские взгляды это способ получить признание в культуре, то вы упускаете суть. Становление таким человеком, который считает, что эти вещи важны для вас - помогают вам учиться и сохранять мотивацию. Как и во всех творческих искусствах, самый эффективный способ стать мастером - это имитировать образ мыслей мастера - не только интеллектуально, но и эмоционально.

Как гласит следующее современное дзенское стихотворение:

     Чтобы следовать по пути:
     посмотри на мастера,
     следуй за мастером,
     ходи с мастером,
     смотри сквозь мастера,
     стань мастером.

Итак, если вы хотите быть хакером, повторяйте следующие вещи, пока не поверите им:

# 1. Мир полон захватывающих проблем, ожидающих своего решения.

Быть хакером - это весело, но это веселье, требующее больших усилий. Усилие требует мотивации. Успешные атлеты получают мотивацию, заставляя свои тела выходить за пределы своих физических ограничений. Точно так же, чтобы быть хакером, вы должны получать наслаждение от решения проблем, оттачивания своих навыков и тренировки своего интеллекта.

Если вы не такой человек, то вам нужно стать им, чтобы стать хакером. В противном случае вы обнаружите, что ваша хакерская энергия истощается такими вещами, как секс, деньги и социальное одобрение.

(Вы также должны развивать своего рода веру в свои собственные способности к обучению - убеждение, что даже если вы не знаете всего, что вам нужно для решения проблемы, если вы решите только ее часть и извлечете уроки из этого, вы сможете выучить достаточно, чтобы решить следующую часть - и так далее, пока вы не закончите.)

# 2. Ни одна проблема не должна решаться дважды.

Творческий мозг - ценный, ограниченный ресурс. Он не должен быть потрачен на изобретение колеса, когда есть так много захватывающих новых проблем.

Чтобы вести себя как хакер, вы должны понимать, что время других хакеров драгоценно - настолько, что для вас должно быть моральным долгом делиться информацией, решать проблемы, а затем делиться решением так, чтобы другие хакеры могли решать новые проблемы вместо того, чтобы постоянно пересматривать старые.

Обратите внимание, однако, что «**Ни одна проблема не должна решаться дважды**». Это не означает, что вы должны считать все существующие решения правильными или что есть только одно правильное решение для любой данной проблемы. Часто мы узнаем много нового о проблеме, о которой мы не знали раньше, изучая первый вариант решения. Это нормально, и часто необходимо, чтобы решить, что мы можем сделать лучше. Проблемой тут являются искусственные технические, юридические или институциональные барьеры (например, проекты с закрытым исходным кодом), которые препятствуют повторному использованию и вынуждают людей заново изобретать колеса.

(Вы не обязаны отдавать весь свой продукт в open source, хотя хакеры, которые делают это, получают наибольшее уважение от других хакеров. Это согласуется с ценностями хакера, о монетизации софта и получением с этого денег, для покупки компьютера, еды и аренды. Является нормальным использовать свои навыки хакерства, чтобы содержать семью или даже разбогатеть, если вы не забываете о своей преданности искусству и другим хакерам.)

# 3. Скука и тяжелая работа - зло.

Хакерам (и творческим людям в целом) никогда не следует скучать или заниматься глупой повторяющейся работой, потому что, когда это происходит, это означает, что они не делают то, для чего нужны - решать новые проблемы. Эта расточительность вредит всем. Поэтому скука и рутина не просто неприятны, но на самом деле являются злом.

Чтобы вести себя как хакер, вы должны верить, что автоматизируете рутину не только для себя, но и для всех остальных (особенно других хакеров).

(Есть одно очевидное исключение из этого. Хакеры иногда делают вещи, которые могут показаться повторяющимися или скучными для наблюдателя, они это делают, в качестве упражнения для очищения ума, или для того, чтобы приобрести навык или получить какой-то особый опыт, который вы не можете получить иначе. Но здесь есть грань - хакер никогда не должен оказаться в ситуации, которая его утомляет.)

# 4. Свобода это хорошо.

Хакеры - не авторитарны. Любой, кто отдает вам приказы, мешает решать проблему, которой вы увлекаетесь - и, учитывая то, как работают авторитарные умы, обычно найдет глупую причину для этого. Поэтому с авторитарным отношением нужно бороться, где бы вы его ни находили, чтобы оно не задушило вас и других хакеров.

(Это не то же самое, что бороться со всеми существующими авторитетами. Детей нужно направлять, а преступников сдерживать. Хакер может согласиться принять некоторые виды полномочий, чтобы получить что-то, что он хочет больше, чем время, которое он тратит на выполнение приказов. Но эта ограниченная, осознанная сделка - вид личной капитуляции)

Авторитарии процветают, с помощью цензуры и секретности. Они не доверяют добровольному сотрудничеству и обмену информацией - им нравится только «сотрудничество», которое они контролируют. Таким образом, чтобы вести себя как хакер, вы должны развить инстинктивную враждебность к цензуре, секретности и применению силы или обмана. И вы должны быть готовы действовать в соответствии с этим убеждением.

# 5. Отношение не заменит компетентность.

Чтобы быть хакером, вы должны выработать некоторые из этих подходов. Но одно только изменение отношения не сделает вас хакером, не сделает вас чемпионом или рок-звездой. Чтобы стать хакером, потребуется ум, практика, преданность делу и тяжелая работа.

Поэтому вы должны научиться только доверять компетенции конкретного человека. Хакеры не позволят позерам тратить свое время, но они поклоняются компетентности - особенно компетентности в хакинге. Но компетентность во всем также ценится. Особенно хороша компетентность в нужных навыках, которыми могут овладеть немногие, а лучше - в тех навыках, которые включают умственную остроту, мастерство и концентрацию.

Если вы почитаете компетентность, вам понравится развивать ее в себе. Эта тяжелая работа и самоотдача станут своего рода интересной игрой, а не рутиной. Такое отношение жизненно важно для того, чтобы стать хакером.

# Основные навыки хакинга

Хакерское отношение к вещам жизненно важно, но навыки еще более важны. Хакерское отношение не может заменить компетентность, и есть определенный базовый набор навыков, которые вам необходимо иметь, прежде чем любой хакер захочет назвать вас таковым.

Этот набор инструментов медленно меняется с течением времени, так как технологии создают новые навыки и уничтожают старые. Например, раньше список включал программирование на машинном языке и до недавнего времени не включал HTML. Но на данный момент, включается следующее:

## 1. Научитесь программировать.

Это, конечно, фундаментальный хакерский навык. Если вы не знаете языков программирования, я рекомендую начать с Python. Он тщательно спроектирован, хорошо документирован и относительно добр к новичкам. Несмотря на то, что это хороший первый язык, для обучения, это не просто игрушка. Это очень мощный и гибкий инструмент, который хорошо подходит для крупных проектов. Я написал более подробный [гайд к Python](https://www.linuxjournal.com/article/3882). Хорошие учебники доступны на веб-сайте [Python](https://docs.python.org/3/tutorial/). Существуют также хорошие сторонние ресурсы по [Python](cscircles.cemc.uwaterloo.ca).

Раньше я рекомендовал Java как хороший язык для раннего изучения, но [данная критика](www.crosstalkonline.org/storage/issue-archives/2008/200801/200801-Dewar.pdf) изменила мое мнение (можете глянуть в “The Pitfalls of Java as a First Programming Language”). Хакер не может решать проблемы, как c какой-нибудь покупной сантехникой из магазина, он должен понимать за что отвечает каждый компонент в системе. На данный момент, я думаю, что лучше всего сначала изучить C и Lisp, а затем Java.

Данный вопрос, является более широким. Если язык делает многое за вас, он может быть одновременно хорошим инструментом для решения проблем в больших компаниях и плохим инструментом для обучения. С такой проблемой сталкиваются не только вышеупомянутые языки, но и фреймворки веб-приложений, такие как RubyOnRails, CakePHP, Django - они могут казаться простыми в решении простых задач, но обманут вас, когда вам придется решать сложную задачу, или даже упростить задачу.

Если вы серьезно занимаетесь программированием, вам придется изучать C, основной язык Unix. C++ очень тесно связан с C. Если вы знаете один из них, то изучение другого не составит труда. Однако ни один из этих языков не подходит для изучения в качестве первого. И, на самом деле, чем больше вы можете избежать программирования на C, тем более продуктивным вы будете.

_C_ очень эффективен и экономит ресурсы вашей машины. К сожалению, _C_ становится эффективным, и в тоже время требует от вас низкоуровневого, ручного управления ресурсами (например, памятью). Весь этот низкоуровневый код сложен и подвержен ошибкам, и он затрачивает огромное количество вашего времени на отладку. Современные компьютеры настолько мощные, насколько это возможно, что обычно является плохим компромиссом - разумнее использовать язык, который использует время компьютера менее эффективно, но ваше время гораздо эффективнее. Поэтому, выбором является Python.

Другие языки, имеющие особое значение для хакеров, включают в себя Perl и LISP. Perl стоит изучать по практическим причинам. Он очень широко используется для веб-страниц и системного администрирования, поэтому даже если вы не пишете на Perl, вам следует научиться его читать. Многие люди используют Perl, как Python, чтобы избежать программирования на C в тех местах, где не требуется такая эффективность. В общем, вы должны уметь понимать код на данных языках.

LISP стоит изучать по другой причине - глубокому просветлению, которое вы получите, когда наконец изучите его. Этот опыт сделает вас лучшим программистом до конца ваших дней, даже если вы на самом деле никогда не используете сам LISP. (Вы можете довольно легко начать работать с LISP, написав и изменив некоторые компоненты Emacs или плагины Script-Fu для GIMP.)

На самом деле, лучше всего изучить все пять языков: Python, C/C ++, Java, Perl и LISP. Помимо того, что они являются наиболее важными хакерскими языками, они представляют очень разные подходы к программированию, и каждый из них научит вас полезным вещам.

Но знайте, что вы не достигнете уровня навыков хакера или даже просто программиста, просто изучая языки программирования - вам нужно научиться думать о проблемах программирования в общем, независимо от какого-либо языка. Чтобы быть настоящим хакером, вам нужно прийти к тому моменту, когда вы сможете выучить новый язык за несколько дней, связав то, что в документации, с тем, что вы уже знаете. Это означает, что вы должны выучить несколько очень разных языков.

Я не могу дать полную инструкцию о том, как научиться программировать здесь - это сложный навык. Но я могу вам сказать, что книги и курсы этого не сделают - многие, может быть, большинство из лучших хакеров самоучки. Вы можете изучать языковые особенности - кусочки знаний - из книг, но мышление, которое превращает эти знания в жизненный навык, может быть изучено только практикой и обучением у учителя. Что необходимо проделать - чтение кода и написание кода.

Peter Norvig, один из ведущих хакеров Google и соавтор самого широко используемого учебника по искусственному интеллекту, написал превосходное эссе под названием [ Teach Yourself Programming in Ten Years](http://norvig.com/21-days.html). Его «рецепт успеха программирования» заслуживает пристального внимания.

Обучение программированию похоже на обучение написанию хорошего естественного языка. Лучший способ сделать это - почитать книги, написанные мастерами слога, написать некоторые вещи самостоятельно, больше читать, немного писать, прочитать еще немного, написать еще немного ... и повторять до тех пор, пока вы не разовьете силу и чувство меры.

Я рассказал об этом подробнее в [ How To Learn Hacking](http://www.catb.org/~esr/faqs/hacking-howto.html). Там описан понятный набор техник, но не самый простой.

Раньше, найти свободно распостраняемый код было трудно, что было преградой, для хакеров. Это резко изменилось с приходом программного обеспечения с открытым исходным кодом, инструментов программирования и операционных системы (созданных хакерами). Что подводит меня к нашей следующей теме ...

## 2. Найдите один из Unix, с открытым исходным кодом и научитесь его использовать и запускать.

Я предполагаю, что у вас есть персональный компьютер или вы можете получить к нему доступ. (Задумайтесь, как много это значит. Культура хакеров изначально развивалась, когда компьютеры были настолько дорогими, что люди не могли ими владеть.) Единственный самый важный шаг, который любой новичок может сделать для приобретения навыков хакера, - это получить копию Linux или один из дистрибутивов BSD-Unix, установите его на персональный компьютер и запустите.

Да, в мире есть и другие операционные системы, кроме Unix. Но они распространяются в двоичном формате - вы не можете прочитать код, и вы не можете изменить его. Попытка научиться хакерству на компьютере под управлением Microsoft Windows или в любой другой системе с закрытым исходным кодом - это все равно, что пытаться научиться танцевать, будучи загипсованным.

В Mac OS X это возможно, но только часть системы имеет открытый исходный код - вы, вероятно, испытаете много боли, и вы должны быть осторожны, чтобы не выработать вредную привычку зависеть от проприетарного кода Apple. Если вы сосредоточитесь на Unix под капотом, вы можете узнать некоторые полезные вещи.

Unix - это операционная система Интернета. Хотя вы можете научиться пользоваться Интернетом, не зная Unix, вы не можете быть хакером, не разбираясь в Unix. По этой причине, современная хакерская культура довольно сильно ориентирована на Unix. (Это не всегда было правдой, и некоторые старые хакеры все еще не довольны этим, но симбиоз между Unix и Интернетом стал достаточно сильным, так что даже мускулы Microsoft, похоже, не способны серьезно помешать этому.)

Итак, заполучите Unix - я сам люблю Linux, но существует и другой мир, помимо его (и да, вы можете запускать Linux и Microsoft Windows на одной машине). Изучите UNIX. Запустите UNIX. Разберитесь с UNIX. Поговорите с интернетом, с помощью него. Прочитайте исходный код. Отредактируйте исходный код. Используя UNIX, вы получите лучшие инструменты программирования (включая C, LISP, Python и Perl), больше чем любая операционная система Microsoft. Вам будет весело, и вы будете впитывать больше знаний, чем осознаете, и изучайте UNIX до того, как вы посмотрите на него глазами хакера.

Для получения дополнительной информации об изучении Unix см. [The Loginataka](http://catb.org/~esr/faqs/loginataka.html). Возможно, вы также захотите взглянуть на [The Art of UNIX Programming](http://catb.org/~esr/writings/taoup/).

Блог [Let's Go Larval!](https://letsgolarval.wordpress.com/) - это окно в учебный процесс нового пользователя Linux, которое я считаю необычайно понятным и полезным. Пост [How I Learned Linux](https://letsgolarval.wordpress.com/2015/06/23/how-i-learned-linux/)- хорошая отправная точка.

Чтобы получить в свои руки Linux, посетите [Linux Online!](https://www.linux.org/). Вы можете скачать оттуда Linux или найти локальную группу пользователей Linux, которая поможет вам с установкой.

В течение первых десяти лет существования этого HOWTO, я говорил, что с точки зрения нового пользователя все дистрибутивы Linux практически эквивалентны. Но в 2006-2007 годах появился лучший выбор: Ubuntu. В то время как другие дистрибутивы имеют свои сильные стороны, Ubuntu является самым доступным для новичков в Linux. Остерегайтесь, однако, отвратительного и почти непригодного интерфейса рабочего стола "Unity", который Ubuntu представил по умолчанию несколько лет спустя; варианты Xubuntu или Kubuntu лучше.

Справку и ресурсы BSD Unix можно найти на сайте www.bsd.org.

Хороший способ пощупать Linux - это загрузить то, что фанаты называют Linux live CD, дистрибутив, который полностью запускается с CD или USB-накопителя, без необходимости изменять ваш жесткий диск. Это может быть медленным процессом, потому как дисковод сам по себе медленный, но это способ взглянуть на возможности Linux, без деструктива.

[Я написал учебник по основам Unix и Интернета](http://en.tldp.org/HOWTO/Unix-and-Internet-Fundamentals-HOWTO/index.html).

Раньше, я рекомендовал не устанавливать Linux или BSD как единственную систему, если вы новичок. В настоящее время установщики достаточно хорошо справляются с задачей, что позволяют сделать это самостоятельно даже новичку. Тем не менее, я все еще рекомендую связаться с вашей локальной группой пользователей Linux и попросить помощи. Это никак не повредит, и поможет сгладить процесс.

## 3. Узнайте, как использовать World Wide Web и писать HTML.

Большинство вещей, созданных хакерской культурой, делают их работу невидимой, помогая управлять фабриками, офисами и университетами без какого-либо очевидного влияния на жизнь нехакеров (простых пользователей - прим.пер.). Сеть - это одно большое исключение, огромная блестящая хакерская игрушка, которая, как признают даже политики, изменила мир. Только по этой причине (и многим другим) вам нужно научиться работать в Интернете.

Это значит не только научиться управлять браузером (каждый может это сделать), но и научиться писать HTML, язык разметки в Интернете. Если вы не знаете, как программировать, написание HTML научит вас умственным привычкам, которые помогут вам обучаться в дальнейшем. Итак, создайте домашнюю страницу.

Но просто наличие домашней страницы недостаточно хорошо, чтобы сделать вас хакером. Сеть полна домашних страниц. Большинство из них - бесполезны шлак (подробнее [ The HTML Hell Page](http://catb.org/~esr/html-hell.html)).

Чтобы домашняя страница была полезной, ваша страница должна иметь контент - она ​​должна быть интересной и/или полезной для других хакеров. И это подводит нас к следующей теме ...

## 4. Если вы не владеете техническим английским - выучите его.

Будучи американцем и носителем английского языка, я раньше не хотел давать таких советов. Не хотел, чтобы это воспринималось как своего рода культурный империализм. Но несколько носителей других языков призвали меня указать, что английский является рабочим языком хакерской культуры и Интернета, и что вам нужно знать его, чтобы действовать в хакерском сообществе.

Примерно в 1991 году я узнал, что многие хакеры, у которых английский является вторым языком, используют его в технических дискуссиях, даже если они говорят на одном родном языке. В то время мне сообщили, что английский обладает большим количеством технических слов, чем любой другой язык, и поэтому является просто лучшим инструментом для работы. По тем же причинам перевод технических книг, написанных на английском языке, часто бывает неудовлетворительным (если такой существует).

Линус Торвальдс, финн, комментирует свой код на английском языке (ему, очевидно, никогда не приходило в голову поступить иначе). Его свободное владение английским языком было важным фактором в его способности привлекать всемирное сообщество разработчиков для Linux. Это пример, которому стоит следовать.

Быть носителем английского языка, не гарантирует, что вы обладаете достаточно хорошими языковыми навыками, чтобы действовать как хакер. Если вы пишите полуграмотно, неграмотно и ваш текст изобилует орфографическими ошибками, многие хакеры (включая меня) будут склонны игнорировать вас. Неаккуратное письмо не всегда означает неаккуратное мышление, но мы, как правило, обнаруживаем сильную корреляцию. Если вы еще не можете писать грамотно, научитесь.

# Статус в хакерской культуре

Как и большинство культур без денежной экономики, хакерство работает на репутацию. Вы пытаетесь решить интересные проблемы, но насколько они интересны и действительно ли ваши решения хороши, это то, о чем обычно могут судить только ваши технические коллеги или начальство.

Соответственно, когда вы играете в хакерскую игру, вы учитесь вести счет, прежде всего, исходя из того, что другие хакеры думают о вашем умении (вот почему вы на самом деле не хакер, пока другие хакеры не назовут вас таковым). Этот факт скрывается за изображением хакинга, как одиночной работы; также хакерско-культурным табу (постепенно ослабевающим с конца 1990-х годов, но все еще мощным) против признания того, что эго или признание социума вовлечены в мотивацию хакера.

В частности, хакерство - это то, что антропологи называют культурой подарков (gift culture - прим. пер.). Вы получаете статус и репутацию в нем не путем доминирования над другими людьми, не за счет того, что вы красивы, или за счет того, чего хотят другие люди, а за счет того, что вы готовы делиться. В частности, отдавая свое время, свой творческий потенциал и результаты своего мастерства.

Существует пять основных вещей, которые вы можете сделать, чтобы быть уважаемыми хакерами:

### 1. Написать программное обеспечение с открытым исходным кодом

Первая (самая главная и традиционная) - это написание программ, которые другие хакеры считают забавными или полезными, и предоставление исходников программ всей хакерской культуре.

(Мы привыкли называть это «свободным программным обеспечением» (free software), но это сбивало с толку слишком многих людей, которые не знали точно, что должно означать слово «свободное» (free). Большинство из нас сейчас предпочитают термин «программное обеспечение с открытым исходным кодом»).

Наиболее почитаемыми полубогами Хакерства, являются люди, написавшие полезные программы, которые имели потребность в распостранении и теперь их используют все.

Но здесь есть небольшой исторический момент. В то время, как хакеры всегда считали разработчиков открытого кода основным ядром, до середины 1990-х годов большинство хакеров большую часть времени работали с закрытым исходным кодом. Это было актуально, когда я написал первую версию этого HOWTO в 1996 году; после 1997 года потребовалось распостранение программного обеспечения с открытым исходным кодом, чтобы что-то изменить. Сегодня «сообщество хакеров» и «разработчиков, в проектах с открытым исходным кодом» - это два описания одной и той же культуры, но стоит помнить, что это не всегда было так. (Подробнее об этом см. [Historical Note: Hacking, Open Source, and Free Software](http://www.catb.org/~esr/faqs/hacker-howto.html#history))

### 2. Помогите протестировать и отладить программное обеспечение с открытым исходным кодом

В нашем несовершенном мире, мы неизбежно тратим большую часть времени на отладку. Вот почему любой автор проекта, с открытым кодом, которого заботит свой продукт, скажет вам, что хорошие бета-тестеры (которые знают, как хорошо локализовать проблему, могут устранить баги в новых релизах, и готовы применить несколько простых диагностических процедур) всегда на вес золота. Даже один тестер может иметь большое значение между длительными фазами отладки.

Если вы новичок, попробуйте найти интересующую вас программу и станьте хорошим бета-тестером. Существует естественная прогрессия от помощи тестирования программ до их отладки, с целью их модификации. Таким образом, вы многому научитесь и получите много хорошей кармы с людьми, которые помогут вам позже.

### 3. Публикуйте полезную информацию

Еще одна полезная вещь - собирать и фильтровать полезную и интересную информацию на веб-страницах или в таких документах, как списки часто задаваемых вопросов (FAQ), и делать их общедоступными.

Мейнтейнеры основных FAQ'ов получают почти такое же уважение, как и авторы открытого исходного кода.

### 4. Помогите сохранить инфраструктуру в рабочем состоянии

Хакерской культурой (и инженерным развитием Интернета) управляют добровольцы. Для этого нужно проделать много необходимой, но неприятной работы: администрирование списков рассылки, модерирование групп новостей, ведение больших сайтов с архивами программного обеспечения, разработка RFC и других технических стандартов.

Люди, которые хорошо занимаются подобными вещами, получают большое уважение, потому что все знают, что такая деятельность - огромная временная затрата и не такая веселая, как игра с кодом. Данная помощь показывает преданность хакерскому делу.

### 5. Помогать самой хакерской культуре

Наконец, вы можете помогать и распространять саму культуру (например, написав статью о том, как стать хакером :-)). Данная деятельность не применима к тем, кто какое-то время не занимался первыми четырьмя пунктами, описанными выше.

У хакерской культуры нет лидеров, но в ней есть культурные герои, старейшины, историки и представители. Когда вы прошли долгий путь и вложили свою душу в дело, то можете превратиться в одного из них. Всегда помните: хакеры не признают превосходство Эго в рядах старейшин, поэтому стремиться к такой славе опасно. Вместо того, чтобы стремиться к этому, вы должны победить его и сделать своим спутником скромности и любезности, под стать вашему Статусу.