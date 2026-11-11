# ДЗ 3. Docker

Приложение со счётчиком запускается в Docker. Исходники в `app/` взяты из
[шаблона](https://github.com/null-ref-exc/hw_3_template) без изменений.

## Настройки

Нужны Docker с Compose и curl. Команды выполняются из корня репозитория в Bash.

```bash
cp .env.example .env
```

В `.env` задаются:

| Переменная | Значение в примере | Назначение |
| --- | --- | --- |
| `APP_PORT` | `8080` | Порт приложения на компьютере |
| `GREETING` | `Hello from Docker!` | Приветствие в ответе |

Внутри контейнера приложение слушает порт 8000. При изменении `APP_PORT`
нужно заменить порт и в адресах запросов ниже.

## Запуск с Redis

```bash
docker compose up -d --build --wait --wait-timeout 120
docker compose ps
curl --max-time 5 -fsS http://127.0.0.1:8080/
curl --max-time 5 -fsS -X POST http://127.0.0.1:8080/increment
```

В `compose.yaml` два сервиса: `app` и `redis`. Приложению передаются
`STORAGE=redis` и `REDIS_URL=redis://redis:6379/0`. Опубликован только порт приложения.

Volume `redis_data` подключён к `/data` Redis. Запись на диск включена параметрами
`--appendonly yes --appendfsync everysec`.

Healthcheck приложения проверяет `/readyz`, Redis — `redis-cli ping`.
При старте приложение ждёт готовности Redis через `depends_on`.
Настройки DNS ограничивают задержку ответа при недоступности Redis.

Остановка с сохранением данных:

```bash
docker compose down
```

## Один контейнер

```bash
docker build -t docker-hw:local .
docker run -d --name docker-hw-single -p 127.0.0.1:8080:8000 -e "GREETING=Hello from a single container!" docker-hw:local
curl --retry 10 --retry-all-errors --retry-delay 1 --max-time 3 -fsS http://127.0.0.1:8080/
curl --max-time 5 -fsS -X POST http://127.0.0.1:8080/increment
docker logs docker-hw-single
docker rm -f docker-hw-single
```

## Compose с SQLite

```bash
docker compose -p docker-hw-sqlite -f compose.sqlite.yaml up -d --build --wait --wait-timeout 120
curl --max-time 5 -fsS http://127.0.0.1:8080/
docker compose -p docker-hw-sqlite -f compose.sqlite.yaml down
```

В этом варианте используется `STORAGE=sqlite`, а volume `sqlite_data`
подключён к `/data`. Перед сменой варианта запуска нужно остановить предыдущий:
в примерах используется один порт 8080.

## Проверки

Команды и результаты проверок счётчика, сохранения данных и восстановления
после остановки Redis приведены в [REPORT.md](REPORT.md).
