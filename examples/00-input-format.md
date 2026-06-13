# Worked example 00 · Input format
**Уровень внедрения:** L1
Иллюстративный сценарий: реалистичный multi-biomarker input от lab-aggregator'а проходит preconditions validation. Проверяет границу «выше по потоку → ядро нормализации»: что принимается, что отвергается, и в какой форме возвращается результат валидации.
## Контекст и input
Стандартная панель check-up'а из 4 биомаркеров, собранная лабораторией-агрегатором. Покрывает типичные граничные случаи в сырых полях: запятая в числе, below-LOQ значение, пустой unit, null в raw_ref, comment с method-маркером.
```json
{
  "lab": "lab-aggregator-00",
  "date": "2026-05-15",
  "type": "венозная кровь натощак",
  "biomarkers": [
    {
      "raw_name": "Глюкоза",
      "raw_value": "5,4",
      "raw_unit": "ммоль/л",
      "raw_ref": "3.9–6.1",
      "raw_comment": null
    },
    {
      "raw_name": "Гликированный гемоглобин",
      "raw_value": "5.6",
      "raw_unit": "%",
      "raw_ref": "4.0–6.0",
      "raw_comment": "Метод: HPLC (BioRad D-100)"
    },
    {
      "raw_name": "С-реактивный белок",
      "raw_value": "< 0.5",
      "raw_unit": "мг/л",
      "raw_ref": "< 5.0",
      "raw_comment": null
    },
    {
      "raw_name": "Антитела к HBs",
      "raw_value": "отрицательно",
      "raw_unit": "",
      "raw_ref": null,
      "raw_comment": "Качественный тест"
    }
  ]
}
```
## Preconditions validation pipeline
### Шаг 1 · Schema validation (input.schema.json, Draft 2020-12)
- Обязательные поля верхнего уровня: `lab`, `date`, `type`, `biomarkers` — все присутствуют и имеют правильные типы
- Каждый biomarker-объект имеет `raw_name`, `raw_value`, `raw_unit` (все строки), `raw_ref` (string или null), `raw_comment` (string или null). Всё валидно
### Шаг 2 · ISO 8601 date check
- `"2026-05-15"` — valid ISO 8601 date (no time-of-day, no timezone — OK для даты забора)
### Шаг 3 · Type-shape биомаркеров
- `raw_value` — всегда string, даже для чисел. Это принципиально: сохраняет исходную форму (`"5,4"`, `"< 0.5"`, `"отрицательно"`) для семантического парсинга ниже по pipeline
- `raw_unit` — может быть пустой строкой (`""`), но не null. Пустота — валидный сигнал для качественных тестов (Anti-HBs)
- `raw_ref`, `raw_comment` — nullable. null означает «управляется ниже по потоку», пустая строка — «upstream передал явный пустой комментарий» (разный семантический сигнал)
### Шаг 4 · Что НЕ делается на этом уровне
- Не транслитерируется `raw_unit` (`ммоль/л` остаётся как есть) — это работа ADR-0002
- Не парсится `raw_value` (`"5,4"` остаётся строкой) — это работа value-парсера ниже по pipeline
- Не резолвится `raw_name` в LOINC — это работа resolver (ADR-0001/ADR-0003)
- Preconditions — строго формальный слой, не семантический
## Output: результат preconditions validation
Результат — передача validated input в Ядро нормализации. Для этого примера: 4 biomarker-объекта проходят валидацию и передаются в mapping pipeline, каждый получит своё envelope (ADR-0006).
```json
{
  "preconditions": "passed",
  "input_id": "sha256:3b8f1a2c9d4e6f70a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718",
  "envelopes": [
    {
      "status": "success",
      "mapping_id": "map_2026-05-15_glucose_a3f2",
      "primary": {
        "system": "LOINC",
        "code": "14749-6",
        "display": "Glucose [Moles/volume] in Serum or Plasma"
      },
      "value": {
        "numeric": 5.4,
        "string": null,
        "unit": "ммоль/л",
        "ucum_canonical": "mmol/L"
      },
      "context": {
        "sample_type": "blood",
        "fasting_status": "fasting",
        "method": null,
        "context_completeness": "partial"
      },
      "alternatives": [],
      "audit": {
        "loinc_version": "2.82",
        "ucum_version": "2.2",
        "mapping_timestamp": "2026-05-15T08:01:11Z",
        "primary_score": 0.96,
        "top_score_by_resolver": 0.96,
        "score_delta_vs_top": 0,
        "rule_applied": null,
        "rule_overrode_score": false,
        "method_alias_group": "loinc:component:b2c3d4",
        "ucum_conversion": null,
        "remap_source": null,
        "current_version_status": "current",
        "replaced_by": null,
        "candidates_returned": 2,
        "policy_version": "1.0.0",
        "resolver_version": "semantic-resolver-v1"
      }
    }
  ],
  "audit": {
    "request_timestamp": "2026-05-15T08:01:11Z",
    "loinc_version": "2.82",
    "ucum_version": "2.2"
  }
}
```
Это ответ `POST /v1/map` по `map-response.schema.json`: top-level дискриминатор `preconditions: "passed"`, детерминированный `input_id` (sha256 входного байт-стрима — общий ключ для tracing и idempotent retry) и массив `envelopes` — по одному на biomarker (ADR-0006 cardinality). Выше для краткости показан envelope только первого биомаркера (Глюкоза); в полном ответе их 4, каждый со своим `mapping_id` и `status` (success или rejected) — детальные маппинги см. в examples 01–06. Wrapper-`audit` несёт snapshot версий стандартов на момент запроса (ADR-0005).
## Сценарий провала preconditions
Если входные данные не проходят валидацию (например, `biomarkers` — null вместо массива), возвращается rejection envelope на уровне input'а, не по biomarker'у:
```json
{
  "preconditions": "failed",
  "input_id": "sha256:7c4e1b9a2f8d3e6c0b5a4f9e8d7c6b1a3f2e0d9c8b7a6f5e4d3c2b1a0f9e8d7c",
  "error": {
    "code": "INPUT_SCHEMA_VIOLATION",
    "message": "Field 'biomarkers' must be an array of biomarker objects, got null",
    "path": "$.biomarkers"
  },
  "audit": {
    "request_timestamp": "2026-05-15T08:01:11Z",
    "loinc_version": "2.82",
    "ucum_version": "2.2"
  }
}
```
- В этом случае Ядро нормализации не вызывается, mapping_id не создаётся — это синтаксическая ошибка контракта (`INPUT_SCHEMA_VIOLATION` — нарушение типа/схемы входа; семантические инварианты дают `PRECONDITION_FAILED`)
- `input_id` присутствует и в failed-ветке: тот же вход даёт тот же sha256-хеш (tracing/idempotency), но persistent mapping не создаётся
- `path` в JSONPath-форме указывает точку нарушения для отладки выше по потоку
## Что валидирует в preconditions
- Граница источник → нормализация четкая: preconditions — это syntax и type-shape, не семантика. Маппинг-ошибки (UCUM-reject в example 04) живут ниже, не здесь
- raw_\* поля — всегда string (даже для чисел), что сохраняет traceability и предотвращает преждевременную нормализацию на стороне источника
- Различие `""` и `null` в raw_unit/raw_ref/raw_comment фиксирует разный семантический сигнал, как в ADR-0004 различие `null` и `"not_applicable"`
## Выявленные вопросы (новые)
- **`POST /v1/map`**** shape: 1 input = N envelopes или 1 input = 1 envelope?** ADR-0006 явно не разрешил: input.schema имеет `biomarkers` как array, но в example 04 input был с 1 biomarker'ом и envelope выводился в singular shape. Нужен patch ADR-0006: `/v1/map` возвращает `{ preconditions, input_id, envelopes: [...] }` (плюрал), или input ограничивается 1 biomarker'ом (тогда batch — обязательно в v1.0, не Phase 2)
- **Граничные случаи preconditions, не покрытые явно:** `biomarkers: []` (пустой array) — валидный no-op или PRECONDITION_FAILED? `raw_value: ""` (пустая строка) — валидно или нет? Дубли `raw_name` в biomarkers (два «Глюкоза» подряд, реальный кейс retest) — валидно или нет? Нужны явные решения в [preconditions.md](../docs/preconditions.md) или input.schema constraints
- **Rejection envelope на preconditions-уровне отличается от envelope в ADR-0006:** здесь `preconditions: "failed"` на верхнем уровне, не `status: "rejected"` по biomarker'у. Это второй тип rejection (синтаксический против семантического). Закрыто patch ADR-0006 (two-level rejection: preconditions против mapping)
## История изменений
- 2026-05-30 — создан worked example 00 (Input format, multi-biomarker check-up panel)
- 2026-06-05 — output-блоки приведены к map-response.schema: passed-ветка показывает wrapper \{ preconditions, input_id, envelopes, audit \}, failed-ветка дополнена input_id и кодом INPUT_SCHEMA_VIOLATION; input_id в формате sha256
