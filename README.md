# PostgreSQL и Postgres Pro для пространственных данных

Два образа для работы с геометрией, картами и маршрутизацией. Оба рассчитаны на `linux/amd64` и при первом запуске включают PostGIS и pgRouting в базе приложения.

| Сервис Compose | Dockerfile | Компоненты на 18 ноября 2026 г. |
| --- | --- | --- |
| `postgres` | `Dockerfile.postgres` | Официальный образ PostgreSQL 18.7 на Debian 13 (Trixie), PostGIS 3.7.0 и pgRouting 4.0.2 из подписанного репозитория PGDG |
| `postgrespro` | `Dockerfile.postgrespro` | Debian 13.7, актуальные пакеты Postgres Pro Standard ветки 18 (опубликован 18.6.2), PostGIS 3.7.0 и pgRouting 4.0.2 |

В сборке Postgres Pro расширения компилируются с её `pg_config` из релизных архивов с проверкой SHA-256. Профиль сборки включает GeoJSON, MVT и опциональное расширение `postgis_topology`. Растр, SFCGAL и серверный пакет LLVM JIT в сборку Postgres Pro не входят. Скрипт инициализации взят из официального образа PostgreSQL. [PostgreSQL](https://hub.docker.com/_/postgres) · [PGDG](https://apt.postgresql.org/) · [Postgres Pro Standard](https://postgrespro.ru/products/download/postgrespro/latest) · [PostGIS](https://postgis.net/development/source_code/) · [pgRouting](https://github.com/pgRouting/pgrouting/releases/tag/v4.0.2)

## Сборка

Понадобятся Docker с Buildx и Docker Compose v2. Все команды выполняются из каталога с `compose.yaml`.

### Обычный PostgreSQL

```bash
docker compose build postgres
```

### Postgres Pro Standard

Для доступа к подписанному APT-репозиторию Postgres Pro Standard нужен ключ. Администратор набора лицензий может получить его на портале поддержки в разделе «Администрирование → Набор лицензий → Репозитории». [Доступ к репозиторию](https://postgrespro.ru/blog/technical/5971798) · [Инструкция вендора](https://postgrespro.ru/docs/postgrespro/18/binary-installation-on-linux)

```bash
read -rsp 'Ключ репозитория Postgres Pro: ' PGPRO_ACCESS_KEY
printf '\n'
export PGPRO_ACCESS_KEY

docker compose build --no-cache postgrespro

unset PGPRO_ACCESS_KEY
```

Compose передаёт ключ как BuildKit Secret. Временный файл авторизации APT находится в `tmpfs`, ключ не сохраняется в слоях образа. `--no-cache` обновляет индекс репозитория и запрашивает последние пакеты ветки 18. Фактическая версия сервера выводится в журнал сборки. Если runner упирается в память при компиляции, добавьте `--build-arg BUILD_JOBS=1`. [Docker Build secrets](https://docs.docker.com/build/building/secrets/)

При обновлении проверьте digest базовых образов. Для PostgreSQL обновите версии пакетов PGDG, а для Postgres Pro — URL и SHA-256 релизных архивов. Если изменились версии расширений, обновите теги образов в `compose.yaml`.

## Запуск и проверка

У каждого сервиса свои том, пароль и порт на хосте. Оба порта привязаны к loopback.

| Сервис | Адрес на хосте | Том | Переменная с паролем |
| --- | --- | --- | --- |
| `postgres` | `127.0.0.1:5432` | `postgres-data` | `POSTGRES_PASSWORD` |
| `postgrespro` | `127.0.0.1:5433` | `postgrespro-data` | `POSTGRESPRO_PASSWORD` |

Сначала задайте пароль и запустите обычный PostgreSQL:

```bash
read -rsp 'Пароль postgres: ' POSTGRES_PASSWORD
printf '\n'
export POSTGRES_PASSWORD

docker compose up -d --no-build --wait postgres
```

Для запуска Postgres Pro после его сборки используйте отдельный пароль:

```bash
read -rsp 'Пароль Postgres Pro: ' POSTGRESPRO_PASSWORD
printf '\n'
export POSTGRESPRO_PASSWORD

docker compose up -d --no-build --wait postgrespro
```

После сборки обоих образов и ввода обоих паролей команда `docker compose up -d --no-build --wait` запускает два сервиса. Compose монтирует пароли как файлы секретов. При первом запуске на пустом томе создаётся база `gis` с суперпользователем `postgres` и двумя расширениями. По умолчанию используется SCRAM для TCP и Unix-сокета. При повторном запуске с тем же томом база и её пароль сохраняются. Для приложения создайте отдельную роль с нужными правами. Каталог данных внутри тома — `/var/lib/postgresql/18/docker`.

Проверка выводит версии, преобразует координаты и ищет кратчайший маршрут на небольшом графе. Ошибка любого из запросов прерывает проверку.

```bash
docker compose exec -T -e "PGPASSWORD=$POSTGRES_PASSWORD" postgres \
  psql -h 127.0.0.1 -U postgres -d gis -v ON_ERROR_STOP=1 <<'SQL'
SELECT version(), postgis_full_version(), pgr_version();

DO $$
DECLARE
    route_cost double precision;
BEGIN
    IF ST_AsText(ST_Transform(ST_SetSRID(ST_MakePoint(0, 0), 4326), 3857))
       IS DISTINCT FROM 'POINT(0 0)' THEN
        RAISE EXCEPTION 'ST_Transform returned an unexpected result';
    END IF;

    SELECT agg_cost INTO route_cost
    FROM pgr_dijkstra(
        'SELECT * FROM (VALUES
            (1, 1, 2, 1.0::float8),
            (2, 2, 3, 2.0::float8),
            (3, 1, 3, 7.0::float8)
        ) AS edges(id, source, target, cost)',
        1::bigint, 3::bigint, true
    )
    WHERE node = 3;

    IF route_cost IS DISTINCT FROM 3.0::float8 THEN
        RAISE EXCEPTION 'Expected route cost 3, got %', route_cost;
    END IF;
END
$$;
SQL
```

Для Postgres Pro замените сервис на `postgrespro`, а переменную пароля на `POSTGRESPRO_PASSWORD`.

В других базах включите расширения командой `CREATE EXTENSION IF NOT EXISTS postgis;`, затем `CREATE EXTENSION IF NOT EXISTS pgrouting;`. Для GIS-топологии дополнительно выполните `CREATE EXTENSION IF NOT EXISTS postgis_topology;`. При обновлении образа сделайте резервную копию и проверьте обновление SQL-расширений в каждой базе на тестовой копии. Для PostGIS используйте `SELECT postgis_extensions_upgrade();`, для pgRouting — `ALTER EXTENSION pgrouting UPDATE;`. [PostGIS](https://postgis.net/docs/manual-3.7/postgis_extensions_upgrade.html) · [pgRouting](https://github.com/pgRouting/pgrouting/releases/tag/v4.0.2)

## Настройка для пространственных запросов

Compose задаёт каждому экземпляру 4 ГиБ RAM, 2 vCPU и 512 МиБ `/dev/shm`. Для одновременного запуска двух экземпляров выделите хосту больше 8 ГиБ RAM с учётом остальных процессов. При изменении лимитов подстройте параметры `command` для реальной нагрузки.

| Что настраивать | С чего начать |
| --- | --- |
| Кэш и память | `shared_buffers=1GB` — четверть лимита из примера. `effective_cache_size=3GB` — оценка доступного кэша для планировщика. Держите глобальный `work_mem` умеренным. Он применяется к каждой сортировке или хеш-операции, в том числе у параллельных воркеров. Для проверенного тяжёлого запроса можно задать `SET LOCAL work_mem = '64MB'` внутри транзакции |
| Индексы и импорт | Создавайте GiST на геометрии. Для первой массовой загрузки через `COPY` создавайте индекс после импорта, затем выполняйте `ANALYZE`. `maintenance_work_mem` помогает при создании индекса, `max_wal_size` может снизить частоту контрольных точек при больших загрузках |
| Статистика и autovacuum | После крупных обновлений выполните `ANALYZE`. Для больших часто изменяемых таблиц настройте параметры на уровне таблицы, например `autovacuum_analyze_scale_factor=0.02` и `autovacuum_vacuum_scale_factor=0.05`, если статистика устаревает или накапливаются мёртвые строки |
| Запросы по расстоянию | Для локальных расчётов используйте `geometry` в подходящей метрической проекции и `ST_DWithin`. Для глобальных расстояний подходит `geography`. Храните рабочий SRID в столбце либо создайте функциональный индекс под регулярно используемое преобразование |
| Большие полигоны | Если пространственный индекс возвращает много кандидатов и точные проверки дороги, храните индексируемые части `ST_Subdivide` вместе с идентификатором исходного объекта. Проверьте необходимость устранять дубли при объединении результатов |
| Маршрутизация | Передавайте pgRouting компактный набор рёбер `id, source, target, cost, reverse_cost`. Измеряйте время выборки рёбер и время построения маршрута. При ограничении выборки по области проверьте, что за её границей не остались нужные пути. При нагрузочных тестах учитывайте память на граф при нескольких одновременных маршрутах |

Например, для таблицы линий `roads` с геометрией в EPSG:32637 (зона подходит для Москвы) запрос в пределах 200 метров может использовать GiST:

```sql
CREATE INDEX roads_geom_gix ON roads USING gist (geom);
ANALYZE roads;

EXPLAIN (ANALYZE, BUFFERS)
SELECT id
FROM roads
WHERE ST_DWithin(
  geom,
  ST_Transform(ST_SetSRID(ST_MakePoint(37.62, 55.75), 4326), 32637),
  200
);
```

Используйте `EXPLAIN (ANALYZE, BUFFERS)` на типичных запросах и проверяйте расход памяти контейнера под ожидаемой конкурентной нагрузкой. [Параметры памяти](https://www.postgresql.org/docs/18/runtime-config-resource.html) · [Настройки планировщика](https://www.postgresql.org/docs/18/runtime-config-query.html) · [Autovacuum](https://www.postgresql.org/docs/18/runtime-config-vacuum.html) · [ST_DWithin](https://postgis.net/docs/manual-3.7/ST_DWithin.html) · [ST_Subdivide](https://postgis.net/docs/manual-3.7/ST_Subdivide.html) · [pgRouting](https://docs.pgrouting.org/latest/en/pgr_dijkstra.html)