# 📘 Ansible: Полный справочник (Master Guide)

Этот документ является главным справочником по Ansible. Он охватывает всё: от базовых концепций до продвинутых паттернов, необходимых для реальной работы и технических собеседований.

## 📋 Оглавление

1. [Фундамент](#1-фундамент)
2. [Инвентарь (Inventory)](#2-инвентарь-inventory)
3. [Ad-Hoc команды](#3-ad-hoc-команды)
4. [Playbook (Плейбуки)](#4-playbook-плейбуки)
5. [Handlers (Обработчики)](#5-handlers-обработчики)
6. [Переменные и их приоритеты](#6-переменные-и-их-приоритеты)
7. [Facts (Факты)](#7-facts-факты)
8. [Условия (when) и Циклы (loop)](#8-условия-when-и-циклы-loop)
9. [Шаблоны (Jinja2)](#9-шаблоны-jinja2)
10. [Роли (Roles)](#10-роли-roles)
11. [Best Practices и Отладка](#11-best-practices-и-отладка)

---

## 1. Фундамент

- **Agentless**: Не требует установки агентов на целевые серверы. Нужен только SSH и Python (обычно Python 3). Ansible подключается, загружает модуль (Python-скрипт), выполняет его и удаляет.
- **Декларативный подход**: Мы описываем *желаемое состояние* системы, а не пошаговые инструкции для его достижения.
- **Идемпотентность**: Повторный запуск одной и той же задачи не ломает систему, а приводит её к нужному состоянию. Если состояние уже достигнуто, задача возвращает `OK` (зелёный), а не `CHANGED` (жёлтый).
- **Модули**: Всегда используй специализированные модули (`apt`, `copy`, `systemd`), а не `shell` или `command`, чтобы гарантировать идемпотентность.

---

## 2. Инвентарь (Inventory)

Список серверов, управляемых Ansible. Поддерживает форматы INI и YAML.

**INI формат:**
~~~ini
[webservers]
web1.example.com ansible_user=admin ansible_port=22

[dbservers]
db1.example.com ansible_user=postgres
~~~

**YAML формат (современный):**
~~~yaml
all:
  children:
    webservers:
      hosts:
        web1.example.com:
          ansible_user: admin
    dbservers:
      hosts:
        db1.example.com:
          ansible_user: postgres
~~~

---

## 3. Ad-Hoc команды

Быстрые однострочники для выполнения задач без создания плейбуков.

**Синтаксис:** `ansible <pattern> -m <module> -a "<arguments>" [options]`

**Примеры:**
~~~bash
# Проверка связи
ansible all -m ping

# Выполнение команды (command - безопаснее, shell - поддерживает пайпы |)
ansible all -a "uptime"
ansible all -m shell -a "ps aux | grep nginx | wc -l"

# Управление пакетами (с sudo через флаг -b)
ansible webservers -m apt -a "name=nginx state=present update_cache=yes" -b

# Копирование файла
ansible webservers -m copy -a "src=./nginx.conf dest=/etc/nginx/ mode=0644" -b

# Перезапуск сервиса
ansible webservers -m systemd -a "name=nginx state=restarted" -b
~~~

**Ключевые флаги:**
- `-i inventory.yml` : Путь к файлу инвентаря
- `-b` (become) : Выполнить команду через sudo
- `-u username` : Пользователь для SSH-подключения
- `-f 10` : Количество параллельных потоков (forks, по умолчанию 5)
- `--check` : Dry-run (показать, что изменится, без реальных действий)

---

## 4. Playbook (Плейбуки)

YAML-файлы с декларативным описанием задач.

**Базовая структура:**
~~~yaml
---
- hosts: webservers          # Кому применяем (группа из inventory)
  become: yes                # Выполнять с sudo
  vars:                      # Переменные для этого Play
    app_port: 8080
    
  tasks:                     # Список задач
    - name: Install Nginx
      apt:
        name: nginx
        state: present
      notify: Restart Nginx  # Вызов handler'а

  handlers:                  # Обработчики
    - name: Restart Nginx
      systemd:
        name: nginx
        state: restarted
~~~

**Запуск:**
~~~bash
ansible-playbook -i inventory.yml playbook.yml
ansible-playbook playbook.yml -e "env=production"  # Передача extra vars
ansible-playbook playbook.yml --check --diff       # Проверка с показом изменений
ansible-playbook playbook.yml --tags "install"     # Запуск только задач с тегом
~~~

---

## 5. Handlers (Обработчики)

Специальные задачи, которые выполняются **только при статусе `CHANGED`** и **строго в конце Play**.

**Ключевые правила:**
1. Срабатывают только если задача, вызвавшая их через `notify`, вернула `CHANGED`.
2. Выполняются **один раз**, даже если `notify` был вызван 100 раз (дедупликация).
3. Выполняются в порядке их **объявления** в секции `handlers:`, а не в порядке вызова.
4. При ошибке (`FAILED`) в задачах handlers **не выполняются** (если не задан `force_handlers: yes`).

**Принудительный вызов (не дожидаясь конца Play):**
~~~yaml
  tasks:
    - name: Update DB config
      template: { src: db.j2, dest: /etc/db.conf }
      notify: Restart DB
      
    - name: Force handler execution NOW
      meta: flush_handlers
      
    - name: Run migrations (теперь БД уже перезагружена с новым конфигом)
      command: /opt/app/migrate.sh
~~~

---

## 6. Переменные и их приоритеты

Иерархия приоритетов (от **низшего** к **высшему**). Чем выше, тем сильнее перебивает остальные:

1. 🟢 **Role Defaults** (`roles/x/defaults/main.yml`) — самые слабые, значения по умолчанию.
2. 🟡 **Inventory Vars** (`group_vars/`, `host_vars/`) — переменные групп или хостов.
3. 🟠 **Playbook Vars** (блок `vars:` или `vars_files:` внутри плейбука).
4. 🔴 **Role Vars** (`roles/x/vars/main.yml`) — внутренние переменные роли.
5. ⚫ **Extra Vars** (флаг `-e` или `--extra-vars` при запуске) — **абсолютный король**, перебивает всё.

**Использование:** `port: "{{ app_port }}"` (всегда бери в кавычки, если это единственное значение).

---

## 7. Facts (Факты)

Переменные, которые Ansible автоматически собирает с целевого сервера при подключении. 
Посмотреть все: `ansible <host> -m setup`

**Самые популярные:**
- `ansible_hostname` : Короткое имя хоста
- `ansible_fqdn` : Полное доменное имя
- `ansible_default_ipv4.address` : Основной IP-адрес
- `ansible_os_family` : Семейство ОС (`Debian`, `RedHat`, `Suse`)
- `ansible_distribution` : Дистрибутив (`Ubuntu`, `CentOS`, `AlmaLinux`)
- `ansible_memtotal_mb` : Объем RAM в МБ
- `ansible_processor_vcpus` : Количество ядер CPU

---

## 8. Условия (when) и Циклы (loop)

**Базовый цикл:**
~~~yaml
- name: Install packages
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - curl
    - git
~~~

**Базовое условие:**
~~~yaml
- name: Install only on Debian
  apt:
    name: certbot
    state: present
  when: ansible_os_family == "Debian"
~~~

**🔥 Продвинутый сценарий (разные пакеты для разных ОС):**
~~~yaml
- name: Install OS-specific packages
  package:  # Универсальный модуль (сам выберет apt или yum)
    name: "{{ item.name }}"
    state: present
  loop:
    - { name: 'nginx', os: 'all' }
    - { name: 'certbot', os: 'Debian' }
    - { name: 'epel-release', os: 'RedHat' }
  when: item.os == 'all' or item.os == ansible_os_family
~~~

---

## 9. Шаблоны (Jinja2)

Модуль `template` генерирует уникальные конфигурационные файлы, подставляя переменные.

**Файл `templates/app.conf.j2`:**
~~~jinja2
server {
    listen {{ app_port }};
    server_name {{ ansible_fqdn }};
    
    {% if enable_ssl %}
    ssl_certificate /etc/ssl/certs/{{ ansible_hostname }}.crt;
    {% endif %}
}
~~~

**Задача в плейбуке:**
~~~yaml
- name: Deploy config
  template:
    src: templates/app.conf.j2
    dest: /etc/app/app.conf
    mode: '0644'
  notify: Restart App
~~~

**Полезные фильтры Jinja2:**
- `{{ var | default('value') }}` : Значение по умолчанию
- `{{ list | join(',') }}` : Объединение списка в строку
- `{{ path | basename }}` : Извлечение имени файла из пути

---

## 10. Роли (Roles)

Стандартный способ организации кода для переиспользования и модульности.

**Структура папок роли:**
~~~text
roles/nginx/
  tasks/main.yml      # Основные задачи (обязательно)
  handlers/main.yml   # Обработчики
  templates/          # Шаблоны (.j2)
  files/              # Статические файлы
  vars/main.yml       # Переменные (высокий приоритет)
  defaults/main.yml   # Дефолтные значения (низкий приоритет)
  meta/main.yml       # Зависимости от других ролей
~~~

**Использование в плейбуке:**
~~~yaml
- hosts: webservers
  roles:
    - common
    - nginx
    - { role: postgresql, when: ansible_os_family == "Debian" }
~~~

---

## 11. Best Practices и Отладка

### ✅ Хорошие практики
- Всегда давай задачам понятные имена (`name:`).
- Используй специализированные модули вместо `shell`/`command`.
- Применяй `notify` и `handlers` для перезапуска сервисов при изменении конфигов.
- Разбивай сложный код на роли.
- Храни секреты в Ansible Vault, а не в открытом виде.
- Используй `--check` перед реальным запуском в продакшене.

### ❌ Антипаттерны
- Использование `shell` без крайней необходимости (теряется идемпотентность).
- Дублирование кода вместо использования `loop`.
- Жёсткое кодирование (hardcoding) значений вместо переменных.
- Ожидание, что handler выполнится мгновенно (он выполняется в конце Play).

### 🔍 Команды для отладки
~~~bash
# Проверка синтаксиса YAML
ansible-playbook playbook.yml --syntax-check

# Максимальная детализация вывода
ansible-playbook playbook.yml -vvv

# Запуск начиная с конкретной задачи (если плейбук упал посередине)
ansible-playbook playbook.yml --start-at-task="Name of the task"
~~~

---
*Документ готов к использованию. Сохраните его как основу вашей базы знаний по Ansible.*
