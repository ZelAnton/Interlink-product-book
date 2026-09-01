# 12. Открытые вопросы, допущения, риски и проверки

## 1. Назначение и правила реестра

Этот документ превращает неопределённости проекта в программу исследований и решений. Он не объявляет гипотезы возможностями продукта и не переоткрывает уже принятые ADR без новых данных.

Для каждого пункта явно разделены три слоя:

- **Факт** — подтверждённый контракт, реализованное поведение или наблюдение из источника с указанной границей применимости.
- **Допущение** — рабочая гипотеза, на которой можно временно продолжать проектирование, но нельзя строить внешнее обещание.
- **Предложение** — рекомендуемый способ проверки или решение-кандидат; оно становится контрактом только после принятого ADR, спецификации либо продуктового решения.

Поле **«этап решения»** означает последний безопасный gate, а не календарное обещание. Обозначения фаз Interlink и gates Alvatune приведены в [карте реализации, §5](02-current-state-and-roadmap.md) и [границе с Alvatune, §15](09-alvatune-and-agents.md). Сила источников и их короткие ID определены в [реестре источников](15-source-register.md).

Приоритеты:

- **P0** — без ответа нельзя безопасно зафиксировать следующий контракт или выпускать бизнес-свидетельство;
- **P1** — ответ нужен до соответствующего product/implementation gate;
- **P2** — можно отложить до измеримого масштаба или подтверждённого спроса.

Закрытие вопроса требует не только текста. В записи должны появиться: решение и владелец; ссылка на ADR/продуктовый протокол; fixture, benchmark, differential test или эксплуатационный отчёт; перечень изменённых допущений и рисков.

## 2. Нормативные вопросы Interlink

