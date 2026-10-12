# ЛР 1. Loki + Zabbix + Grafana

**Задача:** подключить к тестовому сервису Nextcloud мониторинг и логирование, визуализировать в Grafana.

## Состав
| Файл | Что это |
|---|---|
| `docker-compose.yml` | Nextcloud, Loki, Promtail, Grafana, Zabbix (server + web) и Postgres для Zabbix |
| `promtail_config.yml` | Promtail читает `nextcloud.log` и отправляет в Loki |
| `zabbix_template.yml` | Шаблон Zabbix: HTTP-проверка `status.php` Nextcloud + триггер на maintenance mode |

## Ход работы

### Часть 1. Логирование
1. `docker compose up -d`, проверяем `docker compose ps` — все контейнеры `running`.
   ![](screenshots/01-compose-ps.png)
2. Заходим в Nextcloud (`http://localhost:8080`), создаём учётку. Логи пишутся в `/var/www/html/data/nextcloud.log`.
   ![](screenshots/02-nextcloud.png)
3. `docker logs promtail` — есть строка `msg=Seeked /opt/nc_data/nextcloud.log`, значит Promtail подхватил файл.
   ![](screenshots/03-promtail.png)

### Часть 2. Мониторинг
1. Zabbix: `http://localhost:8082`, логин `Admin` / `zabbix`.
2. *Data collection → Templates → Import* — импортируем `zabbix_template.yml`.
3. Разрешаем имя `nextcloud` внутри docker-сети:
   ```bash
   docker exec -u www-data -it nextcloud php occ config:system:set trusted_domains 1 --value="nextcloud"
   ```
4. *Data collection → Hosts → Create host*: имя `nextcloud`, группа Applications, шаблон *Test ping template*.
5. *Monitoring → Latest data* — значение `healthy`.
   ![](screenshots/04-zabbix-latest.png)
6. Проверка триггера: `php occ maintenance:mode --on` → в *Monitoring → Problems* появилась проблема; `--off` → проблема решена.
   ![](screenshots/05-zabbix-problem.png)

### Часть 3. Визуализация
1. Плагин Zabbix для Grafana:
   ```bash
   docker exec -it grafana bash -c "grafana cli plugins install alexanderzobnin-zabbix-app"
   docker restart grafana
   ```
   *Administration → Plugins → Zabbix → Enable*.
2. Data source Loki: URL `http://loki:3100` → *Save & test*.
3. Data source Zabbix: URL `http://zabbix-front:8080/api_jsonrpc.php`, `Admin` / `zabbix` → *Save & test*.
   ![](screenshots/06-datasources.png)
4. *Explore*: Loki — `{job="nextcloud_logs"}`, Zabbix — группа Applications, хост nextcloud, item ping.

## Задание: дашборды
1. **Zabbix** — панель *Stat*, значение `Nextcloud: ping service`, value mapping `healthy` → зелёный, `unhealthy` → красный.
   ![](screenshots/07-dashboard-zabbix.png)
2. **Loki** — панель *Logs* / *Table*, запрос `{job="nextcloud_logs"}`.
   ![](screenshots/08-dashboard-loki.png)

## Ответы на вопросы

**Чем SLO отличается от SLA?**
SLA (Service Level Agreement) — внешнее соглашение с клиентом о качестве сервиса с последствиями за нарушение (штрафы, компенсации), например «доступность 99,9 % в месяц». SLO (Service Level Objective) — внутренняя целевая планка команды по конкретной метрике (SLI), обычно строже SLA, чтобы был запас: например, SLO 99,95 % при SLA 99,9 %. Нарушение SLO — сигнал команде, нарушение SLA — юридические и денежные последствия.

**Чем инкрементальный бэкап отличается от дифференциального?**
Оба делаются поверх полного бэкапа. Инкрементальный сохраняет изменения с момента *предыдущего бэкапа любого типа* — каждый маленький и быстрый, но для восстановления нужен полный + вся цепочка инкрементов. Дифференциальный сохраняет изменения с момента *последнего полного* — каждый следующий больше предыдущего, зато для восстановления нужны только полный + последний дифференциальный.

**В чём разница между мониторингом и observability?**
Мониторинг отвечает на заранее известные вопросы: собираем выбранные метрики и проверки, ставим пороги и алерты («сервис жив?», «CPU > 90 %?»). Observability — свойство системы, позволяющее по её внешним данным (метрики, логи, трейсы) понять внутреннее состояние и ответить на *новые*, заранее не предусмотренные вопросы («почему у части пользователей медленно?»). Мониторинг — часть observability. В этой лабе Zabbix даёт мониторинг, а связка с логами в Loki — шаг к observability.
