# Worked example 03 · Multiple LOINC conflict
**Уровень внедрения:** L2
## Что демонстрирует
Расхождение статистического resolver и priority policy: top по score — один кандидат, побеждает другой благодаря правилу priority policy. `rule_overrode_score=true` и audit делает это наблюдаемым.
## Связанные ADR
- ADR-0001 (LOINC первичный идентификатор) — `alternatives[]` держит отвергнутых кандидатов, включая top по score
- ADR-0002 (UCUM mandatory) — `%` парсится как valid UCUM, конверсии нет
- **ADR-0003 (priority policy + alternatives)** — центральный ADR для этого example
- ADR-0004 (semantic context mandatory) — `context.method` были триггером срабатывания правила
## Кейс
HbA1c — хрестоматийный пример «один физический аналит, несколько валидных LOINC-кодов». Разные лабы используют разные методы (HPLC, immunoassay, IFCC-сертифицированный protocol, eAG-вычисление), и LOINC присваивает каждому отдельный код, поскольку это разные измерения на уровне 6-осевой модели (method — одна из 6 осей).
Источник прислал метод в `raw_comment` («Метод: HPLC»), ядро вывело `method=HPLC`. Статистический resolver выведёт вверх «генерический» LOINC для HbA1c (лексический матч на «Hemoglobin A1c» очевидный, method-варианты по лексике менее похожи). Priority policy должен перебить этот выбор.
## Input
```json
{
  "lab": "lab-orchestrator-v3",
  "date": "2026-05-15",
  "type": "цельная кровь",
  "biomarkers": [
    {
      "raw_name": "Hemoglobin A1c",
      "raw_value": "5.8",
      "raw_unit": "%",
      "raw_ref": "4.0–6.0",
      "raw_comment": "Метод: HPLC"
    }
  ]
}
```
## Шаги обработки
### 1. Validation входа
Schema validation по input.schema.json (`lab`, `date`, `type`, `biomarkers[].raw_*`). Pass. Метод приходит не структурно, а в `raw_comment` («Метод: HPLC») — ядро выведет его ниже.
### 2. UCUM preprocessing (ADR-0002)
`%` — valid UCUM (процент как безразмерная величина). Каноническая форма совпадает с исходной. Pass.
### 3. LOINC resolution (ADR-0001)
Resolver возвращает 5 candidates (top-N=5):
```json
[
  {"loinc_code": "4548-4",  "score": 0.92, "display": "Hemoglobin A1c/Hemoglobin.total in Blood"},
  {"loinc_code": "17856-6", "score": 0.83, "display": "Hemoglobin A1c/Hemoglobin.total in Blood by HPLC"},
  {"loinc_code": "41995-2", "score": 0.74, "display": "Hemoglobin A1c/Hemoglobin.total in Blood by IFCC protocol"},
  {"loinc_code": "17855-8", "score": 0.51, "display": "Hemoglobin A1c/Hemoglobin.total in Blood by calculation"},
  {"loinc_code": "71875-9", "score": 0.43, "display": "Hemoglobin A1c/Hemoglobin.total in Blood by Immunoassay"}
]
```
Ключевое наблюдение: **top по score = 4548-4 (генерический, без method)**. Лексически он самый похожий на «Hemoglobin A1c». Но он не использует method-ось — а это важная информация из источника, и в LOINC для этого есть отдельный код (17856-6).
### 4. Priority policy (ADR-0003) — срабатывает override
Применяется правило `prefer_method_specific_loinc_when_context_provides_method`:
> Если `context.method` заполнен и среди candidates есть LOINC, чья method-ось совпадает с context.method → выбрать его как primary, даже если score ниже top.
Matching: «HPLC» в `context.method` → LOINC `system=Bld, method=HPLC` → 17856-6. Primary = 17856-6 (score 0.83), несмотря на то что 4548-4 имеет score 0.92.
**`rule_overrode_score = true`** — явный сигнал в audit.
### 5. Semantic context attachment (ADR-0004)
Выведенный ядром контекст прикрепляется к выходу, включая `method=HPLC` — именно этот метод был триггером правила.
### 6. Output assembly
Ключевое: 4548-4 (top по score!) попадает в `alternatives[]` — не теряется, потребитель видит все candidates и может провести собственную валидацию выбора.
## Output
```json
{
  "status": "success",
  "mapping_id": "map_2026-05-15_9a2c4d11",
  "primary": {
    "system": "LOINC",
    "code": "17856-6",
    "display": "Hemoglobin A1c/Hemoglobin.total in Blood by HPLC"
  },
  "value": {
    "numeric": 5.8,
    "string": null,
    "unit": "%",
    "ucum_canonical": "%"
  },
  "context": {
    "sample_type": "blood",
    "fasting_status": null,
    "method": "HPLC",
    "context_completeness": "partial"
  },
  "alternatives": [
    {"system": "LOINC", "code": "4548-4", "display": "Hemoglobin A1c/Hemoglobin.total in Blood", "score": 0.92},
    {"system": "LOINC", "code": "41995-2", "display": "Hemoglobin A1c/Hemoglobin.total in Blood by IFCC protocol", "score": 0.74},
    {"system": "LOINC", "code": "17855-8", "display": "Hemoglobin A1c/Hemoglobin.total in Blood by calculation", "score": 0.51},
    {"system": "LOINC", "code": "71875-9", "display": "Hemoglobin A1c/Hemoglobin.total in Blood by Immunoassay", "score": 0.43}
  ],
  "input_echo": {
    "raw_name": "Hemoglobin A1c",
    "raw_value": "5.8",
    "raw_unit": "%",
    "raw_ref": "4.0–6.0",
    "raw_comment": "Метод: HPLC"
  },
  "audit": {
    "loinc_version": "2.82",
    "ucum_version": "2.2",
    "mapping_timestamp": "2026-05-15T08:31:00Z",
    "primary_score": 0.83,
    "top_score_by_resolver": 0.92,
    "score_delta_vs_top": -0.09,
    "rule_applied": "prefer_method_specific_loinc_when_context_provides_method",
    "rule_overrode_score": true,
    "method_alias_group": "loinc:component:1a2b3c",
    "ucum_conversion": null,
    "remap_source": null,
    "current_version_status": "current",
    "replaced_by": null,
    "candidates_returned": 5,
    "policy_version": "1.0.0",
    "resolver_version": "semantic-resolver-v1"
  }
}
```
## Комментарий
> ⚡ **Главный маркер conflict-кейса:** primary выбран со score 0.83, при том что top по score (0.92) отправился в alternatives. Это является **сознательным выбором, а не багом**: источник явно сообщил method=HPLC, и в LOINC для этого есть более точный код, чем «генерика». Audit делает это решение полностью прозрачным.
Парный контраст с example 01:
<table fit-page-width="true" header-row="true">
<tr>
<td>Аспект</td>
<td>Example 01 (happy path)</td>
<td>Example 03 (conflict)</td>
</tr>
<tr>
<td>`rule_applied`</td>
<td>не null (rule-based)</td>
<td>не null (rule-based)</td>
</tr>
<tr>
<td>`rule_overrode_score`</td>
<td>`false`</td>
<td>**`true`**</td>
</tr>
<tr>
<td>`primary_score`</td>
<td>0.94 (= top)</td>
<td>0.83 (ниже top)</td>
</tr>
<tr>
<td>`top_score_by_resolver`</td>
<td>0.94 (равно primary)</td>
<td>0.92 (выше primary)</td>
</tr>
<tr>
<td>Что в alternatives\[0\]</td>
<td>Кандидат #2 (слабый)</td>
<td>**Top по score!** (0.92)</td>
</tr>
</table>
Неочевидные детали:
- **`top_score_by_resolver`**** и ****`score_delta_vs_top`**** — новые поля в audit**, выявленные этим example. Когда `rule_overrode_score=true`, потребитель должен видеть размер override — 0.09 в данном случае невелик, но в других сценариях может быть больше 0.5 — это красный флаг. Это обратная связь в ADR-0003.
- **4548-4 (top по score) остался видимым в ****`alternatives[]`****.** Если клиницист при ручной проверке решит, что method-специфика в данном контексте не критична — он может использовать 4548-4 из alternatives без повторного прогона pipeline.
- **Правило «prefer_method_specific» — реалистичный кандидат в priority policy v1.0.** Он имеет ясный триггер (`context.method != null`) и ясное действие; это хороший шаблон для остальных правил.
- **Само матчинг «HPLC» → LOINC method axis** — отдельный вопрос. Для этого нужен доступ к LOINC parts (component, system, method и т. д.), либо fuzzy матчинг по display, либо отдельная маппинг-таблица method-aliases. Ниже в открытых вопросах.
- **Сценарий, где правило не сработает.** Если бы method="electrochemiluminescence" и в candidates не было бы LOINC с та��ой method-осью — правило не сработало бы, primary стал бы 4548-4 с `rule_applied=null` (выбор по score). Контракт и в этом случае возвращает осмысленный результат, просто без method-точности.
## Открытые вопросы из этого example
- [x] **~~Расширение audit-схемы.~~** **Resolved 2026-06-01:** `top_score_by_resolver` и `score_delta_vs_top` — обязательные поля `audit_success` в output.schema.json; туда же по ADR-0003 п.5 добавлены `candidates_returned`, `policy_version`, `resolver_version`.
- [ ] **Threshold для override.** При `score_delta_vs_top` ниже некоторого порога (например, -0.30) выводить ли warning в audit, или вообще переходить на score-based? Это уже было в ADR-0003 открытым вопросом, example делает его осязаемым.
- [ ] **Method matching: fuzzy string vs alias-таблица vs LOINC parts API.** «HPLC» в context и «by HPLC» в LOINC display — очевидный матч, но «immunoassay» в context и «by Immunoassay» или «enzyme immunoassay (EIA)» — уже сложнее. Решить в ADR-0003 или вынести в отдельный (ADR-0007 method aliasing).
- [ ] **`alternatives[]`**** отсортированы по score**, но не по «близости к primary». В этом example top score и primary — разные коды; alternatives\[0\] = top score выглядит правильно. Подтвердить.
## История изменений
- 2026-05-29 — initial draft, status Accepted для v1.0
