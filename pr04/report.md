# ПР №4. Аудит событий: journalctl и auditd

## 1. journalctl — системный журнал

### Что зафиксировал журнал о команде sudo cat /etc/shadow

| Поле | Значение | Что означает |
|------|----------|---------------|
| Время | Mar 30 14:23:15 | Дата и время выполнения |
| Хост | debian-server | Имя сервера |
| Программа | sudo[2456] | Команда sudo с PID 2456 |
| Пользователь | user1 | Кто выполнил sudo |
| TTY | pts/0 | Терминал (удалённая сессия) |
| PWD | /home/user1 | Рабочая директория |
| USER | root | От чьего имени выполнена команда |
| COMMAND | /usr/bin/cat /etc/shadow | Какая команда выполнена |

**Разбор полей:**
- `_HOSTNAME` — debian-server
- `SYSLOG_IDENTIFIER` — sudo
- `_PID` — 2456
- `_UID` — 1000
- `MESSAGE` — выполняемая команда
- `PRIORITY` — 5 (notice)

### Ошибки в системе с последней загрузки

Вывод `journalctl -p err -b` показал:
- `systemd[1]: Failed to start Application Manager`
- `kernel: bluetooth hci0: firmware load failed`
- `cron[789]: permission denied`

### Статистика входов

- **Успешных входов:** 4
- **Неудачных попыток:** 2

(Данные получены через `journalctl _COMM=sshd | grep -E "Accepted|Failed"`)

## 2. auditd — настройка правил

### Применённые правила (вывод `auditctl -l`)
-w /etc/passwd -p rwa -k auth-files
-w /etc/shadow -p rwa -k auth-files
-w /etc/group -p rwa -k auth-files
-w /etc/sudoers -p rwa -k priv-files
-w /etc/sudoers.d/ -p rwa -k priv-files
-w /etc/ssh/sshd_config -p rwa -k ssh-config
-a always,exit -F arch=b64 -S open -S openat -F success=0 -k access-denied
-a always,exit -F arch=b64 -S unlink -S unlinkat -k file-delete
-a always,exit -F arch=b64 -S execve -F euid=0 -F auid!=0 -F auid!=-1 -k priv-exec

### Разбор записи SYSCALL

| Поле | Значение | Что означает |
|------|----------|----------------|
| auid | 1000 | исходный пользователь (логин) |
| uid | 0 | реальный ID (root) |
| euid | 0 | эффективный ID (root) |
| comm | cat | имя команды |
| exe | /bin/cat | путь к исполняемому файлу |

## 3. Расследование инцидента

### Хронология действий bob

| Время | Действие | Результат | Команда поиска |
|-------|----------|-----------|----------------|
| 14:47:02 | cat /etc/shadow | Отказ | `ausearch -k access-denied` |
| 14:47:10 | cat /etc/passwd | Успех | `ausearch -k auth-files` |
| 14:47:15 | touch /tmp/totally-not-malware.sh | Успех | `ausearch -i | grep malware` |
| 14:47:22 | cat /etc/sudoers | Отказ | `ausearch -k access-denied` |

### Как auditd помог расследованию

- Зафиксированы даже неудачные попытки доступа
- Поле `auid` позволило точно идентифицировать bob
- Видны конкретные команды и их аргументы

### Чем auditd лучше journalctl для задач аудита безопасности

| Аспект | journalctl | auditd |
|--------|------------|--------|
| Уровень работы | прикладной | ядерный |
| Неудачные попытки | только если приложение логирует | все |
| Обход приложением | возможен | невозможен |
| Детализация | произвольная | каждый системный вызов |

## 4. Связь с нормативкой

| Мера АУД | Как реализована |
|----------|----------------|
| **АУД.4** Аудит безопасности | Правила на /etc/passwd, /etc/shadow, /etc/sudoers |
| **АУД.7** Мониторинг НСД | Правило `-F success=0` фиксирует все отказы доступа |

## 5. Контрольные вопросы

**1. Чем отличается auditd от journalctl?**
journald — логи приложений (добровольные), auditd — системные вызовы (ядро). journald можно обойти, auditd — нет.

**2. Что такое auid?**
Audit User ID — неизменный ID исходного пользователя. При sudo uid меняется, auid остаётся, что позволяет определить реального исполнителя.

**3. Зачем флаг -k?**
Задаёт ключ для группировки и поиска событий (`ausearch -k ключ`).

**4. Чем -w отличается от -a?**
`-w` — следит за файлом/каталогом, `-a` — добавляет правило на системный вызов с условиями.

**5. Что делать при удалении audit.log?**
Хранить логи удалённо, использовать `chattr +a`, настраивать оповещения.

**6. Как сохранить правила после перезагрузки?**
Поместить правила в `/etc/audit/rules.d/*.rules` и выполнить `augenrules --load`.



