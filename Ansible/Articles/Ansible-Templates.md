# 📝 Ansible Templates & Jinja2: Детальная шпаргалка

## 📋 Оглавление

1. [Что такое Templates и Jinja2](#1-что-такое-templates-и-jinja2)
2. [Базовый синтаксис Jinja2](#2-базовый-синтаксис-jinja2)
3. [Фильтры (Filters)](#3-фильтры-filters)
4. [Тесты (Tests)](#4-тесты-tests)
5. [Модуль template в Ansible](#5-модуль-template-в-ansible)
6. [Продвинутые паттерны](#6-продвинутые-паттерны)
7. [Практические примеры](#7-практические-примеры)
8. [Best Practices](#8-best-practices)
9. [Вопросы для собеседования](#9-вопросы-для-собеседования)

---

## 1. Что такое Templates и Jinja2

**Templates (Шаблоны)** — это файлы конфигурации, в которые Ansible динамически подставляет переменные, результаты вычислений и логику перед тем, как скопировать их на целевой сервер.

**Jinja2** — это движок шаблонизации для Python, который Ansible использует "под капотом". Он позволяет писать не просто текст, а полноценные программы внутри конфига.

**Зачем нужны:**

- ✅ Генерация уникальных конфигов для каждого сервера (IP, hostname, порты).
- ✅ Динамическое создание списков (например, бэкендов для Nginx/HAProxy).
- ✅ Условное включение блоков конфига (например, SSL только если есть сертификаты).
- ✅ Избавление от хардкода и дублирования.

---

## 2. Базовый синтаксис Jinja2

### Переменные (Вывод значений)

Двойные фигурные скобки `{{ ... }}` выводят значение переменной.

```jinja2
# Простая переменная
listen {{ app_port }};

# Вложенные словари
server_name {{ ansible_fqdn }};

# Элементы списка
worker_processes {{ workers[0] }};
```

### Комментарии

Фигурные скобки с хэшем `{# ... #}`. Не попадают в итоговый файл.

```jinja2
# Это комментарий Jinja2, в итоговом файле его не будет
{# TODO: Добавить поддержку IPv6 в будущем #}
listen {{ app_port }};
```

### Условные конструкции (If / Elif / Else)

Процентные скобки `{% ... %}` используются для управляющих конструкций.

```jinja2
{% if enable_ssl %}
    listen 443 ssl;
    ssl_certificate {{ ssl_cert_path }};
{% elif ansible_os_family == "Debian" %}
    listen 80;
{% else %}
    listen 8080;
{% endif %}
```

### Циклы (For)

Итерация по спискам или словарям.

```jinja2
# Цикл по списку
{% for upstream in backends %}
    server {{ upstream }};
{% endfor %}

# Цикл по словарю
{% for key, value in my_dict.items() %}
    {{ key }} = {{ value }};
{% endfor %}

# Цикл с условием (отфильтруем только активные)
{% for server in servers if server.enabled %}
    server {{ server.ip }};
{% endfor %}
```

### Управление пробелами (Whitespace Control)

По умолчанию Jinja2 сохраняет все переносы строк и пробелы вокруг тегов. Минус `-` убирает лишние пустые строки.

```jinja2
# Без контроля пробелов (может быть много пустых строк, если условие false)
{% if condition %}
line1
{% endif %}
line2

# С контролем пробелов (чистый вывод)
{%- if condition %}
line1
{%- endif %}
line2
```

---

## 3. Фильтры (Filters)

Фильтры применяются через пайп `|` и позволяют трансформировать данные прямо в шаблоне.

### Работа со строками

```jinja2
{{ "hello" | upper }}                  # HELLO
{{ "HELLO" | lower }}                  # hello
{{ "hello world" | capitalize }}       # Hello world
{{ "hello world" | title }}            # Hello World
{{ "hello" | replace("h", "j") }}      # jello
{{ "  spaces  " | trim }}              # "spaces" (убирает пробелы по краям)
{{ "a,b,c" | split(",") }}             # ['a', 'b', 'c'] (превращает строку в список)
```

### Работа со списками и словарями

```jinja2
{{ ['a', 'b', 'c'] | join(',') }}      # a,b,c (объединяет список в строку)
{{ ['a', 'b', 'a'] | unique }}         # ['a', 'b'] (убирает дубликаты)
{{ [3, 1, 2] | sort }}                 # [1, 2, 3] (сортировка)
{{ [3, 1, 2] | reverse }}              # [2, 1, 3] (реверс)
{{ my_list | first }}                  # Первый элемент
{{ my_list | last }}                   # Последний элемент
{{ my_list | length }}                 # Количество элементов
```

### Математика и форматирование

```jinja2
{{ 10.5 | round }}                     # 11 (округление)
{{ 10.5 | round(0, 'floor') }}         # 10 (округление вниз)
{{ 1024 | human_readable }}            # 1.0 KB (человекочитаемый размер)
{{ my_var | to_nice_yaml }}            # Красивый YAML
{{ my_var | to_json }}                 # Компактный JSON
{{ my_var | to_nice_json }}            # Красивый JSON с отступами
```

### Регулярные выражения

```jinja2
{{ "foo bar baz" | regex_replace("ba.", "xyz") }}  # foo xyz xyz
{{ "foo123" | regex_search("[0-9]+") }}            # 123 (находит первое совпадение)
{{ "foo123" | regex_findall("[0-9]+") }}           # ['123'] (находит все совпадения)
```

### Специальные Ansible-фильтры

```jinja2
{{ my_path | basename }}               # file.txt (имя файла из пути)
{{ my_path | dirname }}                # /etc/app (директория из пути)
{{ my_path | expanduser }}             # /home/user/.ssh (раскрывает ~)
{{ my_var | mandatory }}               # Упадёт с ошибкой, если my_var не определена!
{{ my_var | default("fallback") }}     # Значение по умолчанию, если не определена
```

---

## 4. Тесты (Tests)

Тесты проверяют состояние переменной и возвращают `true` или `false`. Используются с оператором `is`.

### Проверка существования и типов

```jinja2
{% if my_var is defined %}             # Переменная существует
{% if my_var is undefined %}           # Переменная не существует
{% if my_var is none %}                # Переменная равна None
{% if my_var is string %}              # Это строка
{% if my_var is number %}              # Это число
{% if my_var is mapping %}             # Это словарь (dict)
{% if my_var is iterable %}            # Это список или словарь
{% if my_var is match("^web.*") %}     # Строка полностью соответствует regex
{% if my_var is search("db") %}        # Строка содержит regex
```

### Сравнение версий (Очень важно для Ansible!)

```jinja2
{% if ansible_distribution_version is version('20.04', '>=') %}
    # Действие для Ubuntu 20.04 и новее
{% endif %}

{% if my_app_version is version('1.5.0', '<') %}
    # Действие для старых версий приложения
{% endif %}
```

---

## 5. Модуль `template` в Ansible

Модуль `template` берет `.j2` файл с управляющей машины, рендерит его и кладет на целевой сервер.

### Базовое использование

```yaml
tasks:
  - name: Deploy Nginx config
    template:
      src: templates/nginx.conf.j2 # Путь на Control Node (относительно роли или playbook)
      dest: /etc/nginx/nginx.conf # Путь на Managed Node
      owner: root
      group: root
      mode: "0644"
      backup: yes # Создаст бэкап .bak перед перезаписью
    notify: Reload Nginx
```

### Валидация конфига перед применением (Киллер-фича!)

Можно проверить синтаксис конфига _до_ того, как он перезапишет старый. Если проверка провалится, старый конфиг останется нетронутым.

```yaml
tasks:
  - name: Deploy and validate Nginx config
    template:
      src: templates/nginx.conf.j2
      dest: /etc/nginx/nginx.conf
      validate: /usr/sbin/nginx -t -f %s # %s заменяется на путь к временному файлу
    notify: Reload Nginx
```

### Магическая переменная `ansible_managed`

Ansible автоматически добавляет переменную `{{ ansible_managed }}`, которую полезно вставлять в шапку конфигов, чтобы админы знали, что файл управляется автоматикой.

```jinja2
# {{ ansible_managed }}
# Не редактируйте этот файл вручную!
listen {{ app_port }};
```

_Вывод будет примерно таким: `# Ansible managed: /path/to/template.j2 modified on 2023-10-25 14:30:00 by user`_

---

## 6. Продвинутые паттерны

### Доступ к переменным других хостов (hostvars)

Внутри шаблона можно обращаться к фактам и переменным _других_ серверов из инвентаря.

```jinja2
# Генерируем список всех IP из группы 'dbservers' для конфига балансировщика
{% for host in groups['dbservers'] %}
    server {{ hostvars[host]['ansible_default_ipv4']['address'] }}:5432;
{% endfor %}
```

### Макросы (Jinja2 Functions)

Макросы — это как функции в программировании. Позволяют переиспользовать куски шаблона.

```jinja2
{# Определяем макрос #}
{% macro render_server(ip, port, weight=1) %}
    server {{ ip }}:{{ port }} weight={{ weight }};
{% endmacro %}

# Используем макрос
{{ render_server('10.0.0.1', 8080) }}
{{ render_server('10.0.0.2', 8080, weight=5) }}
```

### Циклы с переменными `loop.index`

Внутри циклов `for` доступны специальные переменные:

- `loop.index` — текущая итерация (начинается с 1)
- `loop.index0` — текущая итерация (начинается с 0)
- `loop.first` — `true`, если это первая итерация
- `loop.last` — `true`, если это последняя итерация

```jinja2
{% for server in backends %}
    server {{ server }}{% if not loop.last %},{% endif %}
{% endfor %}
# Вывод: server1,server2,server3 (без запятой в конце!)
```

---

## 7. Практические примеры

### Пример 1: Динамический Nginx Upstream (Балансировка)

**`templates/upstream.conf.j2`:**

```jinja2
upstream backend_pool {
{% for host in groups['app_servers'] %}
    server {{ hostvars[host]['ansible_default_ipv4']['address'] }}:{{ app_port }};
{% endfor %}
}
```

### Пример 2: Конфиг HAProxy с условиями

**`templates/haproxy.cfg.j2`:**

```jinja2
global
    maxconn {{ haproxy_maxconn | default(4096) }}

defaults
    mode http
    timeout connect 5000ms

frontend http_front
    bind *:80
{% if enable_ssl %}
    bind *:443 ssl crt /etc/ssl/certs/site.pem
    http-request redirect scheme https unless { ssl_fc }
{% endif %}
    default_backend app_servers

backend app_servers
    balance roundrobin
{% for host in groups['app_servers'] %}
    {% set host_ip = hostvars[host]['ansible_default_ipv4']['address'] %}
    server {{ host }} {{ host_ip }}:8080 check
{% endfor %}
```

### Пример 3: Генерация JSON/YAML конфигов

Иногда нужно сгенерировать JSON. Используем фильтры `to_nice_json`.

**В playbook:**

```yaml
tasks:
  - name: Generate JSON config
    template:
      src: config.json.j2
      dest: /etc/app/config.json
```

**`templates/config.json.j2`:**

```jinja2
{{
  {
    "app_name": app_name,
    "port": app_port,
    "features": {
      "ssl": enable_ssl,
      "debug": debug_mode
    },
    "allowed_ips": allowed_ips
  }
| to_nice_json
}}
```

---

## 8. Best Practices

### ✅ Хорошие практики

1. **Всегда валидируй конфиги**
   - Используй параметр `validate` в модуле `template`. Это спасет от падения сервиса из-за опечатки.
2. **Используй `ansible_managed`**
   - Добавляй `# {{ ansible_managed }}` в начало конфигов, чтобы другие админы не правили их руками.
3. **Используй `default()` для необязательных переменных**
   - `{{ my_var | default('fallback') }}` предотвратит падение шаблона, если переменная не передана.
4. **Выноси сложную логику в `set_fact`**
   - Если в шаблоне слишком много `{% if %}` и вычислений, сделай это в задаче Ansible через `set_fact`, а в шаблон передавай уже готовые данные.
5. **Храни шаблоны в `templates/`**
   - При использовании ролей Ansible сам найдет файлы в папке `roles/myrole/templates/`. Не указывай полный путь.

### ❌ Антипаттерны

1. **Хардкодинг значений в шаблоне**
   - ❌ `listen 80;`
   - ✅ `listen {{ http_port }};`
2. **Слишком сложная бизнес-логика в Jinja2**
   - Jinja2 — для форматирования, а не для вычислений. Если нужно посчитать сумму портов, сделай это в Python/Ansible.
3. **Игнорирование пробелов**
   - Забудь про `-` в `{%- if -%}`, и получишь конфиг с кучей пустых строк, который невозможно читать.
4. **Использование `copy` вместо `template`**
   - Если в файле есть хотя бы одна переменная `{{ }}`, используй `template`, а не `copy`.

---

## 9. Вопросы для собеседования

### Базовые вопросы

**Q: В чем разница между модулями `copy` и `template`?**
A: `copy` просто копирует файл "как есть". `template` обрабатывает файл движком Jinja2, подставляет переменные и логику, и только потом кладет на сервер.

**Q: Как задать значение по умолчанию в шаблоне, если переменная не определена?**
A: Использовать фильтр `default`: `{{ my_var | default('fallback_value') }}`.

**Q: Как сделать так, чтобы Ansible не перезаписывал конфиг, если в нем синтаксическая ошибка?**
A: Использовать параметр `validate` в модуле `template`. Например: `validate: nginx -t -f %s`.

**Q: Как в шаблоне получить IP-адрес другого сервера из инвентаря?**
A: Через `hostvars`: `{{ hostvars['other_host']['ansible_default_ipv4']['address'] }}`.

**Q: Что делает конструкция `{%- if condition -%}`? Зачем нужны минусы?**
A: Минусы управляют пробелами (whitespace control). Они убирают переносы строк и пробелы до и после тега, чтобы в итоговом файле не было лишних пустых строк.

### Продвинутые вопросы

**Q: Как сгенерировать список в JSON или YAML формате внутри шаблона?**
A: Использовать фильтры `to_json`, `to_nice_json`, `to_yaml` или `to_nice_yaml`.

**Q: Как отфильтровать список прямо внутри Jinja2 цикла?**
A: Использовать условие в самом цикле: `{% for item in my_list if item.enabled %}` или фильтры `selectattr` / `rejectattr`.

**Q: Что такое `ansible_managed` и зачем она нужна?**
A: Это специальная переменная Ansible, которая содержит информацию о том, что файл управляется автоматикой (путь к шаблону, дата, пользователь). Нужна, чтобы админы случайно не начали править сгенерированный конфиг руками.

**Q: Как проверить версию ОС или пакета прямо в шаблоне?**
A: Использовать тест `version`: `{% if ansible_distribution_version is version('20.04', '>=') %}`.

**Q: Можно ли вызвать Ansible модуль прямо из Jinja2 шаблона?**
A: Нет, Jinja2 работает только с данными. Но можно использовать Lookup-плагины (например, `{{ lookup('file', '/path/to/local/file') }}`), чтобы прочитать файл с управляющей машины прямо в шаблоне.

---

## 📚 Дополнительные ресурсы

- Официальная документация Jinja2: https://jinja.palletsprojects.com/en/3.1.x/templates/
- Ansible Filters: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_filters.html
- Модуль template: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html

---
