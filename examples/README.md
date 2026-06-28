# Examples · индекс
Набор worked examples — конкретных прохождений данных через слой, на которых проверяются архитектурные решения.
## Принципы выбора examples
- Каждый example демонстрирует одно ключевое архитектурное решение в действии
- Все examples используют один и тот же контракт входа и выхода
- В каждом примере явно показано: вход → промежуточные шаги → выход
- В каждом примере один-два маркера, не пытаемся показать всё сразу
- **Бейдж «Уровень внедрения» = уровень сценария** (где он впервые становится обрабатываемым, см. [capability-levels](../docs/capability-levels.md)), а не форма envelope: примеры L1 показывают полный L4-конверт
## Финальный список v1.0
Статус: все 7 examples закрыты. Каждый example выявил 2–3 архитектурных вопроса, которые вошли в соответствующие ADR или ADR-патчи — см. [ADR · индекс](../docs/adr/README.md).
1. **00 · Input format** — каноническая форма входа от источника, без нормализации. Базовый референс; multi-biomarker check-up panel + failure scenario (PRECONDITION_FAILED). Валидирует preconditions contract; вызвал ADR-0006 patch (cardinality + preconditions envelope shape).
2. **01 · Ferritin serum happy path** — простой случай: один маркер, однозначный LOINC, корректный UCUM. Базовый шаблон для остальных; вызвал 3 Schemas resolved decisions (value XOR, unit + ucum_canonical, confidence split).
3. **02 · Glucose context matters** — глюкоза fasting vs random. Демонстрирует ADR-0004 (semantic context mandatory); вызвал not_applicable patch (third state).
4. **03 · Multiple LOINC conflict** — валидны несколько LOINC-кодов (разные методы). Демонстрирует ADR-0003 (priority policy + alternatives); вызвал ADR-0003 patch (score visibility) + ADR-0007 (method aliasing).
5. **04 · UCUM rejection** — единица, которую невозможно привести к UCUM. Демонстрирует ADR-0002 reject-путь; вызвал ADR-0006 (unified envelope с oneOf-дискриминатором).
6. **05 · Version migration** — cortisol illustrative, маппинг на LOINC@2.76 → после апгрейда поведение изменилось. Демонстрирует ADR-0005; вызвал ADR-0005 patch (`audit.remap_source`, snapshot vs computed split, /remap idempotency dedup-key).
7. **06 · UCUM canonical conversion** — альбумин `г/дл` → `g/L` (coefficient 10). Демонстрирует ADR-0002 convert-путь; вызвал `audit.ucum_conversion` (Schemas resolved decision), mass↔molar исключение из задачи в [scope](../docs/scope.md), выделение ucum-transliteration как отдельного документа.
## Что НЕ войдёт в examples v1.0
- Реальные клинические интерпретации
- Случаи, требующие внешних reference ranges
- Microbiology results (отдельное пространство sample types, v1.1)
- OMOP-выгрузка (v1.1)
- mass↔molar conversion examples (Phase 2, ADR-0009 candidate)
## Формат файла example
Каждый example — markdown-файл с разделами:
- **Что демонстрирует:** одна строка
- **Связанные ADR:** список
- **Input:** JSON
- **Шаги обработки:** последовательные промежуточные состояния
- **Output:** JSON
- **Комментарий:** что в этом примере неочевидно или ценно
- [00 · Input format](00-input-format.md)
- [01 · Ferritin serum happy path](01-ferritin-serum-happy-path.md)
- [02 · Glucose context matters](02-glucose-context-matters.md)
- [03 · Multiple LOINC conflict](03-multiple-loinc-conflict.md)
- [04 · UCUM rejection](04-ucum-rejection.md)
- [05 · Version migration](05-version-migration.md)
- [06 · UCUM canonical conversion](06-ucum-canonical-conversion.md)
