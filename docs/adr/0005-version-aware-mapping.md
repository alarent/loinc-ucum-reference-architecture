# ADR-0005 · Version-aware mapping as first-class
## Статус
**Accepted**, версия v1.0.
Дата: 2026-05-29.
Автор: Nikita Zaverach.
**Уровень внедрения:** L2
## Контекст
LOINC выпускает релиз дважды в год. Концепты могут получать статус *deprecated*, заменяться через `REPLACED_BY`, мержиться или менять scope. UCUM версионируется реже, но семантика канонических единиц тоже эволюционирует. Маппинг, выполненный на одной версии справочника, формально перестаёт быть корректным после обновления — но **результат лабораторного анализа immutable**: переписывать prior interpretation поверх исходных данных нельзя.
Констрейнт домена: lab value, однажды интерпретированный, не должен молча менять свой LOINC код или unit-канонизацию при обновлении reference layer. Потребитель должен иметь возможность retrieve тот же result детерминированно, в той же версии, в которой был сделан маппинг. При этом deprecated коды не должны бесконечно жить в активном использовании — нужен явный путь миграции.
## Проблема
Есть две крайности и обе ломают что-то важное:
1. **Eager remapping**: при выпуске новой версии LOINC reference layer автоматически переписывает все существующие результаты маппинга на новые коды. Это ломает immutability lab values, ломает потребителей ниже по потоку с pinned `mapping_id` и создаёт hidden state changes.
2. **Freeze on version**: pin маппинг к версии навсегда без пути обновления. Deprecated коды остаются в активном обращении, потребитель получает устаревшие interpretations, no path forward.
Нужно решение, которое сохраняет immutability конкретного результата маппинга и одновременно даёт явный механизм re-interpretation.
## Решение
Snapshot-versioned маппинг с explicit remap workflow.
1. **Каждый результат маппинга фиксирует свою version-fingerprint в audit и она immutable.** Поля `audit.loinc_version` (уже в ADR-0003) и `audit.ucum_version` (новое в v1.0) обязательны и не подлежат изменению после записи. Это identity конкретного mapping — retrieval по `mapping_id` всегда возвращает тот же result с теми же кодами.
2. **Reference layer не делает auto-remapping при обновлении версии справочника.** Релиз LOINC обновляет внутренние таблицы reference layer (включая `REPLACED_BY` relations), но не трогает уже выполненные результаты маппинга. Существующие записи остаются в своей snapshot-версии.
3. **Новые запросы используют текущую (latest) версию reference layer по умолчанию.** Версия конфигурируется глобально (`reference_layer.loinc_version`, `reference_layer.ucum_version`). Возможен pinned-режим (запрос явно указывает `loinc_version`), но дефолт — latest.
4. **Remap — отдельная явная операция, инициируемая потребителем.** Endpoint `/remap` принимает existing `mapping_id` (или batch), возвращает новый результат маппинга с новой version-fingerprint и новым `mapping_id`. Исходный result остаётся доступен. Потребитель сам решает, какой версией оперировать. Производный маппинг фиксирует `audit.remap_source` с исходным `mapping_id` для провенанса цепочки. Идемпотентность: повторный `/remap` с тем же `(source_mapping_id, loinc_version, ucum_version)` возвращает существующий derived mapping, а не создаёт новый — dedup-ключ стабилен.
5. **Deprecated-aware audit для долгоживущих маппингов.** При retrieval маппинга, выполненного на старой версии, в response включается `audit.current_version_status`:
	- `current` — код всё ещё активен в latest LOINC
	- `deprecated` — код помечен deprecated в latest, но не заменён
	- `replaced` — код заменён, `audit.replaced_by` указывает LOINC код-преемник
	- `removed` — код удалён (редкий кейс)
