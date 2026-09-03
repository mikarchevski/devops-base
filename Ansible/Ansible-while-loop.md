# 🔄 Ansible Conditions & Loops: Детальная шпаргалка

## 📋 Оглавление

1. [Условия (when)](#1-условия-when)
2. [Множественные условия](#2-множественные-условия)
3. [Проверка переменных](#3-проверка-переменных)
4. [Работа с register](#4-работа-с-register)
5. [Циклы (loop)](#5-циклы-loop)
6. [Продвинутые циклы](#6-продвинутые-циклы)
7. [Циклы с условиями](#7-циклы-с-условиями)
8. [Практические примеры](#8-практические-примеры)
9. [Best Practices](#9-best-practices)
10. [Вопросы для собеседования](#10-вопросы-для-собеседования)

---

## 1. Условия (when)

### Базовый синтаксис

```yaml
tasks:
  - name: Install package only on Ubuntu
    apt:
      name: nginx
      state: present
    when: ansible_distribution == "Ubuntu"
```

### Операторы сравнения

```yaml
tasks:
  # Равенство
  - name: Equal
    debug:
      msg: "OS is Ubuntu"
    when: ansible_distribution == "Ubuntu"

  # Не равно
  - name: Not equal
    debug:
      msg: "OS is not Ubuntu"
    when: ansible_distribution != "Ubuntu"

  # Больше/меньше (числа)
  - name: More than 2GB RAM
    debug:
      msg: "Server has enough RAM"
    when: ansible_memtotal_mb > 2048

  - name: Less than 1GB RAM
    debug:
      msg: "Low memory server"
    when: ansible_memtotal_mb < 1024

  # Больше/меньше или равно
  - name: At least 2GB RAM
    debug:
      msg: "Server has 2GB+ RAM"
    when: ansible_memtotal_mb >= 2048

  # Вхождение в список
  - name: OS is in list
    debug:
      msg: "Supported OS"
    when: ansible_distribution in ["Ubuntu", "Debian", "CentOS"]

  # Проверка на истинность
  - name: Boolean true
    debug:
      msg: "SSL is enabled"
    when: enable_ssl

  # Проверка на ложность
  - name: Boolean false
    debug:
      msg: "Debug mode is off"
    when: not debug_mode
```

### Сравнение строк

```yaml
tasks:
  # Точное совпадение
  - name: Exact match
    debug:
      msg: "Production environment"
    when: env == "production"

  # Содержит подстроку
  - name: Contains substring
    debug:
      msg: "OS name contains 'Ubuntu'"
    when: "'Ubuntu' in ansible_distribution"

  # Начинается с
  - name: Starts with
    debug:
      msg: "Hostname starts with 'web'"
    when: ansible_hostname.startswith("web")

  # Заканчивается на
  - name: Ends with
    debug:
      msg: "Hostname ends with 'prod'"
    when: ansible_hostname.endswith("prod")

  # Регулярное выражение
  - name: Regex match
    debug:
      msg: "Hostname matches pattern"
    when: ansible_hostname is match("web[0-9]+")

  # Поиск по regex
  - name: Regex search
    debug:
      msg: "Found pattern in hostname"
    when: ansible_hostname is search("[0-9]+")
```

### Сравнение чисел

```yaml
tasks:
  # Важно: сравнивай числа с числами, а не со строками!

  # ✅ ПРАВИЛЬНО
  - name: Correct number comparison
    debug:
      msg: "Enough memory"
    when: ansible_memtotal_mb >= 2048

  # ❌ НЕПРАВИЛЬНО (строковое сравнение)
  - name: Wrong string comparison
    debug:
      msg: "This is wrong"
    when: ansible_memtotal_mb >= "2048"
    # "10000" < "2048" при строковом сравнении!

  # Приведение типов
  - name: Convert to int
    debug:
      msg: "Converted to integer"
    when: my_var | int > 100

  - name: Convert to float
    debug:
      msg: "Converted to float"
    when: my_var | float > 10.5
```

---

## 2. Множественные условия

### Логическое И (AND)

```yaml
tasks:
  # Способ 1: Список условий (все должны быть true)
  - name: Debian with 2GB+ RAM
    apt:
      name: postgresql
      state: present
    when:
      - ansible_os_family == "Debian"
      - ansible_memtotal_mb >= 2048

  # Способ 2: Оператор and
  - name: Debian with 2GB+ RAM (alternative)
    apt:
      name: postgresql
      state: present
    when: ansible_os_family == "Debian" and ansible_memtotal_mb >= 2048
```

### Логическое ИЛИ (OR)

```yaml
tasks:
  # Способ 1: Оператор or
  - name: Debian or RedHat
    package:
      name: nginx
      state: present
    when: ansible_os_family == "Debian" or ansible_os_family == "RedHat"

  # Способ 2: Проверка вхождения в список
  - name: OS in supported list
    package:
      name: nginx
      state: present
    when: ansible_os_family in ["Debian", "RedHat"]
```

### Комбинация AND и OR

```yaml
tasks:
  - name: Complex condition
    debug:
      msg: "Complex condition met"
    when:
      - ansible_os_family == "Debian"
      - ansible_memtotal_mb >= 2048
      - env == "production" or env == "staging"
      - enable_ssl or enable_https
```

### Отрицание (NOT)

```yaml
tasks:
  # Способ 1: Оператор not
  - name: Not Ubuntu
    debug:
      msg: "This is not Ubuntu"
    when: not ansible_distribution == "Ubuntu"

  # Способ 2: !=
  - name: Not Ubuntu (alternative)
    debug:
      msg: "This is not Ubuntu"
    when: ansible_distribution != "Ubuntu"

  # Отрицание сложного условия
  - name: Not (Debian and 2GB+)
    debug:
      msg: "Condition not met"
    when: not (ansible_os_family == "Debian" and ansible_memtotal_mb >= 2048)
```

---

## 3. Проверка переменных

### Проверка существования переменной

```yaml
tasks:
  # Проверка, определена ли переменная
  - name: Use variable if defined
    debug:
      msg: "Variable is defined: {{ my_var }}"
    when: my_var is defined

  # Проверка, не определена ли переменная
  - name: Use default if not defined
    debug:
      msg: "Variable is not defined"
    when: my_var is not defined

  # Использование значения по умолчанию
  - name: Use default value
    debug:
      msg: "Value: {{ my_var | default('default_value') }}"

  # Обязательная переменная (упадет, если не определена)
  - name: Mandatory variable
    debug:
      msg: "Value: {{ my_var | mandatory }}"
```

### Проверка пустых значений

```yaml
tasks:
  # Проверка на пустую строку
  - name: Not empty string
    debug:
      msg: "String is not empty"
    when: my_string | length > 0

  # Проверка на None
  - name: Not None
    debug:
      msg: "Variable is not None"
    when: my_var is not none

  # Проверка пустого списка
  - name: List not empty
    debug:
      msg: "List has items"
    when: my_list | length > 0

  # Проверка пустого словаря
  - name: Dict not empty
    debug:
      msg: "Dict has keys"
    when: my_dict | length > 0
```

### Проверка типов данных

```yaml
tasks:
  # Проверка, является ли переменная строкой
  - name: Is string
    debug:
      msg: "Variable is a string"
    when: my_var is string

  # Проверка, является ли переменная числом
  - name: Is number
    debug:
      msg: "Variable is a number"
    when: my_var is number

  # Проверка, является ли переменная списком
  - name: Is list
    debug:
      msg: "Variable is a list"
    when: my_var is iterable and my_var is not string and my_var is not mapping

  # Проверка, является ли переменная словарем
  - name: Is dict
    debug:
      msg: "Variable is a dict"
    when: my_var is mapping
```

---

## 4. Работа с register

### Базовое использование register

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

### Проверка статуса задачи

```yaml
tasks:
  - name: Try to install package
    apt:
      name: myapp
      state: present
    register: install_result
    ignore_errors: yes # Не останавливать playbook при ошибке

  - name: Handle success
    debug:
      msg: "Installation successful"
    when: install_result is succeeded

  - name: Handle failure
    debug:
      msg: "Installation failed"
    when: install_result is failed

  - name: Handle changed
    debug:
      msg: "Package was installed"
    when: install_result is changed

  - name: Handle skipped
    debug:
      msg: "Task was skipped"
    when: install_result is skipped
```

### Использование вывода команд

```yaml
tasks:
  - name: Get disk usage
    shell: df -h / | awk 'NR==2 {print $5}' | sed 's/%//'
    register: disk_usage
    changed_when: false

  - name: Show disk usage
    debug:
      msg: "Disk usage: {{ disk_usage.stdout }}%"

  - name: Alert if disk is full
    debug:
      msg: "WARNING: Disk is almost full!"
    when: disk_usage.stdout | int > 80

  - name: Show all output lines
    debug:
      msg: "{{ item }}"
    loop: "{{ disk_usage.stdout_lines }}"
```

### Проверка кода возврата

```yaml
tasks:
  - name: Run command
    command: /opt/app/check.sh
    register: check_result
    failed_when: false # Не считать ошибкой

  - name: Handle success (exit code 0)
    debug:
      msg: "Check passed"
    when: check_result.rc == 0

  - name: Handle specific error (exit code 1)
    debug:
      msg: "Check failed with code 1"
    when: check_result.rc == 1

  - name: Handle any error
    debug:
      msg: "Check failed with code {{ check_result.rc }}"
    when: check_result.rc != 0
```

---

## 5. Циклы (loop)

### Базовый цикл по списку

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
      - htop
```

### Цикл по словарям

```yaml
tasks:
  - name: Create users
    user:
      name: "{{ item.name }}"
      shell: "{{ item.shell }}"
      groups: "{{ item.groups }}"
      state: present
    loop:
      - { name: "alice", shell: "/bin/bash", groups: "sudo,docker" }
      - { name: "bob", shell: "/bin/zsh", groups: "docker" }
      - { name: "charlie", shell: "/bin/fish", groups: "sudo" }
```

### Цикл по словарю (dict)

```yaml
vars:
  users:
    alice:
      shell: /bin/bash
      groups: sudo,docker
    bob:
      shell: /bin/zsh
      groups: docker

tasks:
  - name: Create users from dict
    user:
      name: "{{ item.key }}"
      shell: "{{ item.value.shell }}"
      groups: "{{ item.value.groups }}"
      state: present
    loop: "{{ users | dict2items }}"
    # dict2items преобразует словарь в список:
    # [{key: 'alice', value: {...}}, {key: 'bob', value: {...}}]
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
    ignore_errors: yes

  - name: Show failed services
    debug:
      msg: "Service {{ item.item }} is not running"
    loop: "{{ service_status.results }}"
    when: item is failed

  - name: Show all service statuses
    debug:
      msg: "Service {{ item.item }}: {{ 'running' if item.status.ActiveState == 'active' else 'stopped' }}"
    loop: "{{ service_status.results }}"
```

### Цикл по файлам в директории

```yaml
tasks:
  - name: Copy all config files
    copy:
      src: "{{ item }}"
      dest: "/etc/app/{{ item | basename }}"
    loop: "{{ lookup('fileglob', 'configs/*.conf') }}"

  - name: Show all files in directory
    debug:
      msg: "File: {{ item }}"
    loop: "{{ lookup('fileglob', '/var/log/*.log') }}"
```

---

## 6. Продвинутые циклы

### Цикл с подэлементами (subelements)

```yaml
vars:
  users:
    - name: alice
      authorized:
        - ssh-rsa AAAA...
        - ssh-rsa BBBB...
    - name: bob
      authorized:
        - ssh-rsa CCCC...

tasks:
  - name: Add SSH keys for users
    authorized_key:
      user: "{{ item.0.name }}"
      key: "{{ item.1 }}"
      state: present
    loop: "{{ users | subelements('authorized') }}"
    # item.0 - это элемент из списка users
    # item.1 - это элемент из подсписка authorized
```

### Цикл с control (управление выводом)

```yaml
tasks:
  - name: Install packages with labels
    apt:
      name: "{{ item }}"
      state: present
    loop:
      - nginx
      - curl
      - git
    loop_control:
      label: "{{ item }}" # Что показывать в выводе вместо всего item
      pause: 2 # Пауза между итерациями (в секундах)
      index_var: my_index # Переменная с индексом итерации

  - name: Show index
    debug:
      msg: "Iteration {{ my_index }}: {{ item }}"
    loop:
      - first
      - second
      - third
    loop_control:
      index_var: my_index
```

### Цикл с уникальными значениями

```yaml
tasks:
  - name: Install unique packages
    apt:
      name: "{{ item }}"
      state: present
    loop: "{{ packages | unique }}"
    vars:
      packages:
        - nginx
        - curl
        - nginx # Дубликат
        - git
        - curl # Дубликат
    # Установит только: nginx, curl, git
```

### Цикл с сортировкой

```yaml
tasks:
  - name: Install packages in alphabetical order
    apt:
      name: "{{ item }}"
      state: present
    loop: "{{ packages | sort }}"
    vars:
      packages:
        - nginx
        - curl
        - git
        - apache2
```

### Цикл с ограничением количества

```yaml
tasks:
  - name: Process first 5 items
    debug:
      msg: "Processing {{ item }}"
    loop: "{{ items[:5] }}"
    vars:
      items:
        - item1
        - item2
        - item3
        - item4
        - item5
        - item6
        - item7
```

---

## 7. Циклы с условиями

### Базовый цикл с when

```yaml
tasks:
  - name: Install packages based on OS
    package:
      name: "{{ item.name }}"
      state: present
    loop:
      - { name: "nginx", os: "all" }
      - { name: "certbot", os: "Debian" }
      - { name: "epel-release", os: "RedHat" }
    when: item.os == 'all' or item.os == ansible_os_family
```

### Цикл с проверкой существования

```yaml
tasks:
  - name: Create directories if variable is defined
    file:
      path: "{{ item }}"
      state: directory
    loop: "{{ directories | default([]) }}"
    when: directories is defined
    vars:
      directories:
        - /opt/app
        - /var/log/app
```

### Цикл с фильтрацией

```yaml
vars:
  packages:
    - name: nginx
      enabled: true
    - name: apache2
      enabled: false
    - name: curl
      enabled: true

tasks:
  - name: Install only enabled packages
    apt:
      name: "{{ item.name }}"
      state: present
    loop: "{{ packages | selectattr('enabled', 'equalto', true) | list }}"
    # Установит только: nginx, curl
```

### Цикл с условным пропуском

```yaml
tasks:
  - name: Process items conditionally
    debug:
      msg: "Processing {{ item }}"
    loop:
      - item1
      - item2
      - item3
    when: item != "item2"
    # Пропустит item2
```

---

## 8. Практические примеры

### Пример 1: Установка разных пакетов для разных ОС

```yaml
---
- hosts: all
  become: yes

  tasks:
    - name: Install OS-specific packages
      package:
        name: "{{ item.name }}"
        state: present
      loop:
        - { name: "nginx", os: "all" }
        - { name: "certbot", os: "Debian" }
        - { name: "python3-certbot-nginx", os: "Debian" }
        - { name: "epel-release", os: "RedHat" }
        - { name: "certbot", os: "RedHat" }
        - { name: "python3-certbot-nginx", os: "RedHat" }
      when: item.os == 'all' or item.os == ansible_os_family
```

### Пример 2: Создание пользователей с SSH-ключами

```yaml
---
- hosts: all
  become: yes
  vars:
    users:
      - name: alice
        shell: /bin/bash
        groups: sudo,docker
        ssh_keys:
          - ssh-rsa AAAA...
          - ssh-rsa BBBB...
      - name: bob
        shell: /bin/zsh
        groups: docker
        ssh_keys:
          - ssh-rsa CCCC...

  tasks:
    - name: Create users
      user:
        name: "{{ item.name }}"
        shell: "{{ item.shell }}"
        groups: "{{ item.groups }}"
        append: yes
        create_home: yes
        state: present
      loop: "{{ users }}"

    - name: Add SSH keys
      authorized_key:
        user: "{{ item.0.name }}"
        key: "{{ item.1 }}"
        state: present
      loop: "{{ users | subelements('ssh_keys') }}"
```

### Пример 3: Настройка виртуальных хостов Nginx

```yaml
---
- hosts: webservers
  become: yes
  vars:
    vhosts:
      - domain: example.com
        port: 80
        root: /var/www/example
      - domain: api.example.com
        port: 8080
        root: /var/www/api

  tasks:
    - name: Create document roots
      file:
        path: "{{ item.root }}"
        state: directory
        owner: www-data
        mode: "0755"
      loop: "{{ vhosts }}"

    - name: Deploy vhost configs
      template:
        src: templates/vhost.conf.j2
        dest: "/etc/nginx/sites-available/{{ item.domain }}.conf"
      loop: "{{ vhosts }}"
      notify: Reload Nginx

    - name: Enable vhosts
      file:
        src: "/etc/nginx/sites-available/{{ item.domain }}.conf"
        dest: "/etc/nginx/sites-enabled/{{ item.domain }}.conf"
        state: link
      loop: "{{ vhosts }}"
      notify: Reload Nginx

  handlers:
    - name: Reload Nginx
      systemd:
        name: nginx
        state: reloaded
```

**`templates/vhost.conf.j2`:**

```jinja2
server {
    listen {{ item.port }};
    server_name {{ item.domain }};
    root {{ item.root }};

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Пример 4: Мониторинг и алерты

```yaml
---
- hosts: all
  tasks:
    - name: Check system resources
      block:
        - name: Get memory usage
          set_fact:
            memory_used_percent: "{{ ((ansible_memtotal_mb - ansible_memfree_mb) / ansible_memtotal_mb * 100) | round(2) }}"

        - name: Get disk usage
          shell: df -h / | awk 'NR==2 {print $5}' | sed 's/%//'
          register: disk_usage
          changed_when: false

        - name: Set disk usage fact
          set_fact:
            disk_used_percent: "{{ disk_usage.stdout | int }}"

        - name: Show system status
          debug:
            msg: |
              Host: {{ ansible_hostname }}
              Memory: {{ memory_used_percent }}% used
              Disk: {{ disk_used_percent }}% used

        - name: Alert if memory is low
          debug:
            msg: "WARNING: Low memory on {{ ansible_hostname }}!"
          when: memory_used_percent | float > 90

        - name: Alert if disk is full
          debug:
            msg: "WARNING: Disk almost full on {{ ansible_hostname }}!"
          when: disk_used_percent | int > 85

        - name: Fail if critical
          fail:
            msg: "Critical resource usage on {{ ansible_hostname }}"
          when: memory_used_percent | float > 95 or disk_used_percent | int > 95
```

### Пример 5: Динамическая настройка на основе памяти

```yaml
---
- hosts: all
  become: yes

  tasks:
    - name: Calculate workers based on RAM
      set_fact:
        workers: >-
          {%- if ansible_memtotal_mb < 1024 -%}1
          {%- elif ansible_memtotal_mb < 2048 -%}2
          {%- elif ansible_memtotal_mb < 4096 -%}4
          {%- elif ansible_memtotal_mb < 8192 -%}8
          {%- else -%}16
          {%- endif -%}

    - name: Show calculated workers
      debug:
        msg: "Server {{ ansible_hostname }}: {{ ansible_memtotal_mb }}MB RAM, {{ workers }} workers"

    - name: Deploy app config
      template:
        src: templates/app.conf.j2
        dest: /etc/app/app.conf
      notify: Restart App

  handlers:
    - name: Restart App
      systemd:
        name: myapp
        state: restarted
```

**`templates/app.conf.j2`:**

```jinja2
[app]
workers = {{ workers }}
memory_limit = {{ (ansible_memtotal_mb * 0.8) | int }}MB
```

---

## 9. Best Practices

### ✅ Хорошие практики

1. **Используй специализированные модули вместо shell**
   - ✅ `when: ansible_os_family == "Debian"`
   - ❌ `shell: cat /etc/os-release | grep Ubuntu`

2. **Сравнивай числа с числами**
   - ✅ `when: ansible_memtotal_mb >= 2048`
   - ❌ `when: ansible_memtotal_mb >= "2048"`

3. **Используй `loop_control: label` для читаемого вывода**
   - Показывай только важную информацию в циклах

4. **Проверяй существование переменных перед использованием**
   - ✅ `when: my_var is defined`
   - ❌ `when: my_var` (упадет, если my_var не определена)

5. **Используй `changed_when: false` для команд, которые не меняют состояние**
   - Избегай ложных `CHANGED` статусов

6. **Используй `ignore_errors: yes` с осторожностью**
   - Только когда действительно нужно продолжить при ошибке
   - Всегда проверяй результат через `register`

7. **Группируй связанные задачи через `block`**
   - Логическая группировка + обработка ошибок

8. **Используй `set_fact` для сложных вычислений**
   - Сохраняй промежуточные результаты

### ❌ Антипаттерны

1. **Использование shell для получения информации, доступной через facts**
   - ❌ `shell: hostname`
   - ✅ `{{ ansible_hostname }}`

2. **Дублирование кода вместо циклов**
   - ❌ Три одинаковые задачи для разных пакетов
   - ✅ Одна задача с `loop`

3. **Жесткое кодирование значений**
   - ❌ `when: ansible_distribution == "Ubuntu"` (если нужно поддержать Debian)
   - ✅ `when: ansible_os_family == "Debian"`

4. **Игнорирование ошибок без проверки**
   - ❌ `ignore_errors: yes` без `register`
   - ✅ `ignore_errors: yes` + `register: result` + проверка `result`

5. **Слишком сложные условия**
   - Разбивай сложные `when` на несколько задач или используй `set_fact`

6. **Циклы без `loop_control: label`**
   - Вывод может быть нечитаемым при больших списках

---

## 10. Вопросы для собеседования

### Базовые вопросы

**Q: Как работает условие `when` в Ansible?**
A: Условие `when` определяет, должна ли задача выполняться. Если условие возвращает `true`, задача выполняется, иначе пропускается (статус `SKIPPED`).

**Q: В чем разница между `loop` и `with_items`?**
A: `loop` — это современный синтаксис, рекомендуемый для использования. `with_items` — старый синтаксис, который все еще работает, но считается устаревшим.

**Q: Как выполнить задачу только на Ubuntu?**
A: Использовать условие: `when: ansible_distribution == "Ubuntu"` или `when: ansible_os_family == "Debian"` (для всех Debian-based).

**Q: Как проверить, что переменная определена?**
A: Использовать `when: my_var is defined`.

**Q: Что делает `register` в задаче?**
A: Сохраняет результат выполнения задачи в переменную, которую можно использовать в последующих задачах.

### Продвинутые вопросы

**Q: Как выполнить задачу только если другая задача вернула `CHANGED`?**
A: Использовать `register` и проверить `result.changed`: `when: result is changed`.

**Q: Как обработать ошибку в задаче и продолжить выполнение?**
A: Использовать `ignore_errors: yes` и проверить результат через `register`: `when: result is failed`.

**Q: Как использовать вложенные циклы?**
A: Использовать `subelements`: `loop: "{{ users | subelements('ssh_keys') }}"`.

**Q: Как сделать цикл с паузой между итерациями?**
A: Использовать `loop_control: pause: 2` (пауза 2 секунды).

**Q: Как отфильтровать элементы в цикле?**
A: Использовать фильтры Jinja2: `loop: "{{ packages | selectattr('enabled', 'equalto', true) | list }}"`.

**Q: В чем разница между `include_tasks` и `import_tasks` в контексте циклов?**
A: `include_tasks` поддерживает циклы (можно передать `loop`), а `import_tasks` не поддерживает (статический импорт).

**Q: Как получить индекс текущей итерации в цикле?**
A: Использовать `loop_control: index_var: my_index`, затем использовать `{{ my_index }}` в задаче.

**Q: Как сравнить версии в условиях?**
A: Использовать фильтр `version`: `when: ansible_distribution_version is version('20.04', '>=')`.

**Q: Как выполнить задачу только на первом хосте из группы?**
A: Использовать `when: inventory_hostname == groups['webservers'][0]`.

**Q: Что произойдет, если в цикле одна итерация упадет с ошибкой?**
A: По умолчанию playbook остановится. Можно использовать `ignore_errors: yes` или `block/rescue` для обработки ошибок.

**Q: Как использовать регулярные выражения в условиях?**
A: Использовать тесты `match` (полное совпадение) или `search` (частичное совпадение): `when: hostname is match("web[0-9]+")`.

**Q: Как объединить несколько списков в одном цикле?**
A: Использовать фильтр `concat` или `+`: `loop: "{{ list1 + list2 }}"`.

**Q: Как сделать цикл уникальным (без дубликатов)?**
A: Использовать фильтр `unique`: `loop: "{{ packages | unique }}"`.

---

## 📚 Дополнительные ресурсы

- Официальная документация по условиям: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_conditionals.html
- Официальная документация по циклам: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_loops.html
- Jinja2 filters: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_filters.html

---
