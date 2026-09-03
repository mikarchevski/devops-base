# 🛠️ Ansible: Практические советы, Трюки и Real-World Опыт

## 📋 Оглавление

1. [Ускорение и Производительность](#1-ускорение-и-производительность)
2. [Отладка и Troubleshooting](#2-отладка-и-troubleshooting)
3. [Безопасность и Ansible Vault](#3-безопасность-и-ansible-vault)
4. [Идемпотентность и Защита от ошибок](#4-идемпотентность-и-защита-от-ошибок)
5. [Структура проекта (Best Practices)](#5-структура-проекта-best-practices)
6. [Продвинутые трюки (Assert, Pause, Lookups)](#6-продвинутые-трюки-assert-pause-lookups)

---

## 1. Ускорение и Производительность

Когда у тебя 5 серверов — Ansible летает. Когда 500 — начинает "тупить". Вот как это лечить.

### 🔥 Трюк 1: Включи Pipelining

По умолчанию Ansible scp-ит временные Python-файлы на сервер. Pipelining создает один SSH-сессия и передает код прямо в stdin. Ускоряет выполнение в 2-3 раза!

**В `ansible.cfg`:**

```ini
[ssh_connection]
pipelining = True
```

_Важно: На целевых серверах должно быть отключено `requiretty` в sudoers._

### 🔥 Трюк 2: Увеличь Forks (Параллелизм)

По умолчанию Ansible подключается только к 5 хостам одновременно.

**В `ansible.cfg`:**

```ini
[defaults]
forks = 20  # Или 50, в зависимости от мощности Control Node и сети
```

_Или через CLI:_ `ansible-playbook site.yml -f 50`

### 🔥 Трюк 3: Кэширование Facts

Сбор фактов (setup) занимает время. Если ты запускаешь playbook часто, кэшируй факты.

**В `ansible.cfg`:**

```ini
[defaults]
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts_cache
fact_caching_timeout = 86400  # 24 часа
```

### 🔥 Трюк 4: Отключай Facts, если они не нужны

Если ты просто перезапускаешь сервис или копируешь файл, сбор фактов не нужен.

```yaml
- hosts: webservers
  gather_facts: no # Экономит 2-5 секунд на каждом хосте
  tasks:
    - name: Restart app
      systemd: name=myapp state=restarted
```

---

## 2. Отладка и Troubleshooting

### 🐞 Пошаговое выполнение (Interactive Debug)

Если playbook огромный и падает на 50-й задаче, используй `--step`. Ansible будет спрашивать подтверждение перед КАЖДОЙ задачей.

```bash
ansible-playbook site.yml --step
# Вывод: Perform task: Install Nginx (y/n/c)?
# y - выполнить, n - пропустить, c - продолжить без вопросов
```

### 🐞 Продолжить с упавшей задачи

Не нужно запускать весь playbook заново, чтобы проверить фикс.

```bash
ansible-playbook site.yml --start-at-task="Deploy configuration"
```

### 🐞 "Тихая" отладка (Debug verbosity)

Не засоряй вывод `debug` модулем, если он не нужен в продакшене. Используй `verbosity`.

```yaml
tasks:
  - name: Show internal variable
    debug:
      var: complex_internal_dict
    verbosity: 2 # Выведется только если запустить с -vv
```

### 🐞 Пауза для ручной проверки

Иногда нужно остановить playbook, посмотреть на сервер руками и продолжить.

```yaml
tasks:
  - name: Pause for manual verification
    pause:
      prompt: "Проверь, что БД поднялась. Нажми Enter для продолжения."
      # minutes: 5  # Или просто подождать 5 минут
```

---

## 3. Безопасность и Ansible Vault

Никогда не храни пароли в git! Используй Vault.

### Шифрование файла

```bash
ansible-vault encrypt inventory/group_vars/prod/secrets.yml
# Введи пароль. Файл зашифруется.
```

### Запуск с зашифрованным файлом

```bash
# Интерактивный ввод пароля
ansible-playbook site.yml --ask-vault-pass

# Или через файл с паролем (для CI/CD)
ansible-playbook site.yml --vault-password-file ~/.vault_pass.txt
```

### Шифрование одной переменной (Inline Vault)

Не обязательно шифровать весь файл. Можно зашифровать только значение.

```bash
ansible-vault encrypt_string 'SuperSecretPassword123' --name 'db_password'
```

**Результат в `vars.yml`:**

```yaml
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  6162636465666768696a6b6c6d6e6f70...
```

_Теперь весь файл `vars.yml` можно коммитить в git, а пароль будет в безопасности._

---

## 4. Идемпотентность и Защита от ошибок

### 🛡️ Делаем `shell` идемпотентным

Если уж пришлось использовать `shell` или `command`, используй `creates` или `removes`.

```yaml
tasks:
  - name: Extract archive
    command: tar -xzf app.tar.gz -C /opt/app
    args:
      creates: /opt/app/bin/start.sh # Не выполнит команду, если файл уже есть!
```

### 🛡️ Подавляем ложные "CHANGED"

Команды чтения (grep, cat, df) всегда возвращают `CHANGED`, потому что Ansible не знает, изменили ли они состояние.

```yaml
tasks:
  - name: Get disk usage
    shell: df -h / | awk 'NR==2 {print $5}' | sed 's/%//'
    register: disk_usage
    changed_when: false # Говорим Ansible: "Это не меняет систему, считай это OK"
```

### 🛡️ Кастомные условия падения (failed_when)

Иногда ненулевой код возврата — это нормально (например, grep не нашел строку и вернул 1).

```yaml
tasks:
  - name: Check if user exists
    command: id myuser
    register: user_check
    failed_when:
      - user_check.rc != 0
      - "'No such user' not in user_check.stderr"
    # Упадет только если ошибка НЕ связана с отсутствием пользователя
```

### 🛡️ Валидация конфигов (Спасение продакшена)

Всегда проверяй синтаксис конфига ДО его применения.

```yaml
tasks:
  - name: Deploy Nginx config
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
      validate: /usr/sbin/nginx -t -f %s # %s - это временный файл. Если nginx -t упадет, старый конфиг НЕ будет перезаписан!
      backup: yes # На всякий случай создаст .bak
    notify: Reload Nginx
```

---

## 5. Структура проекта (Best Practices)

Как должен выглядеть реальный Ansible-репозиторий:

```text
my-ansible-project/
├── ansible.cfg               # Настройки Ansible (pipelining, forks, roles_path)
├── requirements.yml          # Зависимости (коллекции и роли из Galaxy)
├── inventory/
│   ├── production/
│   │   ├── hosts.yml         # Список хостов
│   │   ├── group_vars/
│   │   │   ├── all.yml       # Общие переменные
│   │   │   ├── webservers.yml
│   │   │   └── secrets.yml   # Зашифрован через Vault!
│   │   └── host_vars/
│   │       └── web1.prod.yml
│   └── staging/
│       └── ...
├── playbooks/
│   ├── site.yml              # Главный playbook (оркестратор)
│   ├── webservers.yml
│   └── dbservers.yml
├── roles/
│   ├── common/
│   ├── nginx/
│   └── postgresql/
└── files/                    # Глобальные файлы (если не используются в ролях)
    └── banner.txt
```

**Правило:** `site.yml` должен быть максимально коротким. Он только подключает роли.

```yaml
# site.yml
- hosts: webservers
  become: yes
  roles:
    - common
    - nginx
    - app_deploy
```

---

## 6. Продвинутые трюки (Assert, Pause, Lookups)

### 💡 Fail Fast с помощью `assert`

Не жди, пока playbook дойдет до 50-й задачи и упадет. Проверяй условия на берегу.

```yaml
tasks:
  - name: Validate prerequisites
    assert:
      that:
        - ansible_memtotal_mb >= 2048
        - ansible_os_family == "Debian"
        - env is defined and env in ['prod', 'staging']
      fail_msg: "Server does not meet requirements or env is wrong!"
      success_msg: "Server is good to go."
```

### 💡 Чтение локальных файлов (Lookups)

Модуль `lookup` позволяет читать файлы с **Control Node** (твоей машины) и вставлять их содержимое в задачи.

```yaml
tasks:
  - name: Add SSH public key to server
    authorized_key:
      user: deploy
      key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"

  - name: Read local CSV file
    set_fact:
      csv_data: "{{ lookup('file', 'data.csv').splitlines() }}"
```

### 💡 Генерация паролей "на лету"

Ansible может сам сгенерировать случайный пароль и сохранить его.

```yaml
tasks:
  - name: Generate random DB password
    set_fact:
      db_password: "{{ lookup('password', '/dev/null length=16 chars=ascii_letters,digits') }}"

  - name: Create DB user
    postgresql_user:
      name: myapp
      password: "{{ db_password }}"
```

_Важно: Если указать путь к файлу вместо `/dev/null` (например, `credentials/db_pass.txt`), Ansible сгенерирует пароль, сохранит его в этот файл на Control Node и при следующих запусках будет читать его оттуда, не меняя пароль._

### 💡 Ожидание поднятия сервиса (wait_for)

Не используй `sleep`! Используй `wait_for`, чтобы проверить, что порт реально открыт.

```yaml
tasks:
  - name: Start PostgreSQL
    systemd: name=postgresql state=started

  - name: Wait for PostgreSQL to be ready
    wait_for:
      port: 5432
      host: localhost
      timeout: 30
      state: started
```
