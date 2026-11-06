---
id: 20-14
title: IPv6-правила изоляции гостей ставятся вручную выше accept ICMPv6 — без place-before они уходят в конец цепочки и не работают
tags: [ipv6, firewall, гостевая-сеть, place-before, изоляция]
applies_to: RouterOS 7.24.4, /interface wireless
status: проверено
verified: 2026-09-28 — два правила без place-before встали на позиции 24-25 ниже accept на 17; после пересоздания с place-before встали на 3-4 и изоляция заработала
source: практика — живой роутер (методичка 2026-09)
---

# IPv6-правила изоляции гостей ставятся вручную выше accept ICMPv6

## Что происходит

Даже если глобального IPv6 в сети нет, link-local адреса (fe80::) существуют всегда. Гостевые устройства могут достучаться до роутера по IPv6 link-local. Правила фильтрации IPv6 нужны — они дёшевы и надёжны.

Проблема: если добавить IPv6-правила без `place-before`, они встают **в конец** цепочки, после `defconf: accept ICMPv6`, и не срабатывают — разрешающее правило принимает пакет раньше.

## Как правильно

Каждое IPv6-правило — с `place-before`, указывающим на якорь до `defconf: accept ICMPv6`.

### Forward (изоляция транзита)

```
/ipv6 firewall filter add chain=forward in-interface=guest-bridge action=drop comment="GUEST: no IPv6 forwarding" place-before=[find chain=forward and comment="defconf: drop invalid"]

/ipv6 firewall filter add chain=forward out-interface=guest-bridge action=drop comment="GUEST: no IPv6 forwarding" place-before=[find chain=forward and comment="defconf: drop invalid"]
```

Якорь — `defconf: drop invalid`, потому что он стоит в цепочке forward **после** `defconf: accept established,related` и **до** всего остального.

### Input (закрыть роутер)

```
/ipv6 firewall filter add chain=input in-interface=guest-bridge action=drop comment="GUEST: no access to router" place-before=[find chain=input and comment="defconf: accept ICMPv6"]
```

Якорь — `defconf: accept ICMPv6` в цепочке input.

## Как неправильно

Добавить правила без `place-before`:

```
/ipv6 firewall filter add chain=forward in-interface=guest action=drop
/ipv6 firewall filter add chain=forward out-interface=guest action=drop
```

Они встанут после `defconf: accept ICMPv6` и всех остальных defconf-правил — то есть будут бесполезны.

## Проверка позиции

```
/ipv6 firewall filter print where chain=forward
```

Правила изоляции должны быть среди первых (до `accept ICMPv6`), а не в конце списка.

## Чем подтверждено

При первой попытке два IPv6-правила forward без `place-before` встали на позиции 24-25 — ниже `accept ICMPv6` на позиции 17. После удаления и пересоздания с `place-before` встали на позиции 3-4, счётчики начали расти, изоляция заработала.

Источник: раздел IPv6 методички по гостевой сети (2026-09).