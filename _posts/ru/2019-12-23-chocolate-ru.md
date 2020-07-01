---
layout: post
title: Реверс инжиниринг шоколада!
tags: [fun]
category: [ru]
---

## Intro

Сегодня мы будем реверсить темный и горький шоколад (наконец-то)! Как известно, шоколад поднимает настроение и укрепляет иммунитет, что очень важно, если вы подолгу долбите глазами пиксели и выжигаете глазами код. Из-за большого количества антиоксидантных флавоноидов замедляет старение. У нормального, качественного шоколада - низкий гликемический индекс - 25. А значит его можно смело ставить инсталлироваться ночью.

Цель ресерча найти шоколад с наибольшей горчинкой и наименьшей сладостью. Если вы хотите найти именно годный горький на вкус шоколад, то вам сюда. Качественный горький шоколад должен иметь более 72% какао в своем составе. Это довольно высокая цифра, если в документации вы не видите этой цифры, значит какао там мало и процент такой маленький, что его решили не палить.

За критерий оценки мы будем брать следующие фреймворки, используемые при разработке:

**какао-масло, какао тёртое, сахар, лецитин**

За любые непротестированные third-party компоненты мы будем снимать баллы. Например какао-порошок это непонятная библиотека из неофициального репозитория.

Состав я лично перепечатал с упаковки, а не скачал с интернета. Декомпилировал шоколад я тоже лично.

Исследованию подверглись 14 семплов от следующих девелоперов: **KAZAKSHTAN, Nestle, ROSHEN, Казахстанский PREMIUM DARK, Alpen Gold DARK, DOVE, Millenium Orange, АВК DARK, Mark Sevouni Urban Chocolatier, Лакомства для здоровья Slim Style, O'Zera The chocolate factory, Бабаевский, Победа шоколад горький, Fleur de Lys!** + бонус от нашего реверсера-фрилансера

Все фото оберток шоколада я скачал с интернета, а фото самого шоколада распакованного я сделал сам на нищебродский xiaomi mi a1, так что извините.

## Начинаем
**Tools used:** вкусовые рецепторы, декомпилятор (желудок)

Скачиваем из репозитория свежие семплы

`git clone https://magazin-s-shokoladkami`

В итоге должно получиться так

![](/assets/images/ru/shoco/ALL.jpg)

А после распаковки архивов так

![](/assets/images/ru/shoco/UNPACKED.jpg)

## Alpen Gold DARK, темный шоколад, АРОМАТНЫЙ АПЕЛЬСИН

![](/assets/images/ru/shoco/1_alpen_gold.jpg)

**Состав:сахар, какао тертое, масло какао, жир молочный, лецитин, ароматизатор, апельсиновые кусочки, загуститель, натуральный ароматизатор, краситель**

**Какао:40%**

Какой только дичи не использовали при разработки этого шоколада девелоперы! И загуститель, и краситель, и ароматизатор. Хорошо, что на главной странице проекта нас заранее предупредили, что используется апельсин. Низкое содержание какао идет в минус. Но хотя бы все нужные фреймворки используются в полной мере, и масло какао и какао тертое имеется.

![](/assets/images/ru/shoco/1_alpen.jpg)

**Горечь: 1 из 5**

**Сладость: 5 из 5**

**Цена: 1.08$**

Сахар в составе стоит на первом месте не даром, шоколад очень сладкий из-за чего весь остальной вкус не чувствуется вообще. Апельсиной привкус присутствует только если запихать себе в рот большой кусок. Молочный жир вообще зло в составе. Горечи практически нет, опять же из-за сахара. Еще и блестит на свету, что говорит о неправильном составе. Полная дичь!

**Итоговая оценка: 3 из 10**

## Бабаевский шоколад горький

![](/assets/images/ru/shoco/2_%D0%B1%D0%B0%D0%B1%D0%B0%D0%B5%D0%B2%D1%81%D0%BA%D0%B8%D0%B9.jpg)

**Состав: какао тертое, сахар, масло какао, какао порошок, ядро ореха, лецитин, спирт, конъяк, чай, ароматизатор Ваниль и Миндаль.**

**Какао: 70%**

Вы только посмотрите на этот состав! Там есть спирт и коньяк! Вот это реально неожиданно. Хоть эти библиотеки и никогда не устаревают, но присутствие их настораживает. 

![](/assets/images/ru/shoco/2_babaevski.jpg)

**Горечь: 4 из 5**

**Сладость: 2 из 5**

**Цена: 1.27$**

