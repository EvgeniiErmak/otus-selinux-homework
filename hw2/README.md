# Задание 2. Проблема обновления DNS-зоны при включённом SELinux

## Стенд

Развёрнут через Vagrant + libvirt (провайдер virtualbox в исходном
Vagrantfile заменён на libvirt через `vagrant up --provider=libvirt`,
так как VirtualBox 7.0/7.1 несовместим с ядром AlmaLinux 9.8 —
известный баг from_timer в модуле vboxdrv, см. историю ДЗ1).

- ns01 (192.168.50.10) — DNS-сервер (BIND)
- client (192.168.50.15) — клиент, выполняет nsupdate

## Причина неработоспособности

Зона ddns.lab в named.conf объявлена как динамическая:

    zone "ddns.lab" {
        type master;
        allow-update { key "zonetransfer.key"; };
        file "/etc/named/dynamic/named.ddns.lab";
    };

Файлы зоны копируются Ansible-плейбуком в нестандартный путь
/etc/named/ (вместо штатного /var/named/). Директория /etc/named
целиком размечена SELinux-политикой как named_conf_t — тип только
для чтения конфигурации, без права write для домена процесса named_t.

При попытке nsupdate внести изменение в зону, named успешно проходит
TSIG-авторизацию (ключ верный), но при попытке физически записать
обновление на диск получает SELinux-денай — что BIND возвращает
клиенту как SERVFAIL, маскируя истинную причину под "проблему с зоной".

### Подтверждение

    $ ls -laZ /etc/named
    drw-rwx---. root named system_u:object_r:named_conf_t:s0 .
    drw-rwx---. root named unconfined_u:object_r:named_conf_t:s0 dynamic
    -rw-rw----. root named system_u:object_r:named_conf_t:s0 named.dns.lab
    ...

Для сравнения — контекст штатной зоны localhost:

    $ ls -alZ /var/named/named.localhost
    system_u:object_r:named_zone_t:s0 /var/named/named.localhost

Живой AVC-денай, пойманный в момент запроса nsupdate:

    type=AVC msg=audit(...): avc: denied { write } for pid=8392
    comm="isc-net-0000" name="dynamic" dev="vda4" ino=34222264
    scontext=system_u:system_r:named_t:s0
    tcontext=unconfined_u:object_r:named_conf_t:s0
    tclass=dir permissive=0

## Варианты решения

1. semanage fcontext -a -t named_zone_t "/etc/named(/.*)?" + restorecon —
   назначить всей директории /etc/named правильный, штатный для зон
   BIND тип named_zone_t (тот же, что уже используется для /var/named
   по умолчанию в политике). Персистентно, переживает restorecon и
   relabel системы.
2. chcon -R -t named_zone_t /etc/named — то же самое, но временно;
   без предварительного semanage fcontext контекст слетит обратно при
   следующем restorecon (это наглядно проверено: после отката через
   restorecon без предварительного semanage правило возвращается на
   исходный named_conf_t).
3. Кастомный модуль audit2allow — избыточно: allow-правило
   named_t → named_zone_t уже существует в штатной политике,
   проблема не в отсутствии правила, а в неверной разметке пути.
4. Перенос зоны в стандартный /var/named — устранило бы проблему, но
   требует правки named.conf и меняет архитектуру стенда, а не решает
   исходную SELinux-задачу как таковую.

## Выбор и обоснование

Выбран способ 1 (semanage fcontext + restorecon) — персистентный
штатный механизм SELinux для случая, когда легитимный, но нестандартный
путь файла нужно ассоциировать с правильным типом, без изменения самой
политики или архитектуры сервиса. Тип named_zone_t выбран по аналогии
с уже существующей зоной /var/named/named.localhost, размеченной этим
же типом в штатной политике — то есть используется существующий,
предназначенный именно для DNS-зон тип, а не создаётся новый.

## Реализация

    semanage fcontext -a -t named_zone_t "/etc/named(/.*)?"
    restorecon -Rv /etc/named

Результат — вся директория /etc/named и все вложенные файлы/подкаталоги
(включая dynamic) переразмечены в named_zone_t:

    Relabeled /etc/named from ...named_conf_t:s0 to ...named_zone_t:s0
    Relabeled /etc/named/named.dns.lab from ...named_conf_t:s0 to ...named_zone_t:s0
    Relabeled /etc/named/dynamic from ...named_conf_t:s0 to ...named_zone_t:s0
    Relabeled /etc/named/dynamic/named.ddns.lab from ...named_conf_t:s0 to ...named_zone_t:s0
    ...

## Демонстрация работоспособности

    [vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key << 'END'
    server 192.168.50.10
    zone ddns.lab
    update add www.ddns.lab. 60 A 192.168.50.15
    send
    END
    [vagrant@client ~]$

(без ошибок — nsupdate завершается молча при успехе)

    [vagrant@client ~]$ dig @192.168.50.10 www.ddns.lab A
    ;; ANSWER SECTION:
    www.ddns.lab.           60      IN      A       192.168.50.15

Запись успешно добавлена в зону и раздаётся сервером.
