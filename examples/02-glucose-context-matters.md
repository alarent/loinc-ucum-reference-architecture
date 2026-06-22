# Worked example 02 · Glucose context matters
**Уровень внедрения:** L3
## Что демонстрирует
Один и тот же аналит (glucose) с одним и тем же числовым значением маппится в разные LOINC-коды в зависимости от `fasting_status`. Сдвигается и клиническая интерпретация (5.5 mmol/L на тощак — норма; 5.5 mmol/L random — тоже норма, но по другой референсной рамке). Практический смысл ADR-0004.
Пример выбран с `fasting_status = null` — canonical «источник явно не знает». Показывает graceful degradation и audit-сигнал `context_completeness="partial"`. В контрасте ниже — варианты fasting/random со специфичными LOINC.
## Связанные ADR
- ADR-0001 (LOINC первичный идентификатор)
- ADR-0002 (UCUM mandatory) — `mmol/L` парсится valid
- ADR-0003 (priority policy + alternatives)
- **ADR-0004 (semantic context mandatory)** — центральный ADR для этого example
## Input
```json
{
  "lab": "lab-orchestrator-v3",
  "date": "2026-05-15",
  "type": "сыворотка крови",
  "biomarkers": [
    {
      "raw_name": "Глюкоза",
      "raw_value": "5.5",
      "raw_unit": "ммоль/л",
      "raw_ref": "3.9–6.1",
      "raw_comment": null
    }
  ]
}
```
Ключевое: ядро не может вывести `fasting_status` из `type` («сыворотка крови», без маркера натощак) и пустого `raw_comment` → `fasting_status = null` (ADR-0004, п. 3 — «ядро не смогло определить»). Это валидное состояние, а не ошибка.
## Шаги обработки
### 1. Validation входа
Schema validation по input.schema.json (`lab`, `date`, `type`, `biomarkers[].raw_*`). Pass. Контекст на входе не передаётся — его выведет ядро.
### 2. Входной адаптер — локализация
`raw_name="Глюкоза"` → internal canonical `"Glucose"` (внутренняя каноническая форма English, см. architecture).
`raw_unit="ммоль/л"` → транслитерация → `"mmol/L"`.
### 3. UCUM preprocessing (ADR-0002)
`mmol/L` — valid UCUM. Canonical форма совпадает. Pass.
### 4. LOINC resolution (ADR-0001)
Resolver возвращает candidates:
```json
[
  {"loinc_code": "14749-6", "score": 0.91, "display": "Glucose [Moles/volume] in Serum or Plasma"},
  {"loinc_code": "1547-9",  "score": 0.87, "display": "Fasting glucose [Moles/volume] in Serum or Plasma"},
  {"loinc_code": "15074-8", "score": 0.62, "display": "Glucose [Moles/volume] in Blood"}
]
```
### 5. Priority policy (ADR-0003) с учётом context (ADR-0004)
Правило `prefer_fasting_specific_loinc_when_fasting_known`:
> Если `context.fasting_status` ∈ \{"fasting", "random", "post_prandial"\} И среди candidates есть LOINC с соответствующим fasting-axis → выбрать его. Иначе — выбрать top по score.
`fasting_status=null` → правило не срабатывает → fallback на score-based → primary = 14749-6 (generic).
`rule_applied = null` (выбор по score), `rule_overrode_score = false`.
### 6. Context completeness (ADR-0004)
Контекст-полей 3 (sample_type, method, fasting_status), null в 2 (method, fasting_status) → `context.context_completeness = "partial"`.
## Output
```json
{
  "status": "success",
  "mapping_id": "map_2026-05-15_c7e1f042",
  "primary": {
    "system": "LOINC",
    "code": "14749-6",
    "display": "Glucose [Moles/volume] in Serum or Plasma"
  },
  "value": {
    "numeric": 5.5,
    "string": null,
    "unit": "mmol/L",
    "ucum_canonical": "mmol/L"
  },
  "context": {
    "sample_type": "serum",
    "fasting_status": null,
    "method": null,
    "context_completeness": "partial"
  },
  "alternatives": [
    {"system": "LOINC", "code": "1547-9", "display": "Fasting glucose [Moles/volume] in Serum or Plasma", "score": 0.87},
    {"system": "LOINC", "code": "15074-8", "display": "Glucose [Moles/volume] in Blood", "score": 0.62}
  ],
  "input_echo": {
    "raw_name": "Глюкоза",
    "raw_value": "5.5",
    "raw_unit": "ммоль/л",
    "raw_ref": "3.9–6.1",
    "raw_comment": null
  },
  "audit": {
    "loinc_version": "2.82",
    "ucum_version": "2.2",
    "mapping_timestamp": "2026-05-15T11:43:00Z",
    "primary_score": 0.91,
    "top_score_by_resolver": 0.91,
    "score_delta_vs_top": 0,
    "rule_applied": null,
    "rule_overrode_score": false,
    "method_alias_group": "loinc:component:d4e5f6",
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
## Контраст по fasting_status
Как меняется output для того же входа с варьируемым `fasting_status`:
<table fit-page-width="true" header-row="true">
<tr>
<td>`fasting_status`</td>
<td>Primary LOINC</td>
<td>`rule_applied`</td>
<td>`completeness`</td>
<td>Клинический смысл</td>
</tr>
<tr>
<td>`null` (текущий)</td>
<td>14749-6 generic</td>
<td>null (по score)</td>
<td>partial</td>
<td>Референсные рамки неприменимы ниже по потоку</td>
</tr>
<tr>
<td>`"fasting"`</td>
<td>**1547-9** fasting</td>
<td>не null (rule-based)</td>
<td>full</td>
<td>5.5 mmol/L на тощак — норма (3.9–5.5)</td>
</tr>
<tr>
<td>`"random"`</td>
<td>14749-6 generic \*</td>
<td>не null (rule-based)</td>
<td>full</td>
<td>5.5 mmol/L random — норма, ниже порога диабета</td>
</tr>
<tr>
<td>`"post_prandial"`</td>
<td>14771-0 (mass/vol) \*</td>
<td>не null (rule-based)</td>
<td>full</td>
<td>Это же 5.5 — норма через 2ч после еды (&lt;7.8)</td>
</tr>
</table>
\* Открытые вопросы LOINC маппинга в разделе ниже.
## Комментарий
> ⚠️ **Главный маркер:** при `fasting_status=null` контракт выполняется успешно и возвращает осмысленный результат — но с явным сигналом `context_completeness="partial"`. Потребитель сам решает, достаточно ли этого для его use case (CDSS — нет, пациентский dashboard «показать значение» — да). Это и есть graceful degradation по ADR-0004.
Неочевидные детали:
- **`fasting_status=null`**** — это вывод ядра, а не пропуск источника.** Ядро не смогло определить fasting из `type`/`raw_comment` → `null` → success с `context_completeness="partial"`. «Отсутствующего поля» в этой модели нет: источник контекст не передаёт, поэтому единственная альтернатива значению — `null`. Это и есть семантика ADR-0004.
- **`primary_score`**** равен ****`top_score_by_resolver`** — появляется вопрос об экономии: включать ли `top_score_by_resolver` в audit всегда или только когда он отличен от primary_score? (Объединяется с открытым вопросом из example 03.)
- **`fasting_status="random"`** это осознанный выбор generic LOINC — семантически «random» и «без context» разные ситуации, но в LOINC явного «Random glucose» кода нет. Правило выбирает generic, но fasting_status="random" остаётся в context — потребитель видит оба сигнала.
## N/A vs null: отдельный сюжет (валидация открытого вопроса ADR-0004)
Для glucose `fasting_status=null` — дефицит информации. Но для **ferritin** `fasting_status` вообще не влияет на интерпретацию — это семантически иной случай.
<table fit-page-width="true" header-row="true">
<tr>
<td>Сценарий</td>
<td>Glucose</td>
<td>Ferritin</td>
</tr>
<tr>
<td>`fasting_status: null`</td>
<td>partial — реальный дефицит</td>
<td>partial — ложный сигнал (якобы дефицит, а фактически иррелевантно)</td>
</tr>
<tr>
<td>`fasting_status: "not_applicable"` (гипотетически)</td>
<td>Бессмысленно (всегда applicable)</td>
<td>full — источник явно знает, что поле иррелевантно</td>
</tr>
</table>
Без `"not_applicable"` как отдельного значения `context_completeness` для ferritin **всегда** будет partial, хотя фактически дефицита нет. Это реальная проблема для SLO по качеству интеграции — «средний completeness» по источнику будет искусственно занижен.
Альтернатива без «нового» значения: привязывать применимость context-полей к аналиту (per-LOINC-таблица «какие поля релевантны»). Но это быстро становится отдельным комплексом со своей maintenance-нагрузкой.
## Открытые вопросы из этого example
- [ ] **`"not_applicable"`**** как валидное значение context-полей.** Решить в ADR-0004 (уже в открытых вопросах, этот example делает выбор приоритетным — SLO зависит от этого).
- [ ] **LOINC для «Random glucose».** Нет явного кода «non-fasting glucose» в LOINC. Правило `prefer_fasting_specific` для random фактически сводится к generic. Это ok или нужен специальный маркер «правило совпало с generic из-за отсутствия специфичного LOINC»?
- [ ] **LOINC для post-prandial glucose в mmol/L.** Основные post-prandial-коды в mass/volume. Если source в mmol/L — нужна ли UCUM-конверсия mmol/L→mg/dL? (Свяжется с worked example 06.)
- [ ] **`top_score_by_resolver`**** всегда или только при override?** (Объединён с вопросом из example 03.)
## История изменений
- 2026-05-29 — initial draft, status Accepted для v1.0
