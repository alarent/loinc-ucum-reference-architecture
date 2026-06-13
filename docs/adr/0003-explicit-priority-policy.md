# ADR-0003 · Explicit priority policy + alternatives in output
## Метаданные
- **Номер:** 0003
- **Уровень внедрения:** L2
- **Статус:** Accepted (v1.0)
- **Дата:** 2026-05-29
- **Связанные ADR:** ADR-0001 (LOINC primary), ADR-0004 (Semantic context mandatory), ADR-0005 (Version-aware mapping)
- **Связанные документы:** [architecture](../architecture.md), [scope](../scope.md)
## Контекст
Резолвер LOINC — статистический инструмент. На один и тот же аналит с близкими контекстами он часто возвращает несколько кандидатов с близкими score'ами:
- Ferritin → serum (2276-4) vs plasma (20567-4) vs blood (24373-3) — разные sample types
- Glucose → fasting (1558-6) vs random (2345-7) vs 2h post-meal (1521-4) — разный clinical context
- HbA1c → несколько кодов с разными методами (IFCC vs NGSP)
Pipeline должен превратить недетерминированный список кандидатов в детерминированный output. На это решение действуют силы:
- системам ниже по потоку нужен один primary code для join'ов, агрегации, обучения моделей (см. ADR-0001)
- решение должно быть прозрачно: ошибка маппинга, обнаруженная клиницистом постфактум, должна быть отслеживаемой до конкретного правила или score
- resolver-модель будет меняться (BioLORD → следующая версия; lexical → semantic re-ranking); чистый score-based выбор делает поведение зависимым от версии модели — системы ниже по потоку видят «дрифт» без явного версионирования
- хардкод правил вида «ferritin → 2276-4» не масштабируется: правил будут сотни, они хрупкие, покрытие никогда не будет полным
- скрытие близких кандидатов внутри resolver = чёрный ящик: потребитель не знает, что выбор был «на грани», и не может пометить кейсы для ручной проверки
**Архитектурный вызов:** статистический resolver + детерминированный контракт = необходимо явное reconciliation layer, иначе непрозрачность накопится в проде.
## Решение
**Pipeline разделён на два слоя. Resolver возвращает массив кандидатов с score'ами; priority policy выбирает primary из массива по явным правилам. Все остальные кандидаты сохраняются в ****`alternatives[]`****. Применённое правило priority policy фиксируется в audit trail.**
Конкретика:
1. **Resolver output.** Resolver возвращает упорядоченный список candidates: каждый кандидат — `{loinc_code, score, display}`. Минимум 1 элемент, максимум — настраиваемый top-N (по умолчанию 5).
2. **Priority policy.** Декларативный артефакт (YAML), версионированный, лежит в репо. Правила вида «если `context.specimen=serum` и среди кандидатов есть код с `system=Ser` → выбрать его как primary». Правила применяются по порядку, первое подходящее побеждает.
3. **Primary selection.** Primary = первый кандидат, прошедший правило priority policy. Если ни одно правило не сработало → primary = top кандидат по score, и тогда `rule_applied = null` (выбор по score).
4. **Alternatives.** Поле `alternatives[]` — все остальные candidates (кроме primary), отсортированные по score. Сохраняются `loinc_code`, `score` и `display`.
5. **Audit trail.** Фиксируется: какое правило priority policy сработало (имя/ID правила), исходный score primary-кандидата (`primary_score`), top score возвращённый resolver'ом (`top_score_by_resolver`), дельта между ними (`score_delta_vs_top` = `primary_score − top_score_by_resolver`), общее число candidates от resolver, версия priority policy, версия resolver-модели. Поля `top_score_by_resolver` и `score_delta_vs_top` присутствуют **всегда** (а не только при override): консистентность audit-схемы важнее минимальной экономии. При `rule_applied=null` (выбор по score) они тривиально равны `primary_score` и `0` соответственно.
6. **Конфликт rule vs score.** Правило priority policy побеждает score. Если правило выбирает кандидата со score ниже top'а — это явный сигнал, что resolver и priority policy расходятся, и это видно в audit как `rule_applied` = id сработавшего правила, `rule_overrode_score = true`.
7. **Расширяемость audit (extension port).** Audit-trail устроен как гибрид: фиксированное обязательное ядро (см. п.5) плюс необязательный расширяемый bag `audit.extensions`. Ядро остаётся строгим (`additionalProperties: false`) — это гарантия интероперабельности между потребителями. Всё, что не входит в ядро, кладётся в `audit.extensions` (`additionalProperties: true`), так схема растёт без breaking changes. Правило промоушена: поле из `extensions`, на которое потребители начинают полагаться (несущее для интероперабельности), переносится в ядро отдельным ADR. Это механизм роста по уровням зрелости — порт фиксируется в v1.0, наполняется по мере необходимости.
## Альтернативы и почему отвергнуты
### А1. Только score, без priority policy
Resolver возвращает кандидатов, primary = top по score. Никаких правил поверх.
Отвергнуто: (а) непрозрачно — потребитель видит код, но не знает, почему именно его выбрали из нескольких близких; (б) поведение зависит от версии модели — смена модели сдвигает выбор, системы ниже по потоку видят «дрифт» без явного версионирования; (в) клиницист, обнаруживший ошибку маппинга, не может указать на правило, которое нужно исправить — только на «модель плохая», что не операционализируется.
### А2. Только rules engine, без score
Полный hand-crafted маппинг analyte → LOINC; статистический resolver вообще не нужен.
Отвергнуто: (а) LOINC \~95k кодов, ручное покрытие нереально; (б) названия тестов в источниках бесконечно вариативны (опечатки, аббревиатуры, локализованные формы), lexical rules не справятся; (в) даже частичный rules engine требует fallback для неизвестных случаев — то есть всё равно нужен resolver.
### А3. Скрытый выбор внутри resolver, наружу только primary
Resolver сам делает выбор; внешний контракт содержит только primary code, без `alternatives[]`.
Отвергнуто: (а) теряется информация о близких кандидатах — кейсы «на грани» неотличимы от уверенных кейсов; (б) потребитель не может построить QA-pipeline (например, флагать observations, где primary и второй кандидат имеют разницу score \< 0.05); (в) при смене resolver-модели нет способа сравнить старые и новые выборы для regression-валидации.
### А4. Возвращать только массив без primary
Контракт = массив candidates со score'ами; primary не выделяется, downstream сам выбирает.
Отвергнуто: (а) перекладывает решение на каждого потребителя — та же критика, что и co-primary в ADR-0001; (б) фрагментация выбора между системами ниже по потоку разрушает единство данных; (в) типичный CDSS-инженер не должен переоткрывать ��лгоритм приоритезации — это и есть работа референсного слоя.
## Последствия
### Положительные
- **Прозрачность и accountability.** Каждый primary-выбор объясним: либо «сработало правило X», либо «top по score». Ошибки маппинга чинятся в YAML, а не в коде модели.
- **Независимая эволюция.** Resolver-модель и priority policy версионируются отдельно. Можно обновить модель без переписывания правил и наоборот.
- **Близкие кандидаты не теряются.** `alternatives[]` позволяет системам ниже по потоку строить QA-pipelines, флагать пограничные случаи, делать ручную верификацию.
- **Воспроизводимость.** Версия priority policy + версия LOINC + версия resolver-модели фиксируются в audit → любой output можно воспроизвести и сравнить с историческим.
- **Путь миграции.** Когда правил priority policy накапливается достаточно, их можно использовать как ground truth для обучения / fine-tuning новых версий resolver-модели.
### Отрицательные и компромиссы
- **Двухступенчатость.** Pipeline сложнее в reasoning и debugging — нужно разделять «модель выбрала плохо» vs «правило приоритета сработало не так».
- **Поддержка priority policy.** YAML с правилами — отдельный артефакт, нуждается в review, тестировании, версионировании. На старте правил мало; они накапливаются как реакция на найденные ошибки.
- **Риск переобучения.** Каждое правило, добавленное в ответ на ошибку, может сломать другой кейс. Нужны regression-тесты — worked examples частично эту роль играют, но не полностью.
- **Связка с LOINC versioning.** При обновлении LOINC коды в правилах могут стать deprecated — связь с ADR-0005.
### Нейтральные
- Формат priority policy — YAML с pattern-matching по полям `context`. Императивный код (TS-функции) рассмотрен и отвергнут как менее аудируемый.
- Минимальный score для primary — нет жёсткого порога в v1.0. Если все score'ы низкие, primary всё равно выбирается, но с низким `primary_score` (см. open question). Слой не отклоняет по порогу — порог применяет потребитель.
- Правила могут жить в одном файле или быть разбиты по domain (chemistry / hematology / coagulation). В v1.0 — один файл; разбиение — по росту количества правил.
## Открытые вопросы
Все открытые вопросы закрыты на housekeeping pass 2026-05-31:
- **Confidence shape.** Решено (пересмотрено 2026-06-01) в [Schemas · индекс](../../schemas/README.md): контракт несёт только число `audit.primary_score` `[0,1]`. Категории `high|medium|low` в envelope не входят — это производное, которое потребитель при желании вычисляет из числа по документированным порогам (например, `high ≥ 0.9`). Число несёт больше информации и threshold'ится downstream; слой не обещает семантику каждой сотой доли.
- **Минимальный score для primary.** Решено: нет score threshold для reject в v1.0. Reject основан на UCUM/preconditions failures (ADR-0002, ADR-0006), не на низком score. Низкий score — confidence-сигнал для потребителя через `primary_score` audit field; решение о filter/escalation — ответственность потребителя.
- **Versioning priority policy.** Решено: SemVer, decoupled от LOINC version (consistent с ADR-0006 decoupling между API/data versioning и data snapshots).
- **Threshold для override «правило vs low score».** Мигрировано в validation tracker ([Roadmap · v1.1 / Phase 2 / ADR candidates](../roadmap.md)). Требует real-world data для определения безопасного threshold; v1.0 фиксирует  «правило всегда побеждает» (`rule_overrode_score=true` в audit), validation pass наблюдает кейсы.
- **Локализация правил.** Решено: общий файл с `locale` metadata в v1.0 (например, `locale: ru` для правил «Глюкоза натощак» → fasting). Разделение на отдельные файлы — implementation evolution при росте числа правил.
## Worked example связки
См. [Examples · индекс](../../examples/README.md):
- `01 Ferritin serum happy path` — happy path, правило priority policy и top по score совпадают (`rule_overrode_score=false`, `score_delta_vs_top=0`)
- `02 Glucose context matters` — `fasting_status=null` → выбор по score (`rule_applied=null`), `top_score_by_resolver` равен `primary_score`
- `03 Multiple LOINC conflict` — `rule_overrode_score=true`, `score_delta_vs_top=-0.09` визуализирует расхождение resolver и policy
## История изменений
- 2026-05-29 — initial draft, status Accepted для v1.0
- 2026-05-29 — patch audit-схемы: добавлены поля `top_score_by_resolver` и `score_delta_vs_top` (всегда присутствуют). Surfaced by worked examples 02 и 03.
- 2026-05-31 — housekeeping pass: закрыты все 5 open question (4 in-place как явные решения с rationale, 1 мигрирован в validation tracker)
- 2026-06-01 — добавлен extension port: гибридная audit-схема (строгое ядро + опциональный audit.extensions, additionalProperties: true) и правило промоушена поля в ядро через ADR (п.7)
