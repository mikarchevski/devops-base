# 🎯 Ansible Variables & Precedence: Детальная шпаргалка

## 📋 Оглавление

1. [Что такое переменные в Ansible](#1-что-такое-переменные-в-ansible)
2. [Где можно объявлять переменные](#2-где-можно-объявлять-переменные)
3. [Иерархия приоритетов](#3-иерархия-приоритетов)
4. [Детальный разбор каждого уровня](#4-детальный-разбор-каждого-уровня)
5. [Типы данных переменных](#5-типы-данных-переменных)
6. [Практические примеры](#6-практические-примеры)
7. [Best Practices](#7-best-practices)
8. [Вопросы для собеседования](#8-вопросы-для-собеседования)

---

## 1. Что такое переменные в Ansible

**Переменные** — это именованные значения, которые можно использовать в playbook'ах, ролях, шаблонах и условиях для параметризации и гибкости.

**Зачем нужны:**

- ✅ Параметризация конфигурации (порты, пути, версии)
- ✅ Разделение данных и логики
- ✅ Переиспользование кода для разных окружений
- ✅ Динамическая генерация конфигов через Jinja2

**Синтаксис использования:**

```yaml
# В задачах
- name: Start app on port {{ app_port }}
  systemd:
    name: myapp
    state: started

# В шаблонах (Jinja2)
listen {{ app_port }};
server_name {{ ansible_fqdn }};

# В условиях
when: env == "production"

# В циклах
loop: "{{ packages }}"
```

---

## 2. Где можно объявлять переменные

### 📍 Места объявления (от низшего приоритета к высшему)

Role Defaults (roles/x/defaults/main.yml)
↓
Inventory Vars (group_vars/, host_vars/)
↓
Playbook Vars (блок vars:, vars_files:)
↓
Role Vars (roles/x/vars/main.yml)
↓
Extra Vars (флаг -e при запуске)

### Визуальная схема

```
┌─────────────────────────────────────────────────────────┐
│  Extra Vars (-e)                                        │
│  ⚫ ПЕРЕБИВАЕТ ВСЁ!                                     │
└─────────────────────────────────────────────────────────┘
                          ↑
┌─────────────────────────────────────────────────────────┐
│  Role Vars (roles/x/vars/main.yml)                      │
│  🔴 Высокий приоритет                                   │
└─────────────────────────────────────────────────────────┘
                          ↑
┌─────────────────────────────────────────────────────────┐
│  Playbook Vars (vars:, vars_files:)                     │
│  🟠 Средний приоритет                                   │
└─────────────────────────────────────────────────────────┘
                          ↑
┌─────────────────────────────────────────────────────────┐
│  Inventory Vars (group_vars/, host_vars/)               │
│  🟡 Низкий приоритет                                    │
└─────────────────────────────────────────────────────────┘
                          ↑
┌─────────────────────────────────────────────────────────┐
│  Role Defaults (roles/x/defaults/main.yml)              │
│  🟢 САМЫЙ НИЗКИЙ (легко перебить)                      │
└─────────────────────────────────────────────────────────┘
```

---

## 3. Иерархия приоритетов

### Полная таблица приоритетов

| Приоритет      | Уровень              | Где объявляется             | Пример                           |
| -------------- | -------------------- | --------------------------- | -------------------------------- |
| **1 (низший)** | Role Defaults        | `roles/x/defaults/main.yml` | Значения по умолчанию для роли   |
| **2**          | Inventory Group Vars | `group_vars/webservers.yml` | Переменные для группы хостов     |
| **3**          | Inventory Host Vars  | `host_vars/web1.yml`        | Переменные для конкретного хоста |
| **4**          | Playbook Vars        | блок `vars:` в playbook     | Глобальные переменные play       |
| **5**          | Playbook Vars Files  | `vars_files: [secrets.yml]` | Подключенные файлы переменных    |
| **6**          | Role Vars            | `roles/x/vars/main.yml`     | Внутренние переменные роли       |
| **7 (высший)** | Extra Vars           | флаг `-e "var=value"`       | Переопределение при запуске      |

### Запоминалка для собеседования

> **"Defaults < Inventory < Playbook < Role Vars < Extra Vars"**

Или мнемоника:

> **"DIPRE"** (Defaults, Inventory, Playbook, Role vars, Extra vars)

---

## 4. Детальный разбор каждого уровня

### 🟢 Уровень 1: Role Defaults

**Где:** `roles/nginx/defaults/main.yml`

**Приоритет:** Самый низкий

**Назначение:** Значения по умолчанию для роли. Пользователь может легко их перебить.

```yaml
# roles/nginx/defaults/main.yml
---
nginx_port: 80
nginx_worker_processes: auto
nginx_worker_connections: 1024
nginx_keepalive_timeout: 65
```

**Пример переопределения:**

```yaml
# playbook.yml
- hosts: webservers
  roles:
    - role: nginx
      vars:
        nginx_port: 8080 # Перебивает default
```

---

### 🟡 Уровень 2: Inventory Group Vars

**Где:** `inventory/group_vars/webservers.yml`

**Приоритет:** Низкий (выше defaults)

**Назначение:** Переменные для всей группы хостов

```yaml
# inventory/group_vars/webservers.yml
---
http_port: 80
https_port: 443
app_env: production
nginx_worker_processes: 4
```

**Структура директории:**

```
inventory/
  hosts.yml
  group_vars/
    all.yml              # Для всех хостов
    webservers.yml       # Для группы webservers
    dbservers.yml        # Для группы dbservers
    production.yml       # Для группы production
```

---

### 🟡 Уровень 3: Inventory Host Vars

**Где:** `inventory/host_vars/web1.example.com.yml`

**Приоритет:** Низкий (выше group_vars)

**Назначение:** Переменные для конкретного хоста

```yaml
# inventory/host_vars/web1.example.com.yml
---
ansible_user: admin
ansible_port: 2222
nginx_port: 8080
ssl_certificate: /etc/ssl/web1.crt
```

**Структура директории:**

```
inventory/
  hosts.yml
  host_vars/
    web1.example.com.yml   # Для web1
    web2.example.com.yml   # Для web2
    db1.example.com.yml    # Для db1
```

---

### 🟠 Уровень 4: Playbook Vars

**Где:** блок `vars:` в playbook

**Приоритет:** Средний

**Назначение:** Глобальные переменные для конкретного play

```yaml
# playbook.yml
---
- hosts: webservers
  vars:
    app_port: 8080
    app_user: deploy
    app_version: "2.0.0"
    packages:
      - nginx
      - python3
      - git

  tasks:
    - name: Start app
      systemd:
        name: myapp
        state: started
      environment:
        PORT: "{{ app_port }}"
```

---

### 🟠 Уровень 5: Playbook Vars Files

**Где:** `vars_files:` в playbook

**Приоритет:** Средний (как `vars:`)

**Назначение:** Подключение внешних файлов с переменными

```yaml
# playbook.yml
---
- hosts: webservers
  vars_files:
    - vars/common.yml
    - vars/{{ env }}.yml # Динамический путь
    - vars/secrets.yml # Секреты (хранить в Vault!)

  tasks:
    - name: Use variables
      debug:
        var: app_port
```

**Содержимое `vars/common.yml`:**

```yaml
---
app_port: 8080
db_host: localhost
log_level: info
```

---

### 🔴 Уровень 6: Role Vars

**Где:** `roles/nginx/vars/main.yml`

**Приоритет:** Высокий

**Назначение:** Внутренние переменные роли, которые сложно перебить

```yaml
# roles/nginx/vars/main.yml
---
nginx_config_path: /etc/nginx/nginx.conf
nginx_log_path: /var/log/nginx
nginx_user: www-data
```

**Важно:** Role vars перебивают даже inventory и playbook vars!

---

### ⚫ Уровень 7: Extra Vars

**Где:** флаг `-e` или `--extra-vars` при запуске

**Приоритет:** **АБСОЛЮТНЫЙ КОРОЛЬ** — перебивает ВСЁ!

**Назначение:** Переопределение переменных при запуске

```bash
# Передача одной переменной
ansible-playbook deploy.yml -e "app_version=2.0.0"

# Передача нескольких переменных
ansible-playbook deploy.yml -e "env=production app_port=8080"

# Передача переменных из файла
ansible-playbook deploy.yml -e "@vars/extra.yml"

# Передача в формате JSON
ansible-playbook deploy.yml -e '{"app_version": "2.0.0", "env": "prod"}'
```

**Пример использования:**

```yaml
# playbook.yml
---
- hosts: webservers
  vars:
    app_version: "1.0.0" # Значение по умолчанию

  tasks:
    - name: Deploy app
      git:
        repo: https://github.com/user/app.git
        dest: /opt/app
        version: "{{ app_version }}"
```

```bash
# Запуск с переопределением версии
ansible-playbook deploy.yml -e "app_version=2.0.0"
# Используется версия 2.0.0, а не 1.0.0!
```

---

## 5. Типы данных переменных

### Строки (Strings)

```yaml
vars:
  app_name: "myapp"
  app_version: "2.0.0"
  db_host: "localhost"
```

### Числа (Numbers)

```yaml
vars:
  app_port: 8080
  max_connections: 1000
  timeout: 30.5
```

### Булевы (Booleans)

```yaml
vars:
  enable_ssl: true
  debug_mode: false
  maintenance: yes # или no, on, off, y, n
```

### Списки (Lists)

```yaml
vars:
  packages:
    - nginx
    - python3
    - git

  ports:
    - 80
    - 443
    - 8080
```

### Словари (Dictionaries)

```yaml
vars:
  app_config:
    port: 8080
    host: "localhost"
    workers: 4
    debug: false

  users:
    alice:
      shell: /bin/bash
      groups: sudo,docker
    bob:
      shell: /bin/zsh
      groups: docker
```

### Использование в задачах

```yaml
tasks:
  - name: Use string
    debug:
      msg: "App name: {{ app_name }}"

  - name: Use number
    systemd:
      name: myapp
      state: started
    environment:
      PORT: "{{ app_port }}"

  - name: Use boolean
    template:
      src: app.conf.j2
      dest: /etc/app/app.conf
    when: enable_ssl

  - name: Use list
    apt:
      name: "{{ item }}"
      state: present
    loop: "{{ packages }}"

  - name: Use dictionary
    debug:
      msg: "Port: {{ app_config.port }}, Host: {{ app_config.host }}"

  - name: Use nested dictionary
    user:
      name: alice
      shell: "{{ users.alice.shell }}"
      groups: "{{ users.alice.groups }}"
```

---

## 6. Практические примеры

### Пример 1: Переопределение для разных окружений

**Структура проекта:**

```
ansible-project/
├── inventory/
│   ├── production/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       └── all.yml
│   └── staging/
│       ├── hosts.yml
│       └── group_vars/
│           └── all.yml
├── playbooks/
│   └── deploy.yml
└── roles/
    └── app/
        ├── defaults/
        │   └── main.yml
        └── vars/
            └── main.yml
```

**`inventory/production/group_vars/all.yml`:**

```yaml
---
env: production
app_port: 80
db_host: "db.prod.example.com"
log_level: warn
```

**`inventory/staging/group_vars/all.yml`:**

```yaml
---
env: staging
app_port: 8080
db_host: "db.staging.example.com"
log_level: debug
```

**`roles/app/defaults/main.yml`:**

```yaml
---
app_version: "1.0.0"
app_user: deploy
workers: 2
```

**`roles/app/vars/main.yml`:**

```yaml
---
app_config_path: /etc/app/app.conf
app_log_path: /var/log/app
```

**`playbooks/deploy.yml`:**

```yaml
---
- hosts: all
  become: yes
  vars:
    deploy_timestamp: "{{ ansible_date_time.iso8601 }}"

  roles:
    - app
```

**Запуск:**

```bash
# Production
ansible-playbook -i inventory/production/hosts.yml playbooks/deploy.yml

# Staging
ansible-playbook -i inventory/staging/hosts.yml playbooks/deploy.yml

# Переопределение версии через Extra Vars
ansible-playbook -i inventory/production/hosts.yml playbooks/deploy.yml -e "app_version=2.0.0"
```

---

### Пример 2: Динамические переменные в шаблонах

**`templates/app.conf.j2`:**

```jinja2
# App Configuration
# Generated by Ansible at {{ ansible_date_time.iso8601 }}

[app]
name = {{ app_name }}
version = {{ app_version }}
environment = {{ env }}

[server]
host = {{ ansible_default_ipv4.address }}
port = {{ app_port }}
workers = {{ workers }}

[database]
host = {{ db_host }}
port = 5432

[logging]
level = {{ log_level }}
path = {{ app_log_path }}
```

**Задача в playbook:**

```yaml
tasks:
  - name: Deploy app config
    template:
      src: templates/app.conf.j2
      dest: "{{ app_config_path }}"
    notify: Restart App
```

---

### Пример 3: Условные переменные

```yaml
---
- hosts: all
  vars:
    # Базовые переменные
    app_port: 8080

    # Условные переменные через set_fact
    - name: Set OS-specific variables
      set_fact:
        package_manager: "apt"
        service_manager: "systemd"
      when: ansible_os_family == "Debian"

    - name: Set OS-specific variables (RedHat)
      set_fact:
        package_manager: "yum"
        service_manager: "systemd"
      when: ansible_os_family == "RedHat"

    - name: Use variables
      debug:
        msg: "Using {{ package_manager }} and {{ service_manager }}"
```

---

### Пример 4: Приоритеты в действии

**`roles/app/defaults/main.yml`:**

```yaml
---
app_port: 80
```

**`inventory/group_vars/webservers.yml`:**

```yaml
---
app_port: 8080
```

**`playbook.yml`:**

```yaml
---
- hosts: webservers
  vars:
    app_port: 9000

  roles:
    - role: app
      vars:
        app_port: 7000
```

**Запуск:**

```bash
# Без Extra Vars
ansible-playbook playbook.yml
# Результат: app_port = 7000 (Role Vars перебивают всё)

# С Extra Vars
ansible-playbook playbook.yml -e "app_port=3000"
# Результат: app_port = 3000 (Extra Vars — абсолютный король!)
```

---

## 7. Best Practices

### ✅ Хорошие практики

1. **Используй Role Defaults для значений по умолчанию**
   - Делает роль гибкой и переиспользуемой
   - Пользователь может легко перебить значения

2. **Храни секреты в Ansible Vault**
   - Никогда не коммить пароли в git
   - Используй `ansible-vault encrypt vars/secrets.yml`

3. **Используй group_vars для окружений**
   - Разделяй production, staging, development
   - Храни в `inventory/<env>/group_vars/`

4. **Используй host_vars для уникальных настроек**
   - IP-адреса, SSL-сертификаты, специфичные конфиги

5. **Используй Extra Vars для переопределений**
   - Версии приложений, временные настройки
   - Удобно для CI/CD

6. **Документируй переменные**
   - Добавляй комментарии в `defaults/main.yml`
   - Описывай назначение каждой переменной

7. **Используй осмысленные имена**
   - `nginx_port` вместо `port`
   - `db_host` вместо `host`

### ❌ Антипаттерны

1. **Хранение паролей в открытом виде**
   - Нарушение безопасности
   - Используй Vault!

2. **Жесткое кодирование (hardcoding)**
   - Невозможно переиспользовать код
   - Всегда используй переменные

3. **Использование Role Vars без необходимости**
   - Они перебивают всё, кроме Extra Vars
   - Используй только для внутренних настроек роли

4. **Дублирование переменных**
   - Не объявляй одну переменную в нескольких местах
   - Выбирай один уровень приоритета

5. **Слишком общие имена**
   - `port`, `host`, `user` — плохо
   - `nginx_port`, `db_host`, `app_user` — хорошо

---

## 8. Вопросы для собеседования

### Базовые вопросы

**Q: Что такое переменные в Ansible?**
A: Именованные значения, которые можно использовать в playbook'ах, ролях, шаблонах и условиях для параметризации и гибкости.

**Q: Где можно объявлять переменные?**
A: В role defaults, inventory (group_vars/host_vars), playbook (vars/vars_files), role vars, и передавать через Extra Vars (-e).

**Q: Какие приоритеты у переменных?**
A: От низшего к высшему: Role Defaults < Inventory Vars < Playbook Vars < Role Vars < Extra Vars.

**Q: Что такое Extra Vars и как их передать?**
A: Переменные, передаваемые через флаг `-e` при запуске playbook. Имеют самый высокий приоритет и перебивают всё.

**Q: В чем разница между Role Defaults и Role Vars?**
A: Role Defaults имеют низший приоритет и легко перебиваются. Role Vars имеют высокий приоритет и перебивают даже inventory и playbook vars.

### Продвинутые вопросы

**Q: Как переопределить переменную из role defaults?**
A: Через inventory vars, playbook vars, role vars или extra vars. Любой из этих уровней перебивает defaults.

**Q: Можно ли использовать переменные в именах хостов?**
A: Нет, имена хостов должны быть статичными. Но можно использовать переменные в задачах для динамического выбора хостов.

**Q: Как хранить секреты в Ansible?**
A: Использовать Ansible Vault: `ansible-vault encrypt vars/secrets.yml`. При запуске указывать `--ask-vault-pass`.

**Q: Что произойдет, если переменная объявлена в нескольких местах?**
A: Ansible использует значение с наивысшим приоритетом согласно иерархии.

**Q: Как посмотреть все переменные для хоста?**
A: Использовать модуль `debug` с `var: hostvars[inventory_hostname]` или команду `ansible <host> -m debug -a "var=hostvars[inventory_hostname]"`.

**Q: Можно ли использовать переменные в условиях `when`?**
A: Да, например: `when: env == "production"` или `when: app_port is defined`.

**Q: Как передать переменную из одного playbook в другой?**
A: Напрямую нельзя. Но можно использовать `set_fact` для сохранения значения, или передавать через inventory/external files.

**Q: Что такое `hostvars` и как его использовать?**
A: `hostvars` — это словарь со всеми переменными всех хостов. Пример: `{{ hostvars['web1']['ansible_ip'] }}`.

**Q: Как сделать переменную обязательной?**
A: Использовать `{{ my_var | mandatory }}` в Jinja2. Если переменная не определена, playbook упадет с ошибкой.

**Q: В чем разница между `vars:` и `vars_files:`?**
A: `vars:` — объявление переменных прямо в playbook. `vars_files:` — подключение внешних файлов с переменными. Оба имеют одинаковый приоритет.

**Q: Как использовать переменные в циклах?**
A: Определить список в переменных и использовать в `loop: "{{ my_list }}"`.

**Q: Можно ли использовать переменные в именах handlers?**
A: Нет, имена handlers должны быть статичными. Но можно использовать `listen` с переменными.

---

## 📚 Дополнительные ресурсы

- Официальная документация: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html
- Variable Precedence: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable
- Ansible Vault: https://docs.ansible.com/ansible/latest/vault_guide/index.html

---
