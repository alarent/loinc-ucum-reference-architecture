# Уровни зрелости внедрения (capability levels)

> 🧭 Это карта поэтапного внедрения референс-архитектуры LOINC/UCUM. Один и тот же набор ADR, examples и schema можно читать двумя способами: как завершённую спецификацию v1.0 (это Level 4 целиком) и как лестницу уровней, по которой команда движется в своём темпе. Здесь описан второй способ.

## Зачем нужны уровни
Без них спецификация читается как «всё или ничего». Команда открывает 8 ADR, 7 examples и 3 schema и думает: «выглядит сильно, но нам это не построить за квартал — заведём свой абстрактный справочник». То есть полнота артефакта работает против его же цели — чтобы архитектуру реально внедряли.

Уровни превращают артефакт из просто спецификации в модель зрелости плюс спецификацию. Прямая параллель — Richardson Maturity Model для REST: уровни 0 → 1 → 2 → 3, где промежуточный уровень — это настоящая точка остановки, а не «недоделанный следующий». На своём уровне команда может стоять годами без ощущения недоделки.

Второй смысл — переговорный. С полной спецификацией на руках разговор про границы внедрения всегда болезненный. С уровнями появляется развязка: «method aliasing — это Level 4; давайте сделаем Level 1 за квартал и через два квартала оценим, нужен ли Level 4 вообще». Это меняет характер разговора с «продаю архитектуру» на «продаю поэтапный путь» — второе продаётся радикально легче.

## Матрица уровней
Ось чтения простая: смотришь на свою текущую боль (паттерн отказа) и видишь, на каком уровне она снимается. Колонка «точки расширения» показывает, как уровень готовит почву для следующего без переписывания контракта.

<table fit-page-width="true" header-row="true">
<tr>
<td>**Уровень**</td>
<td>**Какой паттерн отказа закрывает**</td>
<td>**Ключевые механизмы (ADR)**</td>
<td>**Точки расширения**</td>
<td>**Что сознательно НЕ решает**</td>
</tr>
<tr>
<td>**L0 — Базовая боль** *(стартовое состояние, не уровень внедрения)*</td>
<td>Ничего. Это зеркало для самооценки: свободно-строковые единицы, маппинг по ситуации в коде, нет версионирования</td>
<td>—</td>
<td>—</td>
<td>Всё. Точка отсчёта, не уровень для построения</td>
</tr>
<tr>
<td>**L1 — UCUM + LOINC на горячих биомаркерах**</td>
<td>Класс багов «значение в свободной строке». Cross-system обмен на \~80% трафика</td>
<td>Канонизация единиц на топ-30 биомаркерах (\>80% трафика); LOINC-резолвер с `confidence` и `alternatives[]`. Минимальный audit: `{loinc_code, ucum_unit, source_format, mapped_at}`</td>
<td>`audit` как расширяемый объект; `alternatives[]` без верхней границы; `additionalProperties: false` на верхнем уровне, но `additionalProperties: true` в namespace `audit.extensions`</td>
<td>Version drift, конфликты нескольких LOINC, семантический контекст, qualitative-результаты</td>
</tr>
<tr>
<td>**L2 — Version-aware + приоритетная политика**</td>
<td>Дрейф версий во времени; недетерминированный выбор при конфликте кандидатов. Audit готов к нормативной проверке</td>
<td>ADR-0003 + ADR-0005. Snapshot-версионирование, явный `/remap`, приоритетная политика в YAML. Audit дополняется до `{loinc_version, ucum_version, mapping_timestamp, policy_applied, top_score_by_resolver, score_delta_vs_top}`</td>
<td>Политика YAML расширяется без рекомпиляции; `policy_applied` — string, не enum; `replaced_by` допускает null</td>
<td>Семантический контекст пробы и метода; qualitative-результаты</td>
</tr>
<tr>
<td>**L3 — Семантический контекст + ветка qualitative**</td>
<td>Клинически неверная интерпретация пограничных случаев (ферритин и натощак, anti-HBs «отрицательно»)</td>
<td>ADR-0004 + ADR-0002 (с qualitative-патчем). Context-поля обязательны (`sample`, `method`, `fasting`, `patient_state` со значениями null / `not_applicable`); qualitative в обход UCUM</td>
<td>Форма контекста расширяется только добавлением новых полей; канонизация qualitative — отдельная заменяемая таблица</td>
<td>Сопоставление по методам (method aliasing); персистентность отклонённых маппингов</td>
</tr>
<tr>
<td>**L4 — Method aliasing + полный контракт API** *(= текущая reference architecture v1.0)*</td>
<td>Дубли LOINC-кодов по методу; единый предсказуемый контракт ответа и для success, и для rejection</td>
<td>ADR-0006 + ADR-0007. `audit.method_alias_group`, единый envelope с `oneOf`-дискриминатором, персистентные отклонённые маппинги</td>
<td>Hash-функция alias-group подключаемая; схема envelope версионируется независимо от snapshot-ов LOINC/UCUM</td>
<td>Substance-aware конверсия (mass↔molar); вторичные терминологии</td>
</tr>
<tr>
<td>**L5 — Phase 2** *(куда расти за пределы v1.0)*</td>
<td>Кросс-размерные конверсии (mass↔molar) и второй слой терминологии</td>
<td>Substance-aware конверсия; вторичный слой SNOMED CT</td>
<td>—</td>
<td>За границами reference v1.0 — отдельный roadmap</td>
</tr>
</table>

