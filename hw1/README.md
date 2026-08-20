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
