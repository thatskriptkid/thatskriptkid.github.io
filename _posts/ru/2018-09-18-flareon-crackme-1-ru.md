---
layout: post
title: FlareOn Task 1 & 2
tags: [windows, crackme]
category: [ru]
---

Компания FireEye ежегодно проводит онлайн соревнование, под названием [FlareOn challenges](http://flare-on.com/). На момент написания, он еще активен и будет продолжаться до 5 октября 2018 года. Тематикой текущего года (2018) является Minesweeper World Championship. 

В первом задании предлагают найти инвайт код, чтобы попасть на этот чемпионат неофициально. 

![](/assets/images/ru/FlareOn/1%20Minesweeper%20Championship%20Registration/Task.png)

Скачиваем бинарник, он оказывается java приложением, а значит легко декомпилируется. Запускаем, нас просят ввести регистрационный код.

![](/assets/images/ru/FlareOn/1%20Minesweeper%20Championship%20Registration/First-screen.png)

Пробуем ввести что-нибудь и получаем окно с ошибкой.

![](/assets/images/ru/FlareOn/1%20Minesweeper%20Championship%20Registration/Failed-window.png)

Декомпилируем, чтобы посомтреть на код, я использую сайт http://www.javadecompilers.com. Загружаем туда наш jar и там же, онлайн, можно посмотреть исходный код классов.

![](/assets/images/ru/FlareOn/1%20Minesweeper%20Championship%20Registration/Decompile-result.png)

Судя по исходникам, идет проверка со строкой "GoldenTicket2018@flare-on.com", что и является верным кодом. Вводим и получаем успешный результат.

![](/assets/images/ru/FlareOn/1%20Minesweeper%20Championship%20Registration/Success-Window.png)

Во втором задании, под названием "Ultimate Minesweeper", нас просят победить в соревновании по сапёру. 

![](/assets/images/ru/FlareOn/2%20Ultimate%20Minesweeper/Task.png)

Скачиваем и запускаем. Видим практически классического сапёра, с таймером и количеством оставшихся мин.

![](/assets/images/ru/FlareOn/2%20Ultimate%20Minesweeper/First-screen.png)

При проигрыше выдается такое окно и приложение закрывается.

![](/assets/images/ru/FlareOn/2%20Ultimate%20Minesweeper/Fail-window.png)

Ясно, что способом "в лоб" мы игру не пройдем, а значит нужно изучить исходный код. Так как приложение написано на C#, мы можем с легкостью его декомпилировать с помощью программы [DotPeek](https://www.jetbrains.com/decompiler/) или любой другой. Лично мне она нравится, потому что может экспортировать декомплиированный исходник сразу в проект Visual Studio.

![](/assets/images/ru/FlareOn/2%20Ultimate%20Minesweeper/Dotpeek.png)

Декомпилируем, экспортируем в проект для Visual Studio и открываем его.

![](/assets/images/ru/FlareOn/2%20Ultimate%20Minesweeper/Prj-in-visual-studio.png)

Изучаем исходники, дебажим если требуется, и находим функцию, которая вызывается каждый рза, когда мы кликаем по полю.

![](/assets/images/ru/FlareOn/2%20Ultimate%20Minesweeper/Click-callback.png)

Как мы видим, в начале идет проверка на то, является ли выбранное поле бомбой и если является приложение закрывается. Если же нет, то идет проверка на то, открыли ли мы все места без бомб (переменная TotalUnrevealedEmptySquares). И если мы открыли их все, то судя по названию (SuccessPopup) мы увидим окно с флагом. Кстати, если посмотреть на эту форму в исходниках и на ее ресурсы, то можно будет убедиться в этом на 100%. В эту форму передается результат функции GetKey, которая подозрительно похожа на функцию, которая генерит какой-то ключ, на основе 3 значений массива RevealedCells.

![](/assets/images/ru/FlareOn/2%20Ultimate%20Minesweeper/GetKey-function.png)

Из этого можно сделать вывод, что пустых поля всего три. На это же указывает значение переменной TotalUnrevealedEmptySquares в рантайме.

![](/assets/images/ru/FlareOn/2%20Ultimate%20Minesweeper/TotalUnrevealed.png)

Теперь посмотрим на функцию BombRevealed.

![](/assets/images/ru/FlareOn/2%20Ultimate%20Minesweeper/Bomb-Revealed-function.png)

Видим массив под названием MinesPresent. Судя по названию, этот массив содержит информацию о типе поля - пустое/бомба. Перепишем эту функцию так, чтобы в рантайме найти все индексы пустых полей.

![](/assets/images/ru/FlareOn/2%20Ultimate%20Minesweeper/Bombs-Revelead-modified-function.png)

В режиме дебага мы увидим 3 набора индексов строка-столбец:

8 29

21 8

29 25

Это и есть поля без бомб. Запускаем заново и выбираем эти поля

![](/assets/images/ru/FlareOn/2%20Ultimate%20Minesweeper/Right-fields.png)

После этого появляется окно с флагом

![](/assets/images/ru/FlareOn/2%20Ultimate%20Minesweeper/Success-window.png)