> ⚖️ **L0 и L5 асимметричны остальным.** L0 ничего не закрывает — это диагностика текущего состояния. L5 уходит за пределы v1.0 — это направление роста. Их не следует воспринимать как такие же ступени внедрения, как L1–L4.

## Карта артефактов по уровням
Эталонная сводка: какой ADR/example на каком уровне активируется. Именно на эту таблицу указывают бейджи «Уровень внедрения» в шапках артефактов. Уровень example — по правилу «где сценарий впервые становится обрабатываемым».

<table fit-page-width="true" header-row="true">
<tr>
<td>**Артефакт**</td>
<td>**Уровень**</td>
<td>**Примечание**</td>
</tr>
<tr>
<td>[ADR-0001 · LOINC as primary identifier](adr/0001-loinc-as-primary-identifier.md)</td>
<td>L1</td>
<td>LOINC как первичный идентификатор</td>
</tr>
<tr>
<td>[ADR-0002 · UCUM mandatory preprocessing + reject on failure](adr/0002-ucum-mandatory-preprocessing.md)</td>
<td>L1</td>
<td>канонизация UCUM; qualitative-ветка — на L3</td>
</tr>
<tr>
<td>[ADR-0003 · Explicit priority policy + alternatives in output](adr/0003-explicit-priority-policy.md)</td>
<td>L2</td>
<td>приоритетная политика + видимость score</td>
</tr>
<tr>
<td>[ADR-0004 · Semantic context mandatory](adr/0004-semantic-context-mandatory.md)</td>
<td>L3</td>
<td>семантический контекст пробы и метода</td>
</tr>
<tr>
<td>[ADR-0005 · Version-aware mapping](adr/0005-version-aware-mapping.md)</td>
<td>L2</td>
<td>snapshot-версии + explicit remap</td>
</tr>
<tr>
<td>[ADR-0006 · API contract + rejection envelope](adr/0006-api-contract-rejection-envelope.md)</td>
<td>L4</td>
<td>единый API-контракт + rejection envelope</td>
</tr>
<tr>
<td>[ADR-0007 · Method aliasing](adr/0007-method-aliasing.md)</td>
<td>L4</td>
<td>method aliasing</td>
</tr>
<tr>
<td>[ADR-0008 · Capability gating (nullable fields)](adr/0008-capability-gating.md)</td>
<td>сквозной</td>
<td>механизм capability gating для L1–L4</td>
</tr>
<tr>
<td>[00 · Input format](../examples/00-input-format.md)</td>
<td>L1</td>
<td>контракт входа / preconditions</td>
</tr>
<tr>
<td>[01 · Ferritin serum happy path](../examples/01-ferritin-serum-happy-path.md)</td>
<td>L1</td>
<td>базовый happy path</td>
</tr>
<tr>
<td>[02 · Glucose context matters](../examples/02-glucose-context-matters.md)</td>
<td>L3</td>
<td>контекст меняет выбор LOINC (fasting)</td>
</tr>
<tr>
<td>[03 · Multiple LOINC conflict](../examples/03-multiple-loinc-conflict.md)</td>
<td>L2</td>
<td>конфликт LOINC, override по политике</td>
</tr>
<tr>
<td>[04 · UCUM rejection](../examples/04-ucum-rejection.md)</td>
<td>L1</td>
<td>reject по UCUM; полная форма rejection-envelope — ADR-0006/L4</td>
</tr>
<tr>
<td>[05 · Version migration](../examples/05-version-migration.md)</td>
<td>L2</td>
<td>snapshot-версии + remap-цикл</td>
</tr>
<tr>
<td>[06 · UCUM canonical conversion](../examples/06-ucum-canonical-conversion.md)</td>
<td>L1</td>
<td>конверсия единиц; mass↔molar — на L5</td>
</tr>
</table>

