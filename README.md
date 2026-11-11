# ДЗ 3: Docker, вариант на 8 баллов

Приложение со счётчиком упаковано в Docker. Основной вариант запускается через
Compose с отдельным Redis. Для последовательной проверки уровня 6 добавлен
отдельный `compose.sqlite.yaml`.

[Условие задания](https://github.com/Nikfilk2030/missing_semester_hse_cs/blob/main/2026/amcs/year_1/hw_3/README.md).
Исходники из [шаблона](https://github.com/null-ref-exc/hw_3_template) в папке `app/`
сохранены без изменений. Результаты проверок находятся в [REPORT.md](REPORT.md).

## Подготовка

Нужны Docker Engine с Compose либо Docker Desktop, а также `curl`.
Команды ниже выполняются из корня репозитория в Bash, например в Ubuntu через WSL.

На этом компьютере Docker установлен в WSL-дистрибутиве `Ubuntu-26.04`.
Из PowerShell можно открыть папку проекта в Ubuntu командой:

```powershell
wsl -d Ubuntu-26.04 --cd "C:\Users\77071\OneDrive\Документы\ChatGPT\Домашнее задание\hw_3_docker"
```

В открывшемся терминале:

```bash
docker version
docker compose version
cp .env.example .env
```

В `.env` можно изменить `APP_PORT` — порт на компьютере, и `GREETING` — приветствие.
Внутри контейнера приложение слушает порт 8000. В примерах используется
`APP_PORT=8080`; при его изменении нужно заменить порт и в адресах запросов.

## Уровень 4: один контейнер

```bash
docker build -t docker-hw:local .
docker run -d --name docker-hw-single -p 127.0.0.1:8080:8000 -e "GREETING=Hello from a single container!" docker-hw:local
curl --retry 10 --retry-all-errors --retry-delay 1 --max-time 3 -fsS http://127.0.0.1:8080/
curl --max-time 3 -fsS -X POST http://127.0.0.1:8080/increment
curl --max-time 3 -fsS http://127.0.0.1:8080/
docker logs docker-hw-single
docker rm -f docker-hw-single
```

В новом контейнере значение `count` меняется с 0 на 1. В ответе присутствует
переданное приветствие, в логах — запуск сервера и HTTP-запросы.

## Уровень 6: Compose и SQLite volume

```bash
docker compose -p docker-hw-sqlite -f compose.sqlite.yaml up -d --build --wait --wait-timeout 120
curl --max-time 3 -fsS -X POST http://127.0.0.1:8080/increment
curl --max-time 3 -fsS -X POST http://127.0.0.1:8080/increment
curl --max-time 3 -fsS http://127.0.0.1:8080/
docker compose -p docker-hw-sqlite -f compose.sqlite.yaml down
docker compose -p docker-hw-sqlite -f compose.sqlite.yaml up -d --wait --wait-timeout 120
curl --max-time 3 -fsS http://127.0.0.1:8080/
docker compose -p docker-hw-sqlite -f compose.sqlite.yaml down
```

Записать `count` до и после пересоздания: значения должны совпасть. На новом
хранилище после двух POST получается 2. Данные SQLite находятся в именованном
volume `sqlite_data`, который подключён к `/data`.

## Уровень 8: приложение и Redis

```bash
docker compose up -d --build --wait --wait-timeout 120
docker compose ps
curl --max-time 3 -fsS http://127.0.0.1:8080/
```

В `compose.yaml` два сервиса: `app` и `redis`. Приложение получает
`STORAGE=redis` и `REDIS_URL=redis://redis:6379/0`. Имя `redis` разрешается во
внутренней сети Compose. На компьютер опубликован только порт приложения.
Для `app` ограничено время DNS-запросов и отключён поиск по доменам хоста:
при остановленном Redis это позволяет быстро получить ответ 503.

Redis подключает именованный volume `redis_data` к `/data`. Параметр
`--appendonly yes` включает журнал AOF, `--appendfsync everysec` задаёт
синхронизацию на диск раз в секунду. При штатной остановке Redis сохраняет
оставшиеся изменения. [Описание AOF](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/).

У обоих сервисов есть healthcheck. Redis проверяется через `redis-cli ping`,
приложение — через `/readyz`, включая доступность хранилища. При первоначальном
старте `app` ждёт готовности Redis через `depends_on: condition: service_healthy`.
[Порядок запуска Compose](https://docs.docker.com/compose/how-tos/startup-order/).

### Сохранение счётчика

```bash
curl --max-time 3 -fsS -X POST http://127.0.0.1:8080/increment
curl --max-time 3 -fsS -X POST http://127.0.0.1:8080/increment
curl --max-time 3 -fsS -X POST http://127.0.0.1:8080/increment
curl --max-time 3 -fsS http://127.0.0.1:8080/
docker compose down
docker compose up -d --wait --wait-timeout 120
curl --max-time 3 -fsS http://127.0.0.1:8080/
```

Записать значение `N` перед `down`: после запуска должно вернуться то же `N`.
На новом хранилище после трёх POST получается 3. Здесь используется `down`
без `-v`: ключ `-v` удаляет volume и его данные.

### Остановка и восстановление Redis

```bash
docker compose stop redis
curl --max-time 5 -sS -i http://127.0.0.1:8080/healthz
curl --max-time 5 -sS -i http://127.0.0.1:8080/readyz
curl --max-time 5 -sS -i -X POST http://127.0.0.1:8080/increment
docker compose start redis
curl --retry 20 --retry-all-errors --retry-delay 1 --max-time 3 -fsS http://127.0.0.1:8080/readyz
curl --max-time 3 -fsS http://127.0.0.1:8080/
curl --max-time 3 -fsS -X POST http://127.0.0.1:8080/increment
docker compose ps
docker compose logs --tail=30 app redis
```

При остановленном Redis ожидаются HTTP 200 для `/healthz` и HTTP 503 для
`/readyz` и `/increment`. После `start redis` приложение само восстанавливает
соединение: GET возвращает `N`, следующий POST — `N + 1`. Контейнер приложения
перезапускать не нужно. После очередной проверки здоровья оба сервиса будут
помечены как `healthy`.

Остановить проект с сохранением данных:

```bash
docker compose down
```

## Сдача

Создать свой репозиторий на GitHub и загрузить в него `app/`, `Dockerfile`,
`.dockerignore`, `.env.example`, оба файла Compose, `README.md` и `REPORT.md`.
В приватном репозитории предоставить доступ преподавателям. Ссылку на репозиторий
отправить через форму из условия задания. Локальный `.env` добавлять не нужно.
