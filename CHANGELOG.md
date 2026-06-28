# Changelog

Все значимые изменения проекта документируются в этом файле.
Формат основан на «Keep a Changelog», проект следует Semantic Versioning.

## [2.1.0] — 2026-06-27

### Changed

- **Устранение дрейфа документации и примеров относительно `schemas/output.schema.json`.** Контрактные схемы по сути не меняются — правятся описания/комментарии и worked examples.
- `value.unit` — семантика уточнена: «единица как сообщил источник, в UCUM-нотации, до конверсии»; дословный ввод — в `input_echo.raw_unit` (`schemas/output.schema.json`, `schemas/README.md`, examples 00/06).
- `context_completeness` — добавлено явное правило (ADR-0004 §4): поле определено, если значение конкретно ИЛИ `not_applicable`; `null` = не определено; `minimal` = определённых нет. Убрана неоднозначность «≤1».
- ADR-0001 — naming приведён к `output.schema.json`: `loinc_code` → `primary.{system,code,display}`; `alternatives[]` — только LOINC (`system: "LOINC"` const); не-LOINC secondary — в audit/input_echo (structured-поле Phase 2).
- ADR-0006 §6 — синтаксический отказ `INPUT_SCHEMA_VIOLATION` отделён от семантического `PRECONDITION_FAILED`; назван код `LOINC_NO_CANDIDATE` (stage `loinc_resolution`).
- `mapping_id` — задокументировано соглашение `map_YYYY-MM-DD_[<slug>_]<hash>` (pattern в схеме не enforced).

### Fixed

- `examples/00` — `raw_unit: ""` (schema-невалидно) → `null` для качественного теста; primary глюкозы `14749-6` → `15074-8` (Blood) под `sample_type=blood`; `candidates_returned` `2` → `1`.
- `examples/05` — `mapping_id` к дефисному формату дат; `resolver_version` унифицирован к `semantic-resolver-v1`; прозовые `primary.loinc_code` → `primary.code`.
- `examples/04` — добавлен альтернативный сценарий отказа `LOINC_NO_CANDIDATE`.

## [2.0.0] — 2026-06-22

### Changed

- **BREAKING:** `input_echo` теперь обязательное поле success-ветки envelope (`schemas/output.schema.json`), симметрично rejected-ветке. Раньше `input_echo` присутствовал только в rejected. Изменение обязательности output-контракта — major bump по versioning.md. Потребители, валидирующие success-ответы по схеме, обязаны учитывать новое required-поле.

### Updated

- `schemas/output.schema.json` — `input_echo` добавлен в `required` и `properties` `success_envelope` (переиспользует существующий `$defs/input_echo`).
- Worked examples 00, 01, 02, 03, 05, 06 — success-envelope'ы дополнены блоком `input_echo` (эхо сырого входа).
- `README.md`, `docs/architecture.md` — success-примеры дополнены `input_echo`.
- `schemas/README.md`, `docs/adr/0006-api-contract-rejection-envelope.md` — зафиксирована симметрия `input_echo` в обеих ветках.

## [1.0.0] — 2026-06-13

### Added

- Первый публичный релиз reference-архитектуры нормализации лабораторных данных (LOINC/UCUM).
- Трёхслойная архитектура: входной адаптер, ядро нормализации, выходной интерфейс.
- 8 ADR (0001–0008), все Accepted.
- 7 worked examples (00–06).
- 3 JSON Schema: input, output, map-response (Draft 2020-12).
- Документы: architecture, scope, preconditions, versioning, glossary, ucum-transliteration, capability-levels, ip-statement.
- Модель уровней зрелости внедрения (capability levels, L1–L4).
- Двойное лицензирование: Apache-2.0 (код/схемы/примеры) AND CC-BY-4.0 (документация).
- Стандарты: LOINC 2.82, UCUM 2.2.