## Принципы, без которых модель — фикция
1. **Только добавление между уровнями (additive-only).** Переход L2 → L3 не переписывает форму audit, а только добавляет поля. Любой breaking change хотя бы в одном месте делает поэтапное внедрение невозможным на практике.
2. **Точки расширения — в schema, а не только в коде.** Namespace `audit.extensions`, `alternatives[]` без верхней границы, `policy_applied` как string, а не enum. Частично это уже есть в текущих schema (nullable-поля в audit), но это нужно зафиксировать как намеренное design-решение, а не как недоопределённость.
3. **Матрица возможностей.** Таблица «уровень × закрываемый паттерн отказа». Именно она отвечает на вопрос «куда расти»: пользователь смотрит на свою боль и видит уровень, на котором она снимается.
4. **Каждый уровень — настоящая точка остановки.** Не «MVP, который надо доделать», а полностью рабочий слой для своего класса задач. Для этого по каждому уровню явно описано, что он сознательно не решает — и почему это нормально.

## Риск: «решает референс-слой» против выбора уровня
Есть реальный конфликт с принципом «решения принимает референс-слой». Если на L1 нет method aliasing, а на L4 есть, потребитель должен понимать, на каком уровне работает слой. Три варианта развязки:

<table fit-page-width="true" header-row="true">
<tr>
<td>**Подход**</td>
<td>**Цена**</td>
</tr>
<tr>
<td>Breaking change при повышении уровня</td>
<td>Нарушает additive-only — поэтапное внедрение становится фикцией</td>
</tr>
<tr>
<td>Runtime-обнаружение возможностей (`GET /v1/capabilities`)</td>
<td>Новая сложность контракта, но честно</td>
</tr>
<tr>
<td>Schema всегда полная (v1.0), поля nullable пока уровень не включён</td>
<td>Чистый вариант, совместим с текущим `output.schema.json`, но требует явного решения</td>
</tr>
</table>

> ✅ **Выбран третий вариант.** `output.schema.json` уже допускает null на части audit-полей (`ucum_conversion`, `remap_source`, `replaced_by`, `rule_applied`), и для них совместимость «бесплатная». Для strict-полей (`loinc_version`, `method_alias_group` и т. п.) gating выражается conditional-валидацией по `audit.capability_level` — детали в ADR-0008. Но это нужно явно зафиксировать как design intent, иначе через год возникнет вопрос «почему `method_alias_group` в schema, если мы не на L4», и ответить будет нечем. Заодно это ретроактивно объясняет nullability-решения, которые сейчас выглядят как простая осторожность, а на деле — это capability gating.

## Два взгляда на один артефакт
Уровни не размывают сообщение о полноте v1.0, если читать артефакт как два взгляда, а не два конкурирующих формата:
- **Взгляд портфолио:** reference architecture v1.0 = полный набор возможностей (Level 4 целиком). Сообщение «дизайн v1.0 завершён» сохраняется.
- **Взгляд внедрения:** capability levels = поэтапный путь к v1.0.

Один и тот же набор ADR, examples и schema, две разные стратегии чтения. README остаётся единой точкой входа; в портфолио остаётся «дизайн v1.0 завершён»; а для внедряющих появляется отдельный документ.

## Где это в репозитории
Не в README (раздуется) и не в `architecture.md` (там описание самой архитектуры, а не путь к ней). Место — отдельный файл `docs/capability-levels.md` с короткой ссылкой из README в разделе «Как использовать». Документ можно делиться отдельно: внедряющие читают только его, без полной спецификации. ADR и examples ссылаются на свой уровень видимым бейджем «Уровень внедрения: LN» в шапке, а эталонный маппинг сведён в таблицу «Карта артефактов по уровням» ниже в этом документе.

## Открытые вопросы
- [x] Зафиксировать nullable-as-design-intent отдельным ADR (capability gating) — оформлено как [ADR-0008 · Capability gating (nullable fields)](adr/0008-capability-gating.md).
- [x] Решить механику ссылок ADR/examples на уровни — выбран гибрид: видимый бейдж «Уровень внедрения: LN» в шапке каждого ADR и example + эталонная таблица «артефакт → уровень» в этом документе (раздел «Карта артефактов по уровням»). HTML-комментарии отвергнуты — не рендерятся на GitHub. (2026-06-05)
- [x] Определить, где в README встаёт ссылка на этот документ — добавлена в раздел «Как использовать» (2026-06-05).
