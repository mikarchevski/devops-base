# 📘 Ansible Playbook: Детальная шпаргалка

## 📋 Оглавление

1. [Что такое Playbook](#1-что-такое-playbook)
2. [Базовая структура](#2-базовая-структура)
3. [Иерархия элементов](#3-иерархия-элементов)
4. [Запуск Playbook](#4-запуск-playbook)
5. [Задачи (Tasks)](#5-задачи-tasks)
6. [Обработчики (Handlers)](#6-обработчики-handlers)
7. [Переменные в Playbook](#7-переменные-в-playbook)
8. [Условия (when)](#8-условия-when)
9. [Циклы (loop)](#9-циклы-loop)
10. [Шаблоны (Jinja2)](#10-шаблоны-jinja2)
11. [Роли (Roles)](#11-роли-roles)
12. [Блоки (block/rescue/always)](#12-блоки-blockrescuealways)
13. [Теги (Tags)](#13-теги-tags)
14. [Include и Import](#14-include-и-import)
15. [Отладка (Debug)](#15-отладка-debug)
16. [Практические примеры](#16-практические-примеры)
17. [Best Practices](#17-best-practices)
18. [Вопросы для собеседования](#18-вопросы-для-собеседования)

---

## 1. Что такое Playbook

**Playbook** — это YAML-файл с декларативным описанием задач, которые Ansible должен выполнить на целевых серверах.

**Ключевые особенности:**

- ✅ Декларативный подход (описываем состояние, а не шаги)
- ✅ Идемпотентность (повторный запуск не ломает систему)
- ✅ Читаемость (YAML легко читать как документацию)
- ✅ Переиспользуемость (можно параметризовать через переменные)

---

## 2. Базовая структура

### Минимальный Playbook

```yaml
---
- hosts: webservers # Кому применяем (группа из inventory)
  become: yes # Выполнять с sudo
  tasks: # Список задач
    - name: Install Nginx
      apt:
        name: nginx
        state: present
```

### Полная структура Play

```yaml
---
- hosts: webservers # Целевые хосты
  become: yes # Выполнять с sudo
  become_user: root # От какого пользователя

  vars: # Переменные для этого Play
    app_port: 8080
    db_host: "localhost"

  vars_files: # Подключение внешних файлов с переменными
    - vars/secrets.yml
    - vars/common.yml

  pre_tasks: # Задачи ДО ролей
    - name: Update apt cache
      apt:
        update_cache: yes

  roles: # Подключение ролей
    - common
    - nginx

  tasks: # Задачи ПОСЛЕ ролей
    - name: Deploy application
      copy:
        src: app.tar.gz
        dest: /opt/app/

  post_tasks: # Задачи в самом конце
    - name: Verify deployment
      uri:
        url: "http://localhost:{{ app_port }}/health"

  handlers: # Обработчики
    - name: Restart Nginx
      systemd:
        name: nginx
        state: restarted
```

---

## 3. Иерархия элементов

Playbook (файл .yml)
└─► Play (блок для группы хостов)
├─► vars (переменные)
├─► pre_tasks (задачи до ролей)
├─► roles (подключение ролей)
├─► tasks (основные задачи)
│ └─► Task (конкретное действие)
│ ├─► name (имя задачи)
│ ├─► module (модуль: apt, copy, etc.)
│ ├─► args (аргументы модуля)
│ ├─► when (условие)
│ ├─► loop (цикл)
│ ├─► register (сохранить результат)
│ └─► notify (вызов handler)
├─► post_tasks (задачи в конце)
└─► handlers (обработчики)

---

## 4. Запуск Playbook

### Базовые команды

```bash
# Базовый запуск
ansible-playbook playbook.yml

# С указанием инвентаря
ansible-playbook -i inventory.yml playbook.yml

# С передачей переменных
ansible-playbook playbook.yml -e "env=production version=2.0"

# С передачей переменных из файла
ansible-playbook playbook.yml -e "@vars/extra.yml"
```

### Проверка и отладка

```bash
# Проверка синтаксиса (без выполнения)
ansible-playbook playbook.yml --syntax-check

# Dry-run (показать, что изменится, без реальных действий)
ansible-playbook playbook.yml --check

# Показать различия в файлах
ansible-playbook playbook.yml --check --diff

# Подробный вывод
ansible-playbook playbook.yml -v        # Базовый verbose
ansible-playbook playbook.yml -vv       # Более подробный
ansible-playbook playbook.yml -vvv      # Максимальная детализация
```

### Фильтрация задач

```bash
# Запуск только определенных тегов
ansible-playbook playbook.yml --tags "install,config"

# Пропустить определенные теги
ansible-playbook playbook.yml --skip-tags "debug"

# Начать с конкретной задачи (если плейбук упал посередине)
ansible-playbook playbook.yml --start-at-task="Install Nginx"

# Список хостов, которые будут затронуты
ansible-playbook playbook.yml --list-hosts

# Список задач без выполнения
ansible-playbook playbook.yml --list-tasks
```

### Ограничение хостов

```bash
# Выполнить только на определенных хостах (даже если в playbook указаны все)
ansible-playbook playbook.yml --limit "web1.example.com"
ansible-playbook playbook.yml --limit "webservers:&prod"
```

---

## 5. Задачи (Tasks)

### Базовая задача

```yaml
tasks:
  - name: Install Nginx
    apt:
      name: nginx
      state: present
    tags:
      - install
      - nginx
```

### Задача с условием

```yaml
tasks:
  - name: Install PostgreSQL only on Debian
    apt:
      name: postgresql
      state: present
    when: ansible_os_family == "Debian"
```

### Задача с циклом

```yaml
tasks:
  - name: Install multiple packages
    apt:
      name: "{{ item }}"
      state: present
    loop:
      - nginx
      - curl
      - git
```

### Задача с регистрацией результата

```yaml
tasks:
  - name: Check if file exists
    stat:
      path: /etc/myapp.conf
    register: file_check

  - name: Create file if not exists
    copy:
      content: "default config"
      dest: /etc/myapp.conf
    when: not file_check.stat.exists
```

### Задача с вызовом handler

```yaml
tasks:
  - name: Update config
    template:
      src: app.conf.j2
      dest: /etc/app/app.conf
    notify: Restart App
```

### Задача с блоком обработки ошибок

```yaml
tasks:
  - name: Try to install package
    block:
      - name: Install main package
        apt:
          name: myapp
          state: present

      - name: Configure app
        template:
          src: app.conf.j2
          dest: /etc/myapp/app.conf

    rescue:
      - name: Handle installation failure
        debug:
          msg: "Installation failed, trying alternative"

      - name: Install alternative package
        apt:
          name: myapp-alt
          state: present

    always:
      - name: Cleanup temp files
        file:
          path: /tmp/myapp_install
          state: absent
```

---

## 6. Обработчики (Handlers)

### Базовая структура

```yaml
tasks:
  - name: Update config 1
    copy:
      src: config1.conf
      dest: /etc/app/config1.conf
    notify: Restart App

  - name: Update config 2
    copy:
      src: config2.conf
      dest: /etc/app/config2.conf
    notify: Restart App

  - name: Force flush handlers now
    meta: flush_handlers

handlers:
  - name: Restart App
    systemd:
      name: myapp
      state: restarted

  - name: Reload Nginx
    systemd:
      name: nginx
      state: reloaded
    listen: "reload web services"
```

### Ключевые правила

1. Handlers выполняются **ТОЛЬКО** при статусе `CHANGED`
2. Handlers выполняются **ТОЛЬКО** в конце Play (или при `meta: flush_handlers`)
3. Если 3 задачи вызвали один handler, он выполнится только **1 раз** (дедупликация)
4. При ошибке в tasks handlers **НЕ выполняются** (если не указан `force_handlers: yes`)
5. Handlers выполняются в порядке их **объявления**, а не вызова

---

## 7. Переменные в Playbook

### Объявление переменных

```yaml
---
- hosts: webservers
  vars:
    app_port: 8080
    db_host: "localhost"
    packages:
      - nginx
      - curl
      - git

  vars_files:
    - vars/common.yml
    - vars/{{ env }}.yml # Динамический путь

  tasks:
    - name: Start app
      systemd:
        name: myapp
        state: started
      environment:
        PORT: "{{ app_port }}"
        DB_HOST: "{{ db_host }}"
```

### Использование переменных

```yaml
tasks:
  - name: Show variable
    debug:
      var: app_port

  - name: Use variable in path
    copy:
      src: "files/{{ env }}/config.conf"
      dest: "/etc/app/{{ app_name }}.conf"

  - name: Conditional with variable
    debug:
      msg: "Production mode"
    when: env == "production"
```

### Приоритеты переменных

От низшего к высшему:

1. 🟢 Role Defaults (`roles/x/defaults/main.yml`)
2. 🟡 Inventory Vars (`group_vars/`, `host_vars/`)
3. 🟠 Playbook Vars (блок `vars:` или `vars_files:`)
4. 🔴 Role Vars (`roles/x/vars/main.yml`)
5. ⚫ Extra Vars (флаг `-e` при запуске) — **перебивает всё!**

---

## 8. Условия (when)

### Простые условия

```yaml
tasks:
  - name: Install on Ubuntu only
    apt:
      name: htop
      state: present
    when: ansible_distribution == "Ubuntu"

  - name: Install on servers with 2GB+ RAM
    apt:
      name: postgresql
      state: present
    when: ansible_memtotal_mb >= 2048
```

### Множественные условия (AND)

```yaml
tasks:
  - name: Install on Debian with 2GB+ RAM
    apt:
      name: myapp
      state: present
    when:
      - ansible_os_family == "Debian"
      - ansible_memtotal_mb >= 2048
```

### Множественные условия (OR)

```yaml
tasks:
  - name: Install on Debian or RedHat
    package:
      name: myapp
      state: present
    when: ansible_os_family == "Debian" or ansible_os_family == "RedHat"
```

### Проверка существования переменной

```yaml
tasks:
  - name: Use variable if defined
    debug:
      msg: "Value is {{ my_var }}"
    when: my_var is defined

  - name: Use variable with default
    debug:
      msg: "Value is {{ my_var | default('default_value') }}"
```

### Проверка регистра результата

```yaml
tasks:
  - name: Check service status
    systemd:
      name: nginx
    register: nginx_status

  - name: Start if stopped
    systemd:
      name: nginx
      state: started
    when: nginx_status.status.ActiveState != "active"

  - name: Show if failed
    debug:
      msg: "Service check failed"
    when: nginx_status is failed
```

### Сложные условия

```yaml
tasks:
  - name: Complex condition
    apt:
      name: myapp
      state: present
    when:
      - ansible_os_family == "Debian"
      - ansible_memtotal_mb >= 2048
      - env == "production" or env == "staging"
      - my_var is defined
      - my_var != "skip"
```

---

## 9. Циклы (loop)

### Простой цикл по списку

```yaml
tasks:
  - name: Install packages
    apt:
      name: "{{ item }}"
      state: present
    loop:
      - nginx
      - curl
      - git
```

### Цикл по словарям

```yaml
tasks:
  - name: Create users
    user:
      name: "{{ item.name }}"
      shell: "{{ item.shell }}"
      groups: "{{ item.groups }}"
    loop:
      - { name: "alice", shell: "/bin/bash", groups: "sudo,docker" }
      - { name: "bob", shell: "/bin/zsh", groups: "docker" }
```

### Цикл с условием

```yaml
tasks:
  - name: Install OS-specific packages
    package:
      name: "{{ item.name }}"
      state: present
    loop:
      - { name: "nginx", os: "all" }
      - { name: "certbot", os: "Debian" }
      - { name: "epel-release", os: "RedHat" }
    when: item.os == 'all' or item.os == ansible_os_family
```

### Цикл с регистрацией результатов

```yaml
tasks:
  - name: Check multiple services
    systemd:
      name: "{{ item }}"
    register: service_status
    loop:
      - nginx
      - postgresql
      - redis

  - name: Show failed services
    debug:
      msg: "{{ item.item }} is not running"
    loop: "{{ service_status.results }}"
    when: item.status.ActiveState != "active"
```

### Цикл по файлам в директории

```yaml
tasks:
  - name: Copy all config files
    copy:
      src: "{{ item }}"
      dest: "/etc/app/{{ item | basename }}"
    loop: "{{ lookup('fileglob', 'configs/*.conf') }}"
```

### Цикл с подсловарями

```yaml
tasks:
  - name: Create multiple directories with different owners
    file:
      path: "{{ item.0 }}/{{ item.1 }}"
      state: directory
      owner: "{{ item.1 }}"
    loop: "{{ dirs | subelements('owners') }}"
    vars:
      dirs:
        - path: /opt
          owners:
            - app1
            - app2
        - path: /var
          owners:
            - www-data
```

---

## 10. Шаблоны (Jinja2)

### Использование в плейбуке

```yaml
tasks:
  - name: Deploy config from template
    template:
      src: templates/nginx.conf.j2
      dest: /etc/nginx/nginx.conf
      owner: root
      group: root
      mode: "0644"
      backup: yes
    notify: Restart Nginx
```

### Файл шаблона `templates/nginx.conf.j2`

```jinja2
# Простая подстановка переменной
server {
    listen {{ nginx_port }};
    server_name {{ ansible_fqdn }};

    # Условие
    {% if enable_ssl %}
    ssl_certificate /etc/ssl/certs/{{ ansible_hostname }}.crt;
    ssl_certificate_key /etc/ssl/private/{{ ansible_hostname }}.key;
    {% endif %}

    # Цикл
    {% for upstream in backends %}
    upstream {{ upstream.name }} {
        {% for server in upstream.servers %}
        server {{ server }};
        {% endfor %}
    }
    {% endfor %}

    # Фильтры
    location / {
        proxy_pass http://{{ backend_ip | default('127.0.0.1') }}:{{ backend_port | default(8000) }};
    }
}
```

### Полезные фильтры Jinja2

```jinja2
{{ var | default('value') }}           # Значение по умолчанию
{{ var | upper }}                      # Верхний регистр
{{ var | lower }}                      # Нижний регистр
{{ var | replace('old', 'new') }}      # Замена
{{ list | join(',') }}                 # Объединение списка
{{ path | basename }}                  # Имя файла из пути
{{ path | dirname }}                   # Директория из пути
{{ var | to_nice_yaml }}               # Форматирование в YAML
{{ var | to_json }}                    # Конвертация в JSON
{{ var | to_nice_json }}               # Красивый JSON
{{ var | regex_replace('pattern', 'replacement') }}  # Regex замена
{{ var | regex_search('pattern') }}    # Regex поиск
{{ list | map(attribute='name') | list }}  # Извлечение атрибутов
{{ list | selectattr('active', 'equalto', true) | list }}  # Фильтрация
```

---

## 11. Роли (Roles)

### Использование ролей в плейбуке

```yaml
- hosts: webservers
  roles:
    - common
    - nginx
    - { role: postgresql, when: ansible_os_family == "Debian" }
    - { role: monitoring, tags: ["monitoring"] }
    - { role: backup, backup_schedule: "0 2 * * *" }

  tasks:
    - name: Additional tasks after roles
      debug:
        msg: "Roles are done"
```

### Структура роли

```
roles/
  nginx/
    tasks/
      main.yml          # Основные задачи (обязательно)
    handlers/
      main.yml          # Обработчики
    templates/
      nginx.conf.j2     # Шаблоны
    files/
      index.html        # Статические файлы
    vars/
      main.yml          # Переменные (высокий приоритет)
    defaults/
      main.yml          # Дефолтные переменные (низкий приоритет)
    meta/
      main.yml          # Зависимости от других ролей
    tests/
      inventory         # Тестовый инвентарь
      test.yml          # Тестовый playbook
```

---

## 12. Блоки (block/rescue/always)

### Базовая структура

```yaml
tasks:
  - name: Install and configure app
    block:
      - name: Install package
        apt:
          name: myapp
          state: present

      - name: Configure app
        template:
          src: app.conf.j2
          dest: /etc/myapp/app.conf
        notify: Restart App

    rescue:
      - name: Handle installation failure
        debug:
          msg: "Installation failed, rolling back"

      - name: Remove failed package
        apt:
          name: myapp
          state: absent

      - name: Notify admin
        mail:
          to: admin@example.com
          subject: "App installation failed"

    always:
      - name: Cleanup temp files
        file:
          path: /tmp/myapp_install
          state: absent
```

### Когда использовать

- **block**: Группа задач, которые должны выполняться вместе
- **rescue**: Обработка ошибок (как try/catch в программировании)
- **always**: Задачи, которые выполняются ВСЕГДА, даже если был error (как finally)

---

## 13. Теги (Tags)

### Добавление тегов к задачам

```yaml
tasks:
  - name: Install Nginx
    apt:
      name: nginx
      state: present
    tags:
      - install
      - nginx
      - web

  - name: Configure Nginx
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    tags:
      - config
      - nginx

  - name: Start Nginx
    systemd:
      name: nginx
      state: started
    tags:
      - service
      - nginx
```

### Специальные теги

```yaml
tasks:
  - name: Always run this
    debug:
      msg: "This always runs"
    tags:
      - always # Выполняется всегда, даже если указаны другие теги

  - name: Never run unless explicitly tagged
    debug:
      msg: "This is for debugging only"
    tags:
      - never # Не выполняется, если не указан явно
```

### Запуск с тегами

```bash
# Только задачи с тегом "install"
ansible-playbook playbook.yml --tags install

# Задачи с тегами "install" ИЛИ "config"
ansible-playbook playbook.yml --tags install,config

# Пропустить задачи с тегом "debug"
ansible-playbook playbook.yml --skip-tags debug

# Список всех тегов в playbook
ansible-playbook playbook.yml --list-tags
```

---

## 14. Include и Import

### include_tasks (динамический)

```yaml
tasks:
  - name: Include additional tasks
    include_tasks: tasks/install.yml
    when: ansible_os_family == "Debian"

  - name: Include with variables
    include_tasks: tasks/deploy.yml
    vars:
      app_version: "2.0"
```

### import_tasks (статический)

```yaml
tasks:
  - name: Import tasks
    import_tasks: tasks/config.yml
```

### include_vars (загрузка переменных)

```yaml
tasks:
  - name: Load variables
    include_vars:
      file: vars/secrets.yml

  - name: Load variables from directory
    include_vars:
      dir: vars/
      extensions:
        - yml
        - yaml
```

### include_role (динамическое подключение роли)

```yaml
tasks:
  - name: Include role conditionally
    include_role:
      name: nginx
    when: install_webserver | default(false)

  - name: Include role in loop
    include_role:
      name: "{{ item }}"
    loop:
      - nginx
      - postgresql
      - redis
```

### import_role (статическое подключение роли)

```yaml
tasks:
  - name: Import role
    import_role:
      name: nginx
```

### Разница между include и import

| Характеристика     | include (динамический)        | import (статический)             |
| ------------------ | ----------------------------- | -------------------------------- |
| Когда парсится     | Во время выполнения           | При загрузке playbook            |
| Условия (when)     | Работают                      | Работают, но парсятся всегда     |
| Циклы (loop)       | Работают                      | Не работают                      |
| Переменные         | Могут использовать переменные | Не могут использовать переменные |
| Теги               | Наследуются                   | Наследуются                      |
| Производительность | Медленнее                     | Быстрее                          |

---

## 15. Отладка (Debug)

### Вывод переменной

```yaml
tasks:
  - name: Show variable
    debug:
      var: my_variable

  - name: Show variable with message
    debug:
      msg: "The value is: {{ my_variable }}"
```

### Вывод результата регистрации

```yaml
tasks:
  - name: Check service
    systemd:
      name: nginx
    register: nginx_status

  - name: Show service status
    debug:
      var: nginx_status

  - name: Show specific field
    debug:
      msg: "Service state: {{ nginx_status.status.ActiveState }}"
```

### Условный вывод

```yaml
tasks:
  - name: Debug only in verbose mode
    debug:
      msg: "This is verbose output"
    verbosity: 2 # Показывается только при -vv или выше
```

### Отладка циклов

```yaml
tasks:
  - name: Install packages
    apt:
      name: "{{ item }}"
      state: present
    loop:
      - nginx
      - curl
    register: install_result

  - name: Show installation results
    debug:
      msg: "Package {{ item.item }}: {{ 'installed' if item.changed else 'already present' }}"
    loop: "{{ install_result.results }}"
```

---

## 16. Практические примеры

### Пример 1: Полный Playbook для деплоя веб-приложения

```yaml
---
- hosts: webservers
  become: yes
  vars:
    app_name: myapp
    app_port: 8080
    app_user: deploy
    app_version: "1.0.0"

  pre_tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

  roles:
    - common
    - nginx

  tasks:
    - name: Create app user
      user:
        name: "{{ app_user }}"
        shell: /bin/bash
        create_home: yes

    - name: Create app directory
      file:
        path: "/opt/{{ app_name }}"
        state: directory
        owner: "{{ app_user }}"
        mode: "0755"

    - name: Deploy application
      git:
        repo: https://github.com/user/app.git
        dest: "/opt/{{ app_name }}"
        version: "{{ app_version }}"
      become_user: "{{ app_user }}"
      notify: Restart App

    - name: Install Python dependencies
      pip:
        requirements: "/opt/{{ app_name }}/requirements.txt"
        virtualenv: "/opt/{{ app_name }}/venv"
      become_user: "{{ app_user }}"

    - name: Deploy Nginx config
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/sites-available/{{ app_name }}
      notify: Reload Nginx

    - name: Enable site
      file:
        src: "/etc/nginx/sites-available/{{ app_name }}"
        dest: "/etc/nginx/sites-enabled/{{ app_name }}"
        state: link
      notify: Reload Nginx

    - name: Deploy systemd service
      template:
        src: templates/app.service.j2
        dest: /etc/systemd/system/{{ app_name }}.service
      notify: Restart App

    - name: Start and enable service
      systemd:
        name: "{{ app_name }}"
        state: started
        enabled: yes
        daemon_reload: yes

  post_tasks:
    - name: Verify deployment
      uri:
        url: "http://localhost:{{ app_port }}/health"
        status_code: 200
      retries: 5
      delay: 5

  handlers:
    - name: Reload Nginx
      systemd:
        name: nginx
        state: reloaded

    - name: Restart App
      systemd:
        name: "{{ app_name }}"
        state: restarted
```

### Пример 2: Playbook с обработкой ошибок

```yaml
---
- hosts: dbservers
  become: yes
  force_handlers: yes

  tasks:
    - name: Backup database
      block:
        - name: Create backup directory
          file:
            path: /backup/postgresql
            state: directory
            mode: "0755"

        - name: Dump database
          shell: pg_dump mydb > /backup/postgresql/mydb_{{ ansible_date_time.iso8601 }}.sql
          args:
            creates: "/backup/postgresql/mydb_{{ ansible_date_time.iso8601 }}.sql"

      rescue:
        - name: Handle backup failure
          debug:
            msg: "Backup failed, sending alert"

        - name: Send alert
          mail:
            to: dba@example.com
            subject: "Database backup failed on {{ ansible_hostname }}"
            body: "Backup failed at {{ ansible_date_time.iso8601 }}"

      always:
        - name: Cleanup old backups
          shell: find /backup/postgresql -name "*.sql" -mtime +7 -delete
```

---

## 17. Best Practices

### ✅ Хорошие практики

1. **Всегда давай задачам понятные имена** (`name:`)
   - Это спасает при отладке и делает playbook читаемым

2. **Используй специализированные модули** вместо `shell`/`command`
   - Они гарантируют идемпотентность

3. **Применяй хендлеры** для перезапуска сервисов
   - Избегай лишних перезапусков

4. **Разбивай код на роли** для переиспользования
   - Каждая роль — одна ответственность

5. **Используй теги** для гибкого запуска
   - Можно запускать только нужные части

6. **Храни секреты в Ansible Vault**
   - Никогда не коммить пароли в git

7. **Тестируй с `--check`** перед реальным запуском
   - Особенно в продакшене

8. **Используй `block/rescue/always`** для обработки ошибок
   - Особенно для критичных операций

9. **Группируй связанные задачи** через `block`
   - Логическая группировка + обработка ошибок

10. **Используй переменные** вместо hardcoding
    - Делает playbook гибким и переиспользуемым

### ❌ Антипаттерны

1. **Использование `shell` без крайней необходимости**
   - Теряется идемпотентность

2. **Дублирование кода вместо циклов**
   - Сложнее поддерживать

3. **Жесткое кодирование значений**
   - Невозможно переиспользовать

4. **Игнорирование идемпотентности**
   - Повторный запуск может сломать систему

5. **Отсутствие обработки ошибок**
   - Система может остаться в неконсистентном состоянии

6. **Хранение паролей в открытом виде**
   - Нарушение безопасности

7. **Огромные playbook без ролей**
   - Сложно читать и поддерживать

8. **Использование `command` когда есть специализированный модуль**
   - Например, `command: systemctl restart nginx` вместо `systemd: name=nginx state=restarted`

---

## 18. Вопросы для собеседования

### Базовые вопросы

**Q: Что такое Playbook?**
A: YAML-файл с декларативным описанием задач, которые Ansible должен выполнить на целевых серверах.

**Q: В чем разница между Play и Task?**
A: Play — это блок для конкретной группы хостов (описывает ЧТО делать и ГДЕ). Task — это конкретное действие (один модуль с аргументами).

**Q: Как запустить playbook только на определенных хостах?**
A: Использовать флаг `--limit`: `ansible-playbook playbook.yml --limit "webservers"`

**Q: Что делает флаг `--check`?**
A: Выполняет dry-run — показывает, что изменится, но не вносит реальных изменений.

**Q: Как передать переменную в playbook при запуске?**
A: Использовать флаг `-e` или `--extra-vars`: `ansible-playbook playbook.yml -e "env=production"`

### Продвинутые вопросы

**Q: В чем разница между `include_tasks` и `import_tasks`?**
A: `include_tasks` — динамический, парсится во время выполнения, поддерживает условия и циклы. `import_tasks` — статический, парсится при загрузке playbook, быстрее, но не поддерживает переменные в условиях.

**Q: Как работает `block/rescue/always`?**
A: `block` — группа задач, `rescue` — выполняется если в block была ошибка (как catch), `always` — выполняется всегда (как finally).

**Q: Что такое `meta: flush_handlers`?**
A: Заставляет Ansible немедленно выполнить все накопленные handlers, не дожидаясь конца play.

**Q: Как заставить handlers выполниться даже если playbook упал с ошибкой?**
A: Указать `force_handlers: yes` на уровне play.

**Q: Что такое теги и зачем они нужны?**
A: Теги — это метки для задач. Позволяют запускать только определенные части playbook: `ansible-playbook playbook.yml --tags install`

**Q: Как отладить playbook, если он упал посередине?**
A: Использовать `--start-at-task="Name of the task"` чтобы начать с конкретной задачи, или `--check --diff` для проверки.

**Q: Что такое `register` и как его использовать?**
A: `register` сохраняет результат выполнения задачи в переменную. Пример: `register: result`, затем можно использовать `result.stdout`, `result.changed`, `result.failed` и т.д.

**Q: Как сделать задачу идемпотентной, если используешь `shell`?**
A: Использовать параметры `creates: /path/to/file` или `removes: /path/to/file`, которые говорят модулю выполнять команду только если файла нет/он есть.

**Q: Что такое `pre_tasks`, `roles`, `tasks`, `post_tasks`?**
A: Порядок выполнения: сначала `pre_tasks`, потом `roles`, потом `tasks`, в конце `post_tasks`. Handlers выполняются после всего.

---

## 📚 Дополнительные ресурсы

- Официальная документация: https://docs.ansible.com/ansible/latest/playbook_guide/
- Примеры playbook: https://github.com/ansible/ansible-examples
- Best Practices: https://docs.ansible.com/ansible/latest/tips_tricks/ansible_tips_tricks.html
