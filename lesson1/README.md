# Занятие 1 — собираем пайплайн вместе с преподавателем

**Что делаем:** проходим всю цепочку от коммита до работающего
приложения. Преподаватель показывает шаг на экране — вы повторяете
у себя. Не отставайте: каждый шаг опирается на предыдущий.

**Итог занятия:**

```
git push → GitHub Actions: lint → test → build → Trivy → push в GHCR
                                                        ↓
                            Artifactory (remote-репозиторий, кэширует GHCR)
                                                        ↓
                            docker compose pull → up → curl /health
```

---

## Шаг 0. Подготовка

**0.1. Свой репозиторий на GitHub.** Создайте **пустой** репозиторий
(без README!) с именем `pipeline-lab` — имя должно быть строчными
буквами, это важно для реестра образов. Затем выгрузите код:

```bash
cd pipeline-lab-students
git init
git add .
git commit -m "init: демо-приложение и стенд"
git branch -M main
git remote add origin git@github.com:<ВПИШИТЕ логин>/pipeline-lab.git
git push -u origin main
```

**0.2. Стенд реестра.** В этой лаборатории — JFrog Container Registry
(JCR): бесплатная версия Artifactory с поддержкой docker-репозиториев
(artifactory-oss из прошлой лабы их не поддерживает). Если стенд
прошлой лабы ещё запущен — сначала остановите его там
(`docker compose down`), иначе порты займет он. Затем:

```bash
docker compose up -d
```

Дождитесь, пока откроется http://localhost:8082 (admin / password).
Контейнер один — база встроенная, postgres здесь не нужен.

**0.3. Токен для GHCR.** GitHub → Settings → Developer settings →
Personal access tokens → Tokens (classic) → Generate new token.
Права: `read:packages`, `write:packages`. Сохраните токен в заметки —
он понадобится в шаге 5 для Artifactory.

**0.4. jfrog CLI** уже настроен с прошлой лабораторной. Проверьте:

```bash
jf c show
```

