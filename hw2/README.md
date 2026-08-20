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
/etc/named/dynamic/ (вместо штатного /var/named/dynamic/).
Директория /etc/named целиком размечена SELinux-политикой как
named_conf_t — тип только для чтения конфигурации, без права write
для домена процесса named_t.

При попытке nsupdate внести изменение в зону, named успешно проходит
TSIG-авторизацию (ключ верный), но при попытке физически записать
обновление на диск получает SELinux-денай — что BIND возвращает
клиенту как SERVFAIL, маскируя истинную причину под "проблему с зоной".

### Подтверждение

    $ ls -Zd /etc/named/dynamic
    unconfined_u:object_r:named_conf_t:s0 /etc/named/dynamic

    $ sesearch -A -s named_t -t named_conf_t -c file -p write
    (пусто — правила нет)

    $ sesearch -A -s named_t -t named_cache_t -c file -p write
    allow named_t named_cache_t:file { append create ... write };

Живой AVC-денай, пойманный в момент запроса nsupdate:

    type=AVC msg=audit(...): avc: denied { write } for pid=8392
    comm="isc-net-0000" name="dynamic" dev="vda4" ino=34222264
    scontext=system_u:system_r:named_t:s0
    tcontext=unconfined_u:object_r:named_conf_t:s0
    tclass=dir permissive=0

## Варианты решения

1. semanage fcontext + restorecon — назначить /etc/named/dynamic(/.*)?
   тип named_cache_t постоянно, переживает restorecon/relabel системы.
2. chcon — то же самое, но временно; слетает при следующем полном
   relabel.
3. Кастомный модуль audit2allow — избыточно: правило allow named_t
   к named_cache_t уже существует в штатной политике, проблема не в
   отсутствии правила, а в неверной разметке пути.
4. Перенос зоны в стандартный /var/named/dynamic — устранило бы
   проблему, но требует правки named.conf и меняет архитектуру стенда,
   а не решает исходную SELinux-задачу как таковую.

## Выбор и обоснование

Выбран способ 1 (semanage fcontext) — это штатный, персистентный
механизм SELinux именно для случая "легитимный, но нестандартный путь
файла нужно ассоциировать с правильным типом", без изменения политики
или архитектуры сервиса.

## Реализация

    semanage fcontext -a -t named_cache_t "/etc/named/dynamic(/.*)?"
    restorecon -Rv /etc/named/dynamic

Результат:

    Relabeled /etc/named/dynamic from ...named_conf_t:s0 to ...named_cache_t:s0
    Relabeled /etc/named/dynamic/named.ddns.lab from ...named_conf_t:s0 to ...named_cache_t:s0
    Relabeled /etc/named/dynamic/named.ddns.lab.view1 from ...named_conf_t:s0 to ...named_cache_t:s0

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