Следующие одиннадцать пунктов перенесены без смыслового расширения из нормативного списка [открытых вопросов `docs/12-decisions.md`](../../12-decisions.md#2-открытые-вопросы). Принятые ограничения вокруг них считаются фактами; открыта только указанная развилка.

### OQ-001. Компактный combined-профиль версионности

**Приоритет:** P2.

- **Факт:** действующая модель хранит Identity и Revision раздельно; compact `identity + revision` в одной таблице не является default и не входит в PoC. [07 §2](07-versioning-changes-snapshots.md#2-восемь-концептов), [IL-ADR, вопрос 1].
- **Допущение:** отдельные таблицы дают более устойчивую семантику и приемлемую цену даже для простых versioned types.
- **Предложение:** не вводить профиль до появления реального типа с измеренной высокой ценой JOIN/объектов БД и простой неизменной семантикой.
- **Почему важно:** новый профиль умножает DDL, codegen, migration, query и compatibility paths и может размыть единую модель версий ради локальной оптимизации.
- **Риск ошибочного решения:** преждевременное добавление создаст второй versioning dialect; слишком поздний отказ может оставить чрезмерную стоимость для массовых простых типов.
- **Кто подтверждает:** архитекторы Interlink, владелец DB/runtime и представитель первого крупного host deployment.
- **Закрывающее свидетельство:** сравнение раздельного и combined-профилей на одинаковом наборе типов и запросов: размер каталога/индексов, write/read latency, миграция identity/revision attributes, сложность generated API; ADR с порогом включения.
- **Этап решения:** после нагрузочной фазы 8, до добавления второго storage profile в production roadmap.

### OQ-002. Полиморфные endpoints связей

**Приоритет:** P1.

- **Факт:** конкретные typed endpoints обеспечивают FK-целостность; для полиморфной цели БД не может выразить обычный FK на несколько таблиц. Сейчас предполагаются application-enforced integrity и generated validation. [IL-META], [IL-ADR, вопрос 2].
- **Допущение:** для PoC и первого набора доменных моделей достаточно ограниченного числа полиморфных связей и fail-closed проверки writer-а.
- **Предложение:** измерить реальные схемы IPS/Alvatune; вводить per-domain registry только если он закрывает подтверждённые связи и не становится универсальной writable object table.
- **Почему важно:** полиморфизм нужен классификации, evidence и общим subject-связям, но общий registry может вернуть архитектуру к отвергнутому generic Record kernel.
- **Риск ошибочного решения:** слабая ссылочная целостность и «висячие» endpoints либо скрытый универсальный реестр с двойной идентичностью.
- **Кто подтверждает:** metamodel/runtime leads, владельцы Requirements/Evidence моделей Alvatune и DBA.
- **Закрывающее свидетельство:** inventory реальных полиморфных relations, fault-injection tests удаления/миграции target, замеры validation cost и принятый ADR о допустимых профилях.
- **Этап решения:** до заморозки первой доменной модели, которой действительно нужен polymorphic endpoint; не позднее I1 для Alvatune.

### OQ-003. Дополнительный профиль многозначных атрибутов

**Приоритет:** P2.

- **Факт:** действующий контракт использует per-type `many table` с порядком, уникальностью и FK для ссылок; JSONB-массив не является альтернативой PoC. [IL-ADR, вопрос 3], [08 §6](08-model-delivery-and-extensibility.md).
- **Допущение:** many-table даёт нужную индексируемость и целостность при приемлемом числе таблиц.
- **Предложение:** обсуждать новый профиль только по нагрузочным данным и вводить отдельным ADR с явной миграцией, query semantics и ограничениями.
- **Почему важно:** выбор влияет на containment, порядок, уникальность, ссылки, индексы, статистику PostgreSQL и стоимость схемы.
- **Риск ошибочного решения:** JSONB с неявной семантикой и слабой целостностью либо взрыв числа малонагруженных tables.
- **Кто подтверждает:** DB/query architects, владельцы representative domain models и эксплуатация.
- **Закрывающее свидетельство:** benchmark на cardinality/updates/filter/containment/join, footprint каталога на сотнях типов, migration prototype между профилями.
- **Этап решения:** после фазы 8 и только перед первой моделью, не укладывающейся в many-table профиль.

### OQ-004. Канонические views против базовых таблиц

**Приоритет:** P1.

- **Факт:** query compiler должен скрывать joined/TPH storage details; не решено, когда обращение через каноническое view ухудшает pushdown или inline resolution. [IL-QUERY], [IL-ADR, вопрос 4].
- **Допущение:** renderer может выбирать безопасную форму без изменения object semantics.
- **Предложение:** подготовить эквивалентные планы view/base для point query, polymorphic root, where-used, recursive traversal и policy resolution.
- **Почему важно:** неверный default может сделать типизированную модель существенно медленнее рукописного SQL или привязать public query к physical tables.
- **Риск ошибочного решения:** плохие планы, неустойчивость к PostgreSQL planner changes либо утечка физического mapping в generated API.
- **Кто подтверждает:** query/compiler owner, PostgreSQL specialist и performance gate owner.
- **Закрывающее свидетельство:** `EXPLAIN (ANALYZE, BUFFERS)` corpus на целевых объёмах, план-regression fixtures и documented selection rule.
- **Этап решения:** до принятия G3/G4 renderer в фазах 4–5; окончательная настройка — фаза 8.

### OQ-005. Онлайн-продвижение runtime extension в строгую колонку

**Приоритет:** P1.

- **Факт:** promotion JSONB/attached value в column — штатное целевое направление, но online backfill, судьба старого ключа и совместимость экспортов не определены. [08 §6](08-model-delivery-and-extensibility.md), [IL-ADR, вопрос 5].
- **Допущение:** первая реализация может быть offline и использовать проверяемый phased backfill.
- **Предложение:** определить состояния `dual-read/backfill/cutover/cleanup`, один authoritative writer на каждом шаге и reversible checkpoint до удаления старого значения.
- **Почему важно:** promotion — обещанный выход из runtime flexibility в строгую производительную модель; потеря значений или двойная запись подрывает это обещание.
- **Риск ошибочного решения:** data loss, divergent values, долгий lock, несовместимый customer patch или невозможный rollback.
- **Кто подтверждает:** migration/runtime leads, DBA, customer customization owner и host operations.
- **Закрывающее свидетельство:** migration test на большой таблице с concurrent reads/writes, interruption/resume, old/new reader compatibility и value-by-value reconciliation.
- **Этап решения:** фаза 7 до первой production promotion; online-вариант — вместе с OQ-011.

### OQ-006. Гранулярность технического `projection` snapshot

**Приоритет:** P1.

- **Факт:** `baseline` и `order` обязаны быть полными самодостаточными графами; только для rebuildable `projection` открыта развилка «полный граф» против «корень + дельта». [07 §10–12](07-versioning-changes-snapshots.md), [IL-ADR, вопрос 6].
- **Допущение:** полный projection проще, надёжнее и достаточен для PoC, пока объём не доказал обратное.
- **Предложение:** реализовать и измерить full projection первым; delta допускать только с явной base dependency, atomic invalidation и bounded reconstruction.
- **Почему важно:** delta экономит запись, но может превратить ускоряющую проекцию в сложную цепочку зависимостей и смешать её с business evidence.
- **Риск ошибочного решения:** медленное восстановление, stale chain, потеря атомарности либо ошибочное использование delta как baseline/order.
- **Кто подтверждает:** snapshot/query owners, DBA/performance owner и продуктовый владелец large-structure use cases.
- **Закрывающее свидетельство:** фаза-8 benchmark full vs delta по build time, storage, update locality, read latency, invalidation/recovery; invariant tests, запрещающие delta для business kinds.
- **Этап решения:** фаза 8 до завершения projection snapshot gate.

### OQ-007. Совместимость `CodegenVersion`

**Приоритет:** P1.

- **Факт:** generated code коммитится; output зависит не только от normalized model/hash, но и от module membership, seeds и exact generator version. `CodegenVersion` отсутствует, generated C# manifest не несёт `modules[]`/membership, а локальный `generate` атомарен по отдельным файлам, но не по всему набору. [08 §7–8](08-model-delivery-and-extensibility.md#7-сгенерированные-артефакты), [IL-ADR, вопрос 7].
- **Допущение:** generator-only regeneration допустима отдельным reviewed change при неизменном normalized model hash и явном version bump.
- **Предложение:** определить compatibility classes: byte-compatible patch, source-compatible regeneration, API-breaking regeneration и migration-coupled change; связать каждую с CI и downstream policy; добавить точный generator identity, module membership в исполнимый manifest и протокол whole-set publication/recovery.
- **Почему важно:** Alvatune будет пиновать artifacts Interlink; неявный generated API churn способен одновременно сломать несколько модулей.
- **Риск ошибочного решения:** ложная детерминированность при одинаковом model hash, шумные массовые diff, silent ABI/API break, невозможность воспроизвести старый snapshot reader, failure multi-delivery revalidation или смешанный каталог generated-файлов после I/O failure.
- **Кто подтверждает:** codegen/public API owners, release engineering и downstream Alvatune maintainers.
- **Закрывающее свидетельство:** compatibility matrix, golden API tests across two versions, clean-environment regeneration из полного compilation lock, исполнимый multi-delivery sample с одинаковыми локальными именами, fault injection при публикации набора, upgrade/rollback rehearsal и ADR/versioning policy.
- **Этап решения:** до стабильного выпуска фазы 2 и обязательно до I0 artifact lock.

### OQ-008. Multi-node cache fence и registry версий

**Приоритет:** P1.

- **Факт:** stale-window запрещено; общий durable, транзакционно наблюдаемый token/version registry обязателен. Открыты table/lease protocol и необязательный wake-up transport; `LISTEN/NOTIFY` не может быть источником корректности. [IL-ADR, вопрос 8], [IL-HOST].
- **Допущение:** single-node in-process реализация достаточна для PoC, но её API не должен закрывать путь к durable registry.
- **Предложение:** смоделировать write fence как данные с monotonic versions/tokens в той же durability boundary; уведомление использовать только для latency.
- **Почему важно:** кэш инженерных данных не может показывать состояние до commit после подтверждённого изменения, особенно между tenant nodes.
- **Риск ошибочного решения:** редкая stale read после write, cross-node race, потерянное уведомление, вечный bypass после uncertain commit.
- **Кто подтверждает:** runtime/cache lead, host transaction owner, distributed-systems reviewer и operations.
- **Закрывающее свидетельство:** adversarial two-node tests с delayed/lost notifications, crash before/after commit, ambiguous outcome и tenant isolation; protocol ADR.
- **Этап решения:** до multi-node production/I2; API boundary — не позднее I1.

### OQ-009. Гранулярность cache invalidation

**Приоритет:** P2.

- **Факт:** per-type dependency vector корректен, но может быть грубым; per-root карта способна повысить hit rate ценой сложного сопровождения. [IL-ADR, вопрос 9].
- **Допущение:** per-type достаточно для первых workloads и безопаснее как default.
- **Предложение:** собирать hit/miss/invalidation fan-out и вводить более точную карту только для доказанного hot path.
- **Почему важно:** слишком грубая инвалидизация уничтожит ценность result cache; слишком точная может сама стать дорогим и ошибочным dependency graph.
- **Риск ошибочного решения:** низкий hit rate либо stale cache из-за пропущенной зависимости.
- **Кто подтверждает:** query/cache owner, performance owner, наблюдаемость host-а.
- **Закрывающее свидетельство:** production-like trace replay, сравнение per-type/per-root cost и mutation fault tests, где каждый изменённый dimension обязан инвалидировать результат.
- **Этап решения:** после рабочих G3/G4 и фазы 8; до специализированной оптимизации крупного клиента.

### OQ-010. Производные read-модели сверх snapshots

**Приоритет:** P2.

- **Факт:** recursive CTE и snapshot range scan являются базовыми механизмами; открыты closure table для произвольного subtree/where-used и PostgreSQL materialized views для отчётов. [05 §14](05-composition-resolution.md), [IL-ADR, вопрос 10].
- **Допущение:** базовых механизмов достаточно до измеренного bottleneck; производная модель не должна становиться второй writable truth.
- **Предложение:** классифицировать кандидаты как disposable projection с явной provenance/invalidation, а не как новый domain store.
- **Почему важно:** составы и where-used могут быть огромными, но скрытая second truth разрушит воспроизводимость.
- **Риск ошибочного решения:** устаревшая closure, дорогая синхронизация, неверный access scope или смешение technical projection с business baseline.
- **Кто подтверждает:** query/snapshot architects, reporting owners и DBA.
- **Закрывающее свидетельство:** benchmark queries от произвольных roots, refresh/invalidation failure tests, rebuild proof и ownership ADR для каждой read-model family.
- **Этап решения:** после фазы 8 по фактическим планам; до обещания interactive latency на соответствующем масштабе.

### OQ-011. Online/zero-downtime migrations

**Приоритет:** P1.

- **Факт:** текущий контракт — reviewed offline chain с code steps/checkpoints; down migration и zero-downtime не обещаны. [08 §9](08-model-delivery-and-extensibility.md), [IL-ADR, вопрос 11].
- **Допущение:** offline-модель достаточна для PoC и первых контролируемых installations, но не для fleet Alvatune с независимыми tenants.
- **Предложение:** сначала доказать offline correctness, затем определить expand/migrate/contract, rolling reader/writer compatibility, admission и rollback-by-forward-fix.
- **Почему важно:** схема и generated code поставляются совместно; несовместимый rolling window может остановить запись или повредить данные.
- **Риск ошибочного решения:** mixed-version corruption, долгий outage, необратимый partial rollout или притворный rollback после destructive transform.
- **Кто подтверждает:** migration/release owners, DBA/SRE, host deployment owner и владельцы SLA.
- **Закрывающее свидетельство:** multi-version rehearsal на копии production-scale data, interrupted rollout/resume, compatibility matrix, backup/restore drill и formal go/no-go conditions.
- **Этап решения:** проектировать после опыта фазы 3/7; закрыть до I2 fleet rollout или раньше, если первый SLA запрещает offline window.

## 3. Эквивалентность формирования состава IPS

### OQ-012. Точный порядок resolver-а в целевой установке IPS

**Приоритет:** P0.

- **Факт:** источники расходятся. Консолидированная карточка даёт `hard concretion → applicability → editing context → soft concretion → rule`; ТЗ 2013 года — `applicability → concretion → editing context → configurator → rule`, причём прошедшие applicability versions помечаются и дальше в подборе не участвуют, а непрошедшие удаляются; документ contexts — `concretion → context → rule`. [05 §5](05-composition-resolution.md), [IPS-A04], [IPS-K03], [IPS-K04].
- **Допущение:** расхождение вызвано версиями продукта, resolver profiles, modules или разной трактовкой hard/soft concretion, а не ошибкой одного документа.
- **Предложение:** не объявлять ни один порядок «поведением IPS вообще»; создать customer-specific resolution profile и differential fixture corpus.
- **Почему важно:** перестановка двух шагов меняет выбранную revision и может изменить производство, заказ и историческую интерпретацию состава.
- **Риск ошибочного решения:** ложная IPS parity, незаметно другой 100% состав и недостоверные migration results.
- **Кто подтверждает:** продуктовый эксперт IPS целевого клиента, владелец/администратор его конфигурации и инженер Interlink resolver-а.
- **Закрывающее свидетельство:** версия/SP/modules customer installation; black-box cases с конфликтующими hard/soft pin, applicability, context, configurator и rule; зафиксированные inputs, selected revision, status/reason.
- **Этап решения:** до freeze post-PoC resolver/effectivity semantics и до любого обещания IPS-equivalence конкретному клиенту.

### OQ-013. Видимые статусы, fallback и допустимость «не совсем подходящей» версии

**Приоритет:** P0.

- **Факт:** исторический IPS может выбрать fallback из не прошедших обычные критерии и пометить результат как «не совсем подходящий»; Interlink требует explicit reason/warning или `NoMatch`, а не silent success. [05 §4.2–4.3 и §15](05-composition-resolution.md).
- **Допущение:** fallback полезен для диагностического чтения и части planning flows, но не всегда допустим для production handoff/snapshot.
- **Предложение:** каталогизировать IPS status icons/outcomes и установить по каждому host operation режим `forbid | warn | allow-with-approval`.
- **Почему важно:** одинаковый выбранный объект с разным status означает разную бизнес-достоверность.
- **Риск ошибочного решения:** производство принимает diagnostic fallback как approved configuration либо пользователи теряют полезный кандидат и получают лишний NoMatch.
- **Кто подтверждает:** IPS product specialist, production planner, quality/change owner и host policy owner.
- **Закрывающее свидетельство:** реальные сценарии/выгрузки со статусами, downstream acceptance rules, UX prototype с reason/warning и snapshot publication tests.
- **Этап решения:** до фазы 6 policy/fallback и до business `Order` Snapshot.

### OQ-014. Семантика применяемости: series, date, head, wildcard и аннулирование

**Приоритет:** P0.

- **Факт:** IPS использует series/date/head applicability, но исторически часть значений кодировалась строками. Нормативный target Interlink уже фиксирует typed `Effectivity + Coordinates`, приоритет `serial` над `date`, конкретного head над wildcard и запрет overlap при публикации. Открыта не эта целевая очередность, а её соответствие customer IPS, точные границы и дополнительные axes. [05 §4.4 и §7.5](05-composition-resolution.md), [IL-VERS-SPEC].
- **Допущение:** минимальный OC-A покрывается осями head/serial/date с constant coordinates; customer wildcard, annulled ranges и boundary cases можно однозначно нормализовать в принятый target.
- **Предложение:** построить differential truth table customer IPS против нормативного Interlink precedence, определить closed/open bounds, timezone/date precision, wildcard и отмену; несовпадение оформить customer profile либо новым ADR, не скрытой policy branch.
- **Почему важно:** ошибка на границе серии или даты выбирает неправильную revision и делает snapshot невоспроизводимым.
- **Риск ошибочного решения:** overlap/gap, off-by-one, различие online/offline resolver, невозможность индексировать или объяснить выбор.
- **Кто подтверждает:** IPS configuration specialist, production planner, domain modeller и DB/resolver leads.
- **Закрывающее свидетельство:** anonymized applicability rows, boundary fixtures (`100/101`, dates, head wildcard, annulled), overlap rejection и differential results.
- **Этап решения:** до OC-A schema/command freeze; базовые carrier constraints — до фазы 3, execution — до business snapshot.

### OQ-015. Контексты состава: EBOM/MBOM/production view и их комбинирование

**Приоритет:** P1.

- **Факт:** occurrence может быть условно присутствующим в разных composition contexts; удаление из одного представления не должно уничтожать source relation. [05 §3.3](05-composition-resolution.md), [06 §8](06-ips-to-interlink-delta.md).
- **Допущение:** большинство customer cases можно выразить declared coordinate/presence predicate без отдельного копируемого графа на каждый view.
- **Предложение:** собрать каталог контекстов, композиционные правила (`AND/OR`, inheritance, exclusivity) и owner каждого перехода EBOM→MBOM→order.
- **Почему важно:** это определяет, хранится ли одна 150% структура или несколько согласуемых structures, и где возникает технологическая структура.
- **Риск ошибочного решения:** потеря позиции при переключении view, twin graphs без reconciliation, неконтролируемая копия EBOM в MBOM.
- **Кто подтверждает:** конструктор, технолог, production planner, IPS product specialist и domain owner Alvatune.
- **Закрывающее свидетельство:** 3–5 реальных изделий, где одна позиция включается/исключается или преобразуется по context; golden EBOM/MBOM/order outputs и ownership map.
- **Этап решения:** до OC-A; минимальная distinction должна быть подтверждена до выбора первой product slice.

### OQ-016. Опции, значения по умолчанию и распространение по пути

**Приоритет:** P0.

- **Факт:** IPS различает option catalog, allowed values, incompatibilities, implications, occurrence presence и propagation; нужно различать отсутствующее значение, explicit none и default. Target Interlink содержит carrier contracts, но full option IR/editor и nested propagation открыты. [05 §6.1 и §7.7](05-composition-resolution.md).
- **Допущение:** OC-A может начать с constant root coordinates, а path-scoped overrides/nested configurations отложить в OC-B без потери адресуемости благодаря ThreadPath.
- **Предложение:** формально задать value lattice, default timing, implication/conflict order, scope и inheritance; проверить отдельно selection/presence occurrence и selection target Revision. Excluded edge отсутствует в выбранном graph, не участвует в fallback/`NoMatch` и получает отдельную configuration trace, а не `ResolutionReason` версии.
- **Почему важно:** опции превращают 150% source graph в выбранный 100% graph; неявные defaults делают результат невоспроизводимым.
- **Риск ошибочного решения:** override протекает в sibling path, absent превращается в default, несовместимые values дают частичный graph, resolver смешивает variant exclusion с `NoMatch` revision.
- **Кто подтверждает:** IPS configurator expert, product configuration owner, metamodel/resolver architects и UX owner.
- **Закрывающее свидетельство:** nested product fixtures с missing/none/default, implications, conflicts и sibling override; canonical normalized context bytes и explain trace.
- **Этап решения:** value semantics до OC-A; path propagation и nested config до OC-B.

### OQ-017. Точное представление N:M допустимых замен

**Приоритет:** P0.

- **Факт:** допустимая замена принадлежит конкретному composition scope и моделируется group/variant/member; один variant может заменять N positions на M positions. Это не `replacement_id` и не список ссылок. [05 §6.2 и §12](05-composition-resolution.md).
- **Допущение:** group scoped by exact parent composition/ThreadPath, explicit selected variant и members by occurrence thread достаточно для production patterns.
- **Предложение:** проверить модель на реальных 1:1, 1:N, N:1, N:M, kit и path-reused cases до фиксации IR/DDL.
- **Почему важно:** замена меняет topology и quantities, а не только выбранную identity одной позиции.
- **Риск ошибочного решения:** некорректные количества/пути, неоднозначный 100% graph, потеря области действия при reuse subassembly.
- **Кто подтверждает:** IPS product specialist, configuration/production experts, metamodel owner и snapshot owner.
- **Закрывающее свидетельство:** anonymized production substitution groups, round-trip import/export, resolver trace и order snapshot с выбранными members.
- **Этап решения:** до OC-B IR/DDL freeze; representative fixtures собрать до завершения OC-A.

### OQ-018. Глобальные аналоги и их отличие от локальных замен

**Приоритет:** P1.

- **Факт:** аналог в IPS — глобальная возможность замены object с period, priority и selection mode; локальная допустимая замена ограничена composition. В принятом Interlink metamodel базовый candidate set аналога уже выражается identity-scoped `list<reference<T>>`; это не N:M substitution. Открыт полный IPS behavioral policy — период, priority, selection mode и governance. [05 §6.3](05-composition-resolution.md), [06 §8](06-ips-to-interlink-delta.md), [IL-META §8г].
- **Допущение:** базового reference set достаточно для хранения топологии кандидатов, а дополнительная versioned domain policy/attributes смогут задать полное поведение без нового primitive ядра.
- **Предложение:** определить authority, direction/symmetry, qualification, effectivity, priority ties, approval и взаимодействие с local substitution; отдельно проверить, какие данные должны принадлежать identity, relation/policy revision или конкретному решению выбора.
- **Почему важно:** глобальная замена может повлиять на множество изделий и заказов; её риск и governance выше локальной альтернативы.
- **Риск ошибочного решения:** аналог автоматически применяется там, где запрещён; local rule неожиданно проигрывает global; исторический order пересчитывается.
- **Кто подтверждает:** component governance, quality/procurement/production owners, IPS specialist и domain policy owner.
- **Закрывающее свидетельство:** каталог реальных analog records и режимов, conflict matrix analog vs local variant vs pin, approved policy fixtures.
- **Этап решения:** после OC-A, до первого автоматического выбора аналога или production use.

### OQ-019. Заказ: живой объект, exact publication, units/lots и пересчёт

**Приоритет:** P0.

- **Факт:** target разделяет versioned live Order и immutable `Order` Snapshot каждого handoff; один order может иметь несколько publications. Полный IPS parity также требует units/lots, kits/spares и production-specific payload. [05 §11](05-composition-resolution.md), [07 §10](07-versioning-changes-snapshots.md), [IPS-A13].
- **Допущение:** live order изменяется только явной командой/новой revision, а ранее опубликованный snapshot никогда не re-resolve; `Unit`/`Lot` развиваются параллельно после PoC.
- **Предложение:** разделить product order, configured order lines, unit/lot/as-built facts и immutable handoff evidence; установить, когда live order recalculates и кто принимает delta.
- **Почему важно:** production должен получить точный неизменный состав, но planner должен продолжать управлять evolving order.
- **Риск ошибочного решения:** старый заказ меняется после новой revision/rule, один SnapshotId перезаписывается, quantity/unit/lot semantics теряются между PLM и ERP.
- **Кто подтверждает:** production planning, ERP/MRP integration, IPS order specialist, snapshot/domain owners.
- **Закрывающее свидетельство:** сквозной сценарий order create→configure→publish→source change→replan→second publish; byte/semantic identity первого snapshot; reconciliation with ERP payload.
- **Этап решения:** domain model до business Order API; exact publication contract — до первого production handoff, после OC-A.

### OQ-020. Стабильная идентичность места использования и сравнение составов

**Приоритет:** P1.

- **Факт:** Interlink принял `occurrence_thread_id` и полный адрес `(StructureDefinitionId, ExactRootRevisionId, ThreadPath)`; это контракт, но production evolution/matching ещё не доказаны. [05 §7.8](05-composition-resolution.md), [IL-ADR ADR-050/052].
- **Допущение:** thread переживает revision evolution одной логической позиции, а path disambiguates reuse той же subassembly.
- **Предложение:** определить операции split/merge/move/copy/reuse, правила сохранения thread и отображение IPS position identity при миграции.
- **Почему важно:** diff, substitutions, comments, evidence и agents должны ссылаться на то же место, а не угадывать по номеру позиции.
- **Риск ошибочного решения:** ссылка переезжает на другую позицию, diff показывает delete+add вместо change, path нестабилен после перестройки верхнего уровня.
- **Кто подтверждает:** composition/metamodel owners, CAD/PLM integration experts и продуктовые пользователи сравнения.
- **Закрывающее свидетельство:** evolution fixture suite (move, reorder, reuse, split, merge, replace), round-trip IPS mapping и human review of diff.
- **Этап решения:** базовые write invariants до фазы 3/5; полная evolution semantics до OC-B и migration tooling.

### OQ-021. AutoMatch/Expert: воспроизводимость без второго rule engine

**Приоритет:** P1.

- **Факт:** целевой контракт требует от Interlink typed refs/query/commands/context/evidence, но рабочий runtime этих primitives ещё не поставлен; AutoMatch/Expert остаётся versioned domain/host service. `suggest` и `apply` должны быть раздельны. [05 §13](05-composition-resolution.md), [06 §9](06-ips-to-interlink-delta.md), [02 §3–5](02-current-state-and-roadmap.md).
- **Допущение:** exact scenario/rule version, normalized inputs, candidate/exclusion trace и typed Changeset proposal достаточны для IPS outcome parity без произвольного SQL/plugin path в ядре.
- **Предложение:** выбрать один critical AutoMatch scenario и восстановить весь decision trace, включая units, exclusions, fallback и approval.
- **Почему важно:** технологический подбор — значимая IPS-функция; перенос только финального результата не доказывает воспроизводимость.
- **Риск ошибочного решения:** opaque recommendations, невозможность повторить результат, agent/domain service обходит writer или незаметно меняет composition.
- **Кто подтверждает:** технолог/AutoMatch expert, domain service owner, Interlink query/command owners и agent governance.
- **Закрывающее свидетельство:** golden candidate corpus, pinned rule/input artifacts, exclusion trace comparison, proposal/approval/apply test через Changeset.
- **Этап решения:** discovery до выбора Alvatune L1/L2 use case; implementation после G3/G4 и full Changeset/I1.

### OQ-022. Customer Configuration Baseline и граница обещания IPS parity

**Приоритет:** P0.

- **Факт:** IPS отличается по версии, service pack, modules, configuration, handlers и customer plugins; общая реконструкция не доказывает конкретную installation. [04 §2](04-ips-functional-baseline.md#2-важная-оговорка-ips-не-является-одной-конфигурацией), [04 §9](04-ips-functional-baseline.md#9-customer-configuration-baseline).
- **Допущение:** parity можно определить как набор customer-specific golden outcomes и migration invariants, а не полное копирование внутренней физики IPS.
- **Предложение:** для каждого клиента вести baseline: product/version/SP, modules, resolver profile, custom metadata/plugins, data anomalies, critical scenarios, accepted differences.
- **Почему важно:** иначе «сохраняем функциональность IPS» становится непроверяемым и потенциально ложным обещанием.
- **Риск ошибочного решения:** пропущенный plugin меняет resolver/order, migration принимает исключение за норму, sales обещает универсальную совместимость.
- **Кто подтверждает:** customer product owner, IPS administrator/partner, migration lead и Interlink product owner.
- **Закрывающее свидетельство:** подписанный baseline, anonymized fixtures, differential test report и список осознанных semantic deltas.
- **Этап решения:** до contract/scope конкретной миграции и до design freeze соответствующего capability.

## 4. Версионность, изменение и доказательность

### OQ-023. Что означает `Current`, `Released`, `Latest` и Maturity у первого клиента

**Приоритет:** P0.

- **Факт:** нормативный контракт Interlink различает политики `Current`, `Released`, `Latest`, `MinMaturity` и `PinnedOnly`; Maturity — шкала пригодности контента, а Workflow — состояние процесса. Исполняемый selection runtime относится к планам фаз 4–6. [07 §7–9](07-versioning-changes-snapshots.md), [02 §5](02-current-state-and-roadmap.md).
- **Допущение:** одна упорядоченная Maturity scale может отобразить нужные IPS lifecycle levels, если customer-specific workflow states остаются в host-е.
- **Предложение:** составить mapping каждой IPS litera/state/iteration к `Draft/fixed`, Maturity, disposition и Workflow, отдельно описав selection eligibility.
- **Почему важно:** слово «текущая» часто означает разные вещи для конструктора, производства и архива; смешение меняет resolver outcome.
- **Риск ошибочного решения:** отозванная revision выбирается как Latest, process state навсегда попадает в kernel scale или Released не соответствует юридически выпущенной версии.
- **Кто подтверждает:** change/release owner, IPS lifecycle expert, quality/legal representative и Interlink versioning owner.
- **Закрывающее свидетельство:** state-transition matrix на реальных объектах, selection fixtures для каждой policy и signed domain glossary.
- **Этап решения:** Maturity mapping до фаз 3/6; юридическая трактовка — до Baseline/Order publication.

### OQ-024. Конкурирующие Changesets, urgent change и linked notices

**Приоритет:** P1.

- **Факт:** целевая модель предпочитает один explicit Changeset/ordered bundle скрытым linked editing contexts; базовое допущение — exclusive affected-object editing. [07 §5–6](07-versioning-changes-snapshots.md).
- **Допущение:** эксклюзивная блокировка affected identity приемлема для первого релиза и лучше неоднозначного merge revision content.
- **Предложение:** измерить частоту parallel notices, urgent preemption и multi-notice bundles; определить takeover, suspend, abandon, rebase/merge и audit semantics.
- **Почему важно:** модель изменения должна обеспечивать coherent future graph, но не парализовать срочные производственные исправления.
- **Риск ошибочного решения:** deadlock рабочего процесса, потеря правок, два drafts одного object в одном resolution context или неаудируемое ручное слияние.
- **Кто подтверждает:** change manager, engineering leads, IPS notice expert и Changeset/workflow owners.
- **Закрывающее свидетельство:** customer concurrency telemetry/interviews, tabletop emergency scenarios, adversarial command tests и принятый conflict policy.
- **Этап решения:** до full Changeset/I1; minimal lock semantics должны быть известны до writer API freeze.

### OQ-025. Soft concretion: временная preference или публикуемая связь

**Приоритет:** P1.

- **Факт:** target mapping трактует hard concretion как exact pin, а soft concretion — как Changeset override, который до commit должен исчезнуть или стать pin. [05 §7.4 и §10](05-composition-resolution.md).
- **Допущение:** soft concretion не является самостоятельным долговечным published state.
- **Предложение:** проверить физическое и пользовательское поведение soft concretion в целевых IPS installations, включая propagation, conflict и audit.
- **Почему важно:** неверное отображение может либо потерять пользовательское намерение, либо сохранить неявную preference, которую позднее невозможно воспроизвести.
- **Риск ошибочного решения:** draft неожиданно становится exact pin, commit меняет выбранную revision или temporary override протекает за пределы Changeset.
- **Кто подтверждает:** IPS resolver/context specialist, конструктор и Interlink versioning/resolver owners.
- **Закрывающее свидетельство:** black-box cases create/edit/commit/cancel soft concretion, database/export sample и agreed mapping to command lifecycle.
- **Этап решения:** до full Changeset overlay и resolver phase 6 parity claims.

### OQ-026. Какие Snapshot обязаны быть автономным доказательством

**Приоритет:** P0.

- **Факт:** нормативный Snapshot contract хранит topology, exact refs и provenance, но намеренно не дублирует все immutable attributes/files; для автономного legal evidence host должен создать отдельный evidence artifact. Business Baseline/Order API ещё не реализован. [07 §10–12 и §18](07-versioning-changes-snapshots.md).
- **Допущение:** технических exact refs достаточно для большинства внутренних baselines, пока source revisions, artifact vault и readers сохраняются по retention policy.
- **Предложение:** классифицировать use cases: acceleration, engineering baseline, order handoff, legal archive, external exchange; для каждого определить self-contained payload и допустимые dependencies.
- **Почему важно:** ссылка на вечный объект и автономная копия доказательства имеют разную цену, срок жизни и юридический смысл.
- **Риск ошибочного решения:** исторический документ нельзя прочитать после schema evolution/deletion; либо snapshots бесконтрольно дублируют весь vault.
- **Кто подтверждает:** product/change/quality/legal records owners, snapshot architect и vault owner.
- **Закрывающее свидетельство:** evidence-content matrix, retention/legal opinion, restore/read drill после удаления active schema reader и accepted artifact format.
- **Этап решения:** до business Baseline/Order API; legal subset — до первого подписанного/архивного handoff.

### OQ-027. Retention model для provenance, readers, rules, files и keys

**Приоритет:** P0.

- **Факт:** воспроизводимость Snapshot и agent result требует exact model/policy/customization artifacts, schema-versioned readers и file hashes. [07 §11–14](07-versioning-changes-snapshots.md), [09 §11](09-alvatune-and-agents.md).
- **Допущение:** append-only artifact registry с versioned readers и external vault retention может обеспечить чтение без сохранения всего старого runtime environment.
- **Предложение:** определить dependency closure каждого evidence kind, retention periods, cryptographic key rotation, legal hold, deletion exceptions и reader conformance suite.
- **Почему важно:** immutable ID без доступного reader/policy/file не даёт воспроизводимого свидетельства.
- **Риск ошибочного решения:** исторический snapshot становится unreadable, hash невозможно проверить после key/algorithm retirement или retention нарушает privacy obligations.
- **Кто подтверждает:** architecture/release, security/crypto, records/legal, vault and operations owners.
- **Закрывающее свидетельство:** archive manifest specification, long-horizon compatibility test, restore from cold storage и approved retention schedule.
- **Этап решения:** минимальный technical retention до фазы 8; business/legal retention до I1/A3.

### OQ-028. Whole-artifact authorization и sensitive nodes

**Приоритет:** P0.

- **Факт:** business Snapshot нельзя строить или читать как security-pruned graph; доступ — allow/deny целиком. Read queries могут использовать host access predicates. [05 §15](05-composition-resolution.md), [06 §14](06-ips-to-interlink-delta.md).
- **Допущение:** host может принять решение на весь artifact без раскрытия forbidden node и без materialize-then-filter.
- **Предложение:** определить preflight authorization по closure/labels, правила ownership/visibility metadata в header и безопасный UX отказа.
- **Почему важно:** pruned snapshot уже не является тем составом, который был утверждён; при этом само раскрытие existence может быть чувствительным.
- **Риск ошибочного решения:** неполное «доказательство», covert leakage через errors/counts, кэш между principals или невозможность легально передать разрешённую часть.
- **Кто подтверждает:** host authorization/security owner, snapshot owner, customer security administrator и audit/legal.
- **Закрывающее свидетельство:** adversarial graph tests с одним forbidden node, no-side-channel review, cache fingerprint tests и product policy for derived/redacted export distinct from Snapshot.
- **Этап решения:** access seam до I1; mandatory before business snapshot and agent context use.

### OQ-029. ContentHash, files и граница одобрения

**Приоритет:** P0.

- **Факт:** нормативный approval contract относится к exact ContentHash; при наличии artifacts hash должен охватывать file hashes, а upload/scan/signature/vault принадлежат host-у. Исполняемый CommitPermit/host seam ещё предстоит доказать на I1. [07 §13](07-versioning-changes-snapshots.md), [06 §13](06-ips-to-interlink-delta.md).
- **Допущение:** canonical content manifest может объединить object changes, relations, file digests, model/policy context и approval scope без загрузки binaries в Interlink.
- **Предложение:** стандартизовать manifest ordering/encoding, late file upload behavior, malware scan state, signature binding и invalidation of permit.
- **Почему важно:** одобрение метаданных при изменившемся файле не является одобрением того же инженерного результата.
- **Риск ошибочного решения:** TOCTOU между preview и commit, approval reuse на иной файл, неодинаковый hash в host/kernel или невозможность доказать состав подписи.
- **Кто подтверждает:** Interlink host-seam owner, Alvatune workflow/audit, vault/security и records/legal.
- **Закрывающее свидетельство:** canonical hash vectors, edit-after-approval tests, cross-process hash agreement, file replacement/scan failure cases и signed permit specification.
- **Этап решения:** до I1 CommitPermit/atomic commit и до document/order approval.

### OQ-030. Корректность publication при `NoMatch`, crash и ambiguous commit

**Приоритет:** P0.

- **Факт:** по нормативному publication contract любой recursive `NoMatch` должен отклонить snapshot целиком; header, entries и archived provenance должны публиковаться атомарно. Исполнение technical path относится к фазе 8. [07 §12 и §18](07-versioning-changes-snapshots.md).
- **Допущение:** одна transaction/MVCC snapshot и idempotency key достаточно для local publication; external vault/outbox требуют coordinated host protocol.
- **Предложение:** специфицировать idempotent publish states, cleanup of abandoned attempts и recovery после client timeout/unknown outcome без повторного создания бизнес-свидетельства.
- **Почему важно:** частичный граф или два разных snapshots на один handoff недопустимы.
- **Риск ошибочного решения:** orphan header/entries, duplicate publication, host считает fail при фактическом commit или retry использует уже изменившийся graph.
- **Кто подтверждает:** snapshot/runtime owners, host transaction/outbox owner и operations.
- **Закрывающее свидетельство:** fault injection at every publication boundary, kill/reconnect tests, exactly-once business key behavior и reconciliation runbook.
- **Этап решения:** technical path — фаза 8; business/external atomicity — I1 до Baseline/Order release.

## 5. Tenancy, security, host seam, agents и federation

### OQ-031. Полный tenant storage profile

**Приоритет:** P0.

- **Факт:** PoC Interlink односхемный; Alvatune приняла `control` и per-tenant `t_<opaque>_{meta,data,app}`. Request-derived schema names запрещены, cache/pool keys должны включать exact `StorageIdentity`. [09 §4](09-alvatune-and-agents.md), [ALV-ADR069].
- **Допущение:** schema-set-per-tenant даст требуемую изоляцию и управляемость для первого масштаба без database-per-tenant.
- **Предложение:** зафиксировать provisioning, trusted name resolution, roles/grants/default ACL, pool admission, migration ownership, noisy-neighbor limits, deprovision и restore.
- **Почему важно:** tenant boundary проходит через SQL identifiers, credentials, connections, caches, migrations и operations, а не только через `tenant_id` в API.
- **Риск ошибочного решения:** cross-tenant read/write/cache leak, migration не той схемы, shared superuser path или неуправляемое число pools.
- **Кто подтверждает:** Alvatune platform/security/SRE, Interlink storage/runtime/migration owners и независимый security reviewer.
- **Закрывающее свидетельство:** adversarial two-tenant suite, guessed-schema attempts, pool/cache poisoning tests, independent backup/restore/migration и least-privilege inspection.
- **Этап решения:** I2 до любой multi-tenant production data; public `StorageContext` boundary — I1.

### OQ-032. Authorization predicate и `AccessFingerprint`

**Приоритет:** P0.

- **Факт:** принятая граница не даёт Interlink собственного ACL: host должен авторизовать commands и передавать access predicate/fingerprint для reads. Actor attribution не даёт authority; production seam ещё не доказан. [06 §14](06-ips-to-interlink-delta.md), [09 §14 и §16](09-alvatune-and-agents.md).
- **Допущение:** host policy может быть скомпилирована в bounded typed predicate, а fingerprint меняется при любой policy/membership/classification mutation, влияющей на result.
- **Предложение:** определить поддерживаемую authorization algebra, deny-before-fetch guarantees, fingerprint dimensions/versioning и поведение unsupported policy.
- **Почему важно:** query, navigation memo, result cache и agents должны видеть один и тот же разрешённый граф.
- **Риск ошибочного решения:** cache leak, post-filter disclosure, разные решения между G3/G4/snapshot или fail-open при неизвестном predicate.
- **Кто подтверждает:** host IAM/policy owner, Interlink query/cache owner и security review.
- **Закрывающее свидетельство:** policy conformance corpus, mutation-driven fingerprint tests, forbidden traversal cases, cache isolation и fail-closed unsupported tests.
- **Этап решения:** минимальный contract до G3/G4 integration; production-complete до I1/A1.

### OQ-033. Caller-owned transaction, outbox и `CommitPermit`

**Приоритет:** P0.

- **Факт:** Interlink должен поддержать object commit и host audit/event/outbox в одной tenant transaction; permit связывает authorization decision с exact content, но сам не является permission. [07 §13](07-versioning-changes-snapshots.md), [09 §5](09-alvatune-and-agents.md).
- **Допущение:** enlisted session с явными `NotifyCommitted/NotifyRolledBack`, fence protocol и owned connection/transaction может сохранить границы ответственности.
- **Предложение:** специфицировать lifecycle, cancellation/timeout, savepoints, retry, uncertain outcome и запрет Interlink самостоятельно commit/rollback host-owned transaction.
- **Почему важно:** без общего atomic boundary появится окно «объект изменён, audit/outbox отсутствует» или наоборот.
- **Риск ошибочного решения:** dual outcome, преждевременная cache invalidation, reused permit, потерянное событие или connection poisoning.
- **Кто подтверждает:** Interlink runtime/writer, Alvatune ChangeAudit/platform, DB transaction reviewer и operations.
- **Закрывающее свидетельство:** integration tests на one connection/transaction, crash at each boundary, outbox dispatch/replay и content-change permit invalidation.
- **Этап решения:** API shape до фазы 3 writer; complete proof — I1 до permanent Alvatune writes.

### OQ-034. Содержимое, privacy и retention `ContextSnapshot` агента

**Приоритет:** P0.

- **Факт:** принятый target Alvatune требует до первого model call создать immutable ContextSnapshot с exact Interlink refs/context, external observations/freshness, access decisions, prompt/tool hashes, omissions и truncation; каждый read должен переавторизовываться до включения. Agent runtime и этот path пока не реализованы. [09 §11 и §16](09-alvatune-and-agents.md), [ALV-AGENT].
- **Допущение:** reproducibility можно обеспечить manifest/digests и selected fragments, не сохраняя избыточную чувствительную копию всего графа.
- **Предложение:** определить per-purpose minimization, encrypted payload vs digest, redaction, retention, subject access/deletion, model-provider residency и legal hold.
- **Почему важно:** ContextSnapshot — ключ к объяснению решения агента, но одновременно концентрирует наиболее чувствительный engineering context.
- **Риск ошибочного решения:** невозможность повторить run, чрезмерное хранение секретов/PII, forbidden node попадает в prompt или digest становится side channel.
- **Кто подтверждает:** agent runtime/product, IAM/security/privacy/legal, customer data owner и Interlink context owner.
- **Закрывающее свидетельство:** data classification/threat model, minimal manifests for pilot scenarios, prompt-boundary inspection, deletion/retention tests и reproduction exercise.
- **Этап решения:** до A4 любого реального agent L1/L2; baseline data contract — A0/A1.

### OQ-035. Уровень автономности и side-effect governance

**Приоритет:** P0.

- **Факт:** принятый target допускает L0 read, L1 analysis и L2 proposal; L3 требует доказанный reversible low-risk write, L4 вне baseline. Целевой write path: model output → typed proposal → Action intent → policy/approval → Changeset/connector. [09 §12–13](09-alvatune-and-agents.md).
- **Допущение:** первый материальный эффект продукта можно доказать на L1/L2 без autonomous commit/release.
- **Предложение:** для каждого Action задать side-effect class, capability, approval, idempotency, compensation, budget и escalation; запретить generic HTTP/SQL credentials.
- **Почему важно:** агентская полезность не должна создавать новый непроверяемый путь изменения engineering truth.
- **Риск ошибочного решения:** proposal принимается за approval, агент повышает capability, повторяет необратимую операцию или имитирует human signer.
- **Кто подтверждает:** product/risk owner, workflow/agent platform, domain approver, security и Interlink command owner.
- **Закрывающее свидетельство:** action catalog, policy tests, red-team prompts, approval/hash invalidation, bounded pilot metrics и incident drill.
- **Этап решения:** L0/L1 до A4 pilot; каждая L2/L3 action — до её включения, L3 не раньше I1.

### OQ-036. Неоднозначный внешний side effect и reconciliation

**Приоритет:** P0.

- **Факт:** принятый federated target оставляет внешнюю PLM authority; blind retry ambiguous write запрещён. Close case требует observation/reconciliation exact outcome; первый connector path пока не реализован. [09 §6, §12 и §16](09-alvatune-and-agents.md).
- **Допущение:** connector может использовать idempotency/correlation, external version token и read-after-write reconciliation либо переводить operation в human-resolvable unknown state.
- **Предложение:** определить operation state machine `planned/dispatched/acknowledged/observed/reconciled/unknown/compensated`, authority-specific retry matrix и UI evidence.
- **Почему важно:** timeout не доказывает failure; повтор может дважды выпустить/изменить объект.
- **Риск ошибочного решения:** duplicate irreversible action, закрытый ChangeCase без внешнего результата, local state ложно считается authority.
- **Кто подтверждает:** first connector owner, external PLM administrator, Alvatune operations/workflow и risk owner.
- **Закрывающее свидетельство:** независимый simulator с lost/late/duplicate responses, idempotency tests, manual reconciliation UX и incident runbook.
- **Этап решения:** A2 до первого connector write; read-only federation может начаться раньше с freshness contract.

### OQ-037. External identity, authority transfer и freshness

**Приоритет:** P0.

- **Факт:** по принятой границе Interlink — sole internal kernel, но не automatic owner external objects. External data должно оставаться reference/observation с version/freshness до explicit import/authority transfer; исполняемый federation contract ещё открыт. [06 §17](06-ips-to-interlink-delta.md), [09 §6 и §16](09-alvatune-and-agents.md).
- **Допущение:** stable connector namespace + external ID/version token/observed-at/authority mode достаточно, чтобы не маскировать stale observation под local Revision.
- **Предложение:** определить identity mapping cardinality, freshness SLA, deletion/tombstone, conflict/rekey, source switch и auditable transfer protocol.
- **Почему важно:** «удалённый item выглядит локальным» скрывает ownership и момент наблюдения, что опасно для решений и agents.
- **Риск ошибочного решения:** dual writable truth, stale data используется как current, два external IDs сливаются или authority transfer теряет provenance.
- **Кто подтверждает:** integration/data governance, external system owner, Interlink identity owner и product workflow.
- **Закрывающее свидетельство:** mapping table contract, stale/offline/rekey/delete fixtures, transfer rehearsal with before/after authority evidence и UI differentiation.
- **Этап решения:** read contract до A1/A2; transfer/write contract до первой delegated/internal adoption.

### OQ-038. Граница durable engineering artifacts и operational state

**Приоритет:** P1.

- **Факт:** AgentDefinition, ProcessDefinition и domain evidence должны быть versioned Interlink artifacts; AgentRun, AgentRequest, ProcessInstance, WorkItem и connector cursor — mutable Alvatune app state. [09 §9–10](09-alvatune-and-agents.md).
- **Допущение:** это разделение выдержит reporting/audit без копирования full object payload в app tables.
- **Предложение:** для каждого нового типа применять тест: является ли это долговечным инженерным смыслом или высокочастотной координацией; cross-boundary reference всегда exact и direction owned.
- **Почему важно:** размывание границы быстро создаст второй Record/Revision/Changeset store, запрещённый ADR-068.
- **Риск ошибочного решения:** dual truth, независимые revisions, невозможность атомарного удаления/retention или operational churn в object history.
- **Кто подтверждает:** Interlink/Alvatune architects, owners соответствующего domain module и data governance.
- **Закрывающее свидетельство:** type ownership catalog, schema review gate, test denying generic writable records и one-way dependency inspection.
- **Этап решения:** A0 module composition и далее до добавления каждого persistent type; полный audit — I1.

### OQ-039. Продуктовая ценность Agents controls pane до зрелости I1/I2

**Приоритет:** P0.

- **Факт:** Alvatune target — Engineering Process Command Center с четырьмя pilot surfaces, но текущие repositories в основном skeletons; техническая возможность не доказывает спрос. [09 §7 и §16](09-alvatune-and-agents.md).
- **Допущение:** requirements-to-change journey и agent L1/L2 insight могут дать paid/manual value до autonomous writes; если I1/I2 задержатся, federated mode может быть первым wedge.
- **Предложение:** провести concierge pilot на точном manual flow, измеряя cycle time, missing evidence, rework и decision latency до автоматизации.
- **Почему важно:** иначе Interlink может оптимизировать foundation для продукта, чья первичная ценность не подтверждена, или cockpit превратится в generic inbox.
- **Риск ошибочного решения:** преждевременная full-platform стройка, agent demo без outcome, неверный выбор internal vs federated first customer.
- **Кто подтверждает:** product lead, 3–5 target organizations, engineering/change personas и commercial owner.
- **Закрывающее свидетельство:** documented current journey, clickable/concierge test, willingness-to-pay/commitment, baseline and post-pilot metrics, explicit go/no-go for A3/A4.
- **Этап решения:** до A3 paid alpha; wedge selection — до расширения Alvatune implementation beyond A0/A1.

## 6. Поставка модели, миграции и эксплуатация

### OQ-040. Какие customer customizations должны остаться live-editable

**Приоритет:** P0.

- **Факт:** Interlink различает sealed, customizable и runtime extensions; unrestricted production edit core model не является целью. Customer delta должен экспортироваться и проходить three-way reconcile. [08 §6 и §11](08-model-delivery-and-extensibility.md).
- **Допущение:** labels, lists и ограниченный typed extension pool покрывают большую часть безопасной live customization, а schema/behavior changes требуют reviewed module release.
- **Предложение:** инвентаризировать реальные IPS Database Configurator changes по частоте, критичности и upgrade pain; назначить каждому tier и approval.
- **Почему важно:** слишком узкий envelope лишит customer agility, слишком широкий снова создаст hidden production source of truth.
- **Риск ошибочного решения:** customer fork невозможно обновить, runtime field нельзя индексировать/мигрировать или администратор меняет семантику без code review.
- **Кто подтверждает:** IPS administrators/partners, customer product owner, Interlink metamodel/migration и security governance.
- **Закрывающее свидетельство:** anonymized customization inventory нескольких installations, tier mapping, export/import/reconcile prototype и approval UX test.
- **Этап решения:** baseline до фазы 7; customer-specific mapping — до migration contract.

### OQ-041. Module resolution, exact lock и ownership UX

**Приоритет:** P1.

- **Факт:** module — independently versioned delivery unit с namespace/prefix/requires; host выбирает exact pinned artifact set. Реализованы composition contracts, но production resolver/lock UX не доказаны. [08 §5 и §16](08-model-delivery-and-extensibility.md).
- **Допущение:** minimum-compatible dependencies within major плюс exact deployment lock достаточны, если combined manifest deterministically validated.
- **Предложение:** определить resolver owner, lockfile format, conflict diagnostics, artifact trust/signing, rollback set и customer override policy.
- **Почему важно:** фактическое поведение системы определяется всем module set, а не только одним DSL hash.
- **Риск ошибочного решения:** non-reproducible environment, physical-name collision, dependency diamond выбирает иной artifact или host запускает code/schema mismatch.
- **Кто подтверждает:** Interlink module/release owners, Alvatune platform/build owners и customer deployment admin.
- **Закрывающее свидетельство:** clean environment reconstruction from lock, conflicting dependency fixtures, artifact tamper test, upgrade/rollback rehearsal.
- **Этап решения:** до I0/A0 composition lock; production trust/signing — до A3/I1.

### OQ-042. Production drift policy: block, quarantine или warning

**Приоритет:** P0.

- **Факт:** generation mismatch должен ломать CI; managed DB drift должен быть точно диагностирован. Документы пока допускают warning/off startup modes, что конфликтует с более сильным production fail-closed intent. [08 §13](08-model-delivery-and-extensibility.md).
- **Допущение:** production default должен блокировать unsafe writes при model/catalog/physical/artifact mismatch, но может разрешать bounded read-only quarantine.
- **Предложение:** классифицировать drift по влиянию и определить matrix `start/read/write/migrate/repair`, никогда не исправляя неизвестный drift автоматически.
- **Почему важно:** typed code безопасен только при точном соответствии deployed model и physical DB.
- **Риск ошибочного решения:** silent corruption в warning mode либо unnecessary outage из-за harmless observational drift.
- **Кто подтверждает:** architecture, migration/runtime, SRE/support и customer change governance.
- **Закрывающее свидетельство:** drift fault corpus (column/index/FK/ACL/owner/module/codegen), expected admission decisions, repair runbook и production profile ADR.
- **Этап решения:** strict baseline до фазы 3 `validate-db`; host policy окончательно до I1/I2.

### OQ-043. Rollout migrations по множеству tenants и version skew

**Приоритет:** P0.

- **Факт:** tenant schemas мигрируют независимо; exact module/model/codegen/migration hashes должны быть известны. I2 ещё не реализован. [09 §4](09-alvatune-and-agents.md), [08 §9](08-model-delivery-and-extensibility.md).
- **Допущение:** control plane может вести desired/actual versions, admission leases, cohorts и resumable checkpoints без cross-tenant transaction.
- **Предложение:** определить compatibility window, rollout cohorts/canary, per-tenant failure isolation, pause/resume, capacity budgets и application routing by actual version.
- **Почему важно:** один медленный/сломанный tenant не должен блокировать fleet или получить несовместимый runtime.
- **Риск ошибочного решения:** code обслуживает schema другой версии, partial fleet нельзя откатить, migration storm перегружает DB или tenant cache смешивается.
- **Кто подтверждает:** Alvatune control plane/SRE, Interlink migration/runtime, release manager и support.
- **Закрывающее свидетельство:** multi-tenant simulator with mixed versions, failed cohort/resume, routing/admission tests, capacity model и operational dashboard/runbook.
- **Этап решения:** до I2 fleet rollout; compatible artifact policy связана с OQ-007/OQ-011.

### OQ-044. Что означает rollback после необратимого преобразования

**Приоритет:** P0.

- **Факт:** текущая миграционная модель делает forward chain с checkpoints; down migrations не обещаны. Historical fixed data и business snapshots нельзя разрушать. [08 §9 и §18](08-model-delivery-and-extensibility.md).
- **Допущение:** после destructive/data-dependent cutover rollback обычно означает restore в новый environment или forward repair, а не автоматический reverse SQL.
- **Предложение:** для каждого destructive step требовать preflight, backup/restore point, compatibility cutoff, reconciliation count/hash и declared recovery objective.
- **Почему важно:** слово «rollback» без точной семантики создаёт ложную уверенность в безопасности deploy.
- **Риск ошибочного решения:** irreversible loss, split-brain после restore, replay host events twice или возврат code к schema, которую он не понимает.
- **Кто подтверждает:** migration owner, DBA/SRE, product data owner, host event/outbox owner.
- **Закрывающее свидетельство:** destructive rehearsal on representative data, timed restore, event reconciliation, documented point-of-no-return and decision authority.
- **Этап решения:** до первой destructive migration; общий policy — фаза 3/7, fleet form — I2.

### OQ-045. Backup, restore, DR и cryptographic verification

**Приоритет:** P1.

- **Факт:** object DB, model artifacts, snapshot provenance, external vault и Alvatune app state имеют разные owners, но вместе образуют восстанавливаемое engineering evidence. [07 §11](07-versioning-changes-snapshots.md), [09 §3](09-alvatune-and-agents.md).
- **Допущение:** coordinated restore points и post-restore verification могут восстановить consistency без distributed atomic backup всех external systems.
- **Предложение:** определить RPO/RTO по data class, backup closure, ordering restore, key availability, outbox/connector reconciliation и immutable artifact verification.
- **Почему важно:** восстановленная БД без exact model/readers/files либо app state с другим commit point не является восстановленной системой.
- **Риск ошибочного решения:** unreadable snapshots, duplicate integrations, lost audit link, tenant restored into wrong module set или hashes нельзя проверить.
- **Кто подтверждает:** SRE/DBA, vault/security, Interlink/Alvatune data owners и business continuity owner.
- **Закрывающее свидетельство:** full tenant disaster-recovery drill from cold backups, checksum/provenance validation, connector reconciliation and measured RPO/RTO.
- **Этап решения:** PoC не блокирует; minimum before A3 production data, tenant-isolated DR before I2 GA.

### OQ-046. Наблюдаемость и объяснимость resolver/query/migration/agent paths

**Приоритет:** P1.

- **Факт:** target APIs несут ResolutionReason/trace, exact hashes, actor/correlation; operations across kernel/host/connectors need common correlation but no production telemetry contract exists yet. [05 §15](05-composition-resolution.md), [09 §10–12](09-alvatune-and-agents.md).
- **Допущение:** structured events с stable reason codes и privacy-aware references достаточны без логирования domain payload/SQL values по умолчанию.
- **Предложение:** определить metrics/traces/audit разделение, correlation propagation, cardinality budgets, explain artifact и redaction; пользовательский reason не должен зависеть от raw logs.
- **Почему важно:** сложный выбор версии, migration или agent action невозможно поддерживать по одному stack trace.
- **Риск ошибочного решения:** необъяснимый состав, секреты в telemetry, unbounded labels, отсутствие forensic chain или смешение mutable logs с audit evidence.
- **Кто подтверждает:** runtime/query/migration owners, Alvatune observability/security, support и product UX.
- **Закрывающее свидетельство:** golden explain traces, incident exercises, telemetry privacy review, bounded-cardinality load и end-to-end correlation test.
- **Этап решения:** reason codes до фаз 4–6; migration telemetry до фазы 3; agent/connector path до A2/A4.

### OQ-047. Масштаб, на котором архитектурная гипотеза считается доказанной

**Приоритет:** P0.

- **Факт:** фаза 8 требует измерить depth/width/table count, UUID cost, snapshot scan и plans с ориентиром не хуже `1.5×` рукописного эталона для заявленных сценариев. Green contract tests с phase gates не доказывают runtime throughput. [02 §12](02-current-state-and-roadmap.md).
- **Допущение:** representative synthetic corpus плюс один anonymized customer-shaped corpus смогут выявить основные limits typed tables, recursive resolution и snapshots.
- **Предложение:** до реализации benchmark зафиксировать workload envelope: types/attributes/relations, identities/revisions, BOM depth/fanout, tenant count, concurrency, cache state и latency/throughput SLO.
- **Почему важно:** без заранее объявленного envelope любой benchmark можно назвать успешным, а продуктовые ожидания останутся неопределёнными.
- **Риск ошибочного решения:** оптимизация игрушечной схемы, пропущенный PostgreSQL catalog/table explosion, memory/compile/IDE cost или resolver latency на реальном graph.
- **Кто подтверждает:** product owner, first customer/IPS data expert, performance engineer, DB/query/codegen owners.
- **Закрывающее свидетельство:** versioned benchmark spec and dataset generator, hand-written comparison, repeatable CI/perf environment, plan regression and published limits.
- **Этап решения:** workload spec до фазы 3 data model freeze; итоговые thresholds и evidence — фаза 8/I0.

### OQ-048. Cutover с IPS и удаление старого generic kernel Alvatune

**Приоритет:** P0.

- **Факт:** ADR-068 запрещает dual writable generic Record/Revision/Relation/ChangeSet truth; Alvatune skeletons ещё содержат устаревшие descriptors, а migration from a customer IPS installation требует explicit parity baseline. [09 §1 и §16](09-alvatune-and-agents.md), [06 §19](06-ips-to-interlink-delta.md).
- **Допущение:** безопасный переход возможен через inventory→read-only extraction→identity mapping→differential validation→single cutover, без длительного dual write.
- **Предложение:** разделить два cutover playbook: внутренний Alvatune legacy skeleton/store и внешний IPS customer migration; для каждого определить authority switch, freeze window, reconciliation и rollback point.
- **Почему важно:** самый правильный target kernel можно скомпрометировать временным мостом, который станет постоянной второй системой истины.
- **Риск ошибочного решения:** divergent histories, duplicate identities, lost links/contexts/customizations или код продолжает писать старые tables.
- **Кто подтверждает:** migration lead, Interlink/Alvatune architecture, IPS customer owner, DBA/SRE и audit.
- **Закрывающее свидетельство:** full inventory, mapping ledger, dual-read differential results, write-path deny tests, rehearsed cutover/restore and signed authority switch.
- **Этап решения:** Alvatune internal path до I1 permanent writes; customer path до каждого migration contract.

## 7. Реестр ключевых допущений

Эта таблица собирает допущения, которые повторяются в нескольких вопросах. Она не заменяет подробные записи выше.

| ID | Допущение | Связанные вопросы | Что его опровергнет | Действие при опровержении |
|---|---|---|---|---|
| `A-01` | Раздельные Identity/Revision и per-type tables масштабируются до целевого envelope. | OQ-001, 003, 047 | Каталог, storage или планы систематически выходят за принятый SLO. | Новый measured storage ADR; не вводить generic EAV автоматически. |
| `A-02` | Customer-specific parity можно доказать outcomes/fixtures без bug-compatible копии внутренней физики IPS. | OQ-012–022, 048 | Критичный downstream зависит от неописанного внутреннего side effect. | Расширить baseline/adapter либо явно исключить capability из scope. |
| `A-03` | Own Changeset Draft должен быть видим инженеру раньше Effectivity, а outside context не видим. | OQ-012, 024, 025 | Пользовательские tests показывают требование иного precedence. | Отдельный versioned policy/profile и новый ADR, не скрытая ветка. |
| `A-04` | Constant coordinates достаточно для OC-A; path propagation можно безопасно отложить в OC-B. | OQ-015–017 | Первый целевой продукт требует nested/path overrides для минимального сценария. | Поднять OC-B prerequisites и пересмотреть порядок roadmap. |
| `A-05` | Full Snapshot приемлем по объёму для business evidence и первого technical projection. | OQ-006, 026, 047 | Build/read/storage benchmark не проходит envelope. | Оптимизировать projection отдельно; не ослаблять full baseline/order. |
| `A-06` | Host способен выдать complete access predicate/fingerprint и whole-artifact decision. | OQ-028, 032, 034 | Policy нельзя выразить до fetch либо fingerprint не покрывает mutation. | Отключить cache/snapshot/agent scenario либо расширить explicit seam. |
| `A-07` | Schema-set-per-tenant удовлетворяет isolation/cost первому масштабу. | OQ-031, 043, 045 | Adversarial test или pool/catalog cost нарушает SLO. | Рассмотреть database/cell profile отдельным решением. |
| `A-08` | Offline migrations допустимы до появления fleet/SLA, требующего rolling upgrade. | OQ-011, 043, 044 | Первый production contract не допускает window. | Поднять expand/contract work до go-live; не маскировать outage. |
| `A-09` | L1/L2 agents дают ценность до autonomous writes. | OQ-034–039 | Concierge/pilot не улучшает outcome или пользователи не доверяют proposal. | Сменить wedge или остановить agent scope до нового evidence. |
| `A-10` | Durable definition/runtime state boundary достаточна без второго object store в Alvatune. | OQ-033, 038, 048 | Reporting/transaction requirement требует копии mutable domain payload. | Добавить projection/reference contract, не второй writable kernel. |

## 8. Консолидированный реестр рисков

| ID | Риск | Вероятность до проверки | Ущерб | Ранний индикатор | Основной контроль | Владелец | Связанные вопросы |
|---|---|---:|---:|---|---|---|---|
| `R-01` | Ошибочная IPS parity из-за версии/configuration resolver-а. | Высокая | Критический | Один fixture выбирает разные revisions в источниках/installation. | Customer baseline + differential suite. | Product/migration | OQ-012–022 |
| `R-02` | 100% состав неверен из-за options/substitutions/path semantics. | Средняя | Критический | Sibling override leak, ambiguous variant, N:M mismatch. | OC-A/OC-B golden structures. | Configuration/metamodel | OQ-015–020 |
| `R-03` | Order/Baseline перестаёт быть точным историческим свидетельством. | Средняя | Критический | Старый результат меняется после rule/model/source update. | Full immutable Snapshot + retained provenance. | Snapshot/records | OQ-019, 026–030 |
| `R-04` | Generated/schema profile не выдерживает масштаб. | Средняя | Высокий | Catalog/compile/plan cost растёт нелинейно. | Predeclared phase-8 envelope and benchmarks. | Performance/DB | OQ-001–004, 047 |
| `R-05` | Customer customization превращается в скрытый fork/drift. | Высокая | Высокий | Runtime edits нельзя экспортировать/reconcile. | Ownership tiers, patch export, strict drift. | Metamodel/migration | OQ-005, 040, 042 |
| `R-06` | Migration повреждает fixed history или оставляет mixed-version fleet. | Средняя | Критический | Partial checkpoint, incompatible reader/writer, failed restore. | Preflight, journal, cohort rollout, restore drill. | Migration/SRE | OQ-011, 043–045 |
| `R-07` | Cross-tenant data/cache leak. | Низкая до теста, недопустимая | Критический | StorageIdentity/schema/access mismatch. | Adversarial two-tenant suite and least privilege. | Platform/security | OQ-008, 028, 031–032 |
| `R-08` | Host/kernel commit и audit/outbox расходятся. | Средняя | Критический | Object exists without event or duplicate retry. | One enlisted transaction + recovery protocol. | Runtime/ChangeAudit | OQ-030, 033 |
| `R-09` | Agent получает лишний context или выполняет неутверждённый side effect. | Средняя | Критический | Broad fetch then filter, proposal treated as approval. | Reauthorization, ContextSnapshot minimization, typed Actions. | Agents/security | OQ-034–036 |
| `R-10` | External PLM observation маскируется под current local truth. | Высокая | Высокий | Missing version/freshness/authority label. | Explicit authority mode and reconciliation. | Integrations | OQ-036–037 |
| `R-11` | Alvatune создаёт второй generic object/version kernel. | Средняя | Критический | Writable Record/Revision/ChangeSet tables reappear. | ADR-068 schema/type review and deny tests. | Architecture | OQ-038, 048 |
| `R-12` | Техническая реализация не подтверждает рыночную ценность. | Высокая | Высокий | Agent demo без manual outcome/paid commitment. | Concierge pilot and go/no-go gates. | Product | OQ-039 |
| `R-13` | Старое evidence невозможно прочитать/проверить. | Средняя | Критический | Missing reader/model/file/key after restore. | Dependency-closure retention and cold restore. | Records/SRE | OQ-026–027, 029, 045 |
| `R-14` | Наблюдаемость раскрывает данные либо не объясняет решение. | Средняя | Высокий | Raw payload/SQL in logs; no stable reason. | Structured redacted telemetry + explain artifact. | Observability/security | OQ-046 |

## 9. Очередность закрытия

### До принятия фазы 2 / I0 artifact lock

- OQ-007: `CodegenVersion` compatibility.
- OQ-041: exact module lock/resolution.
- OQ-047: benchmark envelope должен быть определён до того, как реализация начнёт оптимизироваться под случайный corpus.

### До writer и реальной БД, фаза 3

- API границы OQ-002, OQ-005, OQ-020, OQ-023, OQ-033.
- OQ-042: strict drift/admission.
- Offline foundations OQ-044 и migration journal.

### До G3/G4/resolver, фазы 4–6

- OQ-004, OQ-012–014, OQ-023, OQ-025, OQ-032 и reason часть OQ-046.
- Customer fixtures OQ-022 должны существовать до заявления parity.

### До projection и завершения PoC, фаза 8/I0

- OQ-006, OQ-009–010 и OQ-047.
- Technical publication/recovery/retention части OQ-027 и OQ-030.

### До full versioning/host seam, I1

- OQ-024, OQ-028–033, OQ-038 и внутренний cutover OQ-048.
- Agent writes выше L2 не открываются до доказательства этого gate.

### До tenant fleet, I2

- OQ-008, OQ-011, OQ-031, OQ-043–045.

### До OC-A/OC-B и business order

- OC-A: OQ-013–016, OQ-019, OQ-023, OQ-026–030.
- OC-B: OQ-016–018 и полная OQ-020.
- AutoMatch/domain automation: OQ-021 после usable typed query/Changeset path.

### До Alvatune A2–A4

- Federation A2: OQ-036–037.
- Agent L1/L2 A4: OQ-032, OQ-034–035, OQ-039.
- A3 paid alpha требует OQ-039, operational baseline OQ-045–046 и честно выбранный authority mode.

## 10. Протокол закрытия вопроса

Вопрос переводится в «закрыт» только когда одновременно выполнены условия:

1. Назван конкретный scope: версия IPS/customer profile, module set, storage profile и use case.
2. Факт подтверждён источником/кодом, а не только пересказом этой базы.
3. Допущение было подвергнуто попытке опровержения.
4. Эксперимент воспроизводим: inputs, environment, expected/actual и artifacts сохранены.
5. Product specialist подтвердил outcome, а technical owner — реализуемость и invariants.
6. Решение закреплено в подходящем нормативном документе/ADR и связано с tests.
7. Обновлены [карта возможностей](02-current-state-and-roadmap.md), [сопоставление IPS](06-ips-to-interlink-delta.md), [реестр источников](15-source-register.md) и Customer Configuration Baseline, если вопрос customer-specific.
8. Указано, какие риски сняты, какие приняты и какие новые вопросы возникли.

До выполнения этого протокола формулировка для внешнего общения должна оставаться: **«целевая гипотеза/открытый вопрос; подтверждается на таком-то gate»**, а не «Interlink уже обеспечивает».

## 11. Ключевые первичные маршруты проверки

Короткие ID во всех пунктах раскрываются в [реестре источников](15-source-register.md). Для вопросов с наибольшим риском ошибочной интерпретации следует начинать со следующих файлов, а не с пересказа базы знаний:

- нормативные решения Interlink: [`docs/12-decisions.md`](../../12-decisions.md), [`docs/versioning/02-specification.md`](../../versioning/02-specification.md), [`docs/06-migrations.md`](../../06-migrations.md), [`docs/15-host-integration.md`](../../15-host-integration.md);
- современная реконструкция подбора IPS: [аспект 04](<../../../../PLM Systems/IPS/Aspects/04 — Подбор версий, применяемость, конкретизация и аналоги.md>) и [аспект 05](<../../../../PLM Systems/IPS/Aspects/05 — Контексты редактирования, варианты и допустимые замены.md>);
- исторические алгоритмы, которые нужно проверять differential tests: [подбор версий](<../../../../PLM Systems/IPS/Knowledge/03 Подбор версий и фильтрация состава.md>) и [контексты редактирования](<../../../../PLM Systems/IPS/Knowledge/04 Контексты редактирования и проведение изменений.md>);
- заказ/производственная конфигурация IPS: [аспект 13](<../../../../PLM Systems/IPS/Aspects/13 — Конфигурации, заказы, экземпляры, партии и MRP.md>);
- принятая downstream-граница Alvatune: [ADR-068](../../../../Alvatune/.root/Adr/ADR-068_interlink_object_foundation.md), [ADR-069](../../../../Alvatune/.root/Adr/ADR-069_schema_per_tenant_isolation.md), [ADR-070](../../../../Alvatune/.root/Adr/ADR-070_agent_centric_command_center.md), [паспорт перехода](../../../../Alvatune/.root/Passport/42_INTERLINK_FOUNDATION_TRANSITION.md) и [agent runtime/evaluation](../../../../Alvatune/.root/Passport/18_AGENT_RUNTIME_AND_EVALUATION.md).

Если прямой источник и эта продуктовая база расходятся, действует иерархия доказательств из [раздела 2 реестра](15-source-register.md#2-иерархия-доказательств), а расхождение регистрируется новым вопросом, а не сглаживается редактурой.
