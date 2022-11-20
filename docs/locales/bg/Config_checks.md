# Проверки на конфигурацията

Този документ съдържа списък със стъпки, които помагат да се потвърдят настройките на щифтовете във файла Klipper printer.cfg. Добре е да преминете през тези стъпки, след като сте изпълнили стъпките в [документа за инсталиране](Installation.md).

По време на това ръководство може да се наложи да се направят промени в конфигурационния файл на Klipper. Не забравяйте да издавате команда RESTART (рестартиране) след всяка промяна в конфигурационния файл, за да се уверите, че промяната влиза в сила (въведете "restart" (рестартиране) в терминалния раздел на Octoprint и след това щракнете върху "Send" (изпрати)). Добре е също така да издавате команда STATUS след всеки RESTART, за да проверите дали конфигурационният файл е зареден успешно.

## Проверка на температурата

Започнете с проверка на правилното отчитане на температурите. Преминете към раздела Octoprint temperature (Температура).

![octoprint-temperature](img/octoprint-temperature.png)

Проверете дали температурата на дюзата и леглото (ако е приложимо) е налице и не се увеличава. Ако тя се увеличава, изключете захранването от принтера. Ако температурите не са точни, прегледайте настройките "sensor_type" и "sensor_pin" за дюзата и/или леглото.

## Проверка на M112

Преминете към раздела на терминала Octoprint и издайте команда M112 в полето на терминала. Тази команда изисква Klipper да премине в състояние "изключване". Тя ще накара Octoprint да прекъсне връзката с Klipper - преминете към областта "Връзка" и щракнете върху "Свързване", за да накарате Octoprint да се свърже отново. След това преминете към раздела за температурата на Octoprint и проверете дали температурите продължават да се актуализират и дали не се увеличават. Ако температурите се увеличават, изключете захранването от принтера.

Командата M112 кара Klipper да премине в състояние "изключване". За да изчистите това състояние, издайте команда FIRMWARE_RESTART в терминалния раздел Octoprint.

## Проверка на нагревателите

Отидете в раздела за температурата на Octoprint и въведете 50, последвано от enter в полето за температурата на "Инструмент". Температурата на екструдера в графиката трябва да започне да се увеличава (в рамките на около 30 секунди). След това отидете в падащото поле "Tool" temperature (Температура на инструмента) и изберете "Off" (Изкл.). След няколко минути температурата трябва да започне да се връща към първоначалната си стойност на стайна температура. Ако температурата не се увеличава, проверете настройката "heater_pin" в конфигурацията.

Ако принтерът има отопляемо легло, извършете горния тест отново с леглото.

## Проверка на щифта за разрешаване на стъпковия двигател

Проверете дали всички оси на принтера могат да се движат свободно ръчно (стъпковите двигатели са изключени). Ако това не е така, издайте команда M84, за да деактивирате двигателите. Ако някоя от осите все още не може да се движи свободно, проверете конфигурацията "enable_pin" на стъпковия принтер за дадената ос. При повечето стокови драйвери за стъпкови двигатели изводът за разрешаване на двигателя е "активен нисък" и затова изводът за разрешаване трябва да има "!" преди извода (например "enable_pin: !ar38").

## Проверка на крайните спирания

Преместете ръчно всички оси на принтера, така че нито една от тях да не е в контакт с крайния ограничител. Изпратете командата QUERY_ENDSTOPS чрез раздела на терминала Octoprint. Той трябва да отговори с текущото състояние на всички конфигурирани крайни ограничители и всички те трябва да съобщят за състояние "отворен". За всеки от крайните стопове изпълнете отново командата QUERY_ENDSTOPS, като ръчно задействате крайния стоп. Командата QUERY_ENDSTOPS трябва да отчете крайния стоп като "TRIGGERED" (задействан).

Ако крайният ограничител изглежда инвертиран (съобщава за "отворен", когато е задействан, и обратно), добавете "!" към дефиницията на извода (например "endstop_pin: ^!ar3") или премахнете "!", ако вече има такъв.

Ако крайният ограничител изобщо не се променя, това обикновено означава, че крайният ограничител е свързан към друг щифт. Това обаче може да изисква и промяна на настройката за издърпване на извода (символът "^" в началото на името на endstop_pin - повечето принтери използват резистор за издърпване и символът "^" трябва да присъства).

## Проверка на стъпковите двигатели

