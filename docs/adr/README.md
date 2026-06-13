# ADR · индекс
Architecture Decision Records. Каждое значимое архитектурное решение фиксируется в формате Nygard:
- **Status:** Accepted / Superseded by ADR-XXXX
- **Context:** какой разрыв или вопрос решаем
- **Decision:** что выбрали
- **Alternatives considered:** с краткой оценкой
- **Consequences:** положительные и отрицательные следствия
- **References:** ссылки на architecture, examples
## Шаблон ADR
```markdown
# ADR-NNNN: <Title>

Status: Accepted | Proposed | Superseded by ADR-YYYY
Date: YYYY-MM-DD

## Context

<gap или вопрос, который решаем; почему этот вопрос вообще возник>

## Decision

<что выбрали; одно-два предложения, без воды>

## Alternatives considered

<список альтернатив с краткой оценкой каждой и почему отвергли>

## Consequences

### Positive
<что становится возможным или упрощается>

### Negative
<что становится сложнее, какие компромиссы приняли>

## References

<ссылки на architecture, examples, внешние материалы>
```
## Список ADR для v1.0
Статус: все 8 ADR закрыты (Accepted) для v1.0. ADR-0007 добавлен по ходу из finding в worked example 03 (изначальный план был 6 ADR); ADR-0008 добавлен при формализации модели capability levels.
1. **ADR-0001:** LOINC as primary identifier — выбор первичной кодовой системы; альтернатива SNOMED CT (снята как категориальная ошибка для лаб-домена)
2. **ADR-0002:** UCUM mandatory preprocessing + reject on failure (+ qualitative bypass patch) — обязательность UCUM-формы для численной ветки; qualitative results минуют UCUM и возвращаются через `value.string`
3. **ADR-0003:** Explicit priority policy + alternatives in output (+ score visibility patch) — двухслойный pipeline (resolver → YAML priority policy); `audit.top_score_by_resolver` и `audit.score_delta_vs_top` всегда присутствуют
4. **ADR-0004:** Semantic context mandatory (+ not_applicable patch) — 3 context-поля в output (sample_type, method, fasting_status) + context_completeness; трёхсостоянные (value / null / `"not_applicable"`). patient_state отложен в Phase 2, collection_datetime — в v1.1
5. **ADR-0005:** Version-aware mapping as first-class (+ remap source patch) — snapshot-versioned audit fields + явный `/remap` endpoint с idempotency dedup-key; nullable `audit.remap_source` в derived mappings
6. **ADR-0006:** API contract + rejection envelope (+ cardinality patch) — endpoints `/v1/map`, `/v1/remap`, `GET /v1/mappings/{id}`; unified envelope с oneOf-дискриминатором `status`; `/v1/map` cardinality 1→N через response wrapper (map-response.schema.json); two-level rejection (preconditions vs mapping)
7. **ADR-0007:** Method aliasing — `audit.method_alias_group` формата `loinc:component:{component_hash}`, deterministically derived from LOINC COMPONENT; snapshot-versioned
8. **ADR-0008:** Capability gating via nullable fields — schema всегда полная (v1.0); поля nullable, пока уровень не включён; nullability = намеренный capability gating, основа модели capability levels
Каждый ADR живёт отдельной страницей-ребёнком этого индекса. Патчи применены in-place внутрь соответствующих ADR (не отдельные ADR), со ссылками на worked examples, которые их вызвали.
## Phase 2 candidates
- **ADR-0009 (кандидат):** Substance-aware unit conversion (mass↔molar через molecular weight lookup) — в v1.0 явное исключение из задачи (см. [scope](../scope.md))
- **ADR-0010 (кандидат):** Подключаемость резолвера + eval harness — формализовать слой резолвера как заменяемый порт (длинное имя LOINC → код) + фреймворк оценки с золотым набором `(raw_name_RU → expected_LOINC)`; отложено до появления данных для оценки и продакшен-резолвера (см. [Roadmap · v1.1 / Phase 2 / ADR candidates](../roadmap.md))
- ADR-0005 patch кандидат: версионирование transliteration-таблицы (отдельный SemVer vs вместе с UCUM snapshot)
- Behaviour-level versioning (non-shape семантические изменения)
- `error.message` localization (machine-readable code + i18n template, v1.1)
## Решения, которые не получают отдельный ADR
Эти решения упоминаются в architecture скобкой, без полного формата ADR — они не несут самостоятельных компромиссов:
- внутренняя каноническая форма — английский (упомянуто во Входном адаптере)
- FHIR-совместимость как мотивация LOINC — упомянуто в ADR-0001
- формат audit trail — задаётся output.schema, не trade-off
- использование доменно-обученных эмбеддингов (BioLORD-класс) — implementation detail Резолвера LOINC, не архитектурное решение
- локализация во Входном адаптере — функциональное требование, не выбор между альтернативами
- [ADR-0001 · LOINC as primary identifier](0001-loinc-as-primary-identifier.md)
- [ADR-0002 · UCUM mandatory preprocessing + reject on failure](0002-ucum-mandatory-preprocessing.md)
- [ADR-0003 · Explicit priority policy + alternatives in output](0003-explicit-priority-policy.md)
- [ADR-0004 · Semantic context mandatory](0004-semantic-context-mandatory.md)
- [ADR-0005 · Version-aware mapping](0005-version-aware-mapping.md)
- [ADR-0006 · API contract + rejection envelope](0006-api-contract-rejection-envelope.md)
- [ADR-0007 · Method aliasing](0007-method-aliasing.md)
- [ADR-0008 · Capability gating (nullable fields)](0008-capability-gating.md)
