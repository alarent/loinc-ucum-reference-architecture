# Worked example 04 · UCUM rejection
**Уровень внедрения:** L1
Иллюстративный сценарий: на вход приходит количественный результат с unit, который технически парсится как UCUM, но не имеет conversion path к каноническому виду для данного теста. Проверяет reject-путь ADR-0002 (`UCUM_NO_CONVERSION_PATH`) и контракт rejection envelope.
## Контекст и input
Лаборатория-агрегатор передаёт результат глюкозы натощак. OCR корректно вытянул численное значение и тип материала, но в поле единицы измерения попало только `ммоль` — знаменатель `/л` потерян на этапе парсинга бланка.
```json
{
  "lab": "lab-aggregator-04",
  "date": "2026-04-12",
  "type": "венозная плазма натощак",
  "biomarkers": [
    {
      "raw_name": "Глюкоза",
      "raw_value": "5.4",
      "raw_unit": "ммоль",
      "raw_ref": "3.9–6.1",
      "raw_comment": null
    }
  ]
}
```
## Pipeline
### Шаг 1 · Транслитерация unit (ADR-0002)
- Исходное `ммоль` → нормализованное `mmol`
- Транслитерация прошла, лексема валидна, переходим к UCUM-парсингу
### Шаг 2 · UCUM parse
- `mmol` — валидный UCUM unit (amount-of-substance, размерность `mol`)
- Парсинг **успешен**, `UCUM_PARSE_FAILED` не срабатывает
### Шаг 3 · Conversion path check
- Канонический unit для глюкозы в крови — `mmol/L` (concentration, размерность `mol/L`)
- Размерность входа `mol` ≠ размерность канона `mol/L`; конверсия из amount в concentration требует объёма, которого нет
- Срабатывает `UCUM_NO_CONVERSION_PATH` → mapping rejected, в output записывается rejection envelope
## Output (rejection envelope)
Конверт отклонения по схеме output.schema.json (rejected-ветка oneOf-дискриминатора `status`):
```json
{
  "status": "rejected",
  "mapping_id": "map_2026-04-12_glucose_4b1a",
  "error": {
    "code": "UCUM_NO_CONVERSION_PATH",
    "message": "Cannot convert 'mmol' (amount) to canonical 'mmol/L' (concentration) for test 14749-6",
    "stage_failed": "conversion_path_check"
  },
  "input_echo": {
    "raw_name": "Глюкоза",
    "raw_value": "5.4",
    "raw_unit": "ммоль",
    "raw_ref": "3.9–6.1",
    "raw_comment": null
  },
  "audit": {
    "loinc_version": "2.82",
    "ucum_version": "2.2",
    "mapping_timestamp": "2026-04-12T09:14:22Z",
    "ucum_conversion": null
  }
}
```
- `value`, `primary`, `context`, `alternatives` **отсутствуют** в rejected-ветке: при rejection нет валидного mapping для отображения
- `mapping_id` присутствует и в rejected-ветке (ADR-0006 rejection persistence) — отклонение получает stable id
- `error.stage_failed` фиксирует, на каком шаге pipeline сработал reject
- `audit.ucum_conversion` = `null`: conversion не выполнялась (path не найден)
- `input_echo` сохраняет все пять raw-полей для traceability и ручной correction на источнике
## Что валидирует в ADR-0002
- Reject-путь действительно отделим от нормализации: рантайм возвращает структурированную ошибку, а не «магическое» догадывание (которое предлагалось в одной из отброшенных альтернатив)
- Три кода (`UCUM_PARSE_FAILED`, `UCUM_NO_CONVERSION_PATH`, `UCUM_AMBIGUOUS_UNIT`) различаются по стадии pipeline: лексическая, размерностная, семантическая. Кейс выше — размерностный
- Транслитерация — отдельный шаг **до** UCUM-парсинга: `ммоль` не отвергается как «не-UCUM», ему даётся шанс через transliteration table
## Альтернативные сценарии rejection
### UCUM_PARSE_FAILED
- `raw_unit = "+++"` для количественного теста; не существует UCUM-лексемы, парсер падает на первом же шаге
- Транслитерация ничего не меняет (нет кириллицы), `UCUM_PARSE_FAILED` уходит в envelope с `stage_failed: "ucum_parse"`
### UCUM_AMBIGUOUS_UNIT
- `raw_unit = "U/mL"` для гормонального теста, где `U` без указания типа активности (IU? mIU? arbitrary units?)
- UCUM формально парсит `U` как единицу активности фермента, но для теста-потребителя это не разрешается без method-context
- Срабатывает `UCUM_AMBIGUOUS_UNIT` с `stage_failed: "ambiguity_resolution"`; candidate-интерпретации перечисляются в `error.message` (структурный список — кандидат на `audit.extensions`; ядро `error` строгое: code / message / stage_failed)
## Выявленные вопросы (новые)
- **Rejection envelope зафиксирован в Schemas.** Закрыто: ADR-0006 ввёл unified `output.schema.json` с `oneOf: [success_envelope, rejected_envelope]` по дискриминатору `status`
- **Локализация error.message.** Сейчас message сформулирован по-английски с inlined raw-данными. Для UI на русском нужен либо переводимый template-id (`error_code → message_template`), либо разделение `error.code` (machine-readable) и `error.message_i18n` (localized). Откладывается в v1.1
- **Rejected-результаты получают полноценный ****`mapping_id`****.** Закрыто: `output.schema.json` требует `mapping_id` в обеих ветках oneOf (ADR-0006 rejection persistence) — отклонение имеет stable id, его можно retrieved и при исправлении выше вызвать `/remap`
## История изменений
- 2026-05-30 — создан worked example 04 (UCUM rejection, scenario `UCUM_NO_CONVERSION_PATH`)
