# Changelog

Все значимые изменения проекта документируются в этом файле.
Формат основан на «Keep a Changelog», проект следует Semantic Versioning.

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