Если пусто — добавьте заново: `jf c add` (URL http://localhost:8082).

**В отчёт:** скриншот пустого репозитория на GitHub после первого пуша.

<img width="1544" height="704" alt="image" src="https://github.com/user-attachments/assets/7ce77507-b0d3-4fdf-b814-fc1aa3e84f34" />

---

## Шаг 1. Локальный прогон — то, что CI будет делать за нас

Пайплайн не должен делать ничего, чего вы не можете сделать руками.
Сначала руками:

```bash
cd app
python3 -m venv venv
. venv/bin/activate
pip install -r requirements-dev.txt
ruff check .
pytest tests -v
```

Всё зелёное? Посмотрите на приложение глазами:

```bash
python -m flask --app app run --port 8000 &
curl localhost:8000
curl localhost:8000/health
kill %1
```

**Вопрос преподавателю (устно):** зачем запускать это локально,
если то же самое сделает CI?

---

## Шаг 2. Первый workflow: lint + test

Создайте файл `.github/workflows/ci.yml`. Набирайте сами, а не
копируйте — обсуждаем каждую строку с преподавателем:

```yaml
name: ci

on:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  lint-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - run: pip install -r app/requirements-dev.txt

      - run: ruff check app

      - run: pytest app/tests -v
```

```bash
git add .github/workflows/ci.yml
git commit -m "ci: lint и тесты"
git push
```

Откройте вкладку **Actions** на GitHub. Найдите прогон, зайдите
внутрь, разверните шаги. Найдите в логе строку `3 passed`.

**Обсуждаем:** что такое `on`, `jobs`, `runs-on`; почему шаги
`uses`, а не только `run`.

**В отчёт:** скриншот зелёного прогона во вкладке Actions.

---

## Шаг 3. Собираем Docker-образ в CI

Добавьте в конец файла вторую джобу:

```yaml
  docker:
    needs: lint-test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: |
          docker build --build-arg APP_VERSION=${{ github.sha }} \
            -t ghcr.io/<ВПИШИТЕ логин>/pipeline-lab:${{ github.sha }} app
```

Пушим, смотрим прогон. Пока образ просто собирается и остаётся
на раннере — мы проверяем, что Dockerfile вообще рабочий.

**Обсуждаем:** `needs` — зачем; что такое `github.sha`; почему тег
образа — это хэш коммита, а не `latest`.

---

## Шаг 4. Trivy: сканируем образ и чиним

В джобу `docker` между сборкой и концом добавьте шаг сканирования:

```yaml
      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@v0.36.0
        with:
          image-ref: ghcr.io/<ВПИШИТЕ логин>/pipeline-lab:${{ github.sha }}
          severity: CRITICAL
          exit-code: "1"
```

Пушим. **Пайплайн должен упасть.** Это ожидаемо: наш Dockerfile
наследует `python:3.8-slim` — образу много лет, поддержка прекращена.

Разбираем с преподавателем отчёт Trivy в логе:
- что такое CVE и severity;
- чем `CRITICAL` отличается от `MEDIUM`;
- почему `exit-code: "1"` роняет пайплайн — и почему это хорошо.

Пока смотрим только `CRITICAL`. На HIGH тоже будут находки (их
десятки даже на свежем образе) — **как жить с ними, разберём
на занятии 2** в задании про `.trivyignore`.

**Чиним.** Откройте `app/Dockerfile` и замените первую строку:

```dockerfile
FROM python:3.____-slim
```

(актуальную версию подскажет преподаватель — возьмите свежий
стабильный Python). Пушим:

```bash
git add app/Dockerfile
git commit -m "fix: свежий базовый образ, уходим от CVE"
git push
```

Пайплайн зелёный. **Вывод:** сканер уязвимостей — часть пайплайна,
а не отдельная процедура «когда-нибудь потом».

**В отчёт:** скриншот упавшего прогона с отчётом Trivy и скриншот
зелёного прогона после починки.

---

## Шаг 5. Публикуем образ в GHCR

Образ нужно где-то хранить. Используем GitHub Container Registry
(ghcr.io) — он бесплатен и рядом с кодом.

**5.1.** Добавьте в начало workflow рядом с `permissions`:

```yaml
permissions:
  contents: read
  packages: write
```

**5.2.** В джобу `docker` после сканирования добавьте логин и пуш:

```yaml
      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Push image
        uses: docker/build-push-action@v6
        with:
          context: app
          push: true
          build-args: APP_VERSION=${{ github.sha }}
          tags: |
            ghcr.io/<ВПИШИТЕ логин>/pipeline-lab:${{ github.sha }}
            ghcr.io/<ВПИШИТЕ логин>/pipeline-lab:latest
```

`GITHUB_TOKEN` создавать не нужно — GitHub выдаёт его каждому прогону
автоматически. Образ пересоберётся из кэша (секунды) и уедет в GHCR.
Ручной `docker push` здесь не используем: с токеном GitHub Actions
он отвечает `denied`, а action делает то же самое надёжно. Пушим.

**5.3.** Проверьте: GitHub → ваш профиль → вкладка **Packages** —
там появился пакет `pipeline-lab`. Откройте его и сделайте **public**
(Package settings → Change visibility) — иначе Artifactory не
сможет его скачать без лишней возни.

**Обсуждаем:** чем реестр образов отличается от git-репозитория;
почему пушим два тега.

**В отчёт:** скриншот страницы пакета в GHCR с двумя тегами.

---

## Шаг 6. Artifactory перед деплоем: единая точка входа

Договорённость «как в большой компании»: машины, где работает
приложение, ходят **только в Artifactory**, наружу — никогда.
Artifactory сам забирает образы из GHCR и кэширует у себя.

**6.1. Remote-репозиторий.** UI → Administration → Repositories →
Add Repositories → Remote Repository → Docker:

- Repository Key: `docker-ghcr`
- URL: `https://ghcr.io`
- User Name: ваш логин GitHub
- Password / Access Token: токен из шага 0.3

**6.2. Virtual-репозиторий.** Add Repositories → Virtual Repository →
Docker:

- Repository Key: `docker`
- В список Included добавьте `docker-ghcr` — виртуальный репозиторий
  станет единой точкой входа поверх него.

**6.3. Доступ по порту.** Administration → Artifactory → General →
HTTP Settings: Docker Access Method = **Port**. Затем откройте настройки
репозитория `docker` и впишите порт **8090** (мы пробросили его в
`docker-compose.yml` заранее).

**6.4. Проверяем pull-through:**

```bash
docker login localhost:8090   # логин/пароль от Artifactory
docker pull localhost:8090/<ВПИШИТЕ логин>/pipeline-lab:latest
```

Образ скачался. Теперь найдите его в UI Artifactory: Application →
Artifactory → Packages — он лежит в кэше `docker-ghcr`. **Artifactory
сходил в GHCR за вас и сохранил копию.**

---

## Шаг 7. Деплой

Скопируйте шаблон и впишите свои значения:

```bash
cp templates/docker-compose.deploy.yml deploy/docker-compose.yml  # или просто в корень
```

```yaml
services:
  app:
    image: localhost:8090/<ВПИШИТЕ логин>/pipeline-lab:latest
    container_name: demo-app
    ports:
      - "8080:8000"
    restart: unless-stopped
```

```bash
docker compose -f docker-compose.deploy.yml up -d
curl localhost:8080/health
curl localhost:8080
```

В поле `version` — хэш вашего коммита. Сверьте с `git log --oneline`.

**Цепочка замкнулась:** коммит → проверки → скан → реестр →
Artifactory → работающий контейнер.

**В отчёт:** вывод `curl localhost:8080` и `docker compose ps`.

---

## Если не получилось

- `repository name must be lowercase` — в имени образа есть заглавные
  буквы (логин GitHub или имя репо). Впишите их строчными.
- Trivy падает с `TOOMANYREQUESTS` — GitHub ограничивает скачивание
  базы уязвимостей. Подождите пару минут и перезапустите джобу кнопкой
  Re-run.
- `denied: permission_denied` при пуше — нет шага Login to GHCR или
  нет `packages: write` в permissions.
- `docker pull localhost:8090/...` отвечает 404 — проверьте, что пакет
  в GHCR публичный, а в remote-репозитории верные логин и токен.
