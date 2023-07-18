---
title: Рішення для резервного копіювання - rsnapshot
author: Steven Spencer
contributors: Ezequiel Bruni
tested_with: 8.5, 8.6, 9.0
tags:
  - резервне копіювання
  - rsnapshot
---

# Рішення для резервного копіювання - _rsnapshot_

## Передумови

  * Знати, як інсталювати додаткові репозиторії та снепшоти з командного рядка
  * Знати про монтування зовнішніх файлових систем по відношенню до вашої машини (зовнішній жорсткий диск, віддалена файлова система тощо)
  * Знати, як користуватися редактором (тут використовується `vi`, але ви можете використовувати свій улюблений редактор)
  * Трохи знати сценарії BASH
  * Знати, як змінити crontab для користувача root
  * Знання відкритих і закритих ключів SSH (тільки якщо ви плануєте запускати віддалене резервне копіювання з іншого сервера)

## Вступ

_rsnapshot_ — це дуже потужна утиліта резервного копіювання, яку можна встановити на будь-якій машині на базі Linux. Він може створити резервну копію машини локально, або ви можете створити резервну копію кількох машин, скажімо, серверів, з однієї машини.

_rsnapshot_ використовує `rsync` і повністю написаний на perl без бібліотечних залежностей, тому немає дивних вимог для встановлення. У випадку Rocky Linux ви, як правило, маєте змогу встановити _rnapshot_ за допомогою репозиторію EPEL. Після першого випуску Rocky Linux 9.0 був період часу, коли EPEL не містив пакета _rsnapshot_. Згодом це було виправлено, але ми включили метод встановлення з джерела на випадок, якщо це повториться.

Ця документація стосується лише встановлення _rsnapshot_ у Rocky Linux.

=== "EPEL Install"

    ## Встановлення _rsnapshot_
    
    Усі наведені тут команди викликаються з командного рядка на вашому сервері чи робочій станції, якщо не зазначено інше.
    
    ### Встановлення репозиторію EPEL
    
    Для встановлення _rsnapshot_ нам потрібен репозиторій програмного забезпечення EPEL із Fedora. Щоб встановити репозиторій, просто скористайтеся цією командою:

    ```
    sudo dnf install epel-release
    ```


    Тепер репозиторій має бути активним.
    
    ### Встановіть пакет _rsnapshot_
    
    Далі встановіть _rsnapshot_ і деякі інші необхідні інструменти, які, ймовірно, вже встановлені:

    ``` 
    sudo dnf install rsnapshot openssh-server rsync
    ```


    Якщо є будь-які відсутні залежності, вони з’являться, і вам просто потрібно буде відповісти на запит, щоб продовжити. Наприклад:

    ```
    dnf install rsnapshot
    Last metadata expiration check: 0:00:16 ago on Mon Feb 22 00:12:45 2021.
    Dependencies resolved.
    ========================================================================================================================================
    Package                           Architecture                 Version                              Repository                    Size
    ========================================================================================================================================
    Installing:
    rsnapshot                         noarch                       1.4.3-1.el8                          epel                         121 k
    Installing dependencies:
    perl-Lchown                       x86_64                       1.01-14.el8                          epel                          18 k
    rsync                             x86_64                       3.1.3-9.el8                          baseos                       404 k

    Transaction Summary
    ========================================================================================================================================
    Install  3 Packages

    Total download size: 543 k
    Installed size: 1.2 M
    Is this ok [y/N]: y
    ```

