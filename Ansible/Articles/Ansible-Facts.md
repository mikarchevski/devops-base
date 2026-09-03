# 🔍 Ansible Facts: Детальная шпаргалка

## 📋 Оглавление

1. [Что такое Facts](#1-что-такое-facts)
2. [Как собирать Facts](#2-как-собирать-facts)
3. [Самые популярные Facts](#3-самые-популярные-facts)
4. [Как использовать Facts в задачах](#4-как-использовать-facts-в-задачах)
5. [Кастомные Facts (set_fact)](#5-кастомные-facts-set_fact)
6. [Отключение и кэширование Facts](#6-отключение-и-кэширование-facts)
7. [Практические примеры](#7-практические-примеры)
8. [Best Practices](#8-best-practices)
9. [Вопросы для собеседования](#9-вопросы-для-собеседования)

---

## 1. Что такое Facts

**Facts (Факты)** — это переменные, которые Ansible **автоматически собирает** с целевого сервера при подключении перед выполнением задач.

**Зачем нужны:**

- ✅ Узнать характеристики сервера (ОС, память, CPU, сеть)
- ✅ Принимать решения на основе состояния системы
- ✅ Генерировать уникальные конфиги для каждого сервера
- ✅ Адаптировать playbook под разные ОС и окружения

**Как это работает:**

1. Ansible подключается к серверу по SSH
2. Загружает Python-скрипт модуля `setup`
3. Скрипт собирает информацию о системе
4. Возвращает данные в формате JSON
5. Ansible сохраняет их в переменные с префиксом `ansible_`

---

## 2. Как собирать Facts

### Автоматический сбор (по умолчанию)

По умолчанию Ansible собирает facts перед выполнением каждого play:

```yaml
---
- hosts: webservers
  # gather_facts: yes (по умолчанию)

  tasks:
    - name: Show OS
      debug:
        msg: "OS is {{ ansible_distribution }}"
```

### Ручной сбор через ad-hoc команду

```bash
# Собрать ВСЕ facts для хоста
ansible web1.example.com -m setup

# Собрать facts для группы
ansible webservers -m setup

# Собрать конкретный fact
ansible web1.example.com -m setup -a "filter=ansible_memtotal_mb"

# Собрать facts с фильтром (wildcard)
ansible web1.example.com -m setup -a "filter=ansible_*_address"
```

### Ручной сбор в playbook

Если `gather_facts: no`, можно собрать facts вручную:

```yaml
---
- hosts: webservers
  gather_facts: no # Отключаем автоматический сбор

  tasks:
    - name: Gather facts manually
      setup:
        filter: ansible_memtotal_mb # Собрать только память

    - name: Use fact
      debug:
        msg: "Memory: {{ ansible_memtotal_mb }} MB"
```

### Вывод facts в JSON

```bash
# Красивый JSON вывод
ansible web1.example.com -m setup | jq

# Сохранить в файл
ansible web1.example.com -m setup > facts.json

# Только конкретные facts
ansible web1.example.com -m setup -a "filter=ansible_os_family" -o
```

---

## 3. Самые популярные Facts

### 🌐 Сеть и хосты

| Fact                           | Описание                  | Пример значения                |
| ------------------------------ | ------------------------- | ------------------------------ |
| `ansible_hostname`             | Короткое имя хоста        | `web1`                         |
| `ansible_fqdn`                 | Полное доменное имя       | `web1.example.com`             |
| `ansible_domain`               | Домен                     | `example.com`                  |
| `ansible_default_ipv4.address` | Основной IPv4 адрес       | `192.168.1.10`                 |
| `ansible_default_ipv4.gateway` | Шлюз по умолчанию         | `192.168.1.1`                  |
| `ansible_all_ipv4_addresses`   | Все IPv4 адреса           | `['192.168.1.10', '10.0.0.5']` |
| `ansible_all_ipv6_addresses`   | Все IPv6 адреса           | `['fe80::1']`                  |
| `ansible_dns.nameservers`      | DNS серверы               | `['8.8.8.8', '8.8.4.4']`       |
| `ansible_interfaces`           | Сетевые интерфейсы        | `['eth0', 'lo']`               |
| `ansible_eth0.ipv4.address`    | IP конкретного интерфейса | `192.168.1.10`                 |
| `ansible_eth0.macaddress`      | MAC адрес интерфейса      | `00:11:22:33:44:55`            |

### 💻 Операционная система

| Fact                                 | Описание               | Пример значения                 |
| ------------------------------------ | ---------------------- | ------------------------------- |
| `ansible_os_family`                  | Семейство ОС           | `Debian`, `RedHat`, `Suse`      |
| `ansible_distribution`               | Дистрибутив            | `Ubuntu`, `CentOS`, `AlmaLinux` |
| `ansible_distribution_version`       | Полная версия          | `20.04`, `8.5`                  |
| `ansible_distribution_major_version` | Мажорная версия        | `20`, `8`                       |
| `ansible_distribution_release`       | Релиз                  | `focal`, `bionic`               |
| `ansible_kernel`                     | Версия ядра            | `5.4.0-42-generic`              |
| `ansible_architecture`               | Архитектура            | `x86_64`, `aarch64`             |
| `ansible_userspace_architecture`     | Архитектура user-space | `x86_64`                        |
| `ansible_pkg_mgr`                    | Менеджер пакетов       | `apt`, `yum`, `dnf`             |
| `ansible_service_mgr`                | Менеджер сервисов      | `systemd`, `sysvinit`           |
| `ansible_selinux.status`             | Статус SELinux         | `enabled`, `disabled`           |

### ⚙️ Железо (Hardware)

| Fact                          | Описание               | Пример значения            |
| ----------------------------- | ---------------------- | -------------------------- |
| `ansible_memtotal_mb`         | Всего RAM (МБ)         | `8192`                     |
| `ansible_memfree_mb`          | Свободно RAM (МБ)      | `4096`                     |
| `ansible_swaptotal_mb`        | Всего swap (МБ)        | `2048`                     |
| `ansible_swapfree_mb`         | Свободно swap (МБ)     | `2048`                     |
| `ansible_processor_vcpus`     | Количество ядер CPU    | `4`                        |
| `ansible_processor_cores`     | Ядер на процессор      | `2`                        |
| `ansible_processor_count`     | Количество процессоров | `2`                        |
| `ansible_processor`           | Модели процессоров     | `['Intel(R) Core(TM)...']` |
| `ansible_machine_id`          | ID машины              | `abc123def456`             |
| `ansible_product_name`        | Название продукта      | `Virtual Machine`          |
| `ansible_virtualization_type` | Тип виртуализации      | `kvm`, `vmware`, `docker`  |
| `ansible_virtualization_role` | Роль (guest/host)      | `guest`                    |

### 📁 Файловая система

| Fact                               | Описание                  | Пример значения                       |
| ---------------------------------- | ------------------------- | ------------------------------------- |
| `ansible_mounts`                   | Список точек монтирования | `[{'mount': '/', 'size_total': ...}]` |
| `ansible_mounts[0].mount`          | Путь монтирования         | `/`                                   |
| `ansible_mounts[0].size_total`     | Размер (байты)            | `53687091200`                         |
| `ansible_mounts[0].size_available` | Свободно (байты)          | `26843545600`                         |
| `ansible_mounts[0].fstype`         | Тип ФС                    | `ext4`, `xfs`                         |
| `ansible_devices`                  | Блочные устройства        | `{'sda': {...}, 'sdb': {...}}`        |

### 🕒 Время и дата

| Fact                        | Описание        | Пример значения        |
| --------------------------- | --------------- | ---------------------- |
| `ansible_date_time.date`    | Дата            | `2024-01-15`           |
| `ansible_date_time.time`    | Время           | `14:30:45`             |
| `ansible_date_time.iso8601` | ISO 8601 формат | `2024-01-15T14:30:45Z` |
| `ansible_date_time.epoch`   | Unix timestamp  | `1705325445`           |
| `ansible_date_time.year`    | Год             | `2024`                 |
| `ansible_date_time.month`   | Месяц           | `01`                   |
| `ansible_date_time.day`     | День            | `15`                   |
| `ansible_date_time.hour`    | Час             | `14`                   |
| `ansible_date_time.minute`  | Минута          | `30`                   |
| `ansible_date_time.second`  | Секунда         | `45`                   |
| `ansible_date_time.weekday` | День недели     | `Monday`               |
| `ansible_date_time.tz`      | Часовой пояс    | `UTC`                  |

### 👤 Пользователи

| Fact                        | Описание              | Пример значения |
| --------------------------- | --------------------- | --------------- |
| `ansible_user_id`           | Текущий пользователь  | `root`          |
| `ansible_user_uid`          | UID пользователя      | `0`             |
| `ansible_user_gid`          | GID пользователя      | `0`             |
| `ansible_user_dir`          | Домашняя директория   | `/root`         |
| `ansible_user_shell`        | Оболочка пользователя | `/bin/bash`     |
| `ansible_effective_user_id` | Эффективный UID       | `0`             |

### 🐍 Python и окружение

| Fact                        | Описание                  | Пример значения    |
| --------------------------- | ------------------------- | ------------------ |
| `ansible_python_version`    | Версия Python             | `3.8.10`           |
| `ansible_python.executable` | Путь к Python             | `/usr/bin/python3` |
| `ansible_env.HOME`          | Переменная окружения HOME | `/root`            |
| `ansible_env.PATH`          | Переменная окружения PATH | `/usr/bin:/bin`    |
| `ansible_env.USER`          | Переменная окружения USER | `root`             |

---

## 4. Как использовать Facts в задачах

### В условиях (when)

```yaml
tasks:
  # Установка только на Ubuntu
  - name: Install Ubuntu package
    apt:
      name: software-properties-common
      state: present
    when: ansible_distribution == "Ubuntu"

  # Установка только на Debian/Ubuntu
  - name: Install Debian family package
    apt:
      name: htop
      state: present
    when: ansible_os_family == "Debian"

  # Установка только на серверах с 2GB+ RAM
  - name: Install PostgreSQL on powerful servers
    apt:
      name: postgresql
      state: present
    when: ansible_memtotal_mb >= 2048

  # Разные действия для разных версий ОС
  - name: Install Python 3.8 on Ubuntu 18.04
    apt:
      name: python3.8
      state: present
    when: ansible_distribution_version == "18.04"

  # Действие только на виртуальных машинах
  - name: Install VM tools
    apt:
      name: qemu-guest-agent
      state: present
    when: ansible_virtualization_type in ['kvm', 'vmware']
```

### В шаблонах (Jinja2)

```jinja2
# templates/nginx.conf.j2
server {
    listen 80;
    server_name {{ ansible_fqdn }};

    # Уникальный конфиг для каждого сервера
    access_log /var/log/nginx/{{ ansible_hostname }}_access.log;
    error_log /var/log/nginx/{{ ansible_hostname }}_error.log;

    # Балансировка на основе IP
    {% if ansible_default_ipv4.address == "192.168.1.10" %}
    location / {
        proxy_pass http://backend1:8080;
    }
    {% else %}
    location / {
        proxy_pass http://backend2:8080;
    }
    {% endif %}
}
```

### В переменных

```yaml
vars:
  # Динамическое имя файла
  backup_filename: "backup_{{ ansible_hostname }}_{{ ansible_date_time.date }}.tar.gz"

  # Путь на основе архитектуры
  app_path: "/opt/app/{{ ansible_architecture }}"

  # Настройки на основе памяти
  workers: "{{ (ansible_memtotal_mb / 1024) | int }}"
```

### В циклах

```yaml
tasks:
  - name: Show info for all interfaces
    debug:
      msg: "Interface {{ item }}: {{ hostvars[inventory_hostname]['ansible_' + item]['ipv4']['address'] }}"
    loop: "{{ ansible_interfaces }}"
    when: hostvars[inventory_hostname]['ansible_' + item]['ipv4'] is defined
```

---

## 5. Кастомные Facts (set_fact)

### Модуль set_fact

Позволяет создавать свои переменные на основе facts или вычислений:

```yaml
tasks:
  # Простое присваивание
  - name: Set custom fact
    set_fact:
      app_port: 8080
      app_env: "production"

  # Вычисление на основе facts
  - name: Calculate memory in GB
    set_fact:
      memory_gb: "{{ (ansible_memtotal_mb / 1024) | round(2) }}"

  # Условное присваивание
  - name: Set OS-specific package manager
    set_fact:
      pkg_mgr: "apt"
    when: ansible_os_family == "Debian"

  - name: Set OS-specific package manager (RedHat)
    set_fact:
      pkg_mgr: "yum"
    when: ansible_os_family == "RedHat"

  # Использование в следующих задачах
  - name: Use custom fact
    debug:
      msg: "Memory: {{ memory_gb }} GB, Package manager: {{ pkg_mgr }}"
```

### Кастомные facts из файлов

Можно размещать кастомные facts в `/etc/ansible/facts.d/` на целевых серверах:

```bash
# На целевом сервере
sudo mkdir -p /etc/ansible/facts.d
sudo nano /etc/ansible/facts.d/custom.fact
```

**Содержимое `/etc/ansible/facts.d/custom.fact`:**

```json
{
  "app_version": "2.0.0",
  "app_config": {
    "port": 8080,
    "workers": 4
  },
  "features": ["ssl", "cache"]
}
```

**Использование в playbook:**

```yaml
tasks:
  - name: Gather custom facts
    setup:
      filter: ansible_local

  - name: Use custom fact
    debug:
      msg: "App version: {{ ansible_local.custom.app_version }}"

  - name: Use nested custom fact
    debug:
      msg: "Port: {{ ansible_local.custom.app_config.port }}"
```

### Динамические facts из команд

```yaml
tasks:
  # Выполнить команду и сохранить результат
  - name: Get disk usage
    shell: df -h / | awk 'NR==2 {print $5}' | sed 's/%//'
    register: disk_usage_result
    changed_when: false

  - name: Set fact from command result
    set_fact:
      disk_usage_percent: "{{ disk_usage_result.stdout | int }}"

  - name: Alert if disk is almost full
    debug:
      msg: "WARNING: Disk usage is {{ disk_usage_percent }}%"
    when: disk_usage_percent > 80
```

---

## 6. Отключение и кэширование Facts

### Отключение сбора facts

Если facts не нужны, можно отключить их сбор для ускорения:

```yaml
---
- hosts: webservers
  gather_facts: no # Отключаем сбор facts

  tasks:
    - name: Simple task without facts
      copy:
        src: file.txt
        dest: /tmp/
```

**Когда отключать:**

- ✅ Простые задачи (копирование файлов, перезапуск сервисов)
- ✅ Когда не нужны данные о системе
- ✅ Для ускорения выполнения

**Когда НЕ отключать:**

- ❌ Если используешь `when: ansible_os_family == ...`
- ❌ Если используешь шаблоны с `{{ ansible_hostname }}`
- ❌ Если нужна информация о системе

### Частичный сбор facts

Можно собирать только нужные facts:

```yaml
---
- hosts: webservers
  gather_facts: yes

  tasks:
    - name: Gather only specific facts
      setup:
        filter:
          - ansible_memtotal_mb
          - ansible_os_family
          - ansible_hostname
```

### Кэширование facts

Для ускорения можно кэшировать facts:

**В `ansible.cfg`:**

```ini
[defaults]
gathering = smart
fact_caching = memory
fact_caching_timeout = 86400  # 24 часа

# Или с сохранением на диск
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 3600  # 1 час
```

**Типы кэширования:**

- `memory` — в памяти (быстро, но теряется при перезапуске)
- `jsonfile` — в JSON файлах на диске
- `redis` — в Redis
- `memcached` — в Memcached

---

## 7. Практические примеры

### Пример 1: Адаптивная установка пакетов

```yaml
---
- hosts: all
  become: yes

  tasks:
    - name: Install packages based on OS
      package:
        name: "{{ item }}"
        state: present
      loop:
        - htop
        - curl
        - git

    - name: Install OS-specific packages
      package:
        name: "{{ item }}"
        state: present
      loop:
        - { name: "software-properties-common", os: "Debian" }
        - { name: "epel-release", os: "RedHat" }
      when: item.os == ansible_os_family
```

### Пример 2: Генерация уникального конфига

```yaml
---
- hosts: webservers
  become: yes
  vars:
    app_name: myapp

  tasks:
    - name: Deploy app config
      template:
        src: templates/app.conf.j2
        dest: "/etc/{{ app_name }}/app.conf"
      notify: Restart App

  handlers:
    - name: Restart App
      systemd:
        name: "{{ app_name }}"
        state: restarted
```

**`templates/app.conf.j2`:**

```jinja2
# Configuration for {{ ansible_hostname }}
# Generated on {{ ansible_date_time.iso8601 }}

[server]
hostname = {{ ansible_fqdn }}
ip_address = {{ ansible_default_ipv4.address }}
port = 8080

[resources]
memory_total = {{ ansible_memtotal_mb }} MB
cpu_cores = {{ ansible_processor_vcpus }}

[environment]
os = {{ ansible_distribution }} {{ ansible_distribution_version }}
kernel = {{ ansible_kernel }}
architecture = {{ ansible_architecture }}
```

### Пример 3: Мониторинг ресурсов

```yaml
---
- hosts: all
  tasks:
    - name: Check memory usage
      set_fact:
        memory_used_percent: "{{ ((ansible_memtotal_mb - ansible_memfree_mb) / ansible_memtotal_mb * 100) | round(2) }}"

    - name: Check disk usage
      shell: df -h / | awk 'NR==2 {print $5}' | sed 's/%//'
      register: disk_usage
      changed_when: false

    - name: Show system status
      debug:
        msg: |
          Host: {{ ansible_hostname }}
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
          Memory: {{ ansible_memfree_mb }} MB free / {{ ansible_memtotal_mb }} MB total ({{ memory_used_percent }}% used)
          Disk: {{ disk_usage.stdout }}% used
          CPU: {{ ansible_processor_vcpus }} cores

    - name: Alert if memory is low
      debug:
        msg: "WARNING: Low memory on {{ ansible_hostname }}!"
      when: ansible_memfree_mb < 512
```

### Пример 4: Настройка swap на основе RAM

```yaml
---
- hosts: all
  become: yes

  tasks:
    - name: Calculate swap size (2x RAM for <4GB, 1x RAM for >=4GB)
      set_fact:
        swap_size_mb: "{{ (ansible_memtotal_mb * 2) if ansible_memtotal_mb < 4096 else ansible_memtotal_mb }}"

    - name: Check if swap exists
      command: swapon --show
      register: swap_check
      changed_when: false

    - name: Create swap file
      command: "fallocate -l {{ swap_size_mb }}M /swapfile"
      when: swap_check.stdout == ""

    - name: Set swap permissions
      file:
        path: /swapfile
        mode: "0600"
      when: swap_check.stdout == ""

    - name: Format swap
      command: mkswap /swapfile
      when: swap_check.stdout == ""

    - name: Enable swap
      command: swapon /swapfile
      when: swap_check.stdout == ""

    - name: Add swap to fstab
      lineinfile:
        path: /etc/fstab
        line: "/swapfile none swap sw 0 0"
        state: present
      when: swap_check.stdout == ""
```

### Пример 5: Динамический inventory на основе facts

```yaml
---
- hosts: all
  tasks:
    - name: Group servers by OS
      group_by:
        key: "os_{{ ansible_os_family | lower }}"

    - name: Group servers by memory
      group_by:
        key: "mem_{{ 'high' if ansible_memtotal_mb >= 8192 else 'low' }}"

    - name: Group servers by virtualization
      group_by:
        key: "virt_{{ ansible_virtualization_type | default('physical') }}"

- hosts: os_debian
  tasks:
    - name: Tasks for Debian servers
      debug:
        msg: "This is a Debian server: {{ ansible_hostname }}"

- hosts: mem_high
  tasks:
    - name: Tasks for high-memory servers
      debug:
        msg: "High memory server: {{ ansible_hostname }} ({{ ansible_memtotal_mb }} MB)"
```

---

## 8. Best Practices

### ✅ Хорошие практики

1. **Используй специфичные facts вместо shell-команд**
   - ✅ `when: ansible_os_family == "Debian"`
   - ❌ `shell: cat /etc/os-release | grep Ubuntu`

2. **Отключай gather_facts, если они не нужны**
   - Ускоряет выполнение playbook

3. **Используй кастомные facts для сложных вычислений**
   - `set_fact` для сохранения промежуточных результатов

4. **Документируй кастомные facts**
   - Комментарии в `facts.d/*.fact`

5. **Используй `changed_when: false` для команд, собирающих facts**
   - Избегай ложных `CHANGED` статусов

6. **Проверяй существование facts перед использованием**
   - `when: ansible_eth0 is defined`

7. **Используй фильтры для обработки facts**
   - `{{ ansible_memtotal_mb | human_readable }}`

### ❌ Антипаттерны

1. **Использование shell для получения информации, которую можно получить через facts**
   - ❌ `shell: hostname`
   - ✅ `{{ ansible_hostname }}`

2. **Сбор всех facts, когда нужны только некоторые**
   - Используй `filter` для точечного сбора

3. **Игнорирование кэширования facts**
   - Для больших инфраструктур кэширование критично

4. **Хардкодинг значений вместо использования facts**
   - ❌ `port: 80` (если порт зависит от ОС)
   - ✅ `port: "{{ 80 if ansible_os_family == 'Debian' else 8080 }}"`

5. **Использование facts без проверки их существования**
   - Некоторые facts могут отсутствовать (например, `ansible_eth1` если нет второго интерфейса)

---

## 9. Вопросы для собеседования

### Базовые вопросы

**Q: Что такое Facts в Ansible?**
A: Facts — это переменные, которые Ansible автоматически собирает с целевого сервера перед выполнением задач. Они содержат информацию о системе: ОС, память, CPU, сеть и т.д.

**Q: Как собрать facts вручную?**
A: Через ad-hoc команду: `ansible <host> -m setup`. Или в playbook через модуль `setup`.

**Q: Как отключить сбор facts?**
A: Указать `gather_facts: no` на уровне play.

**Q: Какие самые популярные facts?**
A: `ansible_hostname`, `ansible_os_family`, `ansible_distribution`, `ansible_memtotal_mb`, `ansible_default_ipv4.address`, `ansible_processor_vcpus`.

**Q: Как использовать facts в условиях?**
A: В блоке `when`: `when: ansible_os_family == "Debian"`.

### Продвинутые вопросы

**Q: Как создать кастомные facts?**
A: Два способа: 1) Использовать модуль `set_fact` в задачах. 2) Разместить JSON файлы в `/etc/ansible/facts.d/` на целевых серверах.

**Q: Как кэшировать facts?**
A: В `ansible.cfg` указать `fact_caching = memory` (или `jsonfile`, `redis`) и `fact_caching_timeout`.

**Q: Что будет, если использовать fact, которого не существует?**
A: Playbook упадет с ошибкой `undefined variable`. Нужно проверять существование: `when: ansible_eth1 is defined`.

**Q: Как собрать только определенные facts?**
A: Использовать `filter` в модуле setup: `setup: filter: ansible_memtotal_mb`.

**Q: В чем разница между `ansible_hostname` и `ansible_fqdn`?**
A: `ansible_hostname` — короткое имя (например, `web1`), `ansible_fqdn` — полное доменное имя (например, `web1.example.com`).

**Q: Как использовать facts для группировки хостов?**
A: Использовать модуль `group_by`: `group_by: key: "os_{{ ansible_os_family | lower }}"`.

**Q: Можно ли использовать facts в шаблонах Jinja2?**
A: Да, facts доступны как обычные переменные: `{{ ansible_hostname }}`, `{{ ansible_memtotal_mb }}`.

**Q: Как получить facts другого хоста?**
A: Через `hostvars`: `{{ hostvars['web1']['ansible_default_ipv4']['address'] }}`.

**Q: Что такое `ansible_local`?**
A: Это словарь с кастомными facts из `/etc/ansible/facts.d/`. Пример: `{{ ansible_local.custom.app_version }}`.

**Q: Как ускорить playbook, если facts не нужны?**
A: Указать `gather_facts: no` на уровне play.

**Q: Как проверить, какие facts собраны?**
A: Использовать модуль `debug`: `debug: var=ansible_facts` или команду `ansible <host> -m setup`.

**Q: Можно ли обновить facts во время выполнения playbook?**
A: Да, вызвать модуль `setup` повторно: `setup:` в задачах.

**Q: Что делать, если fact возвращает неправильное значение?**
A: Проверить, не переопределена ли переменная в inventory или playbook. Использовать `set_fact` для переопределения.

---

## 📚 Дополнительные ресурсы

- Официальная документация: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#variables-discovered-from-systems-facts
- Setup module: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/setup_module.html
- Magic Variables: https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html

---
