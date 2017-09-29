EEPROM emulation in STM32F10x microcontrollers
==============================================

Как известно, в микроконтроллерах STM32 нет энергонезависимой EEPROM памяти. Это сделано для упрощения кристалла, удешевления производства и, как следствие, снижения продажной цены конечной микросхемы - для получения конкурентного преимущества на рынке.

Однако, в embedded приложениях, невозможно обойтись без энергонезависимой памяти: некоторые данные всё же необходимо сохранять на периоды выключения питания устройства (например, "Настройки программы" или "Кастомизуемые пользовательские данные", которые требует логика приложения).

Если нет встроенной EEPROM, то можно подключить внешнюю микросхему EEPROM или FLASH - это обычно и делают, когда требуется сохранять действительно много данных. (Например, "GPS-трекеры" или "Регистраторы данных / Data Loggers" разные... за продолжительное время накапливают очень много телеметрических данных, которые сохраняют локально. Даже те, что имеют встроенный GPRS-модуль и отсылают данные в Интернет - также сохраняют данные локально.)

Но если данных не очень много... Если устройство бюджетное (имеет малые габариты и стоимость)... Если используется готовая макетная плата без внешней EEPROM... Тогда, имеет смысл (или единственное что остаётся) задействовать встроенную FLASH память (предназначенную для программного кода), для сохранения пользовательских данных также.

Но использование FLASH памяти (типа EROM) для Эмуляции EEPROM (в качестве RAM) - имеет ряд ограничений и особенностей... Чтобы предусмотреть которые, требуется особая библиотека-драйвер "Эмулятор".



AN2594 - EEPROM emulation in STM32F10x microcontrollers [ST, August 2009].pdf
-----------------------------------------------------------------------------

ST Microelectronics, для своих контроллеров, в своё время [в 2009 году] выпустила "AppNote 2594", в котором описала концептуальную идею "как это делать правильно". 

Сама идея, изложенная в AN2594, не плоха - наверное, это лучшее, что можно придумать в такой ситуации: для преодоления ограничений FLASH... Но вот "индусский лапша-код", в приводимом "примере-реализации" библиотеки (EEPROM_Emulation/src/eeprom.c v3.1.0) требует ещё очень долгого и тщательного рефакторинга... А лучше, весь его выкинуть и переписать заново! 

Нет, правда! В "eeprom.c" реальный лапша-код в реализации. Но оно работает! Что удивительно: похоже индусы потратили много часов на отладку этого модуля. И много терпения. Я снимаю шляпу перед этим абсурдным упорством. (Мне даже удалось реализовать один довольно ответственный проект с использованием этого модуля - работоспособность протестирована на практике, подтверждаю.)

Тем не менее, отрефакторить этот модуль невозможно: там куча туманных зависимостей - это же автоматное программирование; нужно представлять всю модель, чтобы хоть что-то изменить, и всё не завалилось... Поэтому мне удалось отрефакторить только интерфейс модуля, чуть упростить API, вычистить говнокод из Заголовочного файла (eeprom.h) и инкапсулировать внутрь Файла Реализации (eeprom.c), подальше от глаз. В заголовочном файле оставил только настроечные константы и три высокоуровневых метода.

Однако, код самих методов я не трогал (совсем не трогал) - он остался как в оригинальном AN2594. И его работоспособность гарантируется "инженерами ST" (стрелки закосил).
Ну, я там повставлял свои комментарии в коде: заметки о замеченных "BUG", и предложения об улучшении "TODO"... Но включать правки в код не решился - вдруг я там чего-то недосмотрел. Поэтому оставляю инициативу на ваш страх и риск. И обозначены, конечно, не все проблемы, что там есть: когда фейспалм задолбал, а от мыслей о тщетности бытия совсем испортилось настроение - я бросил разбирать код, решил оставить всё как есть. 

**16bit virtual variables\eeprom.c**

Также, привожу написаный мной высокоуровневый модуль-адаптер (profile.c), реализующий прикладную модель данных и Getter/Setters для переменных. 

