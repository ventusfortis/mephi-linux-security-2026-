# ДЗ по безопасности ОС GNU/Linux (МИФИ, 2026)

Стенд: Fedora Linux 44 (Workstation), SELinux в режиме Enforcing, `/home` на btrfs без опции `nosuid`.
Работа выполнялась от имени администратора `arkom` (группа `wheel`).

Все изменения хранятся в конфигурационных файлах (`/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/sudoers`)
и в метаданных файлов на диске (бит set-UID, расширенный атрибут `security.capability`), поэтому сохраняются после перезагрузки.

## Раздел 1. Создание пользователя

Создан обычный пользователь `user1` с UID 1234, входящий в дополнительную группу `students`.
Максимальный срок действия пароля установлен в 90 дней (три месяца), предупреждение об истечении за 7 дней.

```bash
sudo groupadd students
sudo useradd -u 1234 -G students -m -s /bin/bash user1
sudo passwd user1
sudo chage -M 90 -W 7 user1
sudo chage -l user1
```

Подтверждение:

- `etc/passwd` — строка `user1:x:1234:1234::/home/user1:/bin/bash`;
- `etc/group` — строка `students:x:1001:user1`;
- `etc/shadow` — в записи `user1` поле максимального срока равно 90;
- `stat.out` — домашний каталог `/home/user1` с правами `0700`, владелец `user1:user1`.

## Раздел 2. Мониторинг файлов и процессов

### 2.1. Файлы с битом set-UID

Поиск выполнен по всей файловой системе, каталог `/proc` исключён.

```bash
sudo find / -path /proc -prune -o -type f -perm -4000 -exec ls -l {} + 2>/dev/null
```

Результат сохранён в `setuid-files.out`: найдено 28 файлов, среди них стандартные `passwd`, `su`, `sudo`, `mount`,
а также `/home/arkom/my_cat` из раздела 3.

### 2.2. Процессы с повышенными привилегиями

На втором терминале была запущена команда `passwd` (ожидала ввода пароля).
На первом терминале выполнен поиск процессов, у которых эффективный UID равен 0, а реальный UID принадлежит обычному пользователю.

```bash
ps -eo pid,ruser,euser,ruid,euid,comm | awk 'NR==1 || ($5==0 && $4!=0)'
```

Результат сохранён в `processes.out`: процесс `passwd` имеет RUID 1000 (`arkom`) и EUID 0 (`root`).
Дополнительно в выборку попал `fusermount3` — он также запущен с set-UID.

## Раздел 3. Механизм set-UID

Выбрана утилита `cat`, привилегированная операция — чтение файла `/etc/shadow`.
Обычному пользователю операция запрещена: `cat /etc/shadow` завершается ошибкой `Permission denied`.

Утилита скопирована в домашний каталог, владелец изменён на `root`, установлен бит set-UID.

```bash
cp /usr/bin/cat ~/my_cat
sudo chown root:root ~/my_cat
sudo chmod u+s ~/my_cat
~/my_cat /etc/shadow
```

Подтверждение:

- `stat.out` — файл `/home/arkom/my_cat` с правами `4755` (`-rwsr-xr-x`), владелец `root:root`;
- `setuid-demo.out` — строка `user1` из `/etc/shadow`, прочитанная без `sudo`.

## Раздел 4. Механизм привилегий (capabilities)

Выбрана утилита `chown`, привилегированная операция — смена владельца файла на `root`.
Обычному пользователю операция запрещена: `chown root ~/testfile.txt` завершается ошибкой `Operation not permitted`.

Утилита скопирована в домашний каталог, ей выдана привилегия `CAP_CHOWN`.
Владелец копии остаётся обычным пользователем, бит set-UID не требуется.

```bash
cp /usr/bin/chown ~/my_chown
sudo setcap cap_chown=ep ~/my_chown
getcap ~/my_chown
~/my_chown root ~/testfile.txt
```

Подтверждение:

- `getcap.out` — `/home/arkom/my_chown cap_chown=ep`;
- `stat.out` — файл `/home/arkom/my_chown` с правами `0755`, владелец `arkom:arkom`;
- `capability-demo.out` — владелец `testfile.txt` изменён на `root`.

## Раздел 5. Механизм sudo

В `/etc/sudoers` через `visudo` добавлено правило, разрешающее пользователю `user1` выполнять от имени `root`
только команды установки системного времени:

```
user1 ALL=(root) /usr/bin/date, /usr/bin/timedatectl
```

Проверка:

```bash
sudo visudo -c
sudo -l -U user1
```

От имени `user1` команда `sudo date -s "..."` выполняется успешно, а `sudo cat /etc/shadow` отклоняется
с сообщением `user1 is not allowed to execute`. Вывод обеих проверок сохранён в `sudo-demo.out`,
копия `/etc/sudoers` — в `etc/sudoers` (правило в строке 121).

## Раздел 6. Артефакты

- `mephi-screenshot.png` — скриншот терминала с уникальным номером.
- `history.out` — история выполненных команд (`history > ~/history.out`).
- `stat.out` — вывод `stat` для `/home/user1`, `~/my_cat` и `~/my_chown`.
- `getcap.out` — вывод `getcap ~/my_chown`.
- `etc/passwd`, `etc/shadow`, `etc/group` — копии системных файлов учётных записей (раздел 1).
- `etc/sudoers` — копия файла настроек sudo (раздел 5).
- `setuid-files.out` — вывод команды поиска файлов с set-UID (раздел 2.1).
- `processes.out` — вывод команды мониторинга процессов (раздел 2.2).
- `setuid-demo.out` — результат чтения `/etc/shadow` через `~/my_cat` (раздел 3).
- `capability-demo.out` — результат смены владельца через `~/my_chown` (раздел 4).
- `sudo-demo.out` — результат выполнения `date -s` и отказ для `cat` от имени `user1` (раздел 5).
