# ADR-0007 · Method aliasing
- **Статус:** Accepted
- **Уровень внедрения:** L4
- **Дата:** 2026-05-30
- **Связанные ADR:** ADR-0003 (Priority policy), ADR-0005 (Version-aware mapping), ADR-0006 (API contract)
- **Связанные worked examples:** 03 (HbA1c с HPLC method override)
## Контекст
LOINC содержит множественные коды для одного и того же аналита, различающиеся только аналитическим методом — это family `Component / Property / Time / System / Scale / Method`, где Method-варианты дают разные коды. Example 03 (HbA1c) эмпирически показал: приоритет method-specific кода над генеричным решается через priority rule (ADR-0003), но это закрывает только selection-задачу.
Остаётся вторая задача: как потребителю агрегировать «HbA1c пациента независимо от метода» для cross-method analytics, если каждый mapping сидит на своём LOINC-коде. Без явного alias-понятия это придётся решать каждому потребителю самостоятельно — это прямое нарушение cross-cutting принципа «реф-слой принимает решения».
## Решение
### 1. Method-aware primary selection
- Когда input содержит method-marker (явный в raw_name, raw_comment или lab-type fields), primary = method-specific LOINC код
- Когда method unknown, primary = generic LOINC код для аналита (если в LOINC существует method-agnostic вариант)
- Реализация — через priority rule (ADR-0003); в audit фиксируется `rule_applied = "method_match"`, `rule_overrode_score` устанавливается, если generic имел выше text-similarity score
### 2. Alias group field в audit
- `audit.method_alias_group` (string, required) — канонический id группы method-variants для данного аналита
- Группа выводится из LOINC COMPONENT field equality: коды с одинаковым COMPONENT, но разными METHOD_TYP принадлежат одной alias group
- Group id формат: `loinc:component:{component_hash}`, где component_hash — deterministic hash от нормализованного COMPONENT-string'а
### 3. Snapshot-stability
- Alias group id фиксируется в момент mapping'а вместе с `audit.loinc_version` — он snapshot-versioned, как все audit-поля (ADR-0005)
- `/remap` пересчитывает group id с новым снапшотом. Если LOINC реорганизовал COMPONENT между версиями, новый mapping_id получит свой alias_group; это ожидаемо и документируется в [versioning.md](../versioning.md)
### 4. Cross-method retrieval — v1.0 в audit, endpoint в Phase 2
- v1.0: `audit.method_alias_group` доступен как read-only поле в success envelope (ADR-0006). Потребитель может фильтровать свой локальный хранилище по этому полю
- Phase 2: `GET /v1/mappings?alias_group={group_id}` — возвращает все mappings в группе через method-variants, с pagination
## Альтернативы
### Не различать method (всегда generic primary)
- **Отвергнуто:** теряется clinically significant info. HbA1c HPLC vs immunoassay могут давать систематически разные значения; ref ranges и interpretation method-specific. Снятие правила ADR-0003 для HPLC — явный регресс
### Внешний alias dictionary с ручным maintenance
- **Отвергнуто:** дублирует LOINC COMPONENT/METHOD_TYP. Ручной maintenance подвержен ошибкам и drift'у от LOINC. Деривация из LOINC даёт consistency бесплатно и snapshot-versioned
### Только method-specific код без alias group
- **Отвергнуто:** потребитель для cross-method analytics вынужден сам строить alias map. Прямое нарушение cross-cutting принципа
### Alias-group id как UUID
- **Отвергнуто:** UUID не deterministic, требует alias-group registry в базе. Hash от нормализованного COMPONENT даёт ту же стабильность pure-функционально, без storage
## Последствия
### Положительные
- Потребитель получает оба режима: method-aware primary (ADR-0003) и group id для cross-method aggregation
- Group id deterministically derived — реф-слою не нужен alias-group registry; любой потребитель со снапшотом LOINC может перевычислить то же самое
- Snapshot-versioned — alias-group эволюционирует консистентно с LOINC
### Отрицательные / trade-offs
- COMPONENT-equality hash может схлопывать коды с семантически близкими, но формально разными COMPONENT (граничный случай в LOINC: вариации нормализации string'а). Требует validation на снапшоте
- Если LOINC меняет COMPONENT для аналита, alias_group_id меняется между снапшотами; потребителям, которые индексируют по group_id, нужна политика реиндексации после /remap
## Связь с другими ADR
- **ADR-0003:** method-aware selection реализована через priority rule; этот ADR добавляет второй слой — alias group field
- **ADR-0005:** alias_group snapshot-versioned, как все audit-поля; /remap пересчитывает его
- **ADR-0006:** alias_group — часть success envelope в audit; cross-method endpoint — будущий расширение API в Phase 2
- **Worked example 03:** эмпирическая основа для method-aware selection
## Открытые вопросы
Все открытые вопросы закрыты на housekeeping pass 2026-05-31:
- **Hash function для COMPONENT id.** Решено: SHA-256 рекомендуемый default (deterministic, collision-resistant, standardized). Human-readable slug отвергнут — collision risk при нормализации, длина непредсказуема. Реализация может использовать первые 16 hex-символов SHA-256 для компактности.
- **LOINC меняет COMPONENT без REPLACED_BY** (граничный случай). Решено: документируется как известный граничный случай в `/remap` policy. Если COMPONENT для существующего кода меняется в новом снапшоте без REPLACED_BY, `alias_group_id` пересчитывается на стороне `/remap`; потребителю, индексирующему по group_id, требуется реиндексация после `/remap`. Это явный trade-off в пользу snapshot-consistency.
- **Cross-method endpoint shape** (`GET` с pagination vs `POST` batch). Phase 2 backlog ([Roadmap · v1.1 / Phase 2 / ADR candidates](../roadmap.md)).
- **Локализация COMPONENT field для UI.** v1.1 backlog ([Roadmap · v1.1 / Phase 2 / ADR candidates](../roadmap.md)), решается совместно с ADR-0001 localization-задачей `display_localized`.
## Changelog
- 2026-05-30 — создан ADR-0007 (Method aliasing) на основании finding'а из worked example 03
- 2026-05-31 — housekeeping pass: закрыты все 4 open question (2 in-place, 1 в Phase 2, 1 в v1.1 backlog)