Данный модуль не учебный, а реальный - потому в нём много лишнего кода. Но его хорошо брать за основу для своих проектов. И там много комментариев, всё доступно.

**16bit virtual variables\profile.c**

Модель данных в profile.c реализована достаточно сложная (архитектура имеет потенциально гибкие возможности)::

    Данный модуль эмулирует массив из 30 записей (профилей) по 2 ячейки (length + count). 
    И отдельно от массива, вдобавок, ещё 5 отдельных ячеек-переменных (mode, knives, calibration, frequency, flipangle).

Ещё пара слов про типы данных:

    Все конечные ячейки имеют тип uint16_t (2 байта - это обусловлено структурой FLASH памяти). В них, без проблем, можно было бы сохранять и 1-байтные переменные.
    Но если нужно сохранять данные большей разрядности (типа uint32_t), то в Setter-методах данного модуля, следовало бы разбивать оригинальное значение на фрагменты по 2-байта и сохранять их в отдельных ячейках, а в Getter-методах собирать их обратно.

    На самом деле, стоило бы реализовать разные версии этого модуля (драйвера виртуальной памяти), для эмуляции переменных разных размеров: eeprom_uint8.c, eeprom_uint16.c, eeprom_uint32.c ... Это было бы гораздо удобнее использовать, и рациональнее бы расходовалась память и ресурсы микроконтроллера. 
    Но пока на это нет времени, и ближайшее время не предвидится - потому публикую тот код, который удалось допилить.



2Kb data chunk
--------------

Это полностью моя реализация, с нуля. Уже другой модели данных: здесь нет "виртуальных переменных", а сохраняется BLOB данных целиком в "страницу памяти" FLASH (размером 1K или 2K, в зависимости от модели микроконтроллера).

Но также поддерживается ротация "пары страниц", для защиты от сбоя по питанию при сохранении данных (чтобы хотя бы старая "резервная копия" данных сохранилась). И автовосстановление из аномальных состояний.

Также замечу: драйвер поддерживает адресацию не одного, а сразу нескольких BLOB - для разных массивов!

*Назначение:* в BLOB можно сохранять массивы целиком. А можно даже не только массивы, но и более сложные структуры данных: struct of structs of arrays (сколько фантазии на typedef-ить хватит)... 
Выделить буфер в ОЗУ под такую структуру. Закешировать BLOB из FLASH в этот буфер. Оперативно изменять/редактировать данные сколько требуется. А когда пользователь нажимает кнопочку "Save" - сохранять весь массив в энергонезависимую памяти.

*Размер структуры данных:* 1K или 2K, как уже упоминалось - это требование, кстати, жёсткое! Меньше памяти выделять нельзя - будет "Exception: Out of Range"! (Вернее, даже "Exception" не будет - здесь вам не Windows... Просто похерит данные в оперативе, которым не повезло разместиться сразу за куцым буфером - и пойдёт система вразнос. Так что, поосторожнее с этим.)

**2Kb BLOB data\eeprom_page.c"**

Кстати, в оригинальном AN2594 есть пара классных низкоуровневых методов для непосредственного обращения к FLASH странице памяти: AN2594_ErasePage(стереть страницу); AN2594_ProgramHalfWord(записать данные); 
Я их забрал в свой модуль (копирайты ST указал) "низкоуровневых функций работы с FLASH памятью" (flash.c), в качестве Бонуса. Не то чтобы они там нужны (я их не использую в eeprom_page.c - потому что написал свои альтернативные методы, обёртки-адаптеры для HAL-методов), но они мне понравились (своей автономностью от STM32 HAL-библиотеки, и ещё они немного быстрее выполняются). Ну, и чувствуется в них какая-то простота, и оттого ясность... Хотя, это уже глюки.

**2Kb BLOB data\flash.c"**

Пример использования драйвера BLOB-данных: хранимый в энергонезависимой памяти, двумерный массив из 511 * 2 ячеек, типа uint16_t... 
(Пояснение: этот массив задаёт некую "прикладную программу" из 511 шагов по 2 фазы, для каждой из которых запоминается число - время отработки в секундах.)

**2Kb BLOB data\program.c"**

