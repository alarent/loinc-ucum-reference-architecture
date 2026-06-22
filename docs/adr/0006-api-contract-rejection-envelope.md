# ADR-0006 · API contract + rejection envelope
- **Статус:** Accepted
- **Уровень внедрения:** L4
- **Дата:** 2026-05-30
- **Связанные ADR:** ADR-0002 (UCUM normalize/reject), ADR-0005 (Version-aware mapping), ADR-0001–0004 (success-path)
- **Связанные worked examples:** 04 (rejection), 05 (remap), 01–03 (success)
## Контекст
К v1.0 зафиксирован success-path output (Schemas · индекс, ADR-0001–0005), но не зафиксирована API-поверхность как целое: какие endpoints существуют, как они версионируются, какой envelope получает потребитель на rejection-путях (worked example 04 эмпирически показал, что success-shape для rejection не годится), и попадают ли rejected-результаты в persistent audit log как полноценные mapping_id.
Без этого ADR три проблемы. Потребитель не знает, по какой схеме парсить ответ (success vs rejection). API-версия путается со snapshot-версиями LOINC/UCUM (ADR-0005 разделил их концептуально, но не закрепил policy). Retry-семантика после исправления выше по потоку не определена: если rejection нет в audit log, `/remap` не может его «вылечить».
## Решение
### 1. Endpoints v1.0
- `POST /v1/map` — создать mappings из input. Input содержит массив `biomarkers` (см. worked example 00), ответ — `{ preconditions, input_id, envelopes: [...] }`, где каждый envelope соответствует одному biomarker'у. Cardinality 1-input → N-envelopes (N ≥ 1); каждый envelope независимо принимает `status: "success"` или `"rejected"`
- `POST /v1/remap` — мигрировать существующий mapping на актуальные снапшоты LOINC/UCUM (семантика и idempotency — ADR-0005). Принимает один `mapping_id`, возвращает один envelope (1-to-1)
- `GET /v1/mappings/{mapping_id}` — получить mapping; computed-поля (`current_version_status`, `replaced_by`) вычисляются в момент запроса. Возвращает один envelope
### 2. Unified envelope (oneOf)
- Единая JSON Schema `output.schema.json` с дискриминатором `status` (`"success" | "rejected"`)
- При `status: "success"` envelope содержит `primary`, `value`, `context`, `alternatives`, `input_echo`, `audit` (см. Schemas · индекс)
- При `status: "rejected"` envelope содержит `error`, `input_echo`, `audit` (`primary`/`value`/`alternatives` отсутствуют — их нет по семантике)
- `input_echo` присутствует симметрично в обеих ветках: на success-пути он даёт полный audit trail и verbatim-passthrough `raw_ref` (в т.ч. возрастно/полово-стратифицированных референсов). Введено в v2.0.0 (breaking: success теперь обязан нести `input_echo`)
- `mapping_id` присутствует в обоих вариантах, `audit.loinc_version`/`audit.ucum_version`/`audit.mapping_timestamp` — тоже
### 3. Rejection persistence
- Rejected mappings получают стабильный `mapping_id` и сохраняются в audit log наравне с success
- `POST /v1/remap` для rejected `mapping_id` после исправления выше по потоку создаёт новый success-mapping с `audit.remap_source = <rejected_mapping_id>` (при условии что источник передал исправленный input рядом)
- Альтернатива «rejection эфемерны» отвергнута: потребителю пришлось бы строить свою retry-state-machine, что противоречит cross-cutting принципу
### 4. API versioning policy
- API-версия — SemVer в URL path (`/v1/`, `/v2/`)
- API-версия **decoupled** от снапшотов LOINC/UCUM (это уже зафиксировано ADR-0005, здесь переформулировано как явная policy)
- Breaking changes envelope shape → major bump; аддитивные (новый error code, новое опциональное поле) → minor; уточнения без shape-изменений → patch
### 5. Backward compatibility в пределах major
- Success envelope расширяется только аддитивно (новые опциональные поля)
- Список error codes расширяется только аддитивно; существующий код не меняет `stage_failed` или семантику reject
- Снятие любого поля или сужение enum → требует major bump
### 6. Два слоя отказа: preconditions vs mapping-level rejection
- **Preconditions failure** — input не прошёл синтаксическую валидацию (input.schema). Возвращается на верхнем уровне ответа: `{ preconditions: "failed", error: { code: "PRECONDITION_FAILED", message, path }, audit }`. `envelopes` отсутствует, `mapping_id` не создаётся, Ядро нормализации не вызывается
- **Mapping-level rejection** — input прошёл preconditions, но конкретный biomarker не маппится (UCUM reject, missing context, low-score fail). Появляется внутри `envelopes[i]` как envelope с `status: "rejected"` (секции 2–3). Mapping_id создаётся, persist в audit log
- Один ответ может комбинировать success+rejected внутри `envelopes` по biomarker'ам, но только при successful preconditions. Preconditions-failure всегда terminates до построения envelopes
- Worked example 00 иллюстрирует обе ветки
## Альтернативы
### Разделить success и rejection на HTTP-семантику (200 vs 422)
- **Отвергнуто:** rejection — валидный бизнес-результат нормализационного слоя, не клиентская ошибка. Mixing бизнес-семантики и HTTP-семантики ведёт к hidden coupling. Потребителю всё равно нужна uniform retrieval (`GET /mappings/{id}`), которая не различает варианты по HTTP-коду
### Два отдельных schema-файла (`success.schema.json` + `rejection.schema.json`)
- **Отвергнуто:** потребитель всё равно получает один JSON и должен выбрать схему для валидации — это перекладывает discrimination на потребителя. `oneOf` с явным дискриминатором в одном файле делает то же самое без фрагментации контракта
### Не persistить rejected mappings
- **Отвергнуто:** ломает `/remap`-recovery после upstream fix; превращает rejection в throw-away событие без traceability. Противоречит принципу  «реф-слой принимает решения» — audit-история для retry должна быть в реф-слое, не выше по потоку
### Coupling API-версии к LOINC snapshot
- **Отвергнуто:** ADR-0005 уже зафиксировал, что snapshot — это data versioning, не API contract. Coupling заставлял бы major-bump на каждый LOINC release (2–4 раза в год), что бессмысленно для потребителей
## Последствия
### Положительные
- Потребитель парсит ответ по одной схеме с явным дискриминатором — никаких «угадай схему по HTTP-коду»
- `/remap` может «лечить» rejected mappings после исправления выше по потоку, audit chain прозрачна
- API и снапшоты LOINC/UCUM эволюционируют независимо
### Отрицательные / trade-offs
- Persistent rejections увеличивают размер audit log; нужна retention policy (открытый вопрос)
- `oneOf`-envelope требует от потребителя type-narrowing по `status` — это явно и стандартно для JSON Schema/TypeScript, но всё равно стоит отметить
- API SemVer не предотвращает семантический drift (поведение endpoint меняется без shape-изменений) — это открытый вопрос по behavior-версионированию
## Связь с другими ADR
- **ADR-0002:** rejection envelope формализует то, что ADR-0002 сделал семантически (три error codes); этот ADR фиксирует их wire-format
- **ADR-0005:** наследует decoupling API-версии от снапшотов; `/remap` определён здесь как endpoint, его idempotency и dedup-ключ — в ADR-0005
- **ADR-0001–0004:** заполняют success-side envelope; этот ADR оборачивает их в единый контракт
## Открытые вопросы
Все открытые вопросы закрыты на housekeeping pass 2026-05-31:
- **Retention policy для rejected mappings** (TTL vs бессрочно). v1.1 backlog ([Roadmap · v1.1 / Phase 2 / ADR candidates](../roadmap.md)). v1.0 хранит бессрочно по умолчанию audit log; явная policy фиксируется в v1.1.
- **Batch endpoints** (`POST /v1/map/batch`, `POST /v1/remap/batch`). Phase 2 backlog.
- **Streaming / async** для длинных `/remap`-кампаний. Phase 2 backlog.
- **Behaviour-level versioning** (non-shape семантические изменения API). Мигрировано в ADR candidates backlog ([Roadmap · v1.1 / Phase 2 / ADR candidates](../roadmap.md)); кандидат на отдельный ADR при появлении конкретного случая semantic drift'а в проде.
- **Локализация ****`error.message`** (machine-readable code + i18n template). v1.1 backlog как ADR-кандидат.
- **Граничные случаи preconditions** (`biomarkers: []`, `raw_value: ""`, дубли `raw_name`). Делегировано [preconditions](../preconditions.md): явные constraints зафиксированы в секции  «Семантические граничные случаи и инварианты входа» (`minItems: 1` для `biomarkers`, `minLength: 1` для `raw_name`/`raw_value`, дубли `raw_name` разрешены как валидный сценарий — один analyte разными методами / повторный забор) и зеркалируются в [input.schema.json](../../schemas/input.schema.json).
## История изменений
- 2026-05-30 — создан ADR-0006 (API contract + rejection envelope) на основании findings из worked example 04
- 2026-05-30 — patch: явная cardinality `/v1/map` (1 input → N envelopes), добавлена секция «Два слоя отказа» (preconditions vs mapping-level), уточнены cardinality `/v1/remap` и `GET /v1/mappings/{id}`. Триггер: findings из worked example 00
- 2026-05-31 — housekeeping pass: закрыты все 6 open question (4 deferred в Roadmap, 1 в ADR candidates backlog, 1 делегирован [preconditions.md](../preconditions.md) + input.schema.json)
- 2026-06-22 — v2.0.0 (breaking): `input_echo` стал обязательным в success-ветке envelope, симметрично rejected. Полный audit trail на success-пути + verbatim-сохранение `raw_ref` (включая стратифицированные референсы). Обновлены output.schema.json, worked examples 00–03/05/06, README, architecture, Schemas · индекс