Самые главные ингредиенты (какао тертое и масло какао) стоят в первой тройке по составу, это уже очень хорошо. На вкус чувствуется хорошая горечь, возможно тут помог коньяк (в шоколаде который). Зачем в составе чай непонятно. Шоколад не блестит. Ароматизаторы оказались лишние, если бы не они и побольше коньяка (в шоколаде), это было 10 из 10!

**Итогая оценка: 7 из 10**

## Fleur de Lys

![](/assets/images/ru/shoco/3_fleur_de_Lys.jpg)

**Состав: сахар, тертое какао, масло какао, какао порошок, лецитин соевый, ароматизатор, жаренный миндаль, бисквитная крошка.**

**Какао: 33.5%**

Опять есть все важные ингредиенты, только вот сахар опять на первом месте. Также добавили какао порошок, дабы снизить долю других ингредиентов. Из-за этого тут всего около 30% какао продуктов! А лецитин тут соевый, что недопустимо!

![](/assets/images/ru/shoco/3_fleur.jpg)

**Горечь: 2 из 5**

**Сладость: 5 из 5**

**Цена: 3.14$**

Огромная цена за такую дичь! Из-за сахара на первом месте в составе мы имеем большую сладость, которая затмевает весь вкус. Орехи и бисквит не добавили шарма в это дорогущее месиво.

**Итогая оценка: 4 из 10**

## Победа, шоколад горький

![](/assets/images/ru/shoco/4_%D0%BF%D0%BE%D0%B1%D0%B5%D0%B4%D0%B0.jpg)

**Состав: сахар, тертое какао, масло какао, какао-порошок, лецитин соевый, натуральная ваниль**

**Какао: 55%**

Почему опять запихали кучу сахара непонятно, хотя его можно класть совсем чуть чуть, что вкус был лучше и выразительней. Тем более на main page огромным шрифтов написали ГОРЬКИЙ. Эмульгатор тут соевый, минус. Что такое натуральная ваниль мне тоже непонятно.

![](/assets/images/ru/shoco/4_pobeda.jpg)

**Горечь: 1 из 5**

**Сладость: 4 из 5**

**Цена: 1.27$**

Очень странный привкус у этого шоколада! И прикол в том, что шоколад то совсем не горький! Как так получилось? Я даже специально попробовал несколько плиток. Мне кажется это все из-за сахара, который все тупо заглушает. Оценку занижаем за большую надпись ГОРЬКИЙ.

**Итоговая оценка: 3 из 10**

## Slim style

![](/assets/images/ru/shoco/5_slim_style.jpeg)

**Состав: какао тертое, подсластитель, какао-масло, отруби ржанные, лецитин, ароматизатор**

**Какао: 77%**

Ну наконец-то! На упаковке написано, что сахара нет. Зато есть подсластитель. Большое содержание какао продуктов, что гордо написано на упаковке. Посмотрим как на вкусе отразятся отруби ржанные.

![](/assets/images/ru/shoco/5_slim.jpg)

**Горечь: 5 из 5!**

**Сладость: 2 из 5**

**Цена: 3.61$**

Откусил побольше, чтобы насладиться отрубями (нет). Очень хорошая горечь! Сладости практически нет, что очень хорошо. Но столь большая цена не оправдывает всего этого - минус 1 балл. Зато на упаковке написано, что ты ешь и стройнеешь!

**Итоговая оценка: 9 из 10**

## Mark Sevouni, Urban Chocolatier

![](/assets/images/ru/shoco/6_mark.png)

**Состав: какао тертое, сахар, какао-порошок, какао масло, лецитин, ароматизатор**

**Какао: 72%**

