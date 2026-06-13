# Транслитерация UCUM: правила и тестовые кейсы
Этот документ фиксирует правила преобразования локализованных единиц измерения (преимущественно русскоязычных) в UCUM-валидные строки до передачи в UCUM-парсер. Сам UCUM не определяет транслитерацию или локализационные эквиваленты — это ответственность нормализационного слоя.
## Зачем отдельный документ
UCUM как стандарт оперирует фиксированным набором ASCII-токенов: `g`, `L`, `mol`, `U`, `mmol`, `umol`, `ug`, `IU` и т.д. На входе же `/v1/map` приходят строки вида «г/л», «ммоль/л», «мкг/мл», «МЕ/мл», «×10⁹/л». Без явной таблицы транслитерации UCUM-парсер отвергнет такие строки на этапе лексического разбора, и нормализация выродится в режим сплошных отказов.
Таблица решает три задачи:
- Делает явным набор поддерживаемых локализованных вариантов (что мы поддерживаем — а что reject)
- Фиксирует детерминированные правила (одна и та же исходная строка → один и тот же UCUM-токен)
- Снимает с UCUM-нормализатора заботу о локализации; он работает только с ASCII UCUM-токенами
## Архитектурное место
Транслитерация — это **стадия перед UCUM** внутри нормализационного ядра:
1. Источник передаёт `raw_unit` в исходной форме (см. [preconditions](preconditions.md))
2. **Транслитерация: ****`raw_unit`**** → UCUM-валидная ASCII-строка** ← этот документ
3. UCUM-парсинг и канонизация (`g/dL` → `g/L` с фактором, см. [ADR-0002 · UCUM mandatory preprocessing + reject on failure](adr/0002-ucum-mandatory-preprocessing.md))
4. Запись в `value.unit` + `value.ucum_canonical` + `audit.ucum_conversion`
Если шаг 2 не находит правила трансляции — это **не** UCUM-reject. Это промах транслитерации: возвращается `status: rejected` с кодом `UCUM_PARSE_FAILED` (`stage_failed: "ucum_parse"`; для неоднозначных форм — `UCUM_AMBIGUOUS_UNIT`, см. раздел «Неоднозначные случаи»). Диагностический флаг `raw_unit_unrecognized: true` кладётся в `audit.extensions` — ядро `audit_rejected` строгое (`additionalProperties: false`), нестандартные поля идут только в `extensions`. Для qualitative выходов unit просто игнорируется (см. ADR-0002 пункт 6).
## Базовая таблица (v1.0 draft)
Таблица не финализирована — приведённые правила это рабочий черновик для согласования. Окончательный набор — после первого пилот-прогона на реальных бланках.
### Префиксы (одиночные токены)
<table fit-page-width="true" header-row="true">
<tr>
<td>Исходное</td>
<td>UCUM</td>
<td>Комментарий</td>
</tr>
<tr>
<td>мк</td>
<td>u</td>
<td>micro → u (UCUM ASCII)</td>
</tr>
<tr>
<td>µ / μ</td>
<td>u</td>
<td>Unicode micro sign (две разные кодовые точки)</td>
</tr>
<tr>
<td>м</td>
<td>m</td>
<td>milli (только как префикс перед единицей; «м» отдельно — ambiguous, см. ниже)</td>
</tr>
<tr>
<td>н</td>
<td>n</td>
<td>nano</td>
</tr>
<tr>
<td>п</td>
<td>p</td>
<td>pico</td>
</tr>
<tr>
<td>к</td>
<td>k</td>
<td>kilo (как префикс)</td>
</tr>
</table>
### Базовые единицы
<table fit-page-width="true" header-row="true">
<tr>
<td>Исходное</td>
<td>UCUM</td>
<td>Комментарий</td>
</tr>
<tr>
<td>г</td>
<td>g</td>
<td>грамм</td>
</tr>
<tr>
<td>л</td>
<td>L</td>
<td>литр (UCUM использует L)</td>
</tr>
<tr>
<td>дл</td>
<td>dL</td>
<td>децилитр</td>
</tr>
<tr>
<td>мл</td>
<td>mL</td>
<td>миллилитр</td>
</tr>
<tr>
<td>моль</td>
<td>mol</td>
<td>моль</td>
</tr>
<tr>
<td>ммоль</td>
<td>mmol</td>
<td>миллимоль</td>
</tr>
<tr>
<td>мкмоль</td>
<td>umol</td>
<td>микромоль</td>
</tr>
<tr>
<td>нмоль</td>
<td>nmol</td>
<td>наномоль</td>
</tr>
<tr>
<td>пмоль</td>
<td>pmol</td>
<td>пикомоль</td>
</tr>
<tr>
<td>МЕ / ME / ЕД</td>
<td>\[iU\]</td>
<td>международная единица; UCUM использует bracket notation</td>
</tr>
<tr>
<td>%</td>
<td>%</td>
<td>UCUM поддерживает % напрямую</td>
</tr>
</table>
### Композитные конструкции
<table fit-page-width="true" header-row="true">
<tr>
<td>Исходное</td>
<td>UCUM</td>
<td>Комментарий</td>
</tr>
<tr>
<td>г/л</td>
<td>g/L</td>
<td>mass concentration</td>
</tr>
<tr>
<td>г/дл</td>
<td>g/dL</td>
<td>каноникализуется в g/L (см. [06 · UCUM canonical conversion](../examples/06-ucum-canonical-conversion.md))</td>
</tr>
<tr>
<td>мг/л</td>
<td>mg/L</td>
<td></td>
</tr>
<tr>
<td>мкг/мл</td>
<td>ug/mL</td>
<td></td>
</tr>
<tr>
<td>ммоль/л</td>
<td>mmol/L</td>
<td>molar concentration</td>
</tr>
<tr>
<td>мкмоль/л</td>
<td>umol/L</td>
<td></td>
</tr>
<tr>
<td>10\^9/л / ×10⁹/л / 10\*9/L</td>
<td>10\*9/L</td>
<td>UCUM использует `10*` нотацию для степеней; принимаем три формы Unicode</td>
</tr>
<tr>
<td>10\^12/л / 10\*12/L</td>
<td>10\*12/L</td>
<td>эритроциты</td>
</tr>
<tr>
<td>кл/мкл</td>
<td>\{cells\}/uL</td>
<td>UCUM annotation в фигурных скобках для безразмерных счётчиков</td>
</tr>
<tr>
<td>МЕ/мл</td>
<td>\[iU\]/mL</td>
<td></td>
</tr>
<tr>
<td>МЕ/л</td>
<td>\[iU\]/L</td>
<td></td>
</tr>
</table>
## Неоднозначные случаи (отдельная политика)
Эти случаи требуют либо контекста, либо явного отказа. v1.0 склоняется к отказу при неоднозначности, чтобы не маскировать OCR-проблемы.
- **«м» без следующего токена** — может быть «милли» (префикс) или «метр» (единица). В лабораторных наблюдениях «метр» нерелевантен; вероятно ошибка OCR (потерян суффикс типа «мл», «мг»). v1.0: отказ с `UCUM_AMBIGUOUS_UNIT`
- **«мл» vs «миллилитр» vs «ml»** — все три → `mL`. Смешение латиницы и кириллицы допустимо на стадии транслитерации
- **«U/L» vs «Ед/л»** — обе → `U/L` (UCUM `U` это enzyme unit, в FHIR/LOINC контексте обычно именно это имеется в виду). Если контекст явно про international units — нужно `[iU]/L` через «МЕ»
- **«г %» (грамм-процент)** — историческая нотация для `g/dL`. v1.0: распознаётся как `g/dL`, но с предупреждением в audit (устаревшая нотация)
- **Слитное написание без разделителя** («гл», «ммольл») — отказ. Без `/` граница между числителем и знаменателем не определена
- **Пробелы внутри композитной единицы** («г / л», «г /л») — нормализуются (пробелы вокруг `/` удаляются)
## Что мы НЕ делаем на этом этапе
- **Substance-aware conversion** (mass↔molar через молекулярную массу) — явное исключение из задачи v1.0, см. [scope](scope.md). Транслитерация работает только с unit shape, не с физико-химическими преобразованиями
- **Конверсия префиксов** (mg → g) — делается на следующем этапе UCUM-канонизации, не здесь
- **Регистр имеет значение** (UCUM регистрозависим: `L` ≠ `l`, `M` ≠ `m`). Транслитерация возвращает строки в канонической капитализации UCUM
- **Распознавание неканонического UCUM на входе** (например, `mol/l` со строчной l). Не наша задача — это передаётся в UCUM-парсер as-is, он отдаст error, и Ядро нормализации применит UCUM_PARSE_FAILED
## Тестовые кейсы (v1.0 fixtures)
Каждый кейс — пара `(input, expected_ucum)`. Используется для регрессионных тестов транслитерационной таблицы.
```json
{
  "transliteration_fixtures": [
    { "input": "г/л", "expected": "g/L" },
    { "input": "г/дл", "expected": "g/dL" },
    { "input": "мг/л", "expected": "mg/L" },
    { "input": "мкг/мл", "expected": "ug/mL" },
    { "input": "ммоль/л", "expected": "mmol/L" },
    { "input": "мкмоль/л", "expected": "umol/L" },
    { "input": "нмоль/л", "expected": "nmol/L" },
    { "input": "пмоль/л", "expected": "pmol/L" },
    { "input": "МЕ/мл", "expected": "[iU]/mL" },
    { "input": "МЕ/л", "expected": "[iU]/L" },
    { "input": "ЕД/л", "expected": "[iU]/L" },
    { "input": "10^9/л", "expected": "10*9/L" },
    { "input": "×10⁹/л", "expected": "10*9/L" },
    { "input": "10*9/L", "expected": "10*9/L" },
    { "input": "10^12/л", "expected": "10*12/L" },
    { "input": "кл/мкл", "expected": "{cells}/uL" },
    { "input": "%", "expected": "%" },
    { "input": "г %", "expected": "g/dL" },
    { "input": "г / л", "expected": "g/L" }
  ],
  "reject_fixtures": [
    { "input": "м", "reason": "UCUM_AMBIGUOUS_UNIT" },
    { "input": "гл", "reason": "UCUM_PARSE_FAILED — нет separator" },
    { "input": "ммольл", "reason": "UCUM_PARSE_FAILED — нет separator" }
  ]
}
```
## Открытые вопросы
- **«Ед» / «U» / «МЕ» — разрешение неоднозначности.** Текущий маппинг: «Ед/л» → `[iU]/L`. Но «Ед/л» исторически использовалось и для enzyme units. Нужен поиск контекста по LOINC (для LOINC-кода с COMPONENT enzyme — `U/L`; для immunoassay — `[iU]/L`). v1.0: статический маппинг, в audit предупреждение
- **Локализованные «литр».** «литр» / «liter» / «L» — все → `L`. Но «лит» (обрезанная OCR-форма) — отказ?
- **Юникодные дроби.** `½`, `¼` — встречаются ли в реальных бланках? v1.0: не поддерживаем
- **Поддержка английских единиц в русскоязычных бланках.** Helix / Invitro часто пишут `g/L` сразу — это работает. Но смешанные («g / л», `mL/мин`) — принимаем или отказываем?
- **Версионирование таблицы.** Транслитерация — часть нормализационного слоя, версионируется ли отдельно или вместе с UCUM snapshot? (Кандидат на патч ADR-0005)
## Связи
- [ADR-0002 · UCUM mandatory preprocessing + reject on failure](adr/0002-ucum-mandatory-preprocessing.md) — UCUM mandatory + reject политика (этот документ описывает pre-UCUM stage)
- [06 · UCUM canonical conversion](../examples/06-ucum-canonical-conversion.md) — пример канонизации `г/дл` → `g/L`
- [preconditions](preconditions.md) — upstream передаёт raw_unit в исходной форме
- [scope](scope.md) — substance-aware conversion вынесено в non-goals v1.0