# 🧩 Ansible Roles: Детальная шпаргалка

## 📋 Оглавление

1. [Что такое Роли и зачем они нужны](#1-что-такое-роли-и-зачем-они-нужны)
2. [Стандартная структура директорий](#2-стандартная-структура-директорий)
3. [Как подключать роли в Playbook](#3-как-подключать-роли-в-playbook)
4. [Передача параметров и переопределение переменных](#4-передача-параметров-и-переопределение-переменных)
5. [Зависимости ролей (Role Dependencies)](#5-зависимости-ролей-role-dependencies)
6. [Динамическое vs Статическое подключение](#6-динамическое-vs-статическое-подключение)
7. [Создание и установка ролей (Ansible Galaxy)](#7-создание-и-установка-ролей-ansible-galaxy)
8. [Практический пример: Роль Nginx](#8-практический-пример-роль-nginx)
9. [Best Practices](#9-best-practices)
10. [Вопросы для собеседования](#10-вопросы-для-собеседования)

---

## 1. Что такое Роли и зачем они нужны

**Роли (Roles)** — это стандартный способ организации кода в Ansible для обеспечения модульности, переиспользования и чистоты.

**Проблема без ролей:** Огромный `playbook.yml` на 1000 строк, в котором перемешаны установка пакетов, копирование конфигов, шаблоны и хендлеры. Это невозможно читать и поддерживать.

**Решение (Роли):** Разделение кода на логические блоки. Одна роль = одна ответственность (например, роль `nginx`, роль `postgresql`, роль `firewall`).

**Преимущества:**

- ✅ **Модульность:** Код разбит на независимые части.
- ✅ **Переиспользование:** Роль `nginx` можно подключить в 10 разных плейбуках.
- ✅ **Чистота:** Плейбук становится коротким и читаемым (как список покупок).
- ✅ **Автоматическая загрузка файлов:** Ansible сам знает, где искать шаблоны и файлы внутри роли.

---

## 2. Стандартная структура директорий

Ansible жестко регламентирует структуру папок внутри роли. Если файлы лежат в этих папках, модули `copy`, `template`, `include_tasks` будут искать их там **автоматически** (не нужно писать полные пути).

```text
roles/
  nginx/
    tasks/
      main.yml          # 🟢 ОБЯЗАТЕЛЬНО. Основные задачи роли
    handlers/
      main.yml          # Обработчики (перезапуск сервисов)
    templates/
      nginx.conf.j2     # Шаблоны Jinja2 (для модуля template)
    files/
      index.html        # Статические файлы (для модуля copy)
    vars/
      main.yml          # 🔴 Внутренние переменные роли (ВЫСОКИЙ приоритет)
    defaults/
      main.yml          # 🟢 Дефолтные переменные (НИЗКИЙ приоритет, легко перебить)
    meta/
      main.yml          # Зависимости от других ролей и метаданные
    tests/
      inventory         # Тестовый инвентарь
      test.yml          # Тестовый playbook для проверки роли
```

### Как это работает "под капотом":

Если в `roles/nginx/tasks/main.yml` ты напишешь:
`template: src=nginx.conf.j2 dest=/etc/nginx/nginx.conf`
Ansible **сам** поймет, что шаблон нужно искать в `roles/nginx/templates/nginx.conf.j2`.

---

## 3. Как подключать роли в Playbook

### Базовое подключение (список)

Роли выполняются **ДО** блока `tasks` (и после `pre_tasks`).

```yaml
---
- hosts: webservers
  become: yes

  pre_tasks:
    - name: Update apt cache
      apt: update_cache=yes

  roles:
    - common # Сначала базовая настройка
    - nginx # Потом установка Nginx
    - postgresql # Потом установка БД

  tasks:
    - name: Deploy application code
      git: repo=... dest=/var/www/app
```

### Подключение с условиями и тегами

```yaml
roles:
  - role: nginx
    when: ansible_os_family == "Debian"
    tags: ["web", "nginx"]

  - role: monitoring
    vars:
      monitor_port: 9100 # Передача переменной прямо при подключении
```

---

## 4. Передача параметров и переопределение переменных

Переменные в `defaults/main.yml` созданы для того, чтобы пользователь мог их легко перебить.

### Способ 1: Через `vars` при подключении роли

```yaml
roles:
  - role: nginx
    vars:
      nginx_port: 8080
      nginx_worker_processes: 4
```

### Способ 2: Через `group_vars` или `host_vars`

Просто объяви переменную `nginx_port: 8080` в `group_vars/webservers.yml`. Она автоматически переберет значение из `defaults/main.yml`.

### Способ 3: Через Extra Vars (при запуске)

```bash
ansible-playbook site.yml -e "nginx_port=9000"
```

---

## 5. Зависимости ролей (Role Dependencies)

Если роль `myapp` требует, чтобы на сервере уже был установлен `nginx`, ты можешь прописать это в `meta/main.yml`. Ansible сам установит `nginx` перед `myapp`.

**`roles/myapp/meta/main.yml`:**

```yaml
---
galaxy_info:
  author: your_name
  description: My Awesome App
  min_ansible_version: 2.9

dependencies:
  - role: nginx
    vars:
      nginx_port: 80 # Можно передать переменные в зависимость

  - role: common

  - role: postgresql
    when: enable_db # Зависимость выполнится только при условии
```

**Важно:** По умолчанию Ansible выполнит роль-зависимость только **один раз**, даже если она подключена в нескольких местах. Чтобы разрешить дубликаты, добавь в `meta/main.yml`:

```yaml
allow_duplicates: yes
```

---

## 6. Динамическое vs Статическое подключение

Это **любимая тема на собеседованиях**.

### `import_role` (Статический)

Парсится на этапе **чтения** playbook (до выполнения).

```yaml
tasks:
  - import_role:
      name: nginx
```

- ✅ Быстрее работает.
- ✅ Можно использовать `--list-tasks` для просмотра всех задач.
- ❌ Не поддерживает `loop` (циклы).
- ❌ Не поддерживает `when` на уровне самой конструкции (только внутри задач).

### `include_role` (Динамический)

Парсится на этапе **выполнения** playbook.

```yaml
tasks:
  - include_role:
      name: nginx
    when: install_webserver
    loop:
      - web1
      - web2
```

- ✅ Поддерживает `loop`, `when`, переменные.
- ✅ Можно подключать роли "на лету" в зависимости от условий.
- ❌ Медленнее.
- ❌ `--list-tasks` не покажет задачи внутри включенной роли.

---

## 7. Создание и установка ролей (Ansible Galaxy)

### Создание своей роли

Не нужно создавать папки вручную! Используй CLI:

```bash
ansible-galaxy init roles/nginx
```

Эта команда создаст всю стандартную структуру с заготовками `main.yml`.

### Установка чужих ролей (Ansible Galaxy)

Не пиши велосипеды! В [Ansible Galaxy](https://galaxy.ansible.com/) есть тысячи готовых ролей (Nginx, Docker, Kubernetes, PostgreSQL).

**`requirements.yml`:**

```yaml
---
roles:
  - name: geerlingguy.nginx
    version: 3.1.0
  - name: geerlingguy.docker
    src: https://github.com/geerlingguy/ansible-role-docker
```

**Установка:**

```bash
ansible-galaxy install -r requirements.yml
# Роли скачаются в ~/.ansible/roles/ или ./roles/
```

---

## 8. Практический пример: Роль Nginx

### `roles/nginx/defaults/main.yml`

```yaml
---
nginx_port: 80
nginx_server_name: "{{ ansible_fqdn }}"
nginx_worker_processes: auto
```

### `roles/nginx/tasks/main.yml`

```yaml
---
- name: Install Nginx
  apt:
    name: nginx
    state: present
    update_cache: yes
  when: ansible_os_family == "Debian"

- name: Deploy Nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    validate: /usr/sbin/nginx -t -f %s # Проверка синтаксиса!
  notify: Restart Nginx

- name: Ensure Nginx is started and enabled
  systemd:
    name: nginx
    state: started
    enabled: yes
```

### `roles/nginx/handlers/main.yml`

```yaml
---
- name: Restart Nginx
  systemd:
    name: nginx
    state: restarted
```

### `roles/nginx/templates/nginx.conf.j2`

```jinja2
# {{ ansible_managed }}
worker_processes {{ nginx_worker_processes }};

events {
    worker_connections 1024;
}

http {
    server {
        listen {{ nginx_port }};
        server_name {{ nginx_server_name }};

        location / {
            root /var/www/html;
            index index.html;
        }
    }
}
```

### Использование в `site.yml`

```yaml
---
- hosts: webservers
  become: yes
  roles:
    - role: nginx
      vars:
        nginx_port: 8080
```

---

## 9. Best Practices

### ✅ Хорошие практики

1. **Одна роль = одна ответственность**
   - Не делай "роль для всего". Делай `nginx`, `postgresql`, `firewall`.
2. **Используй `defaults/` для настроек, `vars/` для констант**
   - В `defaults/` пиши то, что пользователь захочет перебить (порты, пути).
   - В `vars/` пиши внутренние пути или имена пакетов, которые менять нельзя.
3. **Всегда валидируй конфиги**
   - Используй `validate: nginx -t -f %s` в модуле `template`.
4. **Используй `ansible_managed`**
   - Добавляй `# {{ ansible_managed }}` в шаблоны, чтобы админы знали, что файл генерируется Ansible.
5. **Тестируй роли локально**
   - Используй папку `tests/` внутри роли для быстрого прогона `test.yml`.
6. **Скачивай готовое из Galaxy**
   - Перед тем как писать роль для Docker или Nginx, проверь, не сделал ли это кто-то до тебя (например, Jeff Geerling).

### ❌ Антипаттерны

1. **Хардкодинг путей в `tasks/`**
   - ❌ `copy: src=roles/nginx/files/index.html ...`
   - ✅ `copy: src=index.html ...` (Ansible сам найдет в `files/`)
2. **Смешивание логики и данных**
   - Не пиши `when: ansible_os_family == "Debian"` в каждой задаче. Вынеси имена пакетов в `vars/Debian.yml` и подключай через `include_vars`.
3. **Гигантские роли**
   - Если роль больше 500 строк, разбей её на под-роли или используй `include_tasks` для разбивки `tasks/main.yml` на отдельные файлы.

---

## 10. Вопросы для собеседования

### Базовые вопросы

**Q: Что такое роль в Ansible и какова её стандартная структура?**
A: Роль — это модуль переиспользуемого кода. Стандартная структура включает папки: `tasks`, `handlers`, `templates`, `files`, `vars`, `defaults`, `meta`.

**Q: В чем разница между `vars/main.yml` и `defaults/main.yml`?**
A: `defaults` имеют самый низкий приоритет и предназначены для значений, которые пользователь может легко перебить. `vars` имеют высокий приоритет и используются для внутренних констант роли, которые перебивать не рекомендуется.

**Q: Как Ansible находит файлы внутри роли?**
A: Если использовать модули `copy`, `template`, `include_tasks` без указания полного пути, Ansible автоматически ищет файлы в соответствующих папках роли (`files/`, `templates/`, `tasks/`).

**Q: Как передать переменную в роль?**
A: Через блок `vars` при подключении роли, через `group_vars`/`host_vars`, или через Extra Vars (`-e`) при запуске.

### Продвинутые вопросы

**Q: В чем разница между `import_role` и `include_role`?**
A: `import_role` — статический, парсится при загрузке playbook (быстрее, но не поддерживает циклы и условия на уровне подключения). `include_role` — динамический, парсится во время выполнения (поддерживает `loop`, `when`, но медленнее).

**Q: Как настроить зависимости между ролями?**
A: В файле `meta/main.yml` роли нужно указать список зависимостей в блоке `dependencies:`. Ansible выполнит их перед самой ролью.

**Q: Что будет, если две разные роли зависят от одной и той же роли? Выполнится ли она дважды?**
A: По умолчанию Ansible выполнит роль-зависимость только один раз (дедупликация). Чтобы разрешить повторное выполнение, нужно добавить `allow_duplicates: yes` в `meta/main.yml` зависимости.

**Q: Как проверить синтаксис конфига, сгенерированного из шаблона, до того как он перезапишет оригинал?**
A: Использовать параметр `validate` в модуле `template`. Например: `validate: /usr/sbin/nginx -t -f %s`, где `%s` заменяется на путь к временному файлу.

**Q: Как установить сторонние роли из Ansible Galaxy?**
A: Создать файл `requirements.yml` со списком ролей и выполнить команду `ansible-galaxy install -r requirements.yml`.

---

## 📚 Дополнительные ресурсы

- Официальная документация по ролям: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html
- Ansible Galaxy: https://galaxy.ansible.com/
- Best Practices: https://docs.ansible.com/ansible/latest/tips_tricks/ansible_tips_tricks.html#role-directory-structure

---
