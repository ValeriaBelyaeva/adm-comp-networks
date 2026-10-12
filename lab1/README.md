# ЛР 1. HA Postgres Cluster (файл задания lab03.pdf)

**Задача:** развернуть высокодоступный кластер Postgres: Patroni + Zookeeper + HAProxy.

## Состав
| Файл | Что это |
|---|---|
| `Dockerfile` | Образ Postgres 15 + Patroni |
| `docker-compose.yml` | Две ноды (`pg-master`, `pg-slave`), Zookeeper, HAProxy |
| `postgres0.yml`, `postgres1.yml` | Конфиги Patroni для двух нод |
| `haproxy.cfg` | Точка входа: трафик идёт на текущего лидера |

## Ход работы

### Часть 1. Поднимаем Postgres
```bash
docker compose up -d --build
docker compose logs -f zoo pg-master pg-slave
```
Zookeeper запустился, одна из нод стала лидером (не обязательно `pg-master`).
![](screenshots/01-leader.png)

### Часть 2. Репликация
1. Подключаемся к обеим нодам: `localhost:5433` и `localhost:5434`, `postgres` / `postgres`.
2. На лидере создаём таблицу и данные:
   ```sql
   CREATE TABLE cars (id serial PRIMARY KEY, plate text, brand text);
   INSERT INTO cars (plate, brand) VALUES ('А001АА', 'Hyundai'), ('В002ВВ', 'Kia');
   ```
3. На реплике та же таблица и данные появились автоматически.
   ![](screenshots/02-replication.png)
4. Запись на реплике — ошибка `cannot execute INSERT in a read-only transaction`.
   ![](screenshots/03-readonly.png)

### Часть 3. HAProxy
Подключаемся через `localhost:5432` (контейнер `postgres_entrypoint`) — попадаем на лидера. Статистика: `http://localhost:7000`.
![](screenshots/04-haproxy.png)

## Задание
> Выбрать один вариант и описать здесь.

**Вариант 1 — отказ мастера.** `docker stop <лидер>` → ждём ~30 с (ttl) → в логах вторая нода становится лидером, HAProxy переключается. Проверяем чтение и запись через `localhost:5432`. Описать, что произошло, со скриншотами.

## Ответы на вопросы

**Чем `expose` отличается от `ports`?**
`expose` открывает порт только внутри docker-сети — к нему могут обращаться другие контейнеры (HAProxy ходит на `:8008` за health-check), но с хоста он недоступен. `ports` публикует порт на хост (`5433:5432`), и к нему можно подключиться снаружи. Порт из `ports` автоматически становится и exposed.

**Пересоберётся ли образ при перезапуске проекта?**
- Обычный `docker compose up` / `restart` — **нет**, используется уже собранный образ `localhost/postgres:patroni`.
- Изменили `postgres0.yml` / `postgres1.yml` — **тоже нет**: файлы копируются в образ на этапе сборки (`COPY`), а compose сам не следит за изменениями контекста. Нужен `docker compose up --build` (или `build`).
- Изменили `Dockerfile` — **тоже нет** без `--build`. С `--build` Docker пересоберёт образ и использует кэш слоёв: слои до первой изменённой инструкции возьмутся из кэша, остальные соберутся заново.
