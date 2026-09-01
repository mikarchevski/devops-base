#  Шпаргалка по CI/CD

> Полное руководство по Continuous Integration и Continuous Deployment для DevOps-инженера

---

##  Содержание

1. [Основные понятия](#основные-понятия)
2. [Принципы построения CI/CD](#принципы-построения-cicd)
3. [Стратегии ветвления](#стратегии-ветвления)
4. [Инструменты интеграции](#инструменты-интеграции)
5. [GitOps](#gitops)
6. [GitHub Actions — быстрый старт](#github-actions--быстрый-старт)
7. [GitLab CI vs GitHub Actions](#gitlab-ci-vs-github-actions)
8. [Топ вопросов на собеседовании](#топ-вопросов-на-собеседовании)

---

## Основные понятия

### CI/CD расшифровка

| Термин | Расшифровка | Суть |
|--------|-------------|------|
| **CI** | Continuous Integration | Частое слияние кода + автотесты |
| **CD** | Continuous Delivery | Код готов к релизу, но деплой **ручной** |
| **CD** | Continuous Deployment | Код попадает в прод **автоматически** |

### Простая аналогия

> **CI** — повар нарезал ингредиенты и проверил их на свежесть  
> **Delivery** — блюдо готово, официант ждёт команды нести гостю  
> **Deployment** — робот-официант сам везёт блюдо, как только оно готово

---

## Принципы построения CI/CD

### 6 ключевых принципов

1. **Автоматизация всего** — если действие повторяется >2 раз, автоматизируй
2. **Fail Fast** — быстрые проверки (линтеры) в начале, долгие (тесты) потом
3. **Воспроизводимость** — одинаковый результат на любой машине (Docker, фиксация версий)
4. **Идемпотентность** — повторный запуск не ломает систему
5. **Безопасность** — секреты только в Variables/Secrets, никогда в коде
6. **Git как источник истины** — конфиги пайплайна версионируются в Git

### Шпаргалка для собеседования

> «Я опираюсь на 4 принципа: **Fail Fast**, **Идемпотентность**, **Безопасность** и **Автоматизация**. Ручное вмешательство допускается только на этапе релиза в Production.»

---

## Стратегии ветвления

### Сравнительная таблица

| Стратегия | Ветки | Релизы | Сложность CI/CD | Для чего |
|-----------|-------|--------|-----------------|----------|
| **GitFlow** | `main`, `develop`, `feature/*`, `release/*`, `hotfix/*` | Раз в месяц/квартал | Высокая | Mobile, Enterprise |
| **GitHub Flow** | `main`, `feature/*` | Раз в неделю/день | Средняя | Web, SaaS, стартапы |
| **Trunk-Based** | Только `main` | Несколько раз в день | Низкая (но нужны тесты) | Highload, зрелые DevOps |

### Как CI/CD зависит от стратегии

    GitFlow:
      feature/* → линтеры + тесты
      develop   → автодеплой на staging
      main      → ручной деплой на prod

    GitHub Flow:
      PR        → автотесты + код-ревью
      merge→main → автодеплой

    Trunk-Based:
      любой коммит в main → полный пайплайн → автодеплой

---

## Инструменты интеграции

### Docker в CI/CD

**Зачем:** Гарантирует воспроизводимость окружения

**Как работает:**

    CI:  docker build → docker push в Registry
    CD:  docker pull → docker compose up -d

**Плюсы:**
-  Нет «работает на моей машине»
-  Мгновенный откат к старой версии
-  Изоляция зависимостей

### Ansible в CI/CD

**Зачем:** Идемпотентное управление конфигурацией

**Как работает:**

    # Вместо хрупких ssh-команд:
    - name: Deploy with Ansible
      run: ansible-playbook -i inventory.ini deploy.yml

**Плюсы:**
-  Декларативность (описываешь *что*, а не *как*)
-  Идемпотентность (повторный запуск безопасен)
-  Читаемость

---

## GitOps

### Суть в одном предложении

> **GitOps** — Git является единственным источником истины, а агент (ArgoCD/Flux) автоматически синхронизирует реальное состояние с Git.

### Push vs Pull модель

    Классический CI/CD (Push):
      Разработчик → git push → GitHub Actions → ssh/kubectl → СЕРВЕР

    GitOps (Pull):
      Разработчик → git push → Git ← ArgoCD ← СЕРВЕР

### 4 принципа GitOps

1. **Декларативность** — описываем желаемое состояние
2. **Git = единственный источник истины**
3. **Автоматическая синхронизация** — агент сам исправляет расхождения
4. **Audit Trail** — вся история в Git

### Когда GitOps НЕ нужен

- Простой проект на одном сервере
- Нет Kubernetes
- Маленькая команда, редкие деплои

---

## GitHub Actions — быстрый старт

### Структура workflow

    name: CI/CD Pipeline

    on:
      push:
        branches: [main]
      pull_request:
        branches: [main]

    env:
      PYTHON_VERSION: '3.11'

    jobs:
      test:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v4
          - name: Set up Python
            uses: actions/setup-python@v5
            with:
              python-version: ${{ env.PYTHON_VERSION }}
              cache: 'pip'
          - name: Install deps
            run: pip install -r requirements.txt
          - name: Lint
            run: flake8 .
          - name: Test
            run: pytest

      deploy:
        needs: test
        runs-on: self-hosted
        environment: production  # ← Manual approval
        steps:
          - uses: actions/checkout@v4
          - name: Deploy
            run: |
              rsync -az --delete ./ ${{ secrets.PROD_HOST }}:/opt/app/
              ssh ${{ secrets.PROD_HOST }} 'systemctl restart app'

### Ключевые концепции

| Концепция | Описание |
|-----------|----------|
| `workflow` | Весь CI/CD процесс |
| `job` | Отдельная задача (test, build, deploy) |
| `steps` | Шаги внутри job |
| `needs` | Зависимость между jobs |
| `secrets` | Секреты (пароли, токены) |
| `vars` | Переменные (не секретные) |
| `environment` | Окружение с manual approval |

### Безопасность: secrets vs vars

    #  Правильно:
    url: ${{ vars.PROD_URL }}           # Публичный URL
    host: ${{ secrets.PROD_HOST }}      # SSH доступ

    #  Неправильно:
    url: https://disk.mk5d.ru           # В коде — опасно!

### Настройка Manual Approval

1. **Settings → Environments → New environment** (`production`)
2. Включить **Required reviewers**
3. Добавить ревьюеров
4. В workflow: `environment: production`

---

## GitLab CI vs GitHub Actions

| GitLab CI | GitHub Actions | Описание |
|-----------|----------------|----------|
| `.gitlab-ci.yml` | `.github/workflows/*.yml` | Файл конфига |
| `pipeline` | `workflow` | Весь процесс |
| `stage` | `jobs` (нет прямого аналога) | Группа задач |
| `job` | `job` | Отдельная задача |
| `script` | `steps` | Команды |
| `rules/only` | `on` | Триггеры запуска |
| `variables` | `env` / `secrets` | Переменные |
| `artifacts` | `actions/upload-artifact` | Передача файлов |
| `when: manual` | `environment` + reviewers | Ручной апрув |
| GitLab Runner | GitHub Runner | Исполнитель |

---

## Топ вопросов на собеседовании

###  Вопрос 1: «Как бы ты построил идеальный пайплайн?»

**Ответ:**
> 1. **Fail Fast** — линтеры и компиляция в начале
> 2. **Параллельность** — независимые задачи выполняются одновременно
> 3. **Идемпотентность** — через Docker и фиксацию версий
> 4. **Безопасность** — секреты в Variables, код чист
> 5. **Ручной контроль на проде** — Continuous Delivery с manual approval

###  Вопрос 2: «В чём разница CI, Continuous Delivery, Continuous Deployment?»

**Ответ:**
> - **CI** — частые мерджи + автотесты
> - **Delivery** — код готов к релизу, но деплой **ручной**
> - **Deployment** — код попадает в прод **автоматически** после тестов

###  Вопрос 3: «Что такое GitOps?»

**Ответ:**
> Подход, где Git — единственный источник истины. Агент (ArgoCD) внутри кластера мониторит Git и автоматически синхронизирует состояние (Pull-модель). В отличие от классического CI/CD (Push-модель), CI-серверу не нужен доступ к прод.

###  Вопрос 4: «Как Docker и Ansible встраиваются в CI/CD?»

**Ответ:**
> - **Docker** решает воспроизводимость: собираем образ в CI, тестируем, пушим в Registry. На проде запускаем тот же образ.
> - **Ansible** решает идемпотентность: вместо хрупких bash-скриптов описываем желаемое состояние декларативно.

###  Вопрос 5: «Какую стратегию ветвления вы используете?»

**Ответ:**
> «GitHub Flow: `main` всегда стабильна, разработка в `feature/*` ветках. PR запускает автотесты, после мерджа — автодеплой на staging, в прод — через manual approval. Стремимся к Trunk-Based, но нужно поднять покрытие тестами.»

---

##  Чек-лист идеального CI/CD

- [ ] Линтеры и компиляция в начале пайплайна
- [ ] Параллельное выполнение независимых задач
- [ ] Кэширование зависимостей (`cache: 'pip'`)
- [ ] Секреты в Variables/Secrets, не в коде
- [ ] Фиксация версий зависимостей
- [ ] Docker для воспроизводимости
- [ ] Manual approval для production
- [ ] Smoke test после деплоя
- [ ] Откат через старый тег образа или `git revert`
- [ ] Конфиги пайплайна в Git

---

##  Полезные ссылки

- [GitHub Actions документация](https://docs.github.com/en/actions)
- [GitLab CI документация](https://docs.gitlab.com/ee/ci/)
- [GitFlow vs GitHub Flow](https://www.atlassian.com/git/tutorials/comparing-workflows)
- [GitOps принципы](https://opengitops.dev/)

---

> 💡 **Совет:** Эта шпаргалка покрывает ~80% вопросов по CI/CD на собеседованиях уровня Middle DevOps. Для Senior уровня добавь глубокое знание Kubernetes, ArgoCD и построение платформенных решений.

**Удачи на собеседованиях! 🚀**
