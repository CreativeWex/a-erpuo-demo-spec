# Design

## Context

Репозиторий сервиса `a-erpuo-demo-service` пуст (только `.gitignore`), сервис создаётся с нуля. Ограничения, которые определяют подход:

- Стек: Java 21, Spring Boot 3, Maven (`./mvnw verify`), стиль — Google Java Style, пакеты `com.example.orders.*`, REST-пути во множественном числе (`CLAUDE.md` репозитория сервиса).
- Архитектура по `domain/03-architecture.md`: controller → service → in-memory repository, без внешних интеграций.
- Хранение в памяти — ADR-0001; БД, авторизация и ПДн запрещены без отдельного ADR (`domain/00-start-here.md`).
- Тесты: JUnit 5 + Spring MVC Test, каждый `AC-n` покрыт тестом (`domain/05-testing-and-quality.md`).

Мотивация и объём — в `proposal.md`, требования — в `specs/orders/order-crud/spec.md`.

## Goals / Non-Goals

**Goals:**
- Реализовать 23 сценария `AC-1…AC-23` минимальным кодом в три слоя.
- Сделать хранилище потокобезопасным, не вводя внешних зависимостей.

**Non-Goals:**
- Всё из Non-goals proposal; дополнительно — абстракция хранилища «на вырост» (интерфейс репозитория остаётся, но других реализаций не делаем).

## Decisions

**Затронутые модули и пакеты (сервис `orders`, репозиторий `a-erpuo-demo-service`):**

| Пакет `com.example.orders` | Содержимое |
|---|---|
| `.` | `OrdersApplication` — точка входа Spring Boot |
| `.api` | `OrderController` (пять эндпоинтов), DTO запроса и ответа, `ApiExceptionHandler` (`@RestControllerAdvice`), `ApiError` |
| `.service` | `OrderService` — бизнес-логика, генерация `id` и `createdAt` |
| `.repository` | `OrderRepository` (интерфейс) и `InMemoryOrderRepository` |
| `.domain` | `Order`, `OrderItem`, `OrderStatus` |

Корень сборки: `pom.xml`, `mvnw`; тесты — `src/test/java/com/example/orders/...`.

**1. Хранилище: `ConcurrentHashMap<UUID, Order>` плюс монотонный счётчик создания.**
Замена — `computeIfPresent`, удаление — `remove`: операции атомарны, `PUT`/`DELETE` одного `id` не дают 500 (AC-23). Список сортируется по `(createdAt, порядковый номер создания)`: счётчик убирает недетерминизм при равных `createdAt` (AC-7). Альтернатива — `synchronized`/`ReentrantLock` на всё хранилище: проще, но сериализует чтения без необходимости.

**2. `id` — `UUID.randomUUID()`, `createdAt` — `Instant`, задаются в `OrderService`.**
Клиентские значения не принимаются: DTO запроса не содержит `id`/`createdAt`, лишние поля Jackson игнорирует (AC-13). Альтернатива — отдельная проверка и 400 на лишний `id`; отвергнута, так как спека требует игнорирования.

**3. `price` — `BigDecimal`, в JSON — число.**
Исключает потерю точности `double` (AC-21). Проверка в тесте сравнивает числовое значение (`compareTo`), а не строковую запись: `10.50` и `10.5` считаются равными. Альтернатива `double` отвергнута из-за точности.

**4. Валидация — Bean Validation на DTO (`@NotBlank`, `@NotEmpty`, `@Valid` на элементах, `@Positive` для `quantity`, `@PositiveOrZero` для `price`).**
`status` — enum; неизвестное значение даёт `HttpMessageNotReadableException`. Для `PUT` используется тот же DTO, что и для `POST`, поэтому неполное тело даёт 400 автоматически (AC-3…AC-6, AC-15).

**5. Единый `@RestControllerAdvice` отображает ошибки в `ApiError {status, error, message}`:**
`MethodArgumentNotValidException`, `HttpMessageNotReadableException`, `MethodArgumentTypeMismatchException` (не-UUID в пути) → 400; `OrderNotFoundException` → 404. Стек-трейс в тело не попадает (AC-20). Не-UUID в пути обрабатывается на уровне привязки параметра до вызова сервиса, поэтому 400 всегда раньше 404 (AC-11, AC-16, AC-19).
Неперехваченные ошибки (500) намеренно не специфицируются в этом change.

**6. Тестовая стратегия (по `domain/05-testing-and-quality.md`):**
срез `@WebMvcTest` + MockMvc для контракта и валидации; `@SpringBootTest` + MockMvc для сценариев с состоянием; конкурентные AC-22/AC-23 — `ExecutorService` и `CountDownLatch` поверх `@SpringBootTest`.

**Соответствие `domain/03-architecture.md`:** слои controller → service → in-memory repository соблюдены, внешних интеграций нет; отступлений нет, ADR не нужен.

**Контракт и обратная совместимость:** API новый, совместимость с предыдущими версиями не требуется. Порядок выкатки — единственный сервис, contract-first не применим: потребителей нет.

## Risks / Trade-offs

- [`customerName` окажется реальным ПДн] → в спеке и глоссарии зафиксировано «условное имя»; при появлении реальных данных класс повышается до R3 и нужен ADR.
- [Конкурентные тесты нестабильны (flaky)] → фиксированное число потоков, ожидание через `CountDownLatch`, проверка инвариантов, а не порядка ответов.
- [Порядок списка зависит от разрешения часов] → tie-break счётчиком создания, а не метками времени.
- [R2 выбран для демо, хотя по осям это R1] → осознанное решение автора; зафиксировано в `proposal.md`.
- [Данные теряются при перезапуске] → принято в ADR-0001.
- [Ограничения, выведенные при составлении intent и не подтверждённые автором напрямую] → вынесены в Open Questions и подтверждаются ревью PR #1.

## Rollback plan

Сервис новый и без состояния за пределами памяти процесса: откат — остановить сервис и откатить (`git revert`) коммит PR #2 в `a-erpuo-demo-service`. Миграций данных нет, потребителей нет. Откат спеки — `git revert` PR #1 до `openspec archive`.

## Open Questions

Не меняют структуру tasks, но их подтверждает ревьюер PR #1; до ответа в спеке действуют принятые значения:

- Непустота `customerName` и `items`, дефолт `NEW`, изменяемость `status` через `PUT`, 400 на неизвестный `status` — выведены при составлении intent; подтвердить.
- Игнорирование `id` из тела `PUT` (альтернатива — 400); подтвердить.
- Поведение при неверном `Content-Type` не специфицируется (Non-goal) — оставить решение Spring по умолчанию?
