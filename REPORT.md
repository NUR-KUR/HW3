# Отчёт о проверках

Дата: 11 ноября 2026 года. Ubuntu 26.04, WSL 2.
Docker Engine 29.9.3, Docker Compose 5.7.1.

## Исходники

Файлы в `app/` совпадают с исходным шаблоном. Команда
`git hash-object app/app.py app/requirements.txt` вернула:

```text
85a3e5807ddf81705fdbbfbe4b0d9a896a7a02a6
0cecdf573aaebc5f35cfe3a524815370f101c8f0
```

## Уровень 4

```bash
docker build -t docker-hw:local .
docker run -d --name docker-hw-single -p 127.0.0.1:8080:8000 -e "GREETING=Hello from a single container!" docker-hw:local
curl --max-time 5 -fsS http://127.0.0.1:8080/
curl --max-time 5 -fsS -X POST http://127.0.0.1:8080/increment
docker logs docker-hw-single
docker rm -f docker-hw-single
```

Образ собрался. GET вернул `count: 0`, POST — `count: 1`, повторный GET — 1.
В ответах было переданное приветствие `Hello from a single container!`.
В логах были `listening port=8000 storage=sqlite` и запрос `POST /increment` со статусом 200.

## Уровень 6

```bash
docker compose -p docker-hw-sqlite -f compose.sqlite.yaml up -d --build --wait --wait-timeout 120
curl --max-time 5 -fsS -X POST http://127.0.0.1:8080/increment
curl --max-time 5 -fsS -X POST http://127.0.0.1:8080/increment
docker compose -p docker-hw-sqlite -f compose.sqlite.yaml down
docker compose -p docker-hw-sqlite -f compose.sqlite.yaml up -d --wait --wait-timeout 120
curl --max-time 5 -fsS http://127.0.0.1:8080/
docker compose -p docker-hw-sqlite -f compose.sqlite.yaml down
```

После двух POST счётчик равнялся 2. После удаления и создания контейнера GET
вернул 2, хотя ID контейнера изменился. Данные сохранились в `sqlite_data`.
Приветствие из `.env` — `Hello from Docker!` — присутствовало в ответах.
Отдельно через `docker compose config --format json` проверена подстановка
других значений порта и приветствия в обоих файлах Compose.

## Уровень 8: сохранение данных

```bash
docker compose up -d --build --wait --wait-timeout 120
curl --max-time 5 -fsS -X POST http://127.0.0.1:8080/increment
curl --max-time 5 -fsS -X POST http://127.0.0.1:8080/increment
curl --max-time 5 -fsS -X POST http://127.0.0.1:8080/increment
curl --max-time 5 -fsS http://127.0.0.1:8080/
docker compose down
docker compose up -d --wait --wait-timeout 120
curl --max-time 5 -fsS http://127.0.0.1:8080/
```

На новом Redis счётчик был 0. После трёх POST получено **N = 3**.
После `down` без `-v` и повторного запуска GET вернул **3**.
Контейнер приложения был создан заново. В ответах указано `storage: redis`.

## Уровень 8: остановка и восстановление Redis

```bash
docker compose stop redis
curl --max-time 5 -sS -i http://127.0.0.1:8080/healthz
curl --max-time 5 -sS -i http://127.0.0.1:8080/readyz
curl --max-time 5 -sS -i -X POST http://127.0.0.1:8080/increment
docker compose start redis
curl --retry 20 --retry-all-errors --retry-delay 1 --max-time 3 -fsS http://127.0.0.1:8080/readyz
curl --max-time 5 -fsS http://127.0.0.1:8080/
curl --max-time 5 -fsS -X POST http://127.0.0.1:8080/increment
docker compose ps
```

| Проверка | Фактический результат |
| --- | --- |
| Redis остановлен: `GET /healthz` | 200, `{"status": "alive"}` |
| Redis остановлен: `GET /readyz` | 503, `{"error": "storage_unavailable"}` |
| Redis остановлен: `POST /increment` | 503, `{"error": "storage_unavailable"}` |
| Redis снова запущен: `GET /` | 200, `count: 3` |
| Следующий `POST /increment` | 200, `count: 4` |
| ID и `State.StartedAt` приложения до и после остановки Redis | Совпадают, приложение не перезапускалось |
| Healthcheck после первоначального запуска | `app`: healthy, `redis`: healthy |

Дополнительно через `docker inspect` подтверждены именованный volume Redis на
`/data` и отсутствие опубликованного порта Redis. Команда
`docker compose exec -T redis redis-cli CONFIG GET appendonly` вернула `appendonly`, `yes`.
Запрос из Windows через `curl.exe` к `http://127.0.0.1:8080/` также прошёл успешно.

Все проверки пройдены. После проверки выполнен `docker compose down` без `-v`.
Данные сохранены: при следующем запуске основной проект продолжит со значения 4.