Минималистичный состав, все важное имеется, но какао масло стоит на четвертом месте *( Также присутсвует ароматизатор.

![](/assets/images/ru/shoco/6_mark.jpg)

![](/assets/images/ru/shoco/6_mark_2.jpg)

**Горечь: 5 из 5!**

**Сладость: 1 из 5**

**Цена: 2.87$**

Очень красивая упаковка с мануалом внутри. Но упаковка нам неинтересна, так как мы хотим просто пожрать шоколада! Отменная горечь, очень небольшая сладость, хотя сахар на втором месте. Приятно тает во рту. Цифра 72% на месте. Сам по себе ну очень вкусный. Цена оправдана (горечь).

**Итоговая оценка: 10 из 10!**

## O'Zera

![](/assets/images/ru/shoco/7_ozera.jpg)

**Состав: тертое какао, сахар, масло какао, лецитин, ароматизатор**

**Какао: 55%**

aka самая огромная картинка. По составу все очень красивое, даже нет какао порошка. Но низкий процент 55 настораживает, видимо недоложили какао продуктов!

![](/assets/images/ru/shoco/7_ozara.jpg)

**Горечь: 2 из 5!**

**Сладость: 4 из 5**

**Цена: 0.68$**

Очень демократичная цена! Шоколад очень сладкий и не сильно горький. Нам такое не подходит. За цену плюс, за состав без лишнего - плюс.

**Итоговая оценка: 7 из 10**

## Nestle горький шоколад

![](/assets/images/ru/shoco/8_nestle_front.jpg)

**Состав: какао тертое, сахар, какао порошок, масло какао, спирт, молочный жир, ароматизаторы**

**Какао: 70%**

Высокий уровень какао продуктов, но опять спирт и жир!

![](/assets/images/ru/shoco/8_nestle.jpg)

**Горечь: 3 из 5!**

**Сладость: 3 из 5**

**Цена: 1.22$**

Знаете что я вам скажу, во вкусе отчетливо чувствуется спирт! Куда делась горечь при таких процентах тоже непонятно. Цена не самая демократичная.

**Итоговая оценка: 5 из 10**

## АВК 1991 DARK, с черникой

![](/assets/images/ru/shoco/9_%D0%90%D0%92%D0%9A_1991.png)

**Состав: сахар, какао тертое, масло какао, кусочки черники, какао порошок, лецитин, соль, ароматизатор**

**Какао:58%**

Сюда даже соль запихали. Состав никакой, тут и какао порошок и сахар на первом месте.

![](/assets/images/ru/shoco/9_avk.jpg)

**Горечь: 1 из 5!**

**Сладость: 5 из 5**

**Цена: 1$**

Много черники во вкусе и куча сладости. Не этого мы искали, горечи совсем нет. Такое чувство, что жуешь чернику с сахаром. Ну хоть спирта не добавили и на том спасибо.

**Итоговая оценка: 0 из 10**

## Millenium Orange 

![](/assets/images/ru/shoco/10_milenium_orange.jpg)

**Состав: какао тертое, сахар, какао масло, какао порошок, апельсиновая цедра, лецитин, масло апельсина, ароматизатор**

**Какао: 74%**

Как и ожидается по названию, тут много апельсинового всего запихали. Имеется какао порошок и ароматизатор *( Но зато высокий процент какао продуктов!

![](/assets/images/ru/shoco/10_milenium.jpg)

**Горечь: 3 из 5!**

**Сладость: 3 из 5**

**Цена: 1.26$**

Горечь еле пробивается сквозь апельсиновый вкус. Средняя сладость и средняя горечь. При таких ингредиентах (масло апельсина) вполне годная цена. Если хочется горького, несладкого и апельсинового, то вполне сойдет

**Итоговая оценка: 8 из 10**

## Dove темный шоколад

![](/assets/images/ru/shoco/11_dove_front.jpg)

**Состав: какао тертое, сахар, какао масло, молочный жир, лецитин, ароматизаторы,.**

**Какао: 47%**

Присутсвует молочный жир, что не есть хорошо. И много разных ароматизаторов.

![](/assets/images/ru/shoco/11_dove.jpg)

**Горечь: 1 из 5!**

**Сладость: 4 из 5**

**Цена: 1.35$**

Из-за обилия ароматизаторов - вкус хороший, но нет горечи! Зато много сладости и цена.

**Итоговая оценка: 4 из 10**

## Казахстанский премиум DARK

![](/assets/images/ru/shoco/12_KZ_PREMIUM.png)

**Состав: какао тертое, сахар, какао масло, лецитин, ваниль**

**Какао: 82%**

Пока лидер по количеству какао продуктов, целых 82%! Состав идеальный!

![](/assets/images/ru/shoco/12_kz_premium.jpg)

**Горечь: 4.5 из 5!**

**Сладость: 2 из 5**

**Цена: 1.12$**

Достаточно горький и не сладкий. Цена нормальная. Топ за свои деньги!

**Итоговая оценка: 10 из 10**

## ROSHEN Dark Bubble

![](/assets/images/ru/shoco/13_roshen_dark_bubble.jpg)

**Состав: сахар, какао тертое, какао масло, эквивалент какао масло, лецитин соевый, ароматизатор**

**Какао: 50%**

Впервые тут видим эквивалент какао масла, нормального видимо недоложили.

![](/assets/images/ru/shoco/13_roshen.jpg)

**Горечь: 2 из 5!**

**Сладость: 5 из 5**

**Цена: 1$**

Так как сахар на первом в месте составе, мы опять получаем тупую сладость. И пузырьки. И всё. По субъективному мнению, на вкус прям дно.

**Итоговая оценка: 0 из 10.**

## KAZAKHSTAN

![](/assets/images/ru/shoco/14_KZ_TRUE.jpg)

**Состав: сахар, какао тертое, какао масло, сыворотка молочная сухая, молоко сухое, лецитин, усилитель вкуса**

**Какао: 45%**

Состав совсем не легендарный. Много какаих-то молочных продуктов. Еще и усилитель вкуса. Сахар на первом месте. Будем надеяться, что какао масло доложили много.

![](/assets/images/ru/shoco/14_kz.jpg)

**Горечь: 0 из 5!**

**Сладость: 5 из 5**

**Цена: 1$**

Совсем не горький. Вкус не похож на тупо сахарный, видимо работает усилитель. Съесть такого много будет трудно, очень сладкий.

**Итоговая оценка: 0 из 10**

## Итог

В самом топе у нас

**Slim style (9 из 10) - 3.61$**

**Mark Sevouni, Urban Chocolatier (10 из 10) - 2.87$**

**Казахстанский премиум DARK (10 из 10) - 1.12$**

Лично для меня, самый топ это "Казахстанский премиум DARK", из-за своей стоимости и доступности. Некоторые виды шоколада из этого списка тяжело встретить в рядовом магазине, а его вполне можно.

В следующий раз, мы будем реверсить концентрированное молоко, не пропустите!

P.S.

## BONUS от **jsn0w**!!!

Вчера был протестирован ещё один экземпляр от местных разработчиков из студии "р4х4т", который может составить конкуренцию этому топу. 

## Рахат 62% 	

![](/assets/images/ru/shoco/bonus1.jpg)

**Состав: какао тёртое, САХАР, какао масло, какао порошок, emulgators+SALT**

**Какао: 62%**

Состав классический, но сахар на втором месте что уже вселяет надежду. Распаковываем стандартным способом используя hands.dll.. Внутри всё стандартно, фольга, стафф, фирменный логотип на плиточках:

![](/assets/images/ru/shoco/bonus2.jpg)

Осторожно кусаем одну плитку, незамечаем как уже во рту тает третья плитка... 
Чувствуется горечь, но не сильно. Сахар к сожалению тоже очень хорошо чувствуется, поэтому ощущения двоякие, 50/50. В целом довольно вкусно, и качественно, но ожидания были попробовать именно горький шоколад, а получился обычный... Баланс какао 62% и сахара делает его универсальным и расширяет целевую аудиторию. Из за невысокой цены и качества этот продукт может сделать конкуренцию вышеупомянутым топам. 

**Горечь: 3 из 5!**

**Сладость: 3 из 5**

**Цена: 1,11$**

**Итоговая оценка: 8 из 10.**

## BONUS от @Franky_T!!!

Пока реверсеры считают байты, крутой пентестер @Franky_T (Telegram) сделала аплоад новых семплов шоколада, чему мы безумно рады! 

![](/assets/images/ru/shoco/rahat_all_2.jpg)

![](/assets/images/ru/shoco/rahat_all_4.jpg)

Целый набор!

**Состав: какао тертое, сахар, какао масло, какао порошок, лецитин, соль, экстракт ванили**

**Какао: 65%, 70%, 80%**

Состав практически классический, за исключением какао порошка и соли. Набор включает в себя три вида, с разным процентным содержанием какао продуктов, такого еще у нас не было! Появилась прекрасная возможность отточить свой навык определения оного! Первым у нас будет самый маленький

## 65%

![](/assets/images/ru/shoco/rahat_65.jpg)

Первые нотки вкуса отдают ванилью, потом начинается горечь, именно то что нам и нужно! Попробовав один маленький квадратик, я не почувствовал особой сладости. Пробуем вторую. И снова ваниль и нет сильной сладости, что однозначно очень хорошо!

**Горечь: 4 из 5!**

**Сладость: 2 из 5**

**Итоговая оценка: 8 из 10.**

## 70%

![](/assets/images/ru/shoco/rahat_11.jpg)

Вкус точно такой же, как и у первого. В начале ваниль, далее горечь. Честно говоря, я не могу сказать, что есть сильное отличие от первого семпла, так что оценки все те же.

**Горечь: 4 из 5!**

**Сладость: 2 из 5**

**Итоговая оценка: 8 из 10.**

## 80%

![](/assets/images/ru/shoco/raghat_80.jpg)

Этот оказался немного тверже, чем другие! Намного более горький, чем другие два и не сладкий! 10 баллов не получает лишь за не совсем каноничный состав.


**Горечь: 5 из 5!**

**Сладость: 2 из 5**

**Итоговая оценка: 9.6 из 10.**