# Schemas · индекс
JSON Schema файлы для входного и выходного контрактов нормализационного слоя.
## Schema-файлы (3/3 DRAFTED)
Формальные JSON Schema (Draft 2020-12) живут отдельными страницами этого индекса. Финальные shape'ы — там, не здесь.
- [input.schema.json](input.schema.json) — формат данных от источника (после OCR/парсинга). Контракт совпадает с [preconditions](../docs/preconditions.md).
- [output.schema.json](output.schema.json) — единый envelope с `oneOf` по дискриминатору `status` (`"success" | "rejected"`); см. ADR-0006 и Resolved decision «Unified envelope» ниже.
- [map-response.schema.json](map-response.schema.json) — top-level response wrapper для `POST /v1/map`: `{ preconditions, input_id, envelopes: [...] }`; см. ADR-0006 patch и Resolved decision «Response wrapper» ниже.
Resolved decisions ниже фиксируют принципы, по которым эти схемы написаны. При патчах схем — обновлять и Resolved decisions, и сами файлы, и соответствующие worked examples.
## Resolved decisions
Решения, зафиксированные после worked examples 01–03. Применяются при написании финального `output.schema.json`.
### `value`: `numeric` XOR `string`
- `value.numeric` (number, опционально) — для количественных результатов
- `value.string` (string, опционально) — для качественных и below-LOQ значений (`"negative"`, `"positive"`, `"<5"`)
- Ровно одно из двух полей обязательно; обе формы взаимоисключающие
- Source: открытый вопрос worked example 01
### `value.unit` и `value.ucum_canonical` — всегда оба
- `value.unit` — исходная единица как пришла в input (после транслитерации, если применима)
- `value.ucum_canonical` — каноническая UCUM-форма (после возможной конверсии)
- Оба поля присутствуют **всегда**, даже если равны. `audit.ucum_conversion` (объект `{ input_unit, canonical_unit, factor, path }` или null) фиксирует факт конверсии явно: непустой объект = конверсия выполнялась
- Удаление любого поля ломает либо traceability (нет `unit`), либо canonical-контракт (нет `ucum_canonical`)
- Source: открытый вопрос worked example 01
### `confidence`: каноническое число, категории — производное
- Контракт несёт **число**: `audit.primary_score` — `[0, 1]`. Это единственный confidence-сигнал в envelope
- Категории `"high" | "medium" | "low"` **в контракт не входят**. Это необязательное производное представление: потребитель при желании вычисляет их из числа по документированным порогам под свою нужду (например, `high ≥ 0.9`)
- Почему так: число несёт больше информации, его можно threshold'ить ниже по потоку под разные кейсы (CDSS строже, дашборд мягче); категория — потеря информации, легко выводится из числа, но не наоборот. Риск «ложной точности» (что значит 0.98 против 0.97) снимается тем, что слой **не обещает** семантику каждой сотой доли — только воспроизводимое число; пороги категорий документируются поверх и в контракт не зашиты
- Согласуется с `alternatives[]`, где тоже стоит число `score`, а не категория
- Слой **не отклоняет** маппинг по порогу score (нет `LOINC_LOW_CONFIDENCE`); решение о пороге — на стороне потребителя. Конкретные границы категорий — provisional до калибровки на validation pass
- Source: worked example 01 + ADR-0003 (порог-вопрос закрыт устранением)
### Version-aware audit fields (ADR-0005)
- `audit.loinc_version` (string, required) и `audit.ucum_version` (string, required) — версии справочников на момент маппинга
- `audit.mapping_timestamp` (string, ISO 8601, required) — момент фиксации маппинга
- Эти три поля — **immutable snapshot**: фиксируются один раз при создании маппинга и никогда не пересчитываются
- `audit.current_version_status` (enum: `"current" | "deprecated" | "replaced" | "removed"`, required) и `audit.replaced_by` (string или null) — **computed at retrieval**, вычисляются при каждом запросе сравнением snapshot-версии с актуальным состоянием LOINC
- `audit.remap_source` (string или null) — заполняется только в маппингах, созданных через `/remap`; содержит `mapping_id` исходного (deprecated/replaced) маппинга. В обычных маппингах — null
- Snapshot vs computed split: snapshot-поля дают reproducibility (тот же score при том же snapshot); computed-поля дают актуальный статус без переписывания истории
- Идемпотентность `/remap`: dedup-ключ `(source_mapping_id, loinc_version, ucum_version)`. Повторный `/remap` для уже мигрированного source возвращает существующий derived маппинг, не создаёт дубль
- Source: worked example 05 + ADR-0005
### Unified envelope (ADR-0006)
- `output.schema.json` — единый файл с `oneOf` по дискриминатору `status` (`"success" | "rejected"`)
- Success-ветка: `primary`, `value`, `context`, `alternatives`, `input_echo`, `audit`
- Rejected-ветка: `error` (`code`, `message`, `stage_failed`), `input_echo`, `audit`
- `input_echo` присутствует симметрично в обеих ветках (success и rejected): эхо сырого входа даёт полный audit trail на success-пути и verbatim-сохранение `raw_ref` (включая возрастно/полово-стратифицированные референсы). Введено в v2.0.0 (breaking)
- В обеих ветках обязательны: `mapping_id`, `audit.loinc_version`, `audit.ucum_version`, `audit.mapping_timestamp`
- Альтернатива  «два отдельных schema-файла (success/rejected)» отвергнута: перекладывает discrimination на потребителя. Альтернатива  «HTTP-семантика (200 vs 422)» отвергнута: rejection — валидный бизнес-результат, не клиентская ошибка
- Source: worked example 04 + ADR-0006
### Method alias group (ADR-0007)
- `audit.method_alias_group` (string, required) — канонический id группы method-variants для данного аналита
- Формат id: `loinc:component:{component_hash}`, выводится из LOINC COMPONENT field детерминистично
- Snapshot-versioned: фиксируется вместе с `audit.loinc_version`, пересчитывается `/remap` (ADR-0005)
- Присутствует в success-ветке envelope; в rejected-ветке отсутствует (нет valid mapping → нет group)
- Source: worked example 03 + ADR-0007
### Response wrapper · map-response.schema.json (ADR-0006 patch)
- Top-level shape для `POST /v1/map`: `{ preconditions, input_id, envelopes, error?, audit }` — отдельный schema-файл `map-response.schema.json`, оборачивающий массив envelopes
- `preconditions: "passed" | "failed"`. На ветке `"failed"`: дополнительно `error` (`{ code, message, path }`) и `audit` (`request_timestamp`, `loinc_version`, `ucum_version`), `envelopes` отсутствует
- На ветке `"passed"`: `envelopes[i]` валидируется через `output.schema.json` (один envelope = success или rejected, oneOf-дискриминатор); `audit` (`request_timestamp`, `loinc_version`, `ucum_version`) присутствует
- Cardinality: 1 input → N envelopes (N ≥ 1, по числу biomarker'ов в input). `/v1/remap` и `GET /v1/mappings/{id}` — возвращают bare envelope (single), без wrapper'а
- Source: worked example 00 + ADR-0006 patch
### audit.ucum_conversion (worked example 06)
- Новое audit-поле, фиксирующее traceability UCUM conversion (для дебага расхождений между `value.unit` и `value.ucum_canonical`)
- Шейп: `{ input_unit, canonical_unit, factor, path }` или `null`
- `input_unit` — UCUM-string после transliteration (например, `g/dL`)
- `canonical_unit` — UCUM canonical из реф-словаря (например, `g/L`)
- `factor` — numeric coefficient (input × factor = canonical numeric)
- `path` — массив тегов conversion-этапов (например, `["mass_prefix_scaling"]`, `["exponent_algebra"]`)
- Поле равно `null`, когда conversion не выполнялась (input уже в canonical UCUM, factor = 1 и тождественные единицы)
- Совместимо с immutable-snapshot принципом ADR-0005: поле фиксируется в момент mapping_id и не пересчитывается при версионных апдейтах LOINC/UCUM
- Source: worked example 06
## Открытые вопросы для schemas
- Расширяемость audit: фиксированная схема vs `additionalProperties: true`
- Локализация полей context — английский внутренний, но как лейблить для UI-потребителей (v1.1)
## Что войдёт в репозиторий
- `schemas/input.schema.json` — формальная JSON Schema (Draft 2020-12)
- `schemas/output.schema.json` — формальная JSON Schema (Draft 2020-12), unified envelope
- `schemas/map-response.schema.json` — формальная JSON Schema (Draft 2020-12), wrapper для `/v1/map`
- `schemas/README.md` — короткое описание, ссылки на спецификации, версия каждого schema, ссылки на ADR-0002/0003/0004/0005/0006/0007 и worked examples 00–06
Все три схемы стабилизированы для v1.0. Дальнейшие изменения — через ADR-патчи.
- [input.schema.json](input.schema.json)
- [output.schema.json](output.schema.json)
- [map-response.schema.json](map-response.schema.json)
