# ⚡ Ansible Ad-Hoc Commands: Детальная шпаргалка

## 📋 Оглавление

1. [Что такое Ad-Hoc команды](#1-что-такое-ad-hoc-команды)
2. [Базовый синтаксис](#2-базовый-синтаксис)
3. [Паттерны выбора хостов (Patterns)](#3-паттерны-выбора-хостов-patterns)
4. [Ключевые флаги и опции](#4-ключевые-флаги-и-опции)
5. [Топ-10 модулей для Ad-Hoc](#5-топ-10-модулей-для-ad-hoc)
6. [Практические сценарии (One-liners)](#6-практические-сценарии-one-liners)
7. [Продвинутые техники (Async и фоны)](#7-продвинутые-техники-async-и-фоны)
8. [Вопросы для собеседования](#8-вопросы-для-собеседования)

---

## 1. Что такое Ad-Hoc команды

**Ad-Hoc команды** — это быстрые однострочники в Ansible для выполнения разовых задач на целевых серверах без написания полноценных Playbook'ов.

**Когда использовать:**

- ✅ Разведка боем (проверить аптайм, место на диске, версию ОС).
- ✅ Срочный фикс (перезапустить зависший сервис на 5 машинах).
- ✅ Массовые рутинные операции (обновить кэш, почистить логи).
- ❌ Не использовать для сложного деплоя или настройки (для этого есть Playbook).

---

## 2. Базовый синтаксис

```bash
ansible <pattern> -m <module> -a "<arguments>" [options]
```

- `<pattern>` — кого трогаем (группа, хост, `all`).
- `-m <module>` — какой модуль используем (если не указано, по умолчанию `command`).
- `-a "<arguments>"` — аргументы для модуля.
- `[options]` — дополнительные флаги (инвентарь, sudo, параллельность).

---

## 3. Паттерны выбора хостов (Patterns)

Ansible позволяет гибко выбирать, на каких серверах выполнять команду.

```bash
# Все хосты из инвентаря
ansible all -m ping

# Конкретная группа
ansible webservers -m ping

# Несколько групп (логическое ИЛИ)
ansible webservers:dbservers -m ping

# Пересечение групп (логическое И) — хосты, которые есть И в webservers, И в prod
ansible 'webservers:&prod' -m ping

# Исключение группы (логическое НЕ) — все webservers, КРОМЕ тех, что в staging
ansible 'webservers:!staging' -m ping

# Wildcards (маски)
ansible '*.example.com' -m ping
ansible 'web*.prod' -m ping

# Регулярные выражения (начинаются с ~)
ansible '~(web|db).*\.prod' -m ping

# Диапазоны (если хосты так названы в inventory)
ansible 'web[01:10]' -m ping
```

---

## 4. Ключевые флаги и опции

### Подключение и права

```bash
-i inventory.yml          # Указать путь к инвентарю
-u admin                  # Подключиться под конкретным SSH-пользователем
-k                        # Запросить SSH-пароль интерактивно (--ask-pass)
--private-key ~/.ssh/key  # Указать путь к приватному SSH-ключу
-b                        # Become (выполнить команду через sudo)
-bu postgres              # Become User (выполнить от имени конкретного юзера)
-K                        # Запросить пароль для sudo (--ask-become-pass)
```

### Производительность и вывод

```bash
-f 20                     # Forks: количество параллельных потоков (по умолч. 5)
-v                        # Подробный вывод (verbose)
-vv / -vvv                # Еще более подробный (для отладки)
--check                   # Dry-run (показать, что изменится, без реальных действий)
--diff                    # Показать разницу в измененных файлах (работает с --check)
-o                        # Вывод в одну строку (compact output)
```

---

## 5. Топ-10 модулей для Ad-Hoc

### 1. `ping` (Проверка связи)

```bash
ansible all -m ping
# Возвращает "pong", если SSH работает и Python на сервере доступен.
```

### 2. `command` и `shell` (Выполнение команд)

```bash
# command (по умолчанию, безопаснее, НО не понимает пайпы | и редиректы >)
ansible all -a "uptime"
ansible all -a "df -h"

# shell (понимает пайпы, переменные окружения, но менее безопасен)
ansible all -m shell -a "ps aux | grep nginx | wc -l"
ansible all -m shell -a "cat /etc/os-release | grep PRETTY_NAME"
```

### 3. `apt` / `yum` (Управление пакетами)

```bash
# Debian/Ubuntu
ansible webservers -m apt -a "name=nginx state=present update_cache=yes" -b

# RHEL/CentOS
ansible webservers -m yum -a "name=httpd state=latest" -b

# Удаление пакета
ansible all -m apt -a "name=old_package state=absent" -b
```

### 4. `copy` (Копирование файлов)

```bash
# Скопировать файл с управляющей машины на целевые
ansible webservers -m copy -a "src=./nginx.conf dest=/etc/nginx/nginx.conf mode=0644" -b

# Создать файл с контентом на лету
ansible all -m copy -a "content='Hello from Ansible' dest=/tmp/hello.txt"
```

### 5. `file` (Файлы и директории)

```bash
# Создать директорию
ansible all -m file -a "path=/opt/app state=directory mode=0755 owner=deploy" -b

# Удалить файл/папку
ansible all -m file -a "path=/tmp/old.log state=absent" -b

# Создать symlink
ansible all -m file -a "src=/etc/nginx/sites-available/app dest=/etc/nginx/sites-enabled/app state=link" -b
```

### 6. `systemd` / `service` (Управление сервисами)

```bash
# Перезапустить сервис
ansible webservers -m systemd -a "name=nginx state=restarted" -b

# Включить в автозагрузку и запустить
ansible webservers -m systemd -a "name=nginx state=started enabled=yes" -b
```

### 7. `user` (Пользователи)

```bash
# Создать пользователя
ansible all -m user -a "name=deploy state=present shell=/bin/bash groups=sudo append=yes" -b

# Удалить пользователя и его home директорию
ansible all -m user -a "name=old_user state=absent remove=yes" -b
```

### 8. `setup` (Сбор фактов)

```bash
# Собрать ВСЕ факты (огромный JSON)
ansible all -m setup

# Собрать только факты про память
ansible all -m setup -a "filter=ansible_memtotal_mb"

# Собрать факты про сеть
ansible all -m setup -a "filter=ansible_*_address"
```

### 9. `lineinfile` (Изменение строк в файлах)

```bash
# Добавить строку в /etc/hosts (идемпотентно!)
ansible all -m lineinfile -a "path=/etc/hosts line='10.0.0.5 db1.local'" -b

# Изменить строку по регулярному выражению
ansible all -m lineinfile -a "path=/etc/ssh/sshd_config regexp='^#Port' line='Port 2222'" -b
```

### 10. `git` (Работа с репозиториями)

```bash
# Клонировать/обновить репозиторий
ansible webservers -m git -a "repo=https://github.com/user/app.git dest=/opt/app version=main" -b
```

---

## 6. Практические сценарии (One-liners)

### 🔍 Диагностика и мониторинг

```bash
# Найти серверы, где мало места на диске
ansible all -m shell -a "df -h / | awk 'NR>1 {print \$5, \$6}'"

# Посмотреть, кто сейчас залогинен на серверах
ansible all -a "w"

# Проверить версию ядра на всех серверах
ansible all -a "uname -r"
```

### 🛠 Обслуживание и фикс

```bash
# Очистить кэш APT на всех Ubuntu серверах
ansible ubuntu_servers -m apt -a "clean=yes" -b

# Массово перезапустить "зависший" php-fpm
ansible webservers -m systemd -a "name=php-fpm state=restarted" -b

# Найти и удалить старые логи (больше 100МБ)
ansible all -m shell -a "find /var/log -type f -size +100M -delete" -b
```

### 🔐 Безопасность и доступы

```bash
# Добавить свой SSH-ключ всем серверам (требует первоначального пароля -k)
ansible all -m authorized_key -a "user=root key='{{ lookup('file', '~/.ssh/id_rsa.pub') }}'" -k -b

# Заблокировать пользователя
ansible all -m user -a "name=compromised_user state=absent" -b
```

---

## 7. Продвинутые техники (Async и фоны)

### Фоновое выполнение (Fire and Forget)

Если команда выполняется очень долго (например, обновление системы или скачивание большого файла), можно запустить её в фоне, чтобы не блокировать терминал.

**Флаги:**

- `-B <seconds>` — максимальное время выполнения (Background).
- `-P <seconds>` — интервал опроса статуса (Poll). Если `0`, Ansible не ждет результата.

```bash
# Запустить обновление в фоне (максимум 3600 сек, опрос каждые 0 сек = не ждать)
ansible all -m apt -a "upgrade=dist" -B 3600 -P 0 -b

# Ansible вернет "job_id" для каждого хоста.
```

### Проверка статуса фоновой задачи

```bash
# Проверить статус конкретной задачи по job_id
ansible all -m async_status -a "jid=123456789.12345"
```

---

## 8. Вопросы для собеседования

**Q: В чем разница между модулями `command` и `shell`?**
A: `command` — более безопасный и быстрый модуль, он выполняет команду напрямую, без обертки в оболочку. Он НЕ понимает пайпы (`|`), перенаправления (`>`, `<`) и переменные окружения (`$VAR`). Модуль `shell` оборачивает команду в `/bin/sh`, поэтому поддерживает все shell-функции, но из-за этого он менее безопасен и может потерять идемпотентность.

**Q: Как проверить, что изменит Ad-Hoc команда, не выполняя её реально?**
A: Использовать флаг `--check` (или `-C`). Для просмотра изменений в файлах добавить `--diff`.

**Q: Как ускорить выполнение Ad-Hoc команды на 100 серверах?**
A: Увеличить количество параллельных потоков (forks) с помощью флага `-f`. Например, `-f 50`. По умолчанию Ansible подключается только к 5 серверам одновременно.

**Q: Что произойдет, если не указать модуль (`-m`) в Ad-Hoc команде?**
A: Ansible по умолчанию использует модуль `command`. То есть `ansible all -a "uptime"` эквивалентно `ansible all -m command -a "uptime"`.

**Q: Как выполнить команду от имени другого пользователя (не того, под которым идет SSH)?**
A: Использовать флаг `-b` (become) для выполнения через sudo. Если нужно стать конкретным пользователем (например, postgres), добавить флаг `-bu postgres` (become user).

---

## 💡 Шпаргалка: Какой модуль выбрать?

| Задача                       | Модуль        | Пример аргументов (`-a`)       |
| :--------------------------- | :------------ | :----------------------------- |
| **Проверить связь**          | `ping`        | _(без аргументов)_             |
| **Выполнить команду**        | `command`     | `"ls -la /tmp"`                |
| **Команда с пайпом (`\|`)**  | `shell`       | `"df -h \| grep sda"`          |
| **Установить пакет**         | `apt` / `yum` | `"name=nginx state=present"`   |
| **Скопировать файл**         | `copy`        | `"src=local.txt dest=/tmp/"`   |
| **Создать папку**            | `file`        | `"path=/opt state=directory"`  |
| **Удалить файл**             | `file`        | `"path=/tmp/old state=absent"` |
| **Изменить строку в файле**  | `lineinfile`  | `"path=/etc/conf line='x=1'"`  |
| **Запустить сервис**         | `systemd`     | `"name=nginx state=started"`   |
| **Создать юзера**            | `user`        | `"name=dev state=present"`     |
| **Склонировать репозиторий** | `git`         | `"repo=url dest=/opt/app"`     |
| **Узнать факты о системе**   | `setup`       | `"filter=ansible_memtotal_mb"` |

---
