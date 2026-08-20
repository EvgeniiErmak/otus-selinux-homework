# Задание 1. nginx на нестандартном порту

Описание трёх способов будет добавлено по мере выполнения.

## Способ 1: setsebool (порт 8091)

По умолчанию `httpd_t` не может биндиться на порты с типом `unreserved_port_t`
(денай `name_bind`). Стандартный `httpd_can_network_connect` тут не подходит —
он относится к исходящим соединениям, а не к прослушиванию порта.

Для `name_bind` на `unreserved_port_t` в targeted-политике `audit2why`
предлагает нестандартный, но рабочий булев переключатель `nis_enabled`
(историческая особенность политики — по названию про NIS, по факту разрешает
httpd_t биндить непривилегированные порты).

### Воспроизведение

1. Конфиг nginx слушает порт 8091 (`/etc/nginx/conf.d/hw1-setsebool.conf`)
2. `systemctl restart nginx` — падает: `bind() to 0.0.0.0:8091 failed (13: Permission denied)`
3. `ausearch -m avc -ts recent` показывает:
   `avc: denied { name_bind } ... tcontext=system_u:object_r:unreserved_port_t:s0`
4. `audit2why` рекомендует: `setsebool -P nis_enabled 1`
5. Применяем: `setsebool -P nis_enabled on`
6. `systemctl restart nginx` — успешный старт
7. `curl http://localhost:8091` → `hw1: setsebool method works on 8091`

## Способ 2: semanage port (порт 8092)

Регистрируем порт 8092 напрямую в существующем типе `http_port_t`
(том же, что уже покрывает 80, 443, 8080 и т.д.) — это самый "штатный"
способ для одиночного сервиса, без побочных эффектов на другие домены.

### Воспроизведение

1. Конфиг nginx слушает порт 8092 (`/etc/nginx/conf.d/hw1-semanage.conf`)
2. Денай: `avc: denied { name_bind } ... src=8092 ... tcontext=unreserved_port_t`
3. `semanage port -a -t http_port_t -p tcp 8092`
4. `semanage port -l | grep http_port_t` → `http_port_t tcp 8092, 80, 81, 443, ...`
5. `systemctl restart nginx` — успешный старт
6. `curl http://localhost:8092` → `hw1: semanage method works on 8092`

### Важное наблюдение

nginx — единый systemd-сервис с одним мастер-процессом: все `listen`-директивы
во всех `server{}`-блоках применяются при одном старте. Если SELinux блокирует
бинд хотя бы одного порта — падает **весь** nginx, а не только проблемный блок.
Это подтвердилось экспериментально: временное отключение `nis_enabled` уронило
не только 8091, но и рестарт всего сервиса целиком, пока оба порта не были
корректно размечены.
