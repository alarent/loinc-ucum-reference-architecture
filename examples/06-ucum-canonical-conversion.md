# Worked example 06 · UCUM canonical conversion
**Уровень внедрения:** L1
Иллюстративный сценарий: biomarker проходит ADR-0002 convert-путь от единиц в российской конвенции к UCUM-canonical форме. Покрывает вторую половину ADR-0002 (worked example 04 покрыл reject-ветку): когда UCUM имеет conversion path, происходит deterministic conversion с factor ≠ 1.
## Контекст и input
Альбумин в российской лабораторной практике иногда репортится в г/дл (US-style), иногда в г/л. LOINC primary 1751-7 (Albumin \[Mass/volume\] in Serum or Plasma); UCUM canonical для этого кода — `g/L`. Пример требует перевода `г/дл` → `g/L` (coefficient 10).
```json
{
  "lab": "lab-aggregator-06",
  "date": "2026-05-12",
  "type": "сыворотка крови натощак",
  "biomarkers": [
    {
      "raw_name": "Альбумин",
      "raw_value": "4.2",
      "raw_unit": "г/дл",
      "raw_ref": "3.5–5.2",
      "raw_comment": null
    }
  ]
}
```
## Pipeline
### Шаг 1 · Preconditions
- Pass (input.schema валиден, см. example 00). Биомаркер передан в Ядро нормализации
### Шаг 2 · Транслитерация UCUM (ADR-0002)
- `"г/дл"` → `"g/dL"` (преобразование кириллических обозначений массы/объёма в UCUM-стандартные)
- Транслитерационная таблица — артефакт реф-слоя (открытый вопрос ADR-0002: где живёт формально)
### Шаг 3 · LOINC resolution (ADR-0001 + ADR-0003)
- raw_name `"Альбумин"` + type `"сыворотка крови"` → primary LOINC 1751-7 (Albumin \[Mass/volume\] in Serum or Plasma)
- `primary_score` 0.95, резолвер не вызвал priority-rule
### Шаг 4 · UCUM conversion path detection
- LOINC 1751-7 canonical UCUM из реф-словаря: `g/L`
- Input UCUM (после transliteration): `g/dL`
- UCUM relation: 1 dL = 0.1 L → conversion factor `g/dL → g/L` = 10. Pure mass-prefix scaling, MW не требуется
### Шаг 5 · Conversion math
- 4.2 g/dL × 10 = 42.0 g/L
- UCUM-deterministic: input precision (`4.2` — две significant digits) сохраняется в output (`42.0`)
- Коэффициент и путь конверсии фиксируются в audit (см. ниже)
## Output (success envelope)
```json
{
  "status": "success",
  "mapping_id": "map_2026-05-12_albumin_8c4f",
  "primary": {
    "system": "LOINC",
    "code": "1751-7",
    "display": "Albumin [Mass/volume] in Serum or Plasma"
  },
  "value": {
    "numeric": 42.0,
    "string": null,
    "unit": "г/л",
    "ucum_canonical": "g/L"
  },
  "context": {
    "sample_type": "serum",
    "fasting_status": "fasting",
    "method": null,
    "context_completeness": "partial"
  },
  "alternatives": [],
  "input_echo": {
    "raw_name": "Альбумин",
    "raw_value": "4.2",
    "raw_unit": "г/дл",
    "raw_ref": "3.5–5.2",
    "raw_comment": null
  },
  "audit": {
    "loinc_version": "2.82",
    "ucum_version": "2.2",
    "mapping_timestamp": "2026-05-12T08:00:00Z",
    "primary_score": 0.95,
    "top_score_by_resolver": 0.95,
    "score_delta_vs_top": 0.0,
    "rule_applied": null,
    "rule_overrode_score": false,
    "method_alias_group": "loinc:component:9f8e7d",
    "ucum_conversion": {
      "input_unit": "g/dL",
      "canonical_unit": "g/L",
      "factor": 10,
      "path": ["mass_prefix_scaling"]
    },
    "remap_source": null,
    "current_version_status": "current",
    "replaced_by": null,
    "candidates_returned": 1,
    "policy_version": "1.0.0",
    "resolver_version": "semantic-resolver-v1"
  }
}
```
- `value.unit` сохраняет российскую конвенцию «г/л» (после conversion `г/дл` → `г/л`, transliteration обратно не выполняется для UI)
- `value.ucum_canonical` — strict UCUM form `g/L` для machine consumption
- `audit.ucum_conversion` — новое audit-поле с traceability conversion: input_unit, canonical_unit, factor, path. Поле существует только когда factor ≠ 1 (иначе null)
## Что валидирует
- ADR-0002 convert path (happy case) — успешная transliteration + canonical conversion с factor 10
- Pure mass-concentration domain: UCUM самодостаточен для mass-prefix scaling, внешняя substance-knowledge (MW) не задействована
- Соответствие primary LOINC и canonical UCUM из реф-словаря (1751-7 ↔ g/L)
- Exponential unit conversions (например, `тыс./мкл` → `10*9/L` для blood counts) работают аналогично через UCUM exponent algebra (coefficient типично = 1 по clinical convention)
## Выявленные вопросы (новые)
- **`audit.ucum_conversion`**** — новое поле, отсутствует в Schemas resolved decisions.** Для traceability conversion (дебаг расхождений между input и canonical) audit должен фиксировать input_unit, canonical_unit, factor, path. Нужно добавить в Schemas как 8-й resolved block. Не блокирует ADR
- **Mass↔molar conversion — явный scope-cut в v1.0.** Этот example работает в pure mass-concentration domain. Для биомаркеров типа креатинина (mg/dL ↔ μmol/L) или билирубина требуется molecular weight lookup, что вне scope ADR-0002 (UCUM-only, не substance-aware). Текущее поведение: mass-LOINC и molar-LOINC — разные `mapping_id`, не конвертимы внутри реф-слоя. Должно быть явно зафиксировано в [scope.md](../docs/scope.md) как сознательный отказ от substance-aware conversion layer; Phase 2 кандидат — ADR-0009 про substance-aware conversion
- **UCUM transliteration table — неформализованный артефакт.** ADR-0002 ввёл preprocessing (`мг/дл` → `mg/dL`), но не указал источник правил. Нужен рабочий артефакт `docs/ucum-transliteration.md` с явными правилами и тестовыми кейсами. Не ADR-level (имплементация), но необходимо для repo-completeness
## История изменений
- 2026-05-30 — создан worked example 06 (UCUM canonical conversion, альбумин г/дл → g/L coefficient 10)
- 2026-06-05 — output envelope приведён к полному канону output.schema: добавлен `value.string` и reproducibility-поля audit (candidates_returned, policy_version, resolver_version); method_alias_group приведён к hex-pattern `^loinc:component:[a-f0-9]+$`