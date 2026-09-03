# 🔧 Ansible Handlers: Детальная шпаргалка

## 📋 Оглавление

1. [Что такое Handlers](#1-что-такое-handlers)
2. [Базовая структура](#2-базовая-структура)
3. [Как это работает под капотом](#3-как-это-работает-под-капотом)
4. [Ключевые правила](#4-ключевые-правила)
5. [Продвинутые паттерны](#5-продвинутые-паттерны)
6. [Handlers в ролях](#6-handlers-в-ролях)
7. [Типичные ошибки](#7-типичные-ошибки)
8. [Отладка](#8-отладка)
9. [Практические сценарии](#9-практические-сценарии)
10. [Вопросы для собеседования](#10-вопросы-для-собеседования)

---

## 1. Что такое Handlers

**Handlers (Обработчики)** — это специальные задачи, которые выполняются **только при статусе `CHANGED`** и **строго в конце Play** (если не использовать `flush_handlers`).

**Зачем нужны:**

- ✅ Перезапуск сервисов только при изменении конфигов (избегаем лишних рестартов)
- ✅ Применение изменений, которые требуют перезагрузки
- ✅ Оптимизация: один handler выполняется один раз, даже если его вызвали 100 раз

**Пример из жизни:**
Ты обновил 5 конфигурационных файлов Nginx. Без handlers Nginx перезапустился бы 5 раз. С handlers он перезапустится **только 1 раз** в самом конце.

---

## 2. Базовая структура

### Минимальный пример

```yaml
---
- hosts: webservers
  become: yes

  tasks:
    - name: Update Nginx config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Restart Nginx # <-- Вызов handler

  handlers:
    - name: Restart Nginx # <-- Объявление handler
      systemd:
        name: nginx
        state: restarted
```

### Полный пример с несколькими handlers

```yaml
---
- hosts: webservers
  become: yes
  force_handlers: yes # <-- Выполнять handlers даже при ошибках

  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present

    - name: Deploy config 1
      template:
        src: config1.conf.j2
        dest: /etc/nginx/conf.d/config1.conf
      notify: Reload Nginx

    - name: Deploy config 2
      template:
        src: config2.conf.j2
        dest: /etc/nginx/conf.d/config2.conf
      notify: Restart Nginx

    - name: Deploy app config
      template:
        src: app.conf.j2
        dest: /etc/app/app.conf
      notify:
        - Restart App
        - Reload Nginx # Один notify может вызывать несколько handlers

  handlers:
    - name: Restart Nginx
      systemd:
        name: nginx
        state: restarted

    - name: Reload Nginx
      systemd:
        name: nginx
        state: reloaded

    - name: Restart App
      systemd:
        name: myapp
        state: restarted
```

---

## 3. Как это работает под капотом

### Механизм очереди

Handlers работают как **отложенные события в очереди**, а не как обычные функции.
┌─────────────────────────────────────────────────────────┐
│ Задача 1 (CHANGED) ──► notify: Restart Nginx │
│ └─► Добавляем "Restart Nginx" │
│ в очередь │
├─────────────────────────────────────────────────────────┤
│ Задача 2 (OK) ────────► notify: Restart Nginx │
│ └─► "Restart Nginx" уже в │
│ очереди, игнорируем │
├─────────────────────────────────────────────────────────┤
│ Задача 3 (CHANGED) ──► notify: Reload Nginx │
│ └─► Добавляем "Reload Nginx" │
│ в очередь │
├─────────────────────────────────────────────────────────┤
│ ВСЕ ЗАДАЧИ ЗАВЕРШЕНЫ │
│ └─► Ansible опустошает очередь: │
│ 1. systemctl restart nginx │
│ 2. systemctl reload nginx │
└─────────────────────────────────────────────────────────┘

### Статусы задач и их влияние на handlers

| Статус    | Цвет       | Handler срабатывает?             |
| --------- | ---------- | -------------------------------- |
| `OK`      | 🟢 Зелёный | ❌ Нет (состояние не изменилось) |
| `CHANGED` | 🟡 Жёлтый  | ✅ Да (состояние изменилось)     |
| `FAILED`  | 🔴 Красный | ❌ Нет (playbook остановился)    |
| `SKIPPED` | 🔵 Голубой | ❌ Нет (задача пропущена)        |

### Дедупликация (магия оптимизации)

Если 3 задачи вызвали один и тот же handler, он выполнится **только 1 раз**:

```yaml
tasks:
  - name: Update config part 1
    copy:
      src: part1.conf
      dest: /etc/app/part1.conf
    notify: Restart App # Добавляется в очередь

  - name: Update config part 2
    copy:
      src: part2.conf
      dest: /etc/app/part2.conf
    notify: Restart App # Уже в очереди, игнорируется

  - name: Update config part 3
    copy:
      src: part3.conf
      dest: /etc/app/part3.conf
    notify: Restart App # Уже в очереди, игнорируется

handlers:
  - name: Restart App
    systemd:
      name: myapp
      state: restarted
    # Выполнится ТОЛЬКО ОДИН РАЗ, несмотря на 3 вызова!
```

---

## 4. Ключевые правила

| Правило                | Описание                                                                                 |
| ---------------------- | ---------------------------------------------------------------------------------------- |
| **Только при CHANGED** | Handler выполняется только если задача вернула статус `CHANGED`                          |
| **Один раз**           | Даже если handler вызвали 100 раз, он выполнится только 1 раз (дедупликация)             |
| **В конце Play**       | Handlers выполняются после всех tasks (если не использовать `flush_handlers`)            |
| **Порядок объявления** | Handlers выполняются в порядке их объявления в секции `handlers:`, а не в порядке вызова |
| **При ошибке — стоп**  | Если задача упала с `FAILED`, handlers не выполняются (если не включен `force_handlers`) |
| **Уникальные имена**   | Имена handlers должны быть уникальными в рамках Play (иначе коллизии)                    |

### Порядок выполнения handlers

Handlers выполняются **в порядке их объявления** в секции `handlers:`, а не в порядке вызова через `notify`:

```yaml
tasks:
  - name: Task 1
    copy: { src: a.conf, dest: /etc/a.conf }
    notify: Handler B # Вызываем B первым

  - name: Task 2
    copy: { src: b.conf, dest: /etc/b.conf }
    notify: Handler A # Вызываем A вторым

handlers:
  - name: Handler A
    debug: { msg: "A" } # Объявлен первым

  - name: Handler B
    debug: { msg: "B" } # Объявлен вторым


# Порядок выполнения: Handler A, затем Handler B
# (несмотря на то, что B был вызван первым!)
```

---

## 5. Продвинутые паттерны

### 🔥 Паттерн 1: `listen` (несколько имён для одного handler)

Когда нужно вызвать один handler разными notify или несколько handlers одним notify:

```yaml
tasks:
  - name: Update nginx config
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: reload web # <-- Вызываем по альтернативному имени

  - name: Update apache config
    template:
      src: apache.conf.j2
      dest: /etc/apache2/apache2.conf
    notify: reload web # <-- Тот же notify

handlers:
  - name: reload nginx
    systemd:
      name: nginx
      state: reloaded
    listen: reload web # <-- "Слушает" событие reload web

  - name: reload apache
    systemd:
      name: apache2
      state: reloaded
    listen: reload web # <-- Тоже "слушает" reload web
```

**Результат:** При изменении любого конфига перезагрузятся ОБА веб-сервера.

### 🔥 Паттерн 2: `meta: flush_handlers` (принудительное выполнение)

Когда нужно выполнить handlers НЕ в конце play, а прямо сейчас:

```yaml
tasks:
  - name: Deploy database config
    template:
      src: db.conf.j2
      dest: /etc/postgresql/db.conf
    notify: Restart PostgreSQL

  - name: Force restart DB now
    meta: flush_handlers # <-- Handler выполнится ЗДЕСЬ, а не в конце

  - name: Run database migrations
    command: /opt/app/migrate.sh
    # Без flush_handlers миграции запустились бы со СТАРЫМ конфигом!

  - name: Deploy app config
    template:
      src: app.conf.j2
      dest: /etc/app/app.conf
    notify: Restart App

  # В конце play выполнится только Restart App
```

### 🔥 Паттерн 3: `force_handlers: yes` (выполнение даже при ошибках)

По умолчанию, если задача упала, handlers не выполняются. Это может оставить систему в сломанном состоянии:

```yaml
- hosts: webservers
  force_handlers: yes # <-- Handlers выполнятся даже при ошибке!

  tasks:
    - name: Update config
      template:
        src: app.conf.j2
        dest: /etc/app/app.conf
      notify: Restart App

    - name: Deploy new binary (может упасть!)
      copy:
        src: broken_binary
        dest: /opt/app/binary
        mode: "0755"
      # Если здесь ошибка — без force_handlers Restart App НЕ выполнится,
      # и сервис останется работать со СТАРЫМ (уже несовместимым) конфигом!

  handlers:
    - name: Restart App
      systemd:
        name: myapp
        state: restarted
```

**Когда использовать:** Когда важно привести систему в согласованное состояние даже при частичной ошибке.

### 🔥 Паттерн 4: Handler для нескольких сервисов

```yaml
tasks:
  - name: Update shared config
    template:
      src: shared.conf.j2
      dest: /etc/shared/config.conf
    notify:
      - Restart App1
      - Restart App2
      - Restart App3

handlers:
  - name: Restart App1
    systemd:
      name: app1
      state: restarted

  - name: Restart App2
    systemd:
      name: app2
      state: restarted

  - name: Restart App3
    systemd:
      name: app3
      state: restarted
```

### 🔥 Паттерн 5: Handler с условием

```yaml
tasks:
  - name: Update config
    template:
      src: app.conf.j2
      dest: /etc/app/app.conf
    notify: Restart App

handlers:
  - name: Restart App (systemd)
    systemd:
      name: myapp
      state: restarted
    when: ansible_service_mgr == "systemd"

  - name: Restart App (sysvinit fallback)
    service:
      name: myapp
      state: restarted
    when: ansible_service_mgr != "systemd"
```

### 🔥 Паттерн 6: Handler с бэкапом

```yaml
tasks:
  - name: Update critical config
    template:
      src: critical.conf.j2
      dest: /etc/app/critical.conf
      backup: yes # Создаст бэкап перед изменением
    notify: Restart App

handlers:
  - name: Restart App
    systemd:
      name: myapp
      state: restarted
    # Если рестарт не удался, можно откатить бэкап вручную
```

---

## 6. Handlers в ролях

### Автоматическое добавление handlers

Когда роль подключается через `roles:`, её handlers **автоматически добавляются** в общий пул handlers play:

```yaml
# playbook.yml
- hosts: webservers
  roles:
    - nginx
    - postgresql
```

```
roles/
  nginx/
    handlers/main.yml:
      - name: Restart Nginx
        systemd: { name: nginx, state: restarted }
  postgresql/
    handlers/main.yml:
      - name: Restart PostgreSQL
        systemd: { name: postgresql, state: restarted }
```

**Важно:** Если в двух ролях есть handler с **одинаковым именем**, выполнится только ПЕРВЫЙ найденный!

**Решение:** Использовать уникальные имена или `listen`.

### `include_role` vs `import_role`

```yaml
tasks:
  # include_role (динамический) — handlers добавляются в момент выполнения
  - include_role:
      name: nginx
    when: install_nginx

  # import_role (статический) — handlers парсятся при загрузке playbook
  - import_role:
      name: postgresql
```

**Разница:** При `include_role` с условием `when: false` handlers роли **не добавляются** в play. При `import_role` — добавляются всегда, даже если роль не выполняется.

---

## 7. Типичные ошибки

### ❌ Ошибка 1: Опечатка в имени handler

```yaml
tasks:
  - name: Update config
    copy:
      src: app.conf
      dest: /etc/app/app.conf
    notify: Restart App # <-- с большой буквы

handlers:
  - name: restart app # <-- с маленькой буквы! Handler НЕ вызовется
    systemd:
      name: myapp
      state: restarted
```

**Решение:** Имена должны совпадать ТОЧНО (или использовать `listen`).

### ❌ Ошибка 2: Ожидание, что handler выполнится сразу

```yaml
tasks:
  - name: Update config
    copy:
      src: app.conf
      dest: /etc/app/app.conf
    notify: Restart App

  - name: Check if app is running with new config
    command: /opt/app/check.sh
    # <-- Handler ЕЩЁ не выполнился! Эта задача выполнится ДО handler'а!
```

**Решение:** Использовать `meta: flush_handlers` перед проверкой.

### ❌ Ошибка 3: Неидемпотентный handler

```yaml
handlers:
  - name: Restart App
    command: /opt/app/restart.sh # <-- ПЛОХО! Всегда CHANGED
    # Лучше использовать systemd/service модуль
```

**Решение:** Использовать идемпотентные модули (`systemd`, `service`).

### ❌ Ошибка 4: Handler не выполняется при ошибке

```yaml
tasks:
  - name: Update config
    template:
      src: app.conf.j2
      dest: /etc/app/app.conf
    notify: Restart App

  - name: Broken task
    command: /nonexistent/script.sh
    # <-- Ошибка! Handler Restart App НЕ выполнится!
```

**Решение:** Использовать `force_handlers: yes` на уровне play.

### ❌ Ошибка 5: Дублирование handlers в разных ролях

```yaml
# roles/nginx/handlers/main.yml
- name: restart web
  systemd: { name: nginx, state: restarted }

# roles/apache/handlers/main.yml
- name: restart web # <-- То же имя!
  systemd: { name: apache2, state: restarted }
```

**Решение:** Использовать уникальные имена (`restart nginx`, `restart apache`) или `listen`.

---

## 8. Отладка

### Посмотреть, какие handlers будут вызваны

```bash
ansible-playbook playbook.yml --list-tasks
# В выводе увидишь:
# TASKS [handlers]
#   Restart Nginx
#   Reload App
```

### Проверить, сработал ли handler

```bash
ansible-playbook playbook.yml -v
# В выводе:
# RUNNING HANDLER [Restart Nginx] ****
# changed: [web1] => {"changed": true, ...}
```

### Чеклист: если handler НЕ срабатывает

1. ✅ Задача вернула статус `CHANGED` (не `OK`)?
2. ✅ Имя в `notify` ТОЧНО совпадает с именем handler'а?
3. ✅ Play завершился успешно (не упал с `FAILED`)?
4. ✅ Handler объявлен в том же play (не в другом)?
5. ✅ Нет ли опечаток в пробелах/регистре?

### Показать разницу в файлах

```bash
ansible-playbook playbook.yml --diff
# Покажет, какие файлы изменились и какие handlers будут вызваны
```

### Отладка с verbose

```bash
ansible-playbook playbook.yml -vvv
# Максимальная детализация: увидишь, когда handler добавляется в очередь
# и когда выполняется
```

---

## 9. Практические сценарии

### Сценарий 1: Каскадный перезапуск (зависимые сервисы)

```yaml
tasks:
  - name: Update shared library
    copy:
      src: libshared.so
      dest: /usr/lib/libshared.so
    notify: Restart All Dependent Services

handlers:
  - name: Restart All Dependent Services
    # Выполняются в порядке объявления!
    - name: Restart App1
      systemd: { name: app1, state: restarted }
    - name: Restart App2
      systemd: { name: app2, state: restarted }
    - name: Restart Worker
      systemd: { name: worker, state: restarted }
```

### Сценарий 2: Reload vs Restart (умная логика)

```yaml
tasks:
  - name: Update minor config
    lineinfile:
      path: /etc/app/app.conf
      line: "debug=false"
    notify: Reload App

  - name: Update major config (требует полного рестарта)
    template:
      src: app.conf.j2
      dest: /etc/app/app.conf
    notify: Restart App

handlers:
  - name: Reload App
    systemd:
      name: myapp
      state: reloaded

  - name: Restart App
    systemd:
      name: myapp
      state: restarted
    # Если сработал Restart, Reload не нужен
    # Ansible выполнит их в порядке объявления: сначала Reload, потом Restart
```

### Сценарий 3: Handler для Docker контейнеров

```yaml
tasks:
  - name: Update docker-compose.yml
    template:
      src: docker-compose.yml.j2
      dest: /opt/app/docker-compose.yml
    notify: Recreate Containers

handlers:
  - name: Recreate Containers
    community.docker.docker_compose:
      project_src: /opt/app
      state: present
      restarted: yes
      build: yes
```

### Сценарий 4: Handler с проверкой конфигурации

```yaml
handlers:
  - name: Restart Nginx
    block:
      - name: Test nginx config
        command: nginx -t
        # Если конфиг невалидный, handler упадет здесь

      - name: Restart nginx
        systemd:
          name: nginx
          state: restarted
    rescue:
      - name: Config test failed
        debug:
          msg: "Nginx config is invalid, not restarting"
        failed_when: true # Пометить задачу как failed
```

### Сценарий 5: Handler с откатом изменений

```yaml
tasks:
  - name: Update critical config
    template:
      src: critical.conf.j2
      dest: /etc/app/critical.conf
      backup: yes
    register: config_update
    notify: Restart App

handlers:
  - name: Restart App
    block:
      - name: Restart application
        systemd:
          name: myapp
          state: restarted

      - name: Wait for app to start
        wait_for:
          port: 8080
          timeout: 30

      - name: Check if app is healthy
        uri:
          url: http://localhost:8080/health
          status_code: 200
    rescue:
      - name: Rollback config
        copy:
          src: "{{ config_update.backup_file }}"
          dest: /etc/app/critical.conf

      - name: Restart with old config
        systemd:
          name: myapp
          state: restarted

      - name: Notify admin
        mail:
          to: admin@example.com
          subject: "App restart failed, rolled back"
```

---

## 10. Вопросы для собеседования

### Базовые вопросы

**Q: Что такое Handlers?**
A: Специальные задачи, которые выполняются только при статусе `CHANGED` и только в конце Play. Используются для перезапуска сервисов при изменении конфигов.

**Q: В каком порядке выполняются handlers?**
A: В порядке их объявления в секции `handlers:`, а не в порядке вызова через `notify`.

**Q: Что будет, если одна задача вызовет handler, а другая упадет с ошибкой?**
A: Handler НЕ выполнится (если не включен `force_handlers: yes`).

**Q: Как заставить handler выполниться прямо сейчас, а не в конце play?**
A: Использовать `meta: flush_handlers`.

### Продвинутые вопросы

**Q: Можно ли вызвать handler из другой роли?**
A: Да, если использовать `listen` или если handler объявлен в общем пуле play.

**Q: Что такое `listen` в handlers?**
A: Механизм, позволяющий вызывать handler по альтернативному имени. Несколько handlers могут "слушать" одно событие.

**Q: Как избежать дублирования handlers в разных ролях?**
A: Использовать уникальные имена, `listen`, или выносить общие handlers в отдельную роль/в play.

**Q: Что такое `force_handlers`?**
A: Параметр на уровне play, который заставляет handlers выполняться даже если playbook упал с ошибкой.

**Q: Handler выполнился 3 раза, хотя я вызвал его 3 раза. Почему?**
A: Скорее всего, это разные handlers с одинаковым именем из разных ролей. Ansible выполняет все найденные handlers с таким именем.

**Q: Можно ли использовать переменные в именах handlers?**
A: Нет, имена handlers должны быть статичными. Но можно использовать `listen` с переменными.

**Q: Как работает дедупликация handlers?**
A: Если одна и та же задача (с одинаковым именем handler) вызывается несколько раз, handler добавляется в очередь только один раз и выполняется один раз.

**Q: Что произойдет, если в handler'е будет ошибка?**
A: Playbook остановится с ошибкой, как и в обычной задаче. Остальные handlers из очереди не выполнятся (если не использовать `block/rescue` внутри handler).

**Q: Как отладить, почему handler не срабатывает?**
A: Проверить: 1) Задача вернула `CHANGED`? 2) Имя в `notify` совпадает с именем handler? 3) Play завершился успешно? 4) Handler в том же play? 5) Нет опечаток?

---

## 📚 Дополнительные ресурсы

- Официальная документация: https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_handlers.html
- Best Practices: https://docs.ansible.com/ansible/latest/tips_tricks/ansible_tips_tricks.html

---
