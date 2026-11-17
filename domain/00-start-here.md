# 00. Start here

## Навигация
Доменные файлы — `domain/`, решения — `adr/`, спеки — `openspec/specs/<service>/<capability>/spec.md`.

## Главные запреты
Не добавлять БД, авторизацию и ПДн без отдельного решения (ADR).

## Инварианты проекта
Заказ однозначно идентифицируется `id`. Существующие эндпоинты не ломаем.

## Реестр сервисов и коды сервисов
| Код | Сервис |
|---|---|
| `orders` | orders-service |

## Правила имён: capability, change, AC
capability — kebab-case (`order-crud`); change — `<service>-<суть>`; AC — `AC-<n>`.

## Правила архивации и группировки change
Один change — одна задача; после merge PR с кодом — `openspec archive`.