Използвайте командата STEPPER_BUZZ, за да проверите свързаността на всеки стъпков двигател. Започнете с ръчно позициониране на дадената ос до средна точка и след това изпълнете `STEPPER_BUZZ STEPPER=stepper_x`. Командата STEPPER_BUZZ ще накара дадения стъпков двигател да се придвижи с един милиметър в положителна посока, след което ще се върне в началното си положение. (Ако крайният ограничител е дефиниран на позиция position_endstop=0, тогава в началото на всяко движение стъпковият механизъм ще се отдалечава от крайния ограничител.) Той ще извърши това колебание десет пъти.

Ако стъпковият механизъм изобщо не се движи, проверете настройките "enable_pin" и "step_pin" за стъпковия механизъм. Ако стъпковият двигател се движи, но не се връща в първоначалното си положение, тогава проверете настройката "dir_pin". Ако стъпковият двигател се колебае в неправилна посока, то това обикновено показва, че "dir_pin" за оста трябва да се обърне. Това става чрез добавяне на символа "!" към "dir_pin" във файла за конфигуриране на принтера (или премахването му, ако вече има такъв). Ако двигателят се движи значително повече или значително по-малко от един милиметър, тогава проверете настройката "rotation_distance".

Изпълнете горния тест за всеки стъпков двигател, дефиниран в конфигурационния файл. (Задайте параметъра STEPPER на командата STEPPER_BUZZ на името на раздела от конфигурацията, който ще бъде тестван.) Ако в екструдера няма нишка, тогава може да се използва STEPPER_BUZZ, за да се провери свързаността на двигателя на екструдера (използвайте STEPPER=extruder). В противен случай е най-добре да се тества отделно двигателят на екструдера (вж. следващия раздел).

След проверка на всички крайни спирачки и проверка на всички стъпкови двигатели механизмът за насочване трябва да се тества. Изпълнете команда G28, за да настроите всички оси. Прекъснете захранването на принтера, ако той не се прибере правилно. Ако е необходимо, повторете стъпките за проверка на крайните ограничители и стъпковите двигатели.

## Проверете двигателя на екструдера

За да тествате двигателя на екструдера, е необходимо да загреете екструдера до температура за печат. Преминете към раздела Octoprint temperature (Температура на Octoprint) и изберете целева температура от падащото поле за температура (или въведете ръчно подходяща температура). Изчакайте принтерът да достигне желаната температура. След това преминете към раздела за управление на Octoprint и щракнете върху бутона "Екструдиране". Проверете дали двигателят на екструдера се върти в правилната посока. Ако не го прави, вижте съветите за отстраняване на проблеми в предишния раздел, за да потвърдите настройките "enable_pin", "step_pin" и "dir_pin" за екструдера.

## Калибриране на PID настройките

Klipper поддържа [PID контрол](https://en.wikipedia.org/wiki/PID_controller) за екструдера и нагревателите на леглото. За да се използва този механизъм за управление, е необходимо да се калибрират PID настройките на всеки принтер (PID настройките, намерени в други фърмуери или в примерните конфигурационни файлове, често работят лошо).

За да калибрирате екструдера, преминете към раздела на терминала OctoPrint и изпълнете командата PID_CALIBRATE. Например: `PID_CALIBRATE HEATER=extruder TARGET=170`

След приключване на теста за настройка стартирайте `SAVE_CONFIG`, за да актуализирате файла printer.cfg с новите настройки на PID.

Ако принтерът има отопляемо легло и той поддържа да се управлява от PWM (Модулация на ширината на импулса) тогава се препоръчва да се използва PID контрол за леглото. (Когато нагревателят за легло се управлява с помощта на алгоритъма PID той може да включва и изключва десет пъти в секунда, което може да не е подходящо за нагреватели с помощта на механичен превключвател.) Типична команда за калибриране на PID за легло е: `PID_CALIBRATE HEATER=heater_bed TARGET=60`

## Следваща стъпка

Това ръководство има за цел да помогне при основната проверка на настройките на пиновете в конфигурационния файл на Klipper. Не забравяйте да прочетете ръководството [Изравняване на леглото](Bed_Level.md). Също така вижте документа [Slicers](Slicers.md) за информация относно конфигурирането на машина за рязане с Klipper.

След като сте се уверили, че основният печат работи, е добре да помислите за калибриране на [аванс на налягането](Pressure_Advance.md).

Възможно е да се наложи да извършите други видове подробна калибрация на принтера - в интернет има редица ръководства, които помагат за това (например, потърсете в интернет "3d printer calibration"). Като пример, ако изпитвате ефекта, наречен звънене, можете да опитате да следвате ръководството за настройка [Resonance compensation](Resonance_Compensation.md).