Этот флаг — *signal*, а не *override*. Result не меняется, но потребитель узнаёт, что mapping технически устарел и может запросить `/remap`.
### Audit-поля v1.0 (дополнение к ADR-0003)
```json
{
	"audit": {
		"loinc_version": "2.82",
		"ucum_version": "2.2",
		"mapping_timestamp": "2026-05-29T14:54:31Z",
		"current_version_status": "current",
		"replaced_by": null
	}
}
```
- `loinc_version`, `ucum_version`, `mapping_timestamp` — всегда присутствуют, immutable.
- `current_version_status` — вычисляется в момент retrieval (не в момент маппинга), отражает relation между snapshot version и latest version reference layer.
- `replaced_by` — заполняется только если `current_version_status = "replaced"`.
- `remap_source` — nullable, присутствует только в маппингах, созданных через `/remap`; для фреш маппинга = `null`. Содержит `mapping_id` исходного маппинга.
**Snapshot vs computed audit-поля.** Audit разделяется на две категории по семантике обновлений:
- *Snapshot* (immutable, фиксируется в момент маппинга): `loinc_version`, `ucum_version`, `mapping_timestamp`, `primary_score`, `top_score_by_resolver`, `score_delta_vs_top`, `rule_applied`, `rule_overrode_score`, `method_alias_group`, `ucum_conversion`, `candidates_returned`, `policy_version`, `resolver_version`, `remap_source`. (Контекст не входит в audit: его состояние живёт в отдельном `context`-объекте envelope — per-field значение / `null` / `not_applicable` + `context_completeness`, per ADR-0004.)
- *Computed* (вычисляется при retrieval, отражает текущее состояние reference layer): `current_version_status`, `replaced_by`.
В JSON output поля не сгруппированы вложенно для backward compatibility, но в schema documentation должны быть явно помечены.
## Альтернативы (отклонённые)
**А1. Eager migration on LOINC release.** Reference layer при обновлении версии auto-rewrite всех existing маппингов на новые коды (через `REPLACED_BY`). Отклонено: ломает immutability lab values (исходная interpretation не должна перезаписываться), ломает потребителей ниже по потоку с pinned `mapping_id`, создаёт hidden state changes без явного триггера от потребителя.
**А2. Freeze mapping to original version forever.** Pin маппинг к версии без пути обновления, никаких `REPLACED_BY` relations не используется. Отклонено: deprecated коды остаются в активном обращении неопределённо долго, потребитель получает устаревшие interpretations без возможности явного re-interpretation, no migration path.
**А3. Hybrid: auto-update только для not-deprecated codes.** Reference layer auto-rewrite только тех маппингов, где код всё ещё active в новой версии (no-op в большинстве случаев), не трогает deprecated/replaced. Отклонено: complexity без выигрыша — большинство кейсов это no-op, оставшиеся всё равно требуют remap по инициативе потребителя; добавляет hidden state changes для подмножества маппингов и нарушает principle «reference layer не делает решений за потребителя» (из ADR-0001 А4).
## Последствия
Положительные:
- Lab value, интерпретированный однажды, остаётся retrievable с той же interpretation навсегда — критично для clinical audit trail и compliance.
- Потребитель контролирует, когда переинтерпретировать historic data — нет surprise migrations.
- `mapping_id` стабилен, потребитель может строить устойчивые references.
- `current_version_status` даёт прозрачность без принуждения — потребитель информирован, но не overridden.
Отрицательные / trade-offs:
- Storage: хранение version-fingerprint по каждому маппингу (loinc_version + ucum_version + timestamp). Стоимость низкая (3 строковых поля), но non-zero.
- Two retrieval modes: "as-mapped" (default, immutable) vs "current-status-aware" (includes `current_version_status`). Documentation должна явно различать.
- Long-tail deprecated codes: если потребитель никогда не вызывает `/remap`, deprecated коды живут в системе. Reference layer не enforce migration — это сознательный trade-off в пользу immutability.
- Remap не идемпотентен в strict sense: новый mapping_id, новая version-fingerprint, потенциально другой primary LOINC code. Потребитель должен явно решать, какой маппинг канонический для данной lab value.
## Открытые вопросы
Все открытые вопросы закрыты на housekeeping pass 2026-05-31:
- **Default retrieval mode** (`current_version_status` always vs by query param). Решено: always. Computation cheap (lookup в latest snapshot relations), signal critical (потребитель информирован без явного запроса).
- **Batch remap policy.** Phase 2. Мигрировано в [Roadmap · v1.1 / Phase 2 / ADR candidates](../roadmap.md). v1.0 — single-mapping `/remap` only; batch-семантика и dedup-логика для массовых операций требуют отдельной разработки.
- **Notification на LOINC release.** Решено: out of reference scope. Notification infrastructure — adopter-side concern (subscription / webhook платформы зависит от deployment). Reference exposes `current_version_status` через retrieval — этого достаточно для pull-based discovery. Если push-нотификация потребуется на платформенном уровне, она строится поверх reference, а не внутри.
- **UCUM version обязательно vs только при non-trivial conversion.** Решено: обязательно. Consistency с `loinc_version` важнее storage экономии (UCUM-snapshot string — десятки байт на mapping); audit предсказуем; нет conditional logic для потребителя.
- **`removed`**** vs ****`replaced`**** semantics.** Решено: отдельная семантика. LOINC формально не удаляет коды, но граничный случай возможен при structural revisions; явная категория `removed` гарантирует, что reference не сворачивает несуществующее в `deprecated` молча.
## Worked example reference
Example 05 (Version migration) — берёт маппинг, выполненный на LOINC 2.76 для кода, который был `REPLACED_BY` в LOINC 2.82. Показывает:
- Исходный маппинг retrievable как есть (immutable), `current_version_status = "replaced"`, `replaced_by` указывает преемника.
- Потребитель вызывает `/remap` с исходным `mapping_id`.
- Получает новый результат маппинга с новым `mapping_id`, новой version-fingerprint, primary LOINC code = преемник.
- Оба mapping_id остаются retrievable. Потребитель решает, какой использовать.
## История изменений
- 2026-05-29 — v1.0 accepted. Snapshot-versioned + explicit remap workflow. Lazy vs eager closed as «snapshot». Audit дополнен `ucum_version`, `mapping_timestamp`, `current_version_status`, `replaced_by`.
- 2026-05-29 — патч после worked example 05: добавлено поле `audit.remap_source` (провенанс remap-цепочки), зафиксирована идемпотентность `/remap` через dedup по `(source_mapping_id, loinc_version, ucum_version)`, в audit-секцию добавлено разделение snapshot vs computed.
- 2026-05-31 — housekeeping pass: закрыты все 5 open question (4 in-place как явные решения, 1 мигрирован в Phase 2 backlog)
- 2026-06-05 — Snapshot vs computed список выровнен с output.schema: убраны `context_fields_null`/`context_fields_not_applicable`/`context_completeness` (контекст теперь в отдельном `context`-объекте per ADR-0004); добавлены `method_alias_group`, `ucum_conversion`, `candidates_returned`, `policy_version`, `resolver_version`.
