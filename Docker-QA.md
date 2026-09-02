 🐳 Шпаргалка по Docker
### 📑 ОГЛАВЛЕНИЕ

- [ ЧАСТЬ 1: ОСНОВНЫЕ КОМАНДЫ](#-часть-1-основные-команды)
  - [Контейнеры](#контейнеры)
  - [Образы](#образы)
  - [Docker Compose](#docker-compose)
  - [Сети](#сети)
  - [Тома (Volumes)](#тома-volumes)
  - [Система](#система)
- [ ЧАСТЬ 2: ТРАБЛШУТИНГ](#-часть-2-траблшутинг)
  - [Контейнер не запускается / падает](#контейнер-не-запускается--падает)
  - [Порт уже занят](#порт-уже-занят)
  - [Нет места на диске](#нет-места-на-диске)
  - [Проблемы с сетью](#проблемы-с-сетью)
  - [Проблемы с томами](#проблемы-с-томами)
  - [Проблемы со сборкой образа](#проблемы-со-сборкой-образа)
  - [Проблемы с Docker Compose](#проблемы-с-docker-compose)
  - [Контейнер не отвечает на запросы](#контейнер-не-отвечает-на-запросы)
  - [Проблемы с правами доступа](#проблемы-с-правами-доступа)
- [ ЧАСТЬ 3: СИНТАКСИС DOCKERFILE](#-часть-3-синтаксис-dockerfile)
  - [Базовый синтаксис](#базовый-синтаксис)
  - [Multi-stage build](#multi-stage-build)
- [ ЧАСТЬ 4: СИНТАКСИС DOCKER-COMPOSE.YML](#-часть-4-синтаксис-docker-composeyml)
  - [Сервисы](#сервисы)
  - [Сети](#сети-1)
  - [Тома](#тома-1)
- [ БЫСТРЫЙ СПИСОК: ЧТО И ЗАЧЕМ](#-быстрый-список-что-и-зачем)
  - [Dockerfile инструкции](#dockerfile-инструкции)
  - [Docker Compose ключи](#docker-compose-ключи)
  - [Флаги docker run](#флаги-docker-run)
- [ ПОЛЕЗНЫЕ ОДНОСТРОЧНИКИ](#-полезные-однострочники)
- [ ЧАСТЫЕ ОШИБКИ И РЕШЕНИЯ](#-частые-ошибки-и-решения)

---

#  ЧАСТЬ 1: ОСНОВНЫЕ КОМАНДЫ

## Контейнеры

### Запуск контейнера
docker run -d -p 8080:80 --name myapp myimage:latest

### Список запущенных контейнеров
docker ps

### Все контейнеры (включая остановленные)
docker ps -a

### Остановить контейнер
docker stop <container_name_or_id>

### Запустить остановленный контейнер
docker start <container_name_or_id>

### Перезапустить контейнер
docker restart <container_name_or_id>

### Удалить контейнер
docker rm <container_name_or_id>

### Удалить все остановленные контейнеры
docker container prune

### Посмотреть логи контейнера
docker logs <container_name_or_id>

### Логи в реальном времени
docker logs -f <container_name_or_id>

### Последние 100 строк логов
docker logs --tail 100 <container_name_or_id>

### Подключиться к работающему контейнеру
docker exec -it <container_name_or_id> bash

### Выполнить команду в контейнере
docker exec <container_name_or_id> ls -la

### Посмотреть статистику использования ресурсов
docker stats

### Посмотреть детали контейнера (IP, mounts, env)
docker inspect <container_name_or_id>

### Копировать файлы между хостом и контейнером
docker cp <file> <container>:/path/
docker cp <container>:/path/file ./local/

## Образы

### Список образов
docker images

### Скачать образ
docker pull nginx:alpine

### Удалить образ
docker rmi <image_name_or_id>

### Удалить все неиспользуемые образы
docker image prune -a

### Собрать образ из Dockerfile
docker build -t myapp:1.0 .

### Собрать без кэша
docker build --no-cache -t myapp:1.0 .

### Посмотреть историю слоёв образа
docker history <image_name>

### Сохранить изменения контейнера в образ (не рекомендуется)
docker commit <container_id> myimage:tag

### Экспорт/импорт образа
docker save myimage:tag -o myimage.tar
docker load -i myimage.tar

## Docker Compose

### Запустить все сервисы (фоновый режим)
docker compose up -d

### Запустить с пересборкой образов
docker compose up -d --build

### Запустить без кэша
docker compose up -d --build --no-cache

### Остановить и удалить контейнеры
docker compose down

### Остановить, удалить контейнеры и тома
docker compose down -v

### Посмотреть статус сервисов
docker compose ps

### Посмотреть логи всех сервисов
docker compose logs

### Логи конкретного сервиса в реальном времени
docker compose logs -f myservice

### Перезапустить сервис
docker compose restart myservice

### Выполнить команду в сервисе
docker compose exec myservice bash

### Собрать образы без запуска
docker compose build

### Показать конфигурацию (проверить YAML)
docker compose config

## Сети

### Список сетей
docker network ls

### Создать сеть
docker network create mynetwork

### Удалить сеть
docker network rm mynetwork

### Подключить контейнер к сети
docker network connect mynetwork <container>

### Отключить контейнер от сети
docker network disconnect mynetwork <container>

### Посмотреть детали сети
docker network inspect mynetwork

##Тома (Volumes)

### Список томов
docker volume ls

### Создать том
docker volume create myvolume

### Удалить том
docker volume rm myvolume

### Удалить все неиспользуемые тома
docker volume prune

### Посмотреть детали тома
docker volume inspect myvolume

## Система

### Общая информация о Docker
docker info

### Версия Docker
docker --version

### Очистить всё неиспользуемое (образы, контейнеры, сети, кэш)
docker system prune -a --volumes

### Посмотреть использование диска
docker system df

---

#  ЧАСТЬ 2: ТРАБЛШУТИНГ

## Контейнер не запускается / падает

### 1. Смотрим статус
docker ps -a

### 2. Читаем логи (самое важное!)
docker logs <container_name>

### 3. Читаем последние 50 строк логов
docker logs --tail 50 <container_name>

### 4. Запускаем в интерактивном режиме для отладки
docker run -it --entrypoint /bin/sh myimage

### 5. Проверяем healthcheck
docker inspect --format='{{json .State.Health}}' <container>

## Порт уже занят

### На Linux: найти процесс на порту 8080
sudo lsof -i :8080
sudo netstat -tulpn | grep 8080

### Убить процесс
sudo kill -9 <PID>

### Или остановить конфликтующий контейнер
docker stop <container_name>

## Нет места на диске

### Посмотреть использование
docker system df

### Очистить всё неиспользуемое
docker system prune -a --volumes

### Удалить все остановленные контейнеры
docker container prune

### Удалить все dangling образы
docker image prune

### Удалить все неиспользуемые тома
docker volume prune

## Проблемы с сетью

### Проверить, в какой сети контейнер
docker inspect --format='{{json .NetworkSettings.Networks}}' <container>

### Проверить IP контейнера
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <container>

### Проверить DNS из контейнера
docker exec <container> nslookup db

### Пересоздать сеть
docker compose down
docker compose up -d

## Проблемы с томами

### Посмотреть, какие тома использует контейнер
docker inspect --format='{{json .Mounts}}' <container>

### Посмотреть содержимое тома
docker run --rm -v myvolume:/data alpine ls -la /data

### Очистить том (ВНИМАНИЕ: удалит данные!)
docker volume rm myvolume

## Проблемы со сборкой образа

### Собрать без кэша
docker build --no-cache -t myapp .

### Посмотреть слои образа
docker history myapp

### Проверить .dockerignore
cat .dockerignore

### Собрать с подробным выводом
docker build --progress=plain -t myapp .

## Проблемы с Docker Compose

### Проверить валидность YAML
docker compose config

### Посмотреть, какие сервисы определены
docker compose config --services

### Перезапустить один сервис
docker compose up -d --force-recreate myservice

### Посмотреть переменные окружения
docker compose exec myservice env

## Контейнер не отвечает на запросы

### Проверить, слушает ли порт
docker exec <container> netstat -tulpn

### Проверить изнутри контейнера
docker exec <container> curl localhost:80

### Проверить с хоста
curl -I http://localhost:8080

### Посмотреть открытые порты контейнера
docker port <container>

## Проблемы с правами доступа

### Запустить от root (для отладки)
docker run -it --user root myimage bash

### Изменить владельца файлов в томе
docker run --rm -v myvolume:/data alpine chown -R 1000:1000 /data

### Добавить пользователя в группу docker (Linux)
sudo usermod -aG docker $USER
newgrp docker

---

#  ЧАСТЬ 3: СИНТАКСИС DOCKERFILE

## БАЗОВЫЙ ОБРАЗ
### FROM задаёт базовый образ, от которого строится наш
### alpine - минимальный Linux (~5MB), ideal для продакшена
FROM python:3.11-alpine


## МЕТАДАННЫЕ (опционально)
LABEL maintainer="benx@example.com"
LABEL version="1.0"
LABEL description="My Python App"

## ПЕРЕМЕННЫЕ ОКРУЖЕНИЯ
### ENV задаёт переменные, доступные во время сборки и в контейнере
ENV PYTHONUNBUFFERED=1
ENV APP_HOME=/app


## РАБОЧАЯ ДИРЕКТОРИЯ
### WORKDIR создаёт папку и переходит в неё
### Все последующие команды будут выполняться здесь
WORKDIR ${APP_HOME}

## КОПИРОВАНИЕ ФАЙЛОВ
### COPY копирует файлы с хоста в образ
### Сначала копируем requirements - это оптимизирует кэш
COPY requirements.txt .

## ВЫПОЛНЕНИЕ КОМАНД
### RUN выполняет команду во время сборки
### --no-cache-dir не сохраняет кэш pip (экономит место)
RUN pip install --no-cache-dir -r requirements.txt

### Копируем весь код после установки зависимостей
### (если код изменится, зависимости не переустановятся)
COPY . .

## ПОЛЬЗОВАТЕЛЬ (безопасность)
### Создаём непривилегированного пользователя
RUN adduser -D myuser
USER myuser

## ДОКУМЕНТАЦИЯ ПОРТОВ
### EXPOSE - только документация, не открывает порт!
### Реальный проброс делается через -p при запуске
EXPOSE 8000

## ТОМА (документация)
### VOLUME - рекомендация, где хранить данные
### Реальный том создаётся при запуске
VOLUME /data

## HEALTHCHECK (проверка здоровья)
### Docker будет проверять, живо ли приложение
### interval - как часто проверять
### timeout - сколько ждать ответа
### retries - сколько попыток до статуса unhealthy
### start_period - время на запуск перед проверками
HEALTHCHECK --interval=30s --timeout=10s --retries=3 --start-period=5s \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/')" || exit 1

## ТОЧКА ВХОДА И КОМАНДА
### ENTRYPOINT - главная команда (глагол)
### CMD - аргументы по умолчанию
### Всегда используем exec form (массив)!
ENTRYPOINT ["python"]
CMD ["app.py"]

## Multi-stage build (оптимизация размера)
### ЭТАП 1: СБОРКА (тяжёлый образ)
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

## ЭТАП 2: ПРОДАКШЕН (лёгкий образ)

FROM nginx:alpine

### Копируем только собранные файлы из первого этапа
COPY --from=builder /app/dist /usr/share/nginx/html

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

---

# 📋 ЧАСТЬ 4: СИНТАКСИС DOCKER-COMPOSE.YML

version: '3.8'

### СЕРВИСЫ (контейнеры)
services:
  
  ### --- ВЕБ-ПРИЛОЖЕНИЕ ---
  web:
    ### build: собрать образ из Dockerfile в текущей папке
    build: .
    
    ### image: использовать готовый образ (вместо build)
    ### image: nginx:alpine
    
    ### container_name: фиксированное имя контейнера
    container_name: my-web-app
    
    ### ports: проброс портов [хост:контейнер]
    ports:
      - "8080:80"
      - "8443:443"
    
    ### environment: переменные окружения
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
      - REDIS_HOST=redis
      ### Или через файл .env:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    
    ### env_file: загрузить переменные из файла
    env_file:
      - .env
      - .env.local
    
    ### volumes: монтирование томов [хост:контейнер]
    volumes:
      ### Именованный том (управляется Docker)
      - pgdata:/var/lib/postgresql/data
      ### Bind mount (папка с хоста)
      - ./src:/app/src
      ### Read-only mount
      - ./config:/app/config:ro
    
    ### depends_on: зависимости (порядок запуска)
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    
    ### restart: политика перезапуска
    ### always - всегда перезапускать
    ### unless-stopped - кроме ручного останова
    ### on-failure - только при ошибке
    restart: always
    
    ### networks: подключение к сетям
    networks:
      - frontend
      - backend
    
    ### healthcheck: проверка здоровья
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
    
    ### deploy: ограничения ресурсов (для Swarm)
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    
    ### logging: настройка логов
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  ### --- БАЗА ДАННЫХ ---
  db:
    image: postgres:15-alpine
    container_name: my-postgres
    
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    
    volumes:
      - pgdata:/var/lib/postgresql/data
      ### Инициализационные скрипты (выполняются при первом запуске)
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    
    networks:
      - backend
    
    restart: always
    
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  ### --- REDIS (КЭШ) ---
  redis:
    image: redis:7-alpine
    container_name: my-redis
    
    ### command: переопределить команду запуска
    command: redis-server --requirepass ${REDIS_PASSWORD}
    
    volumes:
      - redisdata:/data
    
    networks:
      - backend
    
    restart: always
    
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5


### СЕТИ
networks:
  ### frontend: доступна извне
  frontend:
    driver: bridge
  
  ### backend: только для внутренних сервисов
  backend:
    driver: bridge
    internal: true  ### Нет доступа извне!

### ============================================
### ТОМА
### ============================================
volumes:
  ### Именованные тома (управляются Docker)
  pgdata:
    driver: local
  
  redisdata:
    driver: local

---

# ⚡ БЫСТРЫЙ СПИСОК: ЧТО И ЗАЧЕМ

### Dockerfile инструкции

| Инструкция | Зачем нужна |
|------------|-------------|
| `FROM` | Базовый образ (фундамент) |
| `RUN` | Выполнить команду при сборке (установка пакетов) |
| `COPY` | Скопировать файлы с хоста в образ |
| `ADD` | COPY + распаковка архивов + скачивание из URL |
| `WORKDIR` | Установить рабочую директорию |
| `ENV` | Задать переменные окружения |
| `EXPOSE` | Документация: какой порт слушает приложение |
| `VOLUME` | Документация: где хранить данные |
| `CMD` | Команда по умолчанию при запуске (можно переопределить) |
| `ENTRYPOINT` | Главная команда (нельзя переопределить, только дополнить) |
| `USER` | От какого пользователя запускать (безопасность) |
| `HEALTHCHECK` | Как проверить, что приложение живо |
| `LABEL` | Метаданные образа (версия, автор) |
| `ARG` | Переменные только для сборки (не попадают в образ) |

### Docker Compose ключи

| Ключ | Зачем нужен |
|------|-------------|
| `build` | Собрать образ из Dockerfile |
| `image` | Использовать готовый образ |
| `ports` | Проброс портов хост→контейнер |
| `volumes` | Монтирование файлов/папок |
| `environment` | Переменные окружения |
| `env_file` | Загрузить переменные из файла |
| `depends_on` | Зависимости между сервисами |
| `restart` | Политика перезапуска |
| `networks` | Подключение к сетям |
| `healthcheck` | Проверка здоровья сервиса |
| `command` | Переопределить команду запуска |
| `entrypoint` | Переопределить точки входа |

### Флаги docker run

| Флаг | Что делает |
|------|-----------|
| `-d` | Запуск в фоне (detach) |
| `-p 8080:80` | Проброс порта |
| `-v /host:/container` | Монтирование тома |
| `-e VAR=value` | Переменная окружения |
| `--name myapp` | Имя контейнера |
| `--network mynet` | Подключить к сети |
| `--rm` | Удалить контейнер после остановки |
| `-it` | Интерактивный режим + терминал |
| `--restart always` | Автоперезапуск |
| `--env-file .env` | Загрузить переменные из файла |

---

## ПОЛЕЗНЫЕ ОДНОСТРОЧНИКИ

### Удалить все stopped контейнеры
docker rm $(docker ps -aq -f status=exited)

### Удалить все dangling образы
docker rmi $(docker images -q -f dangling=true)

### Посмотреть IP всех контейнеров
docker inspect -f '{{.Name}} - {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $(docker ps -q)

### Остановить все контейнеры
docker stop $(docker ps -q)

### Удалить все контейнеры
docker rm -f $(docker ps -aq)

### Посмотреть размер всех образов
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

### Найти контейнеры по имени
docker ps -f name=myapp

### Посмотреть переменные окружения контейнера
docker exec <container> env

### Копировать файл из контейнера
docker cp <container>:/path/to/file ./local/file

### Запустить bash в контейнере (если нет bash, используй sh)
docker exec -it <container> /bin/bash
docker exec -it <container> /bin/sh

### Пересобрать и перезапустить один сервис
docker compose up -d --build myservice

### Посмотреть логи всех сервисов за последние 100 строк
docker compose logs --tail 100

### Выполнить команду в сервисе
docker compose exec web python manage.py migrate

---

##  ЧАСТЫЕ ОШИБКИ И РЕШЕНИЯ

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `port is already allocated` | Порт занят другим процессом | `lsof -i :PORT` и убить процесс |
| `No such image` | Образ не скачан | `docker pull <image>` |
| `Cannot connect to Docker daemon` | Docker не запущен | `sudo systemctl start docker` |
| `permission denied` | Нет прав на docker.sock | `sudo usermod -aG docker $USER` |
| `container exited with code 1` | Приложение упало | `docker logs <container>` |
| `unknown shorthand flag` | Неправильный синтаксис | Проверить порядок флагов |
| `build context too large` | Много файлов в папке | Добавить `.dockerignore` |
| `service unhealthy` | Healthcheck не проходит | Проверить команду healthcheck |
| `connection refused` | Сервис ещё не готов | Добавить `depends_on` с `condition: service_healthy` |
| `module not found` | Зависимость не установлена | Пересобрать с `--no-cache` |

---

##ЧАСТЬ 5: ВОПРОСЫ С СОБЕСЕДОВАНИЙ
Примечание: Вопросы про чистый Docker сейчас задают редко, обычно они всплывают в контексте Kubernetes. Но база должна быть железной.
###1. Чем контейнеризация отличается от виртуализации?
Краткий ответ:
Виртуальные машины (VM): Имеют полноценную гостевую ОС, гипервизор (Type 1 или Type 2). Тяжёлые, долго стартуют, потребляют много ресурсов (гигабайты RAM/диска).
Контейнеры: Используют ядро хостовой ОС. Изоляция происходит на уровне процессов. Лёгкие, стартуют за миллисекунды, потребляют мегабайты ресурсов.
Итог: VM изолируют на уровне железа (через гипервизор), контейнеры изолируют на уровне ОС.
###2. За счёт каких технологий ядра Linux обеспечивается изоляция контейнеров?
Краткий ответ:
Две основные технологии:
Namespaces (Пространства имён): Отвечают за изоляцию. Они создают иллюзию, что у контейнера есть своё собственное окружение.
PID (процессы)
NET (сеть, интерфейсы)
MNT (файловая система, точки монтирования)
IPC (межпроцессное взаимодействие)
UTS (hostname)
USER (пользователи и группы)
Cgroups (Control Groups): Отвечают за ограничение ресурсов. Не дают контейнеру съесть всю память или CPU хоста (лимиты RAM, CPU, I/O).
UnionFS (OverlayFS): Объединённая файловая система, которая позволяет слоям образа накладываться друг на друга, экономя место и ускоряя сборку.
###. Зачем нужен multi-stage build?
Краткий ответ:
Для уменьшения размера финального образа и повышения безопасности.
Как работает: В первом этапе (stage) используется тяжёлый образ со всеми инструментами сборки (компиляторы, SDK, node_modules). Во втором этапе берётся минимальный базовый образ (например, alpine или scratch), и туда копируется только скомпилированный артефакт/бинарник из первого этапа.
Результат: Образ может похудеть с 1GB до 50MB. В финальном образе нет исходного кода, компиляторов и уязвимостей инструментов сборки.
###4. Какие есть best practice'ы для написания Dockerfile?
Краткий ответ:
Использовать минимальные базовые образы (alpine, slim, distroless).
Правильный порядок слоёв (Layer Caching): Сначала копировать файлы зависимостей (package.json, requirements.txt) и устанавливать их, и только потом копировать исходный код. Это сохраняет кэш при изменении кода.
Не запускать от root: Создавать непривилегированного пользователя (USER) для безопасности.
Использовать .dockerignore: Чтобы не тащить в контекст сборки .git, node_modules, .env.
Использовать COPY вместо ADD (если не нужна распаковка tar или URL).
Использовать Exec Form (["cmd", "arg"]) для CMD и ENTRYPOINT, чтобы избегать проблем с сигналами (SIGTERM).
Одна задача на слой: Объединять команды через && и чистить кэш в одном RUN, чтобы уменьшить количество слоёв и размер образа.
Использовать Multi-stage build для продакшена.
###5. Разница CMD и ENTRYPOINT
Краткий ответ:
ENTRYPOINT — это "глагол", главная команда контейнера. Её нельзя заменить аргументами при запуске docker run, они лишь добавляются к ней.
CMD — это "аргументы по умолчанию". Полностью заменяется аргументами, переданными в docker run.
Взаимодействие: Если используются оба, ENTRYPOINT задаёт команду, а CMD передаёт ей аргументы по умолчанию.
Правило: Всегда использовать Exec Form (массив []), чтобы избежать создания лишнего процесса-оболочки (sh -c), который блокирует передачу сигналов завершения.
###6. Разница COPY и ADD
Краткий ответ:
COPY — просто копирует файлы и папки с хост-машины в образ. Предсказуем и прозрачен.
ADD — умеет всё, что умеет COPY, плюс автоматически распаковывает локальные .tar архивы и может скачивать файлы по URL.
Best Practice: Официальная документация Docker рекомендует всегда использовать COPY, если вам не нужна специфичная магия распаковки tar-архивов. ADD делает поведение Dockerfile менее очевидным.
