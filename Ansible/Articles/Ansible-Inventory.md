# 📋 Ansible Inventory: Детальная шпаргалка

## 📋 Оглавление

1. [Что такое Inventory](#1-что-такое-inventory)
2. [Формат INI](#2-формат-ini)
3. [Формат YAML](#3-формат-yaml)
4. [Переменные в Inventory](#4-переменные-в-inventory)
5. [Динамический Inventory](#5-динамический-inventory)
6. [Best Practices](#6-best-practices)
7. [Продвинутые паттерны](#7-продвинутые-паттерны)
8. [Вопросы для собеседования](#8-вопросы-для-собеседования)

---

## 1. Что такое Inventory

**Inventory (Инвентарь)** — это файл или источник данных, который описывает целевые серверы (Managed Nodes), которыми управляет Ansible.

**Зачем нужен:**

- Определяет, на каких серверах выполнять задачи
- Позволяет группировать хосты по ролям (webservers, dbservers)
- Хранит переменные для конкретных хостов или групп
- Может быть статичным (файл) или динамическим (скрипт/плагин)

**Где хранится:**

- По умолчанию: `/etc/ansible/hosts`
- Можно указать свой: `ansible-playbook -i my_inventory.yml playbook.yml`
- Или в `ansible.cfg`: `inventory = ./inventory/`

---

## 2. Формат INI

Классический формат, простой и читаемый.

### Базовая структура

```ini
# Простой список хостов (без групп)
web1.example.com
web2.example.com
db1.example.com

# Группы хостов
[webservers]
web1.example.com
web2.example.com

[dbservers]
db1.example.com
db2.example.com

# Специальная группа 'all' включает все хосты
# Специальная группа 'ungrouped' включает хосты без группы
```

### Переменные для хостов

```ini
[webservers]
web1.example.com ansible_user=admin ansible_port=2222
web2.example.com ansible_user=deploy ansible_port=22

[dbservers]
db1.example.com ansible_user=postgres ansible_become=yes
db2.example.com ansible_user=postgres
```

### Переменные для групп

```ini
[webservers]
web1.example.com
web2.example.com

# Переменные для всей группы webservers
[webservers:vars]
ansible_user=deploy
ansible_python_interpreter=/usr/bin/python3
ntp_server=ntp.example.com
http_port=80

[dbservers]
db1.example.com
db2.example.com

[dbservers:vars]
ansible_user=postgres
db_port=5432
```

### Вложенные группы (группы групп)

```ini
[webservers]
web1.example.com
web2.example.com

[dbservers]
db1.example.com
db2.example.com

# Группа production включает webservers и dbservers
[production:children]
webservers
dbservers

# Переменные для группы production
[production:vars]
env=production
monitoring_enabled=true

# Группа staging
[staging:children]
webservers
dbservers

[staging:vars]
env=staging
```

### Диапазоны хостов

```ini
[webservers]
web[01:50].example.com

[dbservers]
db[01:10].example.com

# Это развернется в:
# web01.example.com, web02.example.com, ..., web50.example.com
# db01.example.com, db02.example.com, ..., db10.example.com
```

### Полный пример INI

```ini
# Контроллеры Kubernetes
[k8s_masters]
k8s-master-01.example.com
k8s-master-02.example.com
k8s-master-03.example.com

[k8s_masters:vars]
ansible_user=k8sadmin
ansible_become=yes

# Воркеры Kubernetes
[k8s_workers]
k8s-worker-[01:20].example.com

[k8s_workers:vars]
ansible_user=k8sadmin
ansible_become=yes

# Все узлы Kubernetes
[k8s_cluster:children]
k8s_masters
k8s_workers

[k8s_cluster:vars]
k8s_version=1.28.0
container_runtime=containerd

# Базы данных
[databases]
db-postgres-01.example.com db_role=master
db-postgres-02.example.com db_role=replica
db-redis-01.example.com db_type=redis

[databases:vars]
ansible_user=dbadmin
backup_enabled=true

# Мониторинг
[monitoring]
prometheus.example.com
grafana.example.com

[monitoring:vars]
ansible_user=monitoring

# Production окружение
[production:children]
k8s_cluster
databases
monitoring

[production:vars]
env=production
log_level=warn

# Staging окружение
[staging:children]
k8s_cluster
databases

[staging:vars]
env=staging
log_level=debug
```

---

## 3. Формат YAML

Более современный, гибкий и читаемый формат. Рекомендуется для сложных структур.

### Базовая структура

```yaml
all:
  hosts:
    # Хосты на верхнем уровне (в группе 'all')
    standalone.example.com:

  children:
    webservers:
      hosts:
        web1.example.com:
        web2.example.com:

    dbservers:
      hosts:
        db1.example.com:
        db2.example.com:
```

### Переменные для хостов

```yaml
all:
  children:
    webservers:
      hosts:
        web1.example.com:
          ansible_user: admin
          ansible_port: 2222
          http_port: 8080

        web2.example.com:
          ansible_user: deploy
          ansible_port: 22
          http_port: 80

    dbservers:
      hosts:
        db1.example.com:
          ansible_user: postgres
          ansible_become: yes
          db_role: master

        db2.example.com:
          ansible_user: postgres
          db_role: replica
```

### Переменные для групп

```yaml
all:
  children:
    webservers:
      hosts:
        web1.example.com:
        web2.example.com:
      vars:
        ansible_user: deploy
        ansible_python_interpreter: /usr/bin/python3
        ntp_server: ntp.example.com
        http_port: 80

    dbservers:
      hosts:
        db1.example.com:
        db2.example.com:
      vars:
        ansible_user: postgres
        db_port: 5432
        backup_enabled: true
```

### Вложенные группы

```yaml
all:
  children:
    webservers:
      hosts:
        web1.example.com:
        web2.example.com:
      vars:
        role: web

    dbservers:
      hosts:
        db1.example.com:
        db2.example.com:
      vars:
        role: database

    # Группа production включает webservers и dbservers
    production:
      children:
        webservers:
        dbservers:
      vars:
        env: production
        monitoring_enabled: true

    # Группа staging
    staging:
      children:
        webservers:
        dbservers:
      vars:
        env: staging
        monitoring_enabled: false
```

### Полный пример YAML

```yaml
all:
  vars:
    # Глобальные переменные для всех хостов
    ansible_python_interpreter: /usr/bin/python3
    ntp_server: ntp.example.com

  children:
    # Kubernetes кластер
    k8s_cluster:
      children:
        k8s_masters:
          hosts:
            k8s-master-01.example.com:
              ansible_user: k8sadmin
              ansible_become: yes
              k8s_role: master

            k8s-master-02.example.com:
              ansible_user: k8sadmin
              ansible_become: yes
              k8s_role: master

            k8s-master-03.example.com:
              ansible_user: k8sadmin
              ansible_become: yes
              k8s_role: master

          vars:
            k8s_component: master

        k8s_workers:
          hosts:
            k8s-worker-01.example.com:
              ansible_user: k8sadmin
              ansible_become: yes

            k8s-worker-02.example.com:
              ansible_user: k8sadmin
              ansible_become: yes

            k8s-worker-03.example.com:
              ansible_user: k8sadmin
              ansible_become: yes

          vars:
            k8s_component: worker

      vars:
        k8s_version: "1.28.0"
        container_runtime: containerd
        cni_plugin: calico

    # Базы данных
    databases:
      children:
        postgres:
          hosts:
            db-postgres-01.example.com:
              ansible_user: dbadmin
              db_role: master

            db-postgres-02.example.com:
              ansible_user: dbadmin
              db_role: replica

          vars:
            db_type: postgres
            db_port: 5432

        redis:
          hosts:
            db-redis-01.example.com:
              ansible_user: dbadmin

          vars:
            db_type: redis
            db_port: 6379

      vars:
        backup_enabled: true
        backup_schedule: "0 2 * * *"

    # Мониторинг
    monitoring:
      hosts:
        prometheus.example.com:
          ansible_user: monitoring
          monitoring_role: prometheus

        grafana.example.com:
          ansible_user: monitoring
          monitoring_role: grafana

      vars:
        retention_days: 30

    # Окружения
    production:
      children:
        k8s_cluster:
        databases:
        monitoring:
      vars:
        env: production
        log_level: warn
        alerting_enabled: true

    staging:
      children:
        k8s_cluster:
        databases:
      vars:
        env: staging
        log_level: debug
        alerting_enabled: false
```

---

## 4. Переменные в Inventory

### Где можно хранить переменные

**1. В самом файле inventory (inline):**

```yaml
webservers:
  hosts:
    web1.example.com:
      ansible_user: admin
      http_port: 8080
```

**2. В директории `group_vars/`:**

```
inventory/
  hosts.yml
  group_vars/
    all.yml           # Для всех хостов
    webservers.yml    # Для группы webservers
    dbservers.yml     # Для группы dbservers
    production.yml    # Для группы production
```

**Содержимое `group_vars/webservers.yml`:**

```yaml
---
ansible_user: deploy
http_port: 80
https_port: 443
nginx_worker_processes: auto
nginx_worker_connections: 1024
```

**3. В директории `host_vars/`:**

```
inventory/
  hosts.yml
  host_vars/
    web1.example.com.yml   # Для конкретного хоста
    web2.example.com.yml
    db1.example.com.yml
```

**Содержимое `host_vars/web1.example.com.yml`:**

```yaml
---
ansible_user: admin
ansible_port: 2222
http_port: 8080
ssl_certificate: /etc/ssl/web1.crt
ssl_key: /etc/ssl/web1.key
```

### Приоритет переменных

От низшего к высшему:

1. Переменные в самом файле inventory (inline)
2. `group_vars/all.yml`
3. `group_vars/<group_name>.yml`
4. `host_vars/<host_name>.yml`
5. Переменные, переданные через `-e` (Extra Vars)

### Пример структуры проекта

```
ansible-project/
├── inventory/
│   ├── production/
│   │   ├── hosts.yml
│   │   ├── group_vars/
│   │   │   ├── all.yml
│   │   │   ├── webservers.yml
│   │   │   └── dbservers.yml
│   │   └── host_vars/
│   │       ├── web1.prod.yml
│   │       └── db1.prod.yml
│   └── staging/
│       ├── hosts.yml
│       ├── group_vars/
│       │   ├── all.yml
│       │   └── webservers.yml
│       └── host_vars/
│           └── web1.staging.yml
├── playbooks/
│   ├── site.yml
│   ├── webservers.yml
│   └── dbservers.yml
├── roles/
│   ├── nginx/
│   ├── postgresql/
│   └── common/
└── ansible.cfg
```

**Запуск для разных окружений:**

```bash
# Production
ansible-playbook -i inventory/production/hosts.yml playbooks/site.yml

# Staging
ansible-playbook -i inventory/staging/hosts.yml playbooks/site.yml
```

---

## 5. Динамический Inventory

Динамический инвентарь позволяет получать список хостов из внешних источников (облачные провайдеры, CMDB, API).

### Типы динамического инвентаря

**1. Inventory-скрипты (старый способ):**

```bash
#!/usr/bin/env python3
# my_inventory.py

import json

inventory = {
    "webservers": {
        "hosts": ["web1.example.com", "web2.example.com"],
        "vars": {
            "http_port": 80
        }
    },
    "dbservers": {
        "hosts": ["db1.example.com"],
        "vars": {
            "db_port": 5432
        }
    },
    "_meta": {
        "hostvars": {
            "web1.example.com": {
                "ansible_user": "admin"
            }
        }
    }
}

print(json.dumps(inventory, indent=2))
```

**Использование:**

```bash
chmod +x my_inventory.py
ansible-playbook -i my_inventory.py playbook.yml
```
