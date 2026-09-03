## 📋 Оглавление

1. [Архитектура Ansible](#1-архитектура-ansible)
2. [Ключевые концепции](#2-ключевые-концепции)
3. [Инвентарь (Inventory)](#3-инвентарь-inventory)
4. [Модули](#4-модули)
5. [Подключение и аутентификация](#5-подключение-и-аутентификация)
6. [Конфигурационный файл (ansible.cfg)](#6-конфигурационный-файл-ansiblecfg)
7. [Практические примеры](#7-практические-примеры)
8. [Вопросы для собеседования](#8-вопросы-для-собеседования)

---

## 1. Архитектура Ansible

### Компоненты системы

**Control Node (Управляющая машина):**

- Машина, на которой установлен Ansible
- Отсюда запускаются плейбуки и ad-hoc команды
- Требования: Python 3.8+, Linux/macOS (Windows не поддерживается как Control Node)
- Хранит инвентарь, плейбуки, роли

**Managed Nodes (Целевые серверы):**

- Серверы, которыми управляет Ansible
- Требования: Python 2.7+ или Python 3.5+, SSH-сервер
- НЕ требуют установки агентов (agentless)

**Inventory (Инвентарь):**

- Файл со списком целевых серверов
- Поддерживает INI и YAML форматы
- Может быть статичным или динамическим

### Как работает Ansible (пошагово)

┌─────────────────────────────────────────────────────────────┐
│ 1. Ansible читает inventory и определяет целевые хосты │
├─────────────────────────────────────────────────────────────┤
│ 2. Подключается к хостам по SSH (параллельно) │
├─────────────────────────────────────────────────────────────┤
│ 3. Загружает Python-скрипт (код модуля) на целевой сервер │
├─────────────────────────────────────────────────────────────┤
│ 4. Выполняет скрипт на целевом сервере │
├─────────────────────────────────────────────────────────────┤
│ 5. Получает результат в формате JSON │
├─────────────────────────────────────────────────────────────┤
│ 6. Удаляет временный скрипт с целевого сервера │
├─────────────────────────────────────────────────────────────┤
│ 7. Выводит результат на Control Node │
└─────────────────────────────────────────────────────────────┘

### Поток данных

```
Control Node                    Managed Node
    │                               │
    │── SSH connection ────────────►│
    │                               │
    │── Upload module (Python) ────►│
    │                               │
    │                               │── Execute module
    │                               │
    │◄── Return JSON result ────────│
    │                               │
    │── Delete temp files ─────────►│
    │                               │
    │── Close SSH connection ──────►│
```

---

## 2. Ключевые концепции

### 🤖 Agentless (Без агентов)

**Что это:** Ansible не требует установки специального программного обеспечения (агентов) на целевых серверах.

**Как работает:**

- Использует стандартный SSH для подключения
- Временные Python-скрипты загружаются на лету и удаляются после выполнения
- Нет фоновых процессов, которые нужно обслуживать

**Преимущества:**

- ✅ Нет дополнительной нагрузки на сервер в простое
- ✅ Не нужно обновлять агенты на сотнях серверов
- ✅ Меньше точек отказа
- ✅ Быстрый старт (не нужно настраивать агентов)

**Недостатки:**

- ❌ Требует Python на целевых серверах
- ❌ Зависит от SSH (если SSH недоступен — Ansible не работает)
- ❌ Каждый запуск требует передачи кода модуля по сети

**Сравнение с агентовыми системами:**

| Характеристика       | Ansible (Agentless) | Puppet/Chef (Agent-based) |
| -------------------- | ------------------- | ------------------------- |
| Установка на клиенте | Не требуется        | Требуется агент           |
| Фоновые процессы     | Нет                 | Есть (демон)              |
| Обновление           | Не нужно            | Нужно обновлять агенты    |
| Нагрузка в простое   | Нулевая             | Постоянная                |
| Начальная настройка  | Быстрая             | Требует времени           |

---

### 📝 Декларативный подход

**Что это:** Ты описываешь **желаемое состояние** системы, а не пошаговые инструкции для его достижения.

**Сравнение подходов:**

**Императивный (Bash-скрипт):**

```bash
# Как сделать (шаги)
if ! dpkg -l | grep -q nginx; then
    apt-get update
    apt-get install -y nginx
fi

if ! systemctl is-active --quiet nginx; then
    systemctl start nginx
fi

if ! systemctl is-enabled --quiet nginx; then
    systemctl enable nginx
fi
```

**Декларативный (Ansible):**

```yaml
# Что хотим получить (состояние)
- name: Ensure Nginx is installed and running
  apt:
    name: nginx
    state: present

- name: Ensure Nginx is started and enabled
  systemd:
    name: nginx
    state: started
    enabled: yes
```

**Преимущества декларативного подхода:**

- ✅ Код читается как документация
- ✅ Идемпотентность "из коробки"
- ✅ Меньше кода — меньше ошибок
- ✅ Ansible сам решает, какие шаги нужны

---

### 🔄 Идемпотентность (САМОЕ ВАЖНОЕ!)

**Что это:** Результат выполнения операции не зависит от того, сколько раз она была выполнена. Повторный запуск приводит систему к тому же состоянию, что и первый запуск.

**Формула:** `f(f(x)) = f(x)`

**Примеры:**

✅ **Идемпотентные операции:**

- "Файл `/etc/config.conf` должен содержать строку `port=8080`"
  - 1-й запуск: строки нет → добавляем → `CHANGED`
  - 2-й запуск: строка есть → ничего не делаем → `OK`
- "Пакет `nginx` должен быть установлен"
  - 1-й запуск: пакета нет → устанавливаем → `CHANGED`
  - 2-й запуск: пакет есть → ничего не делаем → `OK`

❌ **Не идемпотентные операции:**

- "Добавить строку `port=8080` в файл" (без проверки существования)
  - 1-й запуск: добавляем строку
  - 2-й запуск: добавляем ЕЩЁ одну такую же строку
  - 3-й запуск: добавляем ТРЕТЬЮ строку
  - Результат: файл засорен дубликатами

**Как Ansible обеспечивает идемпотентность:**

1. **Проверка текущего состояния:** Перед выполнением задачи Ansible проверяет, нужно ли вообще что-то делать
2. **Сравнение состояний:** Сравнивает желаемое состояние с текущим
3. **Возврат статуса:**
   - `OK` — состояние уже соответствует желаемому (ничего не делаем)
   - `CHANGED` — состояние изменилось (применили изменения)
   - `FAILED` — ошибка при применении
   - `SKIPPED` — задача пропущена (из-за условия `when`)

**Примеры идемпотентных модулей:**

| Модуль        | Что проверяет                                    |
| ------------- | ------------------------------------------------ |
| `apt` / `yum` | Установлен ли пакет нужной версии                |
| `copy`        | Совпадает ли контрольная сумма файла             |
| `template`    | Совпадает ли содержимое после рендеринга         |
| `file`        | Существует ли файл/директория с нужными правами  |
| `systemd`     | Запущен ли сервис, включен ли в автозагрузку     |
| `user`        | Существует ли пользователь с нужными параметрами |

**Антипаттерн: использование `shell`/`command`**

```yaml
# ❌ ПЛОХО: Не идемпотентно
- name: Add line to config
  shell: echo "port=8080" >> /etc/app.conf
  # Каждый запуск добавляет новую строку!

# ✅ ХОРОШО: Идемпотентно
- name: Ensure port is set in config
  lineinfile:
    path: /etc/app.conf
    line: "port=8080"
    state: present
  # Проверяет, есть ли строка, и добавляет только если её нет
```

---

### 📦 Модули

**Что это:** Готовые переиспользуемые единицы кода, которые выполняют конкретные задачи.

**Категории модулей:**

| Категория        | Примеры                                  | Назначение                              |
| ---------------- | ---------------------------------------- | --------------------------------------- |
| **Системные**    | `apt`, `yum`, `service`, `systemd`       | Управление пакетами и сервисами         |
| **Файловые**     | `copy`, `file`, `template`, `lineinfile` | Работа с файлами и директориями         |
| **Сетевые**      | `uri`, `get_url`                         | Работа с HTTP, скачивание файлов        |
| **Пользователи** | `user`, `group`, `authorized_key`        | Управление пользователями и SSH-ключами |
| **Базы данных**  | `postgresql_db`, `mysql_db`              | Управление БД                           |
| **Облачные**     | `ec2`, `azure_rm_vm`                     | Управление облачными ресурсами          |
| **Контейнеры**   | `docker_container`, `k8s`                | Работа с Docker и Kubernetes            |

**Как использовать модули:**

```yaml
- name: Install package
  apt: # <-- Это модуль
    name: nginx # <-- Параметры модуля
    state: present
```

**Золотое правило:** Всегда используй специализированный модуль, если он существует. Используй `shell`/`command` только в крайних случаях.

---

## 3. Инвентарь (Inventory)

**Что это:** Файл со списком серверов, которыми управляет Ansible.

### Формат INI

```ini
# Простой список хостов
web1.example.com
web2.example.com

# Группы хостов
[webservers]
web1.example.com
web2.example.com

[dbservers]
db1.example.com
db2.example.com

# Переменные для конкретного хоста
[webservers]
web1.example.com ansible_user=admin ansible_port=2222
web2.example.com ansible_user=deploy

# Переменные для группы
[webservers:vars]
ansible_python_interpreter=/usr/bin/python3
ntp_server=ntp.example.com

# Вложенные группы
[production:children]
webservers
dbservers

[production:vars]
env=production
```

### Формат YAML (рекомендуемый)

```yaml
all:
  children:
    webservers:
      hosts:
        web1.example.com:
          ansible_user: admin
          ansible_port: 2222
        web2.example.com:
          ansible_user: deploy
      vars:
        ansible_python_interpreter: /usr/bin/python3
        ntp_server: ntp.example.com

    dbservers:
      hosts:
        db1.example.com:
        db2.example.com:

    production:
      children:
        webservers:
        dbservers:
      vars:
        env: production
```

### Динамический инвентарь

Ansible может получать список хостов из внешних источников (AWS, GCP, VMware и т.д.) через скрипты или плагины.

```bash
# Пример использования динамического инвентаря
ansible-playbook -i aws_ec2.yml playbook.yml
```

**Пример `aws_ec2.yml`:**

```yaml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
  - us-west-2
filters:
  tag:Environment: production
keyed_groups:
  - key: tags.Role
    prefix: role
```

---

## 4. Модули

### Популярные модули с примерами

#### 📦 Управление пакетами

**APT (Debian/Ubuntu):**

```yaml
- name: Install nginx
  apt:
    name: nginx
    state: present # present, absent, latest
    update_cache: yes # Выполнить apt update
    cache_valid_time: 3600 # Кэш валиден 1 час
```

**YUM (RHEL/CentOS):**

```yaml
- name: Install httpd
  yum:
    name: httpd
    state: present
    enablerepo: epel # Включить дополнительный репозиторий
```

**Универсальный модуль PACKAGE:**

```yaml
- name: Install package (auto-detect package manager)
  package:
    name: nginx
    state: present
  # Сам выберет apt, yum, dnf в зависимости от ОС
```

---

#### 📄 Работа с файлами

**COPY (копирование файлов):**

```yaml
- name: Copy file
  copy:
    src: files/config.conf # Путь на Control Node
    dest: /etc/app/config.conf # Путь на Managed Node
    owner: root
    group: root
    mode: "0644"
    backup: yes # Создать бэкап перед заменой
```

**TEMPLATE (шаблоны с переменными):**

```yaml
- name: Deploy config from template
  template:
    src: templates/app.conf.j2
    dest: /etc/app/app.conf
    mode: "0644"
  # Подставляет переменные {{ var }} из Jinja2
```

**FILE (управление файлами/директориями):**

```yaml
- name: Create directory
  file:
    path: /opt/app
    state: directory
    owner: appuser
    mode: "0755"

- name: Remove file
  file:
    path: /tmp/old.log
    state: absent

- name: Create symlink
  file:
    src: /etc/app/config.conf
    dest: /opt/app/config.conf
    state: link
```

**LINEINFILE (работа со строками в файлах):**

```yaml
- name: Ensure line exists in file
  lineinfile:
    path: /etc/hosts
    line: "192.168.1.10 server1"
    state: present

- name: Replace line using regexp
  lineinfile:
    path: /etc/app.conf
    regexp: "^port="
    line: "port=8080"
```

---

#### 🔧 Управление сервисами

**SYSTEMD (современные Linux):**

```yaml
- name: Start and enable service
  systemd:
    name: nginx
    state: started # started, stopped, restarted, reloaded
    enabled: yes # Добавить в автозагрузку
    daemon_reload: yes # Выполнить systemctl daemon-reload
```

**SERVICE (универсальный):**

```yaml
- name: Restart service
  service:
    name: nginx
    state: restarted
```

---

#### 👥 Пользователи и группы

**USER:**

```yaml
- name: Create user
  user:
    name: deploy
    shell: /bin/bash
    groups: sudo,docker
    append: yes # Добавить в группы, не удаляя из других
    create_home: yes
    state: present
```

**AUTHORIZED_KEY (SSH-ключи):**

```yaml
- name: Add SSH key
  authorized_key:
    user: deploy
    state: present
    key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
```

---

#### 🌐 Сетевые операции

**GET_URL (скачивание файлов):**

```yaml
- name: Download file
  get_url:
    url: https://example.com/app.tar.gz
    dest: /opt/app.tar.gz
    mode: "0644"
    checksum: sha256:abc123... # Проверка контрольной суммы
```

**URI (HTTP-запросы):**

```yaml
- name: Make HTTP request
  uri:
    url: https://api.example.com/health
    method: GET
    return_content: yes
  register: result

- name: Show response
  debug:
    var: result.content
```

---

## 5. Подключение и аутентификация

### SSH-ключи (рекомендуемый способ)

**Генерация ключа на Control Node:**

```bash
ssh-keygen -t ed25519 -C "ansible@control"
# Или для старых систем:
ssh-keygen -t rsa -b 4096 -C "ansible@control"
```

**Копирование ключа на Managed Nodes:**

```bash
# Вручную
ssh-copy-id user@managed-node

# Или через Ansible
ansible all -m authorized_key -a "user=root key='{{ lookup('file', '~/.ssh/id_rsa.pub') }}'" -k
# Флаг -k запросит пароль для первичного подключения
```

### SSH-конфигурация

**Файл `~/.ssh/config` на Control Node:**

```
Host *.example.com
    User ansible
    IdentityFile ~/.ssh/ansible_key
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null

Host db*
    User postgres
    Port 2222
```

### Переменные подключения в inventory

```ini
[webservers]
web1.example.com
  ansible_user=admin
  ansible_port=22
  ansible_ssh_private_key_file=~/.ssh/ansible_key
  ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

### Become (sudo)

**Глобально для Play:**

```yaml
- hosts: webservers
  become: yes # Выполнять все задачи с sudo
  become_user: root # От какого пользователя (по умолчанию root)
  become_method: sudo # Метод (sudo, su, doas и т.д.)
```

**Для конкретной задачи:**

```yaml
- name: Install package
  apt:
    name: nginx
    state: present
  become: yes
```

---

## 6. Конфигурационный файл (ansible.cfg)

Ansible ищет конфигурацию в следующем порядке:

1. `ANSIBLE_CONFIG` (переменная окружения)
2. `./ansible.cfg` (в текущей директории)
3. `~/.ansible.cfg` (в домашней директории)
4. `/etc/ansible/ansible.cfg` (системный)

**Пример `ansible.cfg`:**

```ini
[defaults]
# Путь к инвентарю по умолчанию
inventory = ./inventory.yml

# Отключение сбора фактов (ускоряет выполнение)
gathering = smart
fact_caching = memory
fact_caching_timeout = 86400

# Параллелизм
forks = 20

# Таймауты
timeout = 30

# Логи
log_path = ./ansible.log

# Роли
roles_path = ./roles

# Отключение предупреждений
deprecation_warnings = False

# Цветной вывод
force_color = True

[privilege_escalation]
# Настройки sudo по умолчанию
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
# Ускорение SSH
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
control_path_dir = /tmp/.ansible/cp
```

---

## 7. Практические примеры

### Пример 1: Проверка инфраструктуры

```yaml
---
- hosts: all
  gather_facts: yes

  tasks:
    - name: Show OS info
      debug:
        msg: "Host: {{ ansible_hostname }}, OS: {{ ansible_distribution }} {{ ansible_distribution_version }}"

    - name: Show memory
      debug:
        msg: "RAM: {{ ansible_memtotal_mb }} MB, Free: {{ ansible_memfree_mb }} MB"

    - name: Show disk space
      command: df -h /
      register: disk_info
      changed_when: false

    - name: Display disk info
      debug:
        var: disk_info.stdout_lines
```

### Пример 2: Базовая настройка сервера

```yaml
---
- hosts: webservers
  become: yes

  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install basic packages
      apt:
        name:
          - htop
          - curl
          - git
          - vim
        state: present

    - name: Create deploy user
      user:
        name: deploy
        shell: /bin/bash
        groups: sudo
        append: yes
        create_home: yes

    - name: Add SSH key for deploy user
      authorized_key:
        user: deploy
        state: present
        key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"

    - name: Configure sudoers
      lineinfile:
        path: /etc/sudoers
        line: "deploy ALL=(ALL) NOPASSWD:ALL"
        validate: "visudo -cf %s"

    - name: Set timezone
      timezone:
        name: Europe/Moscow
```

### Пример 3: Установка и настройка Nginx

```yaml
---
- hosts: webservers
  become: yes

  vars:
    nginx_port: 80
    server_name: "{{ ansible_fqdn }}"

  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Deploy Nginx config
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/sites-available/default
      notify: Reload Nginx

    - name: Enable site
      file:
        src: /etc/nginx/sites-available/default
        dest: /etc/nginx/sites-enabled/default
        state: link
      notify: Reload Nginx

    - name: Start and enable Nginx
      systemd:
        name: nginx
        state: started
        enabled: yes

  handlers:
    - name: Reload Nginx
      systemd:
        name: nginx
        state: reloaded
```

---

## 8. Вопросы для собеседования

### Базовые вопросы

**Q: Что такое Ansible и чем он отличается от Puppet/Chef?**
A: Ansible — это инструмент управления конфигурациями без агентов (agentless). В отличие от Puppet/Chef, которые требуют установки агентов на целевых серверах, Ansible использует SSH и временные Python-скрипты.

**Q: Что такое идемпотентность?**
A: Идемпотентность — это свойство операции, при котором повторное выполнение не изменяет результат. В Ansible это означает, что повторный запуск плейбука не ломает систему, а приводит её к желаемому состоянию.

**Q: В чем разница между декларативным и императивным подходом?**
A: Декларативный подход описывает желаемое состояние ("nginx должен быть установлен"), а императивный — шаги для его достижения ("скачай пакет, установи его, проверь код возврата"). Ansible использует декларативный подход.

**Q: Что такое модули в Ansible?**
A: Модули — это переиспользуемые единицы кода, которые выполняют конкретные задачи (установка пакетов, копирование файлов, управление сервисами и т.д.).

**Q: Зачем нужен inventory?**
A: Inventory — это файл со списком серверов, которыми управляет Ansible. Он позволяет группировать хосты и задавать для них переменные.

### Продвинутые вопросы

**Q: Как Ansible обеспечивает идемпотентность?**
A: Ansible проверяет текущее состояние перед выполнением задачи и сравнивает его с желаемым. Если состояние уже соответствует, задача возвращает `OK` и ничего не меняет.

**Q: Почему не рекомендуется использовать модуль `shell`?**
A: Модуль `shell` не идемпотентен по умолчанию и зависит от оболочки. Всегда лучше использовать специализированные модули (`apt`, `copy`, `systemd`), которые гарантируют идемпотентность.

**Q: Как работает подключение по SSH в Ansible?**
A: Ansible подключается по SSH, загружает Python-скрипт (код модуля) на целевой сервер, выполняет его, получает результат в JSON и удаляет временные файлы.

**Q: Что такое `become` и зачем он нужен?**
A: `become` позволяет выполнять задачи с правами другого пользователя (обычно root через sudo). Это нужно для операций, требующих привилегий (установка пакетов, изменение системных конфигов).

**Q: Как ускорить выполнение Ansible playbook?**
A:

- Включить pipelining в `ansible.cfg`
- Увеличить `forks` для параллельного выполнения
- Отключить сбор фактов, если они не нужны (`gather_facts: no`)
- Использовать кэширование фактов
- Использовать SSH ControlMaster для переиспользования соединений

---

## 📚 Дополнительные ресурсы

- Официальная документация: https://docs.ansible.com
- Модули: https://docs.ansible.com/ansible/latest/collections/index.html
- Best Practices: https://docs.ansible.com/ansible/latest/tips_tricks/ansible_tips_tricks.html

---
