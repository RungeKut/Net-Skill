---
id: 20-08
title: Прозрачный прокси-шлюз на RouterOS: mangle mark-routing + отдельная таблица маршрутов — устройствам прокси неизвестен
tags: [mangle, mark-routing, routing-table, tproxy, прозрачный-прокси]
applies_to: RouterOS 7.24.4
status: проверено
verified: 2026-10-26 — схема в постоянной эксплуатации, разборы и команды из рабочей документации
source: практика — рабочая инфраструктура (документация 2026-10)
---

# Прозрачный прокси-шлюз на RouterOS: mangle mark-routing + таблица маршрутов

## Как устроено

Устройства не знают о существовании прокси. Роутер помечает транзитный трафик из выбранных подсетей, направляет его в отдельную таблицу маршрутов, и шлюзом там стоит адрес прокси-сервера.

Главное условие: правила mangle работают в цепочке `forward`, а не `output`. Трафик самого роутера не заворачивается.

## Команды

### 1. Address-list клиентов

Перечисление подсетей, чей трафик должен уходить через прокси:

```
/ip firewall address-list add list=xray-clients address=192.0.2.0/24 comment="LAN"
/ip firewall address-list add list=xray-clients address=198.51.100.0/24 comment="L2TP pool"
/ip firewall address-list add list=xray-clients address=203.0.113.0/24 comment="WireGuard tunnel"
```

### 2. Mangle-правило

Пометить транзитный трафик клиентов, исключив приватные адреса назначения:

```
/ip firewall mangle add chain=forward src-address-list=xray-clients dst-address-list=!PRIVATE action=mark-routing new-routing-mark=via-xray comment="xray: policy route to Xray VM"
```

Список `PRIVATE` включает: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `169.254.0.0/16`, `100.64.0.0/10`. Если его нет — создать:

```
/ip firewall address-list add list=PRIVATE address=10.0.0.0/8
/ip firewall address-list add list=PRIVATE address=172.16.0.0/12
/ip firewall address-list add list=PRIVATE address=192.168.0.0/16
/ip firewall address-list add list=PRIVATE address=169.254.0.0/16
/ip firewall address-list add list=PRIVATE address=100.64.0.0/10
```

### 3. Таблица маршрутов

```
/ip route add routing-mark=via-xray gateway=192.0.2.10 comment="xray: default route for marked traffic"
```

`192.0.2.10` — адрес прокси-сервера в локальной сети. Не путать со шлюзом провайдера.

### 4. Сопутствующие правила

Если прокси принимает трафик TPROXY, на роутере может понадобиться hairpin NAT, чтобы клиенты видели прокси по его локальному адресу (если прокси и клиенты в одной подсети). Если прокси заворачивает трафик сам (Xray), это не требуется.

Проброс порта, если прокси принимает внешние подключения:

```
/ip firewall nat add chain=dstnat protocol=tcp dst-port=443 action=dst-nat to-addresses=192.0.2.10 to-ports=8443 comment="xray: wan vless"
```

## Выключение схемы

Достаточно отключить одно правило mangle. Трафик сразу пойдёт напрямую через шлюз провайдера:

```
/ip firewall mangle disable [find comment~"xray: policy route"]
```

## Расширение

Чтобы добавить новую подсеть под прокси — достаточно добавить её адрес в `xray-clients`. Mangle-правило применяется ко всему списку.

## Проверка

```
/ip firewall mangle print where comment~"xray"
/ip route print where routing-mark=via-xray
/ip firewall address-list print where list=xray-clients
```

Счётчик на mangle-правиле должен расти при обращении клиента к внешнему ресурсу.

## Чем подтверждено

Схема в постоянной эксплуатации: несколько подсетей (LAN, L2TP, WireGuard) маршрутизируются через прокси-сервер; отключение mangle-правила возвращает прямой трафик без перезагрузки; добавление новой подсети в address-list не требует правки правил.