=== "Source Install"

    ## Встановлення _rsnapshot_ з вихідного коду
    
    Встановити _rsnapshot_ з вихідного коду не складно. Однак у нього є недолік: у разі випуску нової версії для оновлення версії знадобиться нова інсталяція з вихідного коду, тоді як метод встановлення EPEL забезпечить вам оновлення за допомогою простого `dnf оновлення`. 
    
    ### Встановлення засобів розробки та завантаження вихідного коду
    
    Як зазначено, першим кроком є встановлення групи «Інструменти розробки»:

    ```
    dnf groupinstall 'Development Tools'
    ```


    Вам також знадобляться кілька інших пакетів:

    ```
    dnf install wget unzip rsync openssh-server
    ```


    Далі нам потрібно буде завантажити вихідні файли зі сховища GitHub. Ви можете зробити це кількома способами, але найпростішим у цьому випадку є просто завантажити файл ZIP зі сховища.

    1. Перейдіть на сторінку https://github.com/rsnapshot/rsnapshot
    2. Натисніть зелену кнопку «Code» праворуч![Код](images/code.png)
    3. Клацніть правою кнопкою миші на «Download ZIP» і скопіюйте розташування посилання![Zip](images/zip.png)
    4. Використовуйте `wget` або `curl`, щоб завантажити скопійоване посилання. Приклад:
    ```
    wget https://github.com/rsnapshot/rsnapshot/archive/refs/heads/master.zip
    ```
    5. Розархівуйте файл `master.zip`
    ```
    unzip master.zip
    ```


    ### Побудова Джерела

    Тепер, коли ми маємо все на нашій машині, наступним кроком є збірка. Коли ми розпакували файл `master.zip`, ми отримали каталог `rsnapshot-master`. Нам потрібно буде змінити це для нашої процедури побудови. Зауважте, що наша збірка використовує всі параметри пакета за замовчуванням, тому, якщо ви хочете щось інше, вам потрібно провести невелике дослідження. Крім того, ці дії виконуються безпосередньо зі сторінки [Встановлення GitHub](https://github.com/rsnapshot/rsnapshot/blob/master/INSTALL.md):

    ```
    cd rsnapshot-master
    ```

    Запустіть сценарій `authogen.sh`, щоб створити сценарій налаштування:

    ```
    ./autogen.sh
    ```

    !!! tip "Підказка"

        Ви можете отримати кілька рядків, які виглядатимуть так:

        ```
        fatal: not a git repository (or any of the parent directories): .git
        ```

        Це не смертельно.

    Далі нам потрібно запустити `configure` із набором каталогу конфігурації:

    ```
    ./configure --sysconfdir=/etc
    ```

    Нарешті, запустіть `make install`:

    ```
    sudo make install
    ```

    Під час усього цього файл `rsnapshot.conf` буде створено як `rsnapshot.conf.default`. Нам потрібно скопіювати це до `rsnapshot.conf`, а потім відредагувати, щоб відповідати тому, що нам потрібно для нашої системи.

    ```
    sudo cp /etc/rsnapshot.conf.default /etc/rsnapshot.conf
    ```

    Це стосується копіювання файлу конфігурації. У розділі нижче про «Налаштування rsnapshot» описано зміни, необхідні в цьому файлі конфігурації.

## Підключення диска або файлової системи для резервного копіювання

У цьому кроці ми покажемо, як підключити жорсткий диск, наприклад зовнішній жорсткий диск USB, який використовуватиметься для резервного копіювання вашої системи. Цей конкретний крок необхідний, лише якщо ви створюєте резервну копію однієї машини або сервера, як показано в нашому першому прикладі нижче.

1. Підключіть USB-накопичувач.
2. Введіть `dmesg | grep sd`, який має показати вам диск, який ви хочете використовувати. У цьому випадку він називатиметься _sda1_.  
   Приклад: `EXT4-fs (sda1): монтування файлової системи ext2 за допомогою підсистеми ext4`.
3. На жаль (або на щастя, залежно від вашої думки), більшість сучасних настільних операційних систем Linux автоматично монтують диск, якщо можуть. Це означає, що залежно від різних факторів _rsnapshot_ може втратити відстеження жорсткого диска. Ми хочемо, щоб диск «монтувався» або робив його файли доступними в тому самому місці кожного разу.  
   Щоб зробити це, візьміть інформацію про диск, наведену в команді `dmesg` вище, і введіть `mount | grep sda1`, який має показати щось на зразок цього: `/dev/sda1 на /media/username/8ea89e5e-9291-45c1-961d-99c346a2628a`
4. Введіть `sudo umount /dev/sda1`, щоб відключити зовнішній жорсткий диск.
5. Далі створіть нову точку підключення для резервної копії: `sudo mkdir /mnt/backup`
6. Тепер підключіть диск до папки резервної копії: `sudo mount /dev/sda1 /mnt/backup`
7. Тепер введіть `mount | grep sda1` знову, і ви повинні побачити щось на зразок цього: `/dev/sda1 на /mnt/backup type ext2 (rw,relatime)`
8. Далі створіть каталог, який повинен існувати для продовження резервного копіювання на підключеному диску. У цьому прикладі ми використовуємо папку під назвою «storage»: `sudo mkdir /mnt/backup/storage`

Зауважте, що для однієї машини вам доведеться або повторювати кроки `umount` і `mount` кожного разу, коли диск знову під’єднується, або кожного разу, коли система перезавантажується, або автоматизувати ці команди зі сценарієм.

Ми рекомендуємо автоматизацію. Автоматизація - це спосіб системного адміністратора.

## Налаштування rsnapshot

Це найважливіший крок. При внесенні змін у файл конфігурації легко зробити помилку. Конфігурація _rsnapshot_ вимагає вкладок для будь-якого розділення між елементами, і попередження про це є у верхній частині файлу конфігурації.

Пробіл спричинить збій усієї конфігурації та вашої резервної копії. Наприклад, у верхній частині файлу конфігурації є розділ для `# SNAPSHOT ROOT DIRECTORY #`. Якби ви додавали це з нуля, ви б ввели `snapshot_root`, потім TAB, а потім введіть `/whatever_the_path_to_the_snapshot_root_will_be/`

Найкраще те, що стандартна конфігурація, яка постачається з _rsnapshot_, потребує лише незначних змін, щоб вона працювала для резервного копіювання локальної машини. Проте завжди доцільно створити резервну копію файлу конфігурації перед початком редагування:

`cp /etc/rsnapshot.conf /etc/rsnapshot.conf.bak`

## Резервне копіювання базової машини або одного сервера

У цьому випадку _rsnapshot_ буде запущено локально для резервного копіювання певної машини. У цьому прикладі ми розберемо конфігураційний файл і покажемо, що саме потрібно змінити.

Щоб відкрити файл _/etc/rsnapshot.conf_, вам знадобиться використати `vi` (або відредагувати його у вашому улюбленому редакторі).

Перше, що потрібно змінити, це параметр _snapshot_root_, який за умовчанням має таке значення:

`snapshot_root   /.snapshots/`

Нам потрібно змінити це на нашу точку монтування, яку ми створили вище, а також додати «сховище».

`snapshot_root   /mnt/backup/storage/`

Ми також хочемо, щоб резервне копіювання НЕ запускалося, якщо диск не підключено. Для цього видаліть знак «#» (який також називають зауваженням, знаком фунта, знаком числа, символом решітки тощо) біля `no_create_root`, щоб він виглядав так:

`no_create_root 1`

Далі перейдіть до розділу під назвою `# EXTERNAL PROGRAM DEPENDENCIES #` і видаліть коментар (знову знак «#») із цього рядка:

`#cmd_cp         /usr/bin/cp`

Так що тепер він читає:

`cmd_cp         /usr/bin/cp`

Хоча нам не потрібен `cmd_ssh` для цієї конкретної конфігурації, він знадобиться для нашої іншої опції нижче, і не завадить її ввімкнути. Отже, знайдіть рядок, який говорить:

`#cmd_ssh        /usr/bin/ssh`

І приберіть знак "#", щоб це виглядало так:

`cmd_ssh        /usr/bin/ssh`

Далі нам потрібно перейти до розділу під назвою `# BACKUP LEVELS / INTERVALS #`

Це було змінено порівняно з попередніми версіями _rsnapshot_ з `щогодини, щодня, щомісяця, щороку` на `альфа, бета, гамма, дельта`. Що трохи заплутало. Що вам потрібно зробити, це додати примітку до будь-якого інтервалу, який ви не використовуватимете. У конфігурації дельта вже виділена.

У цьому прикладі ми не будемо запускати жодних інших інкрементів, окрім нічного резервного копіювання, тому просто додайте примітку до альфа- та гамми, щоб конфігурація виглядала так, коли ви закінчите:

```
#retain  alpha   6
retain  beta    7
#retain  gamma   4
#retain delta   3
```

Тепер перейдіть до рядка файлу журналу, який за замовчуванням має бути таким:

`#logfile        /var/log/rsnapshot`

І видаліть примітку, щоб вона була включена:

`logfile        /var/log/rsnapshot`

Нарешті, перейдіть до розділу `### BACKUP POINTS / SCRIPTS ###` та додайте будь-які каталоги, які ви хочете додати, у розділ `# LOCALHOST`, не забудьте використовувати TAB а не ПРОБІЛ між елементами!

Наразі запишіть свої зміни (`SHIFT :wq!` для `vi`) і вийдіть із файлу конфігурації.

### Перевірка конфігурації

Ми хочемо переконатися, що під час редагування файлу конфігурації ми не додали пробіли чи інші явні помилки. Для цього ми запускаємо _rsnapshot_ для нашої конфігурації за допомогою параметра `configtest`:

`rsnapshot configtest` покаже `Syntax OK`, якщо в конфігурації немає помилок.

Ви маєте виробити звичку запускати `configtest` для конкретної конфігурації. Причина цього стане більш очевидною, коли ми перейдемо до розділу **Резервне копіювання кількох машин або кількох серверів**.

Щоб запустити `configtest` для певного файлу конфігурації, запустіть його з параметром -c, щоб визначити конфігурацію:

`rsnapshot -c /etc/rsnapshot.conf configtest`

## Перший запуск резервного копіювання

Все було перевірено, тож настав час продовжити та запустити резервне копіювання вперше. Ви можете спочатку запустити це в тестовому режимі, щоб побачити, що робитиме сценарій резервного копіювання.

Знову ж таки, для цього вам не обов’язково вказувати конфігурацію в цьому випадку, але ви повинні мати звичку робити це:

`rsnapshot -c /etc/rsnapshot.conf -t beta`

Що має повернути щось на зразок цього, показуючи вам, що станеться, коли резервне копіювання фактично запуститься:

```
echo 1441 > /var/run/rsnapshot.pid
mkdir -m 0755 -p /mnt/backup/storage/beta.0/
/usr/bin/rsync -a --delete --numeric-ids --relative --delete-excluded \
    /home/ /mnt/backup/storage/beta.0/localhost/
mkdir -m 0755 -p /mnt/backup/storage/beta.0/
/usr/bin/rsync -a --delete --numeric-ids --relative --delete-excluded /etc/ \
    /mnt/backup/storage/beta.0/localhost/
mkdir -m 0755 -p /mnt/backup/storage/beta.0/
/usr/bin/rsync -a --delete --numeric-ids --relative --delete-excluded \
    /usr/local/ /mnt/backup/storage/beta.0/localhost/
touch /mnt/backup/storage/beta.0/
```

Коли ви задоволені тестом, запустіть його вручну вперше без тесту:

`rsnapshot -c /etc/rsnapshot.conf beta`

Коли резервне копіювання завершиться, перейдіть до /mnt/backup і подивіться на структуру каталогів, яка там була створена. Буде каталог `storage/beta.0/localhost`, а потім каталоги, які ви вказали для резервного копіювання.

### Подальші пояснення

Кожного разу, коли виконується резервне копіювання, воно створюватиме новий приріст бета-версії, резервні копії тривалістю 0–6 або 7 днів. Найновіша резервна копія завжди буде beta.0, тоді як вчорашня резервна копія завжди буде beta.1.

Здається, що розмір кожної з цих резервних копій займає однаковий (або більший) дисковий простір, але це через те, що _rsnapshot_ використовує жорсткі посилання. Щоб відновити файли з вчорашньої резервної копії, скопіюйте їх назад зі структури каталогів beta.1.

Кожне резервне копіювання є лише додатковим резервним копіюванням із попереднього запуску, АЛЕ через використання жорстких посилань кожен каталог резервного копіювання містить або файл, або жорстке посилання на файл, незалежно від того, у якому каталозі було створено резервну копію.

Отже, для відновлення файлів вам не потрібно вибирати, з якого каталогу чи приросту їх відновлювати, а лише те, яку позначку часу має мати резервна копія, яку ви відновлюєте. Це чудова система, яка займає набагато менше місця на диску, ніж багато інших рішень для резервного копіювання.

## Налаштування автоматичного запуску резервного копіювання

Після того, як усе буде перевірено, і ми знаємо, що все працюватиме без проблем, наступним кроком буде налаштування crontab для користувача root, щоб усе це можна було робити автоматично щодня:

`sudo crontab -e`

Якщо ви раніше цього не запускали, виберіть vim.basic як редактор або власне налаштування редактора, коли з’явиться рядок `Select an editor`.

Ми збираємося налаштувати резервне копіювання на автоматичний запуск о 23:00, тому ми додамо це до crontab:

```
## Running the backup at 11 PM
00 23 *  *  *  /usr/bin/rsnapshot -c /etc/rsnapshot.conf beta`
```

## Резервне копіювання кількох машин або кількох серверів

Створення резервних копій кількох машин із машини з масивом RAID або великою ємністю пам’яті, як локально, так і через Інтернет, працює дуже добре.

Якщо ви виконуєте ці резервні копії через Інтернет, вам потрібно переконатися, що обидва місця мають достатню пропускну здатність для створення резервних копій. Ви можете використовувати _rsnapshot_ для синхронізації локального сервера з зовнішнім масивом резервного копіювання або резервним сервером для покращення резервування даних.

## Припущення

Ми припускаємо, що ви запускаєте _rsnapshot_ із машини віддалено локально. Цю точну конфігурацію можна скопіювати, як зазначено вище, також віддалено поза приміщенням.

У цьому випадку ви захочете інсталювати _rsnapshot_ на машині, яка виконує всі резервні копії. Ми також припускаємо:

* Що сервери, на які ви будете створювати резервні копії, мають правило брандмауера, яке дозволяє віддаленій машині підключитися до нього через SSH
* Що на кожному сервері, резервну копію якого ви збираєтеся, було встановлено останню версію `rsync`. Для серверів Rocky Linux запустіть `dnf install rsync`, щоб оновити версію `rsync` вашої системи.
* Що ви або підключилися до машини як користувач root, або ви запустили `sudo -s`, щоб перейти до користувача root.

## Відкриті/приватні ключі SSH

Для сервера, який виконуватиме резервне копіювання, нам потрібно створити пару ключів SSH для використання під час резервного копіювання. Для нашого прикладу ми будемо створювати ключі RSA.

Якщо у вас уже є згенерований набір ключів, ви можете пропустити цей крок. Ви можете дізнатися, виконавши `ls -al .ssh` і знайшовши пару ключів `id_rsa` та `id_rsa.pub`. Якщо такого немає, скористайтеся наведеним нижче посиланням, щоб налаштувати ключі для вашої машини та серверів, до яких ви хочете отримати доступ:

[Пари відкритих закритих ключів SSH](../security/ssh_public_private_keys.md)

## Конфігурація _rsnapshot_

Файл конфігурації має бути таким же, як той, який ми створили для **резервної копії базової машини або одного сервера** вище, за винятком того, що ми хочемо змінити деякі параметри.

Корінь знімка можна повернути до стандартного так:

`snapshot_root   /.snapshots/`

І цей рядок:

`no_create_root 1`

... можна знову прокоментувати:

`#no_create_root 1`

Інша відмінність полягає в тому, що кожна машина матиме власну конфігурацію. Коли ви звикнете до цього, ви просто скопіюєте один із наявних конфігураційних файлів під новою назвою, а потім зміните його відповідно до будь-яких додаткових машин, резервну копію яких ви хочете створити.

Наразі ми хочемо змінити файл конфігурації, як ми робили вище, а потім зберегти його. Потім скопіюйте цей файл як шаблон для нашого першого сервера:

`cp /etc/rsnapshot.conf /etc/rsnapshot_web.conf`

Ми хочемо змінити новий файл конфігурації та створити журнал і файл блокування з назвою машини:

`logfile /var/log/rsnapshot_web.log`

`lockfile        /var/run/rsnapshot_web.pid`

Далі ми хочемо змінити rsnapshot_web.conf так, щоб він містив каталоги, резервні копії яких ми хочемо створити. Єдине, що тут інше – це мета.

Ось приклад конфігурації web.ourdomain.com:

```
### BACKUP POINTS / SCRIPTS ###
backup  root@web.ourourdomain.com:/etc/     web.ourourdomain.com/
backup  root@web.ourourdomain.com:/var/www/     web.ourourdomain.com/
backup  root@web.ourdomain.com:/usr/local/     web.ourdomain.com/
backup  root@web.ourdomain.com:/home/     web.ourdomain.com/
backup  root@web.ourdomain.com:/root/     web.ourdomain.com/
```

### Перевірка конфігурації та запуск початкової резервної копії

Як і раніше, тепер ми можемо перевірити конфігурацію, щоб переконатися, що вона синтаксично правильна:

`rsnapshot -c /etc/rsnapshot_web.conf configtest`

І, як і раніше, ми шукаємо повідомлення `Syntax OK`. Якщо все добре, ми можемо виконати резервне копіювання вручну:

`/usr/bin/rsnapshot -c /etc/rsnapshot_web.conf beta`

Якщо припустити, що все працює нормально, ми можемо створити файли конфігурації для поштового сервера (rsnapshot_mail.conf) і сервера порталу (rsnapshot_portal.conf), перевірити їх і зробити пробну резервну копію.

## Автоматизація резервного копіювання

Автоматизація резервного копіювання для версії з кількома машинами/серверами дещо відрізняється. Ми хочемо створити сценарій bash для виклику резервних копій по порядку. Коли один закінчить, почнеться наступний. Цей сценарій виглядатиме приблизно так і зберігатиметься в /usr/local/sbin:

`vi /usr/local/sbin/backup_all`

Зі змістом:

```
#!/bin/bash/
# script to run rsnapshot backups in succession
/usr/bin/rsnapshot -c /etc/rsnapshot_web.conf beta
/usr/bin/rsnapshot -c /etc/rsnapshot_mail.conf beta
/usr/bin/rsnapshot -c /etc/rsnapshot_portal.conf beta
```
Потім ми робимо скрипт виконуваним:

`chmod +x /usr/local/sbin/backup_all`

Потім створіть crontab для root для запуску сценарію резервного копіювання:

`crontab -e`

І додайте цей рядок:

```
## Running the backup at 11 PM
00 23 *  *  *  /usr/local/sbin/backup_all
```

## Повідомлення про статус резервного копіювання

Щоб переконатися, що резервне копіювання виконується згідно з планом, ви можете надіслати файли журналу резервного копіювання на свою електронну пошту. Якщо ви виконуєте резервне копіювання кількох машин за допомогою _rsnapshot_, кожен файл журналу матиме власну назву, яку ви можете надіслати на свою електронну пошту для перегляду за допомогою [Використання процедури postfix For Server Process Reporting](../email/postfix_reporting.md).

## Відновлення резервної копії

Відновлення з резервної копії, або кількох файлів, або повне відновлення, передбачає копіювання файлів, які ви хочете, з каталогу з датою, яку ви хочете відновити, назад на вашу машину. Просто!

## Висновки та інші ресурси

Правильне налаштування за допомогою _rsnapshot_ спочатку трохи складне, але може заощадити багато часу на резервне копіювання ваших машин або серверів.

_rsnapshot_ дуже потужний, дуже швидкий і дуже економний у використанні дискового простору. Ви можете знайти більше інформації про _rsnapshot_, відвідавши [rsnapshot.org](https://rsnapshot.org/download.html)