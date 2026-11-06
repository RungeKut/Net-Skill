---
id: 20-09
title: TPROXY на Linux перехватывает транзит: iptables mangle PREROUTING + ip rule fwmark + локальный маршрут через lo
tags: [tproxy, linux, iptables, xray, прозрачный-прокси, systemd]
applies_to: Ubuntu 24.04+, Xray (TPROXY inbound :12345)
status: проверено
verified: 2026-10-26 — схема в постоянной эксплуатации, разборы и команды из рабочей документации
source: практика — рабочая инфраструктура (документация 2026-10)
---

# TPROXY на Linux перехватывает транзит

## Как устроено

Три компонента, каждый нужен:

1. **ip rule** — пакеты с меткой fwmark 1 направляет в таблицу 100.
2. **ip route** — в таблице 100 весь трафик считается локальным (dev lo). Без этого TPROXY молча не работает: ядро не знает, куда доставить пакет с нелокальным адресом назначения.
3. **iptables mangle PREROUTING** — цепочка XRAY: приватные диапазоны пропускает (RETURN), остальное перенаправляет на TPROXY-порт 12345.

## Скрипт

```bash
#!/bin/bash
# TPROXY-перехват транзитного трафика для Xray (порт 12345)
case "$1" in
start)
  ip rule add fwmark 1 table 100 2>/dev/null
  ip route add local 0.0.0.0/0 dev lo table 100 2>/dev/null
  iptables -t mangle -N XRAY 2>/dev/null
  iptables -t mangle -F XRAY
  for net in 0.0.0.0/8 10.0.0.0/8 100.64.0.0/10 127.0.0.0/8 169.254.0.0/16 172.16.0.0/12 192.168.0.0/16 224.0.0.0/4 240.0.0.0/4; do
    iptables -t mangle -A XRAY -d "$net" -j RETURN
  done
  iptables -t mangle -A XRAY -p tcp -j TPROXY --on-ip 127.0.0.1 --on-port 12345 --tproxy-mark 1
  iptables -t mangle -A XRAY -p udp -j TPROXY --on-ip 127.0.0.1 --on-port 12345 --tproxy-mark 1
  iptables -t mangle -C PREROUTING -j XRAY 2>/dev/null || iptables -t mangle -A PREROUTING -j XRAY
  ;;
stop)
  iptables -t mangle -D PREROUTING -j XRAY 2>/dev/null
  iptables -t mangle -F XRAY 2>/dev/null
  iptables -t mangle -X XRAY 2>/dev/null
  ip rule del fwmark 1 table 100 2>/dev/null
  ip route del local 0.0.0.0/0 dev lo table 100 2>/dev/null
  ;;
*)
  echo "usage: $0 start|stop"; exit 1
  ;;
esac
```

## systemd-сервис (oneshot, применяется при загрузке)

```
[Unit]
Description=Xray TPROXY transparent gateway rules
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/root/xray-tproxy.sh start
ExecStop=/root/xray-tproxy.sh stop
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Сервис включается в автозагрузку: `systemctl enable xray-tproxy.service`.

## Что важно

- `ip rule` и `ip route` работают **на уровне ядра**, до фаервола. Без них пакеты, попавшие в TPROXY, не будут доставлены приложению.
- `RemainAfterExit=yes` — systemd считает сервис запущенным даже после завершения скрипта. Без этого `systemctl enable` не сработает как надо.
- Порядок `for net` включает `0.0.0.0/8` — это не опечатка: диапазон `0.0.0.0/8` существует и используется для `THIS NETWORK` (RFC 1122).

## Проверка

```bash
ip rule show | grep fwmark
ip route show table 100
iptables -t mangle -L XRAY -n -v
```

Счётчики на цепочке XRAY должны расти при транзитном трафике.

## Чем подтверждено

Схема в постоянной эксплуатации: Xray с TPROXY inbound на порту 12345 обрабатывает транзитный трафик с роутера MikroTik. Скрипт применяется при каждой загрузке, сервис включён в multi-user.target.