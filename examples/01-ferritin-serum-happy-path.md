# Worked example 01 · Ferritin serum happy path
**Уровень внедрения:** L1
## Что демонстрирует
Базовый happy path: один лабораторный аналит с явным sample type, однозначно мапится в LOINC, единица корректно парсится UCUM. Все базовые ADR срабатывают синхронно, без конфликтов.
## Связанные ADR
- ADR-0001 (LOINC первичный идентификатор)
- ADR-0002 (UCUM mandatory preprocessing + reject on failure)
- ADR-0003 (Explicit priority policy + alternatives in output)
- ADR-0004 (Semantic context mandatory)
## Input
Каноническая форма от системы-источника (лаб-оркестратор или OCR бланка), без нормализации:
```json
{
  "lab": "lab-orchestrator-v3",
  "date": "2026-05-15",
  "type": "сыворотка крови",
  "biomarkers": [
    {
      "raw_name": "Ferritin",
      "raw_value": "85",
      "raw_unit": "ng/mL",
      "raw_ref": "30–400",
      "raw_comment": null
    }
  ]
}
```
## Шаги обработки
### 1. Validation входа
Schema validation по input.schema.json: `lab`, `date`, `type`, `biomarkers[]` и в каждом биомаркере `raw_name`, `raw_value`, `raw_unit`. Все присутствуют. Pass. Контекст (specimen, method) на входе не передаётся — его выведет ядро ниже из `type` и `raw_comment`.
### 2. UCUM preprocessing (ADR-0002)
`ng/mL` автоматически парсится UCUM-парсером. Это уже canonical UCUM-форма — конверсия не требуется, значение `85` сохраняется.
Промежуточное состояние:
```json
{
  "ucum_status": "valid",
  "original_unit": "ng/mL",
  "canonical_unit": "ng/mL",
  "converted": false,
  "value": 85
}
```
### 3. LOINC resolution (ADR-0001)
Ядро выводит контекст: `specimen=serum` из `type` («сыворотка крови»), `method` из `raw_comment` не выводится → `null`. Resolver получает `{name: "Ferritin", specimen: "serum", method: null}` и возвращает top-N candidates:
```json
[
  {"loinc_code": "2276-4",  "score": 0.94, "display": "Ferritin [Mass/volume] in Serum or Plasma"},
  {"loinc_code": "20567-4", "score": 0.71, "display": "Ferritin [Mass/volume] in Plasma"},
  {"loinc_code": "24373-3", "score": 0.45, "display": "Ferritin [Mass/volume] in Blood"}
]
```
### 4. Priority policy (ADR-0003)
Применяется правило `serum_specimen_prefers_serum_loinc`: если `context.specimen=serum` и среди candidates есть LOINC с `system=Ser` или `system=Ser/Plas` → выбрать его как primary.
2276-4 (`system=Ser/Plas`) подходит и одновременно является top по score → primary = 2276-4. Правило и score совпали; `rule_overrode_score=false`.
### 5. Semantic context attachment (ADR-0004)
Выведенный ядром контекст прикрепляется к выходу. Поля, которые ядро не смогло вывести, остаются явным `null` (не вырезаются) — потребитель отличает «метод не выведен» от конкретного значения.
### 6. Output assembly
Сборка финального контракта: primary + value + context + alternatives + audit + source.
## Output
```json
{
  "status": "success",
  "mapping_id": "map_2026-05-15_7f3a9b2e",
  "primary": {
    "system": "LOINC",
    "code": "2276-4",
    "display": "Ferritin [Mass/volume] in Serum or Plasma"
  },
  "value": {
    "numeric": 85,
    "string": null,
    "unit": "ng/mL",
    "ucum_canonical": "ng/mL"
  },
  "context": {
    "sample_type": "serum",
    "fasting_status": null,
    "method": null,
    "context_completeness": "partial"
  },
  "alternatives": [
    {"system": "LOINC", "code": "20567-4", "display": "Ferritin [Mass/volume] in Plasma", "score": 0.71},
    {"system": "LOINC", "code": "24373-3", "display": "Ferritin [Mass/volume] in Blood", "score": 0.45}
  ],
  "audit": {
    "loinc_version": "2.82",
    "ucum_version": "2.2",
    "mapping_timestamp": "2026-05-15T08:31:00Z",
    "primary_score": 0.94,
    "top_score_by_resolver": 0.94,
    "score_delta_vs_top": 0,
    "rule_applied": "serum_specimen_prefers_serum_loinc",
    "rule_overrode_score": false,
    "method_alias_group": "loinc:component:a1b2c3",
    "ucum_conversion": null,
    "remap_source": null,
    "current_version_status": "current",
    "replaced_by": null,
    "candidates_returned": 3,
    "policy_version": "1.0.0",
    "resolver_version": "semantic-resolver-v1"
  }
}
```
## Комментарий
> 💡 **Главный маркер happy path:** `rule_applied=serum_specimen_prefers_serum_loinc` совпадает с top по score. Правило priority policy и статистический resolver согласны. В example 03 (Multiple LOINC conflict) эти два слоя разойдутся — и именно разбор этого расхождения станет полезным для понимания ADR-0003.
Неочевидные детали контракта, которые этот example валидирует:
- **`primary.system`**** и ****`alternatives[].system`**** = «LOINC».** В v1.0 enum жёстко LOINC; явный `system` поддерживает будущее расширение `alternatives[]` на secondary-коды (внутренние/национальные) в v1.1 (ADR-0001, пункт 2).
- **`value.ucum_canonical`**** совпадает с ****`value.unit`****.** `ng/mL` уже canonical, конверсия не произошла. В example 06 (UCUM canonical conversion) будет случай реальной конверсии (`г/л` → `g/L`) с audit trail.
- **`context.method=null`**** сохранён явно**, а не выпущен из объекта. По ADR-0004 контекст выводит ядро; если метод не выводится из `raw_comment`, поле остаётся `null` — потребитель отличает «метод не выведен» от конкретного значения.
- **`audit`**** несёт координаты reproducibility:** `loinc_version`, `ucum_version`, `policy_version`, `resolver_version`, `candidates_returned`, `primary_score`. Это минимум, ниже которого audit теряет смысл — любой output можно воспроизвести и сравнить с историческим проходом.
- **`status: "success"`** — потому что UCUM прошёл и LOINC найден. Example 04 (UCUM rejection) покажет симметричный `status: "rejected"` с объектом `error`.
## Открытые вопросы из этого example
- [x] ~~`value.numeric`~~~~ vs ~~~~`value.string`~~~~ — как контракт выглядит для качественных результатов («positive», «+++»)?~~ **Resolved 2026-05-29:** оба поля опциональны, взаимоисключающие — `numeric` для количественных, `string` для качественных и below-LOQ. См. [Schemas · индекс](../schemas/README.md).
- [x] ~~`confidence`~~~~ поле в primary~~ **Resolved 2026-05-29, пересмотрено 2026-06-01:** в контракте только число `audit.primary_score`; категории `high|medium|low` — необязательное производное с документированными порогами. См. [Schemas · индекс](../schemas/README.md).
- [x] ~~`value.unit`~~~~ vs ~~~~`value.ucum_canonical`~~ **Resolved 2026-05-29:** оба всегда, даже когда равны. См. [Schemas · индекс](../schemas/README.md).
## История изменений
- 2026-05-29 — initial draft, status Accepted для v1.0
- 2026-05-29 — 3 open questions resolved (см. [Schemas · индекс](../schemas/README.md))
