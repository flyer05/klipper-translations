# Приклад конфігурації

Цей документ містить інструкції щодо додавання прикладу конфігурації Klipper до репозиторію Klipper на github (розміщений у [config directory](../config/)).

Зверніть увагу, що це [Klipper Community Discourse server](https://community.klipper3d.org) також корисний ресурс для пошуку та обміну конфігураційними файлами.

## Інструкції

1. Виберіть відповідну префіксацію імені config:
   1. `printer` префікс використовується для принтерів запасів, які продаються виробником основного потоку.
   1. Приставка `generic` використовується для плат, які можуть використовуватись у різних 3D принтерах.
   1. `kit` префікс для 3D принтерів, які зібрані відповідно до широко використовуваної специфікації. Ці принтери "kit" зазвичай відрізняються від звичайних "принтерів", в яких вони не продаються виробником.
   1. `sample` префікс використовується для налаштування "snippets", який може копіювати і приклеїти в основний файл конфігурації.
   1. `example` префікс використовується для опису кінематики принтера. Цей тип конфігурації, як правило, тільки додано разом з кодом для нового типу принтера kinematics.
1. Всі файли конфігурації повинні закінчитися в `.cfg` suffix. `printer` config files повинні закінчитися в рік, після чого `.cfg` (наприклад, `-2019.cfg`). У цьому випадку рік є приблизним роком, який був проданий принтер.
1. Не використовуйте пробіли або спеціальні символи в назві конфігурації. Назва файлу повинна містити тільки символи `A-Z`, `a-z`, `0-9`, `-`, і ``.
1. Klipper має мати можливість запускати файл конфігурації `printer`, `generic` і `kit` без помилок. Ці конфігураційні файли слід додати до регресійного тесту [test/klippy/printers.test](../test/klippy/printers.test). Додайте нові конфігураційні файли до цього тестового випадку у відповідному розділі та в алфавітному порядку в цьому розділі.
1. Прикладна конфігурація повинна бути для конфігурації "панель" принтера. (Примітки дуже багато "згоджені" для відстеження в основній репозитори Klipper.) Аналогічно, ми додаємо лише приклад конфігурації файлів для принтерів, комплектів та дошки, які мають основну популярність (наприклад, є не менше 100 з них у активному використанні). Переглядайте використання [Klipper Community Discourse server](https://community.klipper3d.org) для інших конфігурацій.
1. Only specify those devices present on the given printer or board. Do not specify settings specific to your particular setup.
   1. For `generic` config files, only those devices on the mainboard should be described. For example, it would not make sense to add a display config section to a "generic" config as there is no way to know if the board will be attached to that type of display. If the board has a specific hardware port to facilitate an optional peripheral (eg, a bltouch port) then one can add a "commented out" config section for the given device.
   1. Не вказуйте `pressure_advance` в прикладі config, так як це значення специфічне для подачі, а не обладнання для принтера. Аналогічно, не вказуйте `max_extrude_only_velocity` ні `max_extrude_only_accel` налаштування.
   1. Не вкажіть розділ налаштування, що містить шлях хосту або апаратне обладнання. Наприклад, не вказуйте `[virtual_sdcard]`, ані `[темперія_host]` розділи налаштувань.
   1. Тільки визначаються макроси, які використовують функціональність, специфічні для даного принтера або для визначення g-кодів, які зазвичай видаються скибочками, налаштованими для даного принтера.
1. Where possible, it is best to use the same wording, phrasing, indentation, and section ordering as the existing config files.
   1. The top of each config file should list the type of micro-controller the user should select during "make menuconfig". It should also have a reference to "docs/Config_Reference.md".
   1. Не скопіюйте польову документацію на прикладі конфігураційних файлів. (Для того, щоб створити технічне навантаження, як оновлення до документації, потрібно змінити його в багатьох місцях.)
   1. Приклад конфігураційних файлів не містить розділу "SAVE_CONFIG". При необхідності скопіюйте відповідні поля з розділу SAVE_CONFIG до відповідного розділу в основній області налаштування.
   1. Використовуйте ` поле: значення` синтаксису замість `field=value`.
   1. При додаванні екструдера `rotation_distance` бажано вказати `gear_ratio` якщо екструдер має механізм передачі. Ми очікуємо обертання_distance в прикладі конфігурацій для кореляції з окружністю захоплених передач в екструдера - це зазвичай в діапазоні від 20 до 35 мм. При уточненні `gear_ratio` бажано вказати фактичні редуктори на механізмі (наприклад, віддати перевагу `gear_ratio: 80:20` над `gear_ratio: 4:1`). Дивитися [вихідний документ](Rotation_Distance.md#using-a-gear_ratio) для отримання додаткової інформації.
   1. Уникайте визначення значень поля, які встановлюються на їх значення за замовчуванням. Наприклад, не слід вказати `min_extrude_temp: 170`, оскільки це вже значення за замовчуванням.
   1. Де можна, лінії не повинні перевищувати 80 стовпчиків.
   1. Уникайте додавання повідомлень налаштування або переадресації до файлів налаштувань. (Приміром, не додаючи рядків, таких як "це файл був створений ...". Місце атрибуту і зміни історії в git-комунальному повідомленні.
1. Не використовуйте будь-які застарілі функції у файлі налаштування прикладу.
1. Не відключити систему безпеки за замовчуванням у файлі налаштування прикладу. Наприклад, конфігурація не повинна вказувати на користувача `max_extrude_cross_section`. Не увімкніть функції відключення. Наприклад, не повинно бути `force_move`.
1. Усі відомі плати, які підтримує Klipper, можуть використовувати послідовну швидкість передачі за замовчуванням 250 000 бод. Не рекомендуйте іншу швидкість передачі даних у прикладі файлу конфігурації.

Приклад конфігураційних файлів подаються за допомогою створення github "Запитати запит". Будь ласка, дотримуйтесь інструкцій у [contributing документа](CONTRIBUTING.md).
