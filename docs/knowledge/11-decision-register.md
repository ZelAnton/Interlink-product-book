# 11. Реестр ключевых решений Interlink

## 1. Назначение реестра

Этот документ переводит технические ADR в продуктовые обязательства: что именно Interlink обещает сохранить, каким способом, какую цену принимает и где решение пока существует только как контракт.

Нормативный источник — [12 — Решения и открытые вопросы](../../12-decisions.md). Здесь ADR не переопределяются. Если краткая формулировка ниже расходится с нормативным текстом, действует ADR. Состояние реализации сверяется отдельно с [02 — Текущее состояние и путь реализации](02-current-state-and-roadmap.md).

Дата среза: **31 августа 2026 года**.

### Как читать статус

Все ADR-001…054 приняты как архитектурные решения. Последняя колонка отвечает на другой вопрос — насколько решение исполнено:

- **реализовано** — соответствующий compiler/IR/G1/G2-срез присутствует в `main`;
- **локальная работа** — наблюдается только в незакоммиченном рабочем дереве и ещё не является поставкой;
- **контракт PoC** — форма и инварианты зафиксированы, исполняемый слой относится к фазам 3–8;
- **после PoC** — решение принято, но его основная продуктовая функция сознательно не входит в PoC;
- **смешанный** — части решения имеют разные сроки.

Наличие carrier-типа, сигнатуры или теста с `PhaseNotImplementedException` не считается реализацией функции.

## 2. Цепочка решений

Ключевые решения образуют не независимый список, а причинную цепочку:

```text
DSL как источник + module delivery
  → нормализованный IR и stable identity
  → type-owned PostgreSQL storage и generated API
  → один typed command path
  → Identity / Revision / Occurrence
  → явный ResolutionContext и объяснимый resolver
  → полный Snapshot для воспроизводимого результата
  → host seam для workflow, прав, агентов и tenant control
```

Если удалить звено, меняется продуктовый смысл последующих решений. Например, Snapshot без exact model/policy provenance становится обычной выгрузкой, а Changeset при наличии второго пути записи перестаёт гарантировать атомарность изменения.

## 3. Реестр ADR-001…ADR-054

### 3.1. Источник модели и физика хранения

| KB ID | Решение и источник | Зачем принято | Последствия и ограничения | Исполнение |
|---|---|---|---|---|
| **KB-D-001** | DSL в репозитории — источник sealed-baseline и конверта расширения. [ADR-001](../../12-decisions.md) | Получать reviewable, воспроизводимую модель вместо неконтролируемого изменения production metadata. | `meta` — deployment projection, не второй авторитет; допустимая runtime-дельта обязана быть ограничена и экспортируема в DSL-patch. | DSL/compiler реализованы; DB catalog и export/reconcile — контракт фаз 3/7. |
| **KB-D-002** | Универсальные таблицы допустимы только для метаданных; EAV не является основной моделью. [ADR-002](../../12-decisions.md) | Перенести доменные инварианты в типизированную физику БД и исключить позднюю проверку «всего из JSON». | Больше таблиц/DDL/generated code; изменение модели требует migration, зато доступны FK, CHECK, NOT NULL и прогнозируемый SQL. | Модель/manifest/codegen реализованы; предметный DDL — фаза 3. |
| **KB-D-003** | Concrete type явно объявляет write-table mapping. [ADR-003](../../12-decisions.md) | Не позволять конвенции генератора молча изменить физическое отображение. | Авторы модели отвечают за стабильные физические имена; несовпадение — ошибка, а не эвристика. | Валидация/manifest реализованы; DDL — фаза 3. |
| **KB-D-004** | Наследование: `joined` по умолчанию либо `table-per-hierarchy`; TPC отклонён. [ADR-004](../../12-decisions.md) | Балансировать нормализацию и стоимость полиморфных чтений без дублирования базовых колонок. | Утверждение «каждый concrete type имеет отдельную таблицу» не буквально верно для TPH; joined требует JOIN. | IR/compiler/codegen реализованы; физическая проверка — фаза 3. |
| **KB-D-005** | Relation — first-class typed record со своей identity, таблицей, атрибутами и аудитом. [ADR-005](../../12-decisions.md) | Сохранить PLM-семантику факта связи и данных места использования. | Reference attribute не заменяет relation, если нужны атрибуты, inverse, traversal, pin или собственная identity. | Метамодель/G1/G2 реализованы; writer/query — фазы 3–6. |
| **KB-D-006** | Identity отделена от Revision. [ADR-006](../../12-decisions.md) | Развести «что это» и «каково его состояние», не переписывая связи при выпуске новой версии. | Versioned type получает identity/revision storage; атрибуты обязаны иметь scope. | Контракты и generated shapes реализованы; runtime — с фазы 3. |
| **KB-D-007** | Scope каждого endpoint явен; occurrence — `revision → identity` с optional pin. [ADR-007](../../12-decisions.md) | Делать необходимость подбора версии частью типа связи, а не неявной догадкой запроса. | Identity-target требует `ResolutionContext` только там, где нужна exact revision; revision-target не разрешается повторно. | Метамодель/codegen реализованы; исполняемый resolver — фазы 4–6. |
| **KB-D-008** | Runtime-расширения хранятся в локальном JSONB-контейнере типа и могут продвигаться в колонку. [ADR-008](../../12-decisions.md) | Сохранить управляемую динамичность IPS без превращения всех основных данных в EAV. | Нет произвольных строковых свойств; promotion требует migration/backfill, детали online-перехода открыты. | Формы IR/G2 реализованы; attach/query/promotion — фаза 7 и далее. |

### 3.2. Generated API, запросы и инфраструктурный профиль PoC

| KB ID | Решение и источник | Зачем принято | Последствия и ограничения | Исполнение |
|---|---|---|---|---|
| **KB-D-009** | Descriptor/catalog/query roots — immutable singletons в DI; I/O живёт в scoped session. [ADR-009](../../12-decisions.md) | Не связывать метаданные с сессией и не плодить статические global instances. | Generated registration обязателен; record/descriptor не удерживают connection. | G1 DI/catalog реализованы; query execution — фазы 4–6. |
| **KB-D-010** | Fluent/named query компилируется через AST → SQL IR; values только parameters. [ADR-010](../../12-decisions.md) | Обеспечить безопасность, анализируемость и единое место оптимизации. | Raw SQL и строковая конкатенация не extension point; `Condition` не `bool`, операторы AND/OR языка C# намеренно недоступны. | AST-контракты есть; полноценный compiler/render — фазы 4–6. |
| **KB-D-011** | Recursive CTE — базовый traversal; database cursor — явный opt-in streaming contract. [ADR-011](../../12-decisions.md) | Поддержать глубокие структуры без загрузки всего результата в память. | Max depth обязателен; `SEARCH`/`CYCLE`; cursors относятся к G4 и не должны появляться скрыто. | Контракт PoC; G4 — фаза 5/6. |
| **KB-D-012** | ResolutionContext всегда передаётся явно, результат содержит причину выбора. [ADR-012](../../12-decisions.md) | Устранить session/window-dependent результат и сделать подбор воспроизводимым/объяснимым. | Нельзя иметь невидимый «current context»; cache, trace и agents обязаны включать normalized context. | Carrier-контракт реализован; профили resolver — фазы 4–6 и после PoC. |
| **KB-D-013** | Единственный диалект PoC — PostgreSQL 18.6 с pinned OCI image; dialect не часть DSL. [ADR-013](../../12-decisions.md) | Сузить эксперимент и проверять точную платформу, сохраняя dialect-neutral algebra/IR. | SQL Server-first и матрица 16/18 отменены; upgrade 18.x — осознанное изменение с повтором contract tests. | Bootstrap реализован; generated DDL/query runtime — фазы 3–6. |
| **KB-D-014** | Полная мультитенантность вне PoC; schema выбирает доверенный StorageContext. [ADR-014](../../12-decisions.md) | Не раздувать PoC, но не зашить одну схему в public API/cache identity. | Schema-per-tenant — профиль хоста; request-derived schema names недопустимы. | Host seam принят; Alvatune tenant profile — I2/после PoC. |
| **KB-D-015** | CLI-first codegen; generated C# коммитится и проверяется `generate --check`. [ADR-015](../../12-decisions.md) | Делать изменение generated API видимым в review и не зависеть от Roslyn build magic. | Репозиторий содержит generator output; upgrade CodegenVersion может требовать отдельный diff. Roslyn adapter — только будущий frontend. | G1/G2 реализованы; CLI/check — локальная работа этапа 8, не принятый `main`. |
| **KB-D-016** | Metadata ownership: sealed default, customizable/extensible только явно; three-way reconcile. [ADR-016](../../12-decisions.md) | Совместить vendor upgrades с customer delta без тихого drift. | Runtime editor не может менять sealed semantics; нужны baseline, patch/export, conflict UX и tombstones. | Ownership в IR реализован; operational reconcile/export — фазы 3/7. |
| **KB-D-017** | Один UUID является PK identity/revision/relation; UUIDv7 для операций, UUIDv5 для deterministic seed. [ADR-017](../../12-decisions.md) | Убрать двойную бухгалтерию local ID ↔ global ID, характерную для IPS. | Более крупные FK/index keys; бизнес-порядок не выводится из UUID; внешний UUID можно adopt только для публичной identity/relation/tuple, не revision. | Типы/генерация ID-контракта реализованы; DB/writer и измерения — фазы 3/8. |
| **KB-D-018** | Конкурентность — `row_version bigint` и CAS. [ADR-018](../../12-decisions.md) | Обнаруживать потерянные обновления без скрытой долгой DB-блокировки. | IPS-style checkout может быть отдельным UX поверх, но не альтернативным write path. | RowVersion в generated DTO реализован; CAS writer — фаза 3. |
| **KB-D-019** | Current хранится как `is_current` на revision с partial unique index. [ADR-019](../../12-decisions.md) | Избежать циклической FK identity ↔ revision и удерживать единственность в БД. | Current-read платит JOIN; решение пересматривается только по benchmark. | Контракт DDL/runtime — фазы 3–4. |
| **KB-D-020** | Все DB names — `snake_case`, всегда schema-qualified, без `search_path`. [ADR-020](../../12-decisions.md) | Сделать SQL детерминированным и исключить подмену объекта через окружение сессии. | Manifest/StorageContext обязаны дать полное имя; ручной SQL не поддерживается. | Naming/manifest реализованы; enforcement DDL/query — фазы 3–6. |
| **KB-D-021** | Pin revision обязан принадлежать target identity; это composite FK. [ADR-021](../../12-decisions.md) | Не позволить «точной конкретизации» выбрать версию другого объекта. | Каждая pinnable occurrence-table несёт составной ключ; pin сильнее остальных слоёв resolver. | Contract/G2 реализованы; FK writer — фаза 3, resolver — фаза 5. |
| **KB-D-022** | Бизнес-логика не живёт в triggers; ACL/topology profile закрывает обход runtime-роли fail-closed. [ADR-022](../../12-decisions.md) | Сохранить один язык поведения и не позволить обычной runtime-role обходить command invariants. | БД хранит declarative guards; owner/DBA остаются trusted boundary. ACL drift, unsafe routine/trigger/inheritance closure блокируют deployment. | Детальный контракт; emitter/smoke — фаза 3. |
| **KB-D-023** | Named queries AOT-компилируются в приложение; stored procedures не являются механизмом ядра. [ADR-023](../../12-decisions.md) | Не заводить второй язык правил и второй versioning/deployment path. | Узкая будущая калитка только для generated, hashed model-owned SQL functions по измеренной необходимости. | Контракт; G3/G4 — фазы 4–6. |
| **KB-D-024** | Result cache — только explicit opt-in, с compiler-built dependency vector и write fence. [ADR-024](../../12-decisions.md) | Не допустить скрытой устарелости инженерного ответа после записи. | `Immutable` — default-deny; `projection` не immutable; multi-node требует durable transactional token/version registry. | Контракт PoC; in-process validation — фазы 4/8, multi-node после PoC. |
| **KB-D-025** | Полная стабильная структура загружается из materialized Snapshot, не через clustering базовых таблиц. [ADR-025](../../12-decisions.md) | Получить последовательное range-read и воспроизводимость контекстно выбранного DAG. | Только full snapshot-safe outgoing occurrence graph; arbitrary filter/prune/access запрещены; `baseline`/`order` append-only, `projection` rebuildable. | Carrier/physical contract; `projection` — фаза 8, business kinds — после PoC. |

### 3.3. Расширяемая метамодель и единый versioning path

| KB ID | Решение и источник | Зачем принято | Последствия и ограничения | Исполнение |
|---|---|---|---|---|
| **KB-D-026** | Attachment modes ортогональны ownership: `static`, `on-demand`, `open`. [ADR-026](../../12-decisions.md) | Различить обязательную форму типа и прикрепляемые свойства, не отказываясь от typed definitions. | Три состояния on-demand; open разрешает только определения из pool; значения остаются типизированными. | IR/G2 формы реализованы; runtime attach/query — фаза 7. |
| **KB-D-027** | Runtime attribute — полноценный AttributeDefinition в общем catalog с `cust.` namespace. [ADR-027](../../12-decisions.md) | Не создавать параллельную «вселенную дополнительных полей». | JSONB reference не имеет FK и проверяется command layer; promotion — штатная migration. | Contract; catalog/runtime — фаза 7. |
| **KB-D-028** | TupleType — nonversioned объект со структурной identity и typed slots. [ADR-028](../../12-decisions.md) | Моделировать контекстные комбинации вроде Part-at-Plant без искусственного versioned объекта. | `UNIQUE NULLS NOT DISTINCT`, typed get-or-create; tuple не заменяет occurrence и relation. | IR/G1/G2 реализованы; DDL/writer — фаза 3. |
| **KB-D-029** | Пятнадцать IPS-механизмов сводятся к восьми концептам и одному write path: Identity, Revision, Draft, Changeset, Maturity, Pin, Effectivity, ResolutionContext+Snapshot. [ADR-029](../../12-decisions.md) | Сохранить функции версий/изменений, устранив пересекающиеся сущности и неоднозначный порядок. | Fixed revision immutable; один exclusive affected-object lock; resolver order `snapshot → pin → changeset → effectivity → policy`; process state не Maturity. | Carriers/G1/G2 частично реализованы; single-revision PoC — фазы 3–6; full Changeset/Effectivity — после PoC. |
| **KB-D-030** | Instance navigation — explicit awaited methods; transparent lazy properties отклонены. [ADR-030](../../12-decisions.md) | Сделать I/O, session и ResolutionContext видимыми и избежать N+1/sync-over-async. | Lazy допускается только как AST до execute, async streaming, guarded memo и explicit resolve. | Generated method shapes частично реализованы; execution — фазы 4–6. |
| **KB-D-031** | DSL может seed system instances с deterministic UUID; seed идёт через тот же command path и имеет protection levels. [ADR-031](../../12-decisions.md) | Воспроизводимо поставлять справочники/системные объекты без второго DML-механизма. | ModelId неизменяем; versioned seed имеет одну revision; tombstone не даёт resurrect customer deletion. | Parse/IR/WellKnown реализованы; DB seed/reconcile — фазы 3/7. |
| **KB-D-032** | Computed attributes ограничены deterministic language; Caption обязателен; machine Name отделён от Label; icons — resource keys. [ADR-032](../../12-decisions.md) | Дать типизированное представление и UI metadata, не затаскивая forms/layout/произвольный код в DSL. | Stored computed ограничен одной таблицей; projected живёт в view; icon не binary asset. | Compiler/codegen формы реализованы; DB/view runtime — фаза 3+. |
| **KB-D-033** | Module order явен; model releases образуют chain; code migrations — reviewed offline steps. [ADR-033](../../12-decisions.md) | Сделать composition и data transition детерминированными и возобновляемыми. | Любой semantic IR diff требует model version bump; destructive change обязан сохранить fixed history/readers; rolling migration пока не обещана. | Composition/version checks реализованы; migration journal/steps — фаза 3; online — открыто. |
| **KB-D-034** | Traversal может объединять relation types; семантика `FilterEdge`, `FilterTarget`, `PruneWhen` различна. [ADR-034](../../12-decisions.md) | Корректно работать с неоднородным PLM-графом без скрытого Current для identity-only edges. | Нужен общий identity ancestor; revision enrichment nullable; multi-relation traversal — stretch. | Контракт G4; базовый срез — фаза 5, расширение позже. |
| **KB-D-035** | C# attributes как канонический источник модели отклонены; normalized IR frontend-neutral. [ADR-035](../../12-decisions.md) | Сохранить model-as-data, deterministic hash, seed, patches и будущий editor без Roslyn dependency. | C# builder остаётся test frontend; возможный Roslyn adapter не меняет канон и pipeline. | Реализовано в compiler architecture. |
| **KB-D-036** | Host-neutral seam: standalone/enlisted transaction, trusted StorageContext, ActorContext, CommitPermit. [ADR-036](../../12-decisions.md) | Встраивать object change, host audit/outbox и approval в одну атомарную операцию без зависимости Interlink → Alvatune. | Host owns outer commit/authorization; missing transaction notification leaves safe cache bypass; permit сверяется с exact hash. | Contracts/carriers приняты; runtime — фаза 3 и последующие; Alvatune integration — I0/I1. |
| **KB-D-037** | Types референсируемы как data через read-only metatype facet/`typeref`; полная метацикличность отклонена. [ADR-037](../../12-decisions.md) | Позволить ссылаться на definition и строить relations instance↔type, не редактируя schema через object API. | `meta.definition` — FK anchor; model semantics меняются только DSL/migration. | Contract PoC; registry/typeref DB — фаза 3, расширенные facets по потребности. |
| **KB-D-038** | Reference attribute имеет identity/revision scope; ordered lists — per-type many-table; relation boundary явна. [ADR-038](../../12-decisions.md) | Достичь IPS-паритета ссылок без превращения каждого указателя в relation. | Reference без собственных attributes/inverse/traversal; списки atomically replace; JSONB references без FK. | IR/G2 реализованы; DDL/query/writer — фазы 3–4. |
| **KB-D-039** | Tags — identity-only hierarchical pool; revision tags и tag-as-status/classification отклонены. [ADR-039](../../12-decisions.md) | Дать cross-cutting findability, не создавать теневую lifecycle/property system. | Assignment tables per taggable type; один tag assignment не несёт domain attributes. | Contract/G2 частично реализованы; writer/query — фазы 3–4. |
| **KB-D-040** | Classification — композиция class hierarchy + общего attribute pool + on-demand attachment, не отдельный xProperty universe. [ADR-040](../../12-decisions.md) | Сохранить PLM-классификаторы с едиными типами/валидацией и без дублирования метамодели Aras. | Один class на scheme, assignment на identity; в PoC не входит, нужен только glue Classify/Reclassify после проверки primitives. | После PoC. |
| **KB-D-041** | Перед freeze закреплены: одна DSL maturity scale, typed constraints, per-type writer facades, technical recursion anchor. [ADR-041](../../12-decisions.md) | Не допустить, чтобы PoC «доказал» только read-only carriers без единого способа записи и DB guards. | Нет generic EngineeringCommands/JSON; validators/CHECK/catalog должны происходить из одного constraint node; snapshot command до фазы 8 не исполняется. | Scale/constraints/signatures/codegen реализованы; writer — фаза 3, snapshot — фаза 8. |
| **KB-D-042** | Из IPS заимствуются явный Fallback, wildcard effectivity и file hashes в ContentHash; runtime selection-rule objects не заимствуются. [ADR-042](../../12-decisions.md) | Сохранить полезный outcome «не совсем подходящая» и исправить известные слабости IPS. | Без явного fallback остаётся `NoMatch`; конкретное head сильнее wildcard; file content входит в approval hash после появления vault. | Fallback carrier-контракт после PoC; policies — фаза 6; Effectivity/vault — после PoC. |

### 3.4. Закрытие развилок и заморозка публичных контрактов

| KB ID | Решение и источник | Зачем принято | Последствия и ограничения | Исполнение |
|---|---|---|---|---|
| **KB-D-043** | Границы метамодели закрыты conservatively: relation reification и anonymous tuple values отклонены; localization, pools и evolution имеют объявленный envelope. [ADR-043](../../12-decisions.md) | Не раздувать generic metamodel до появления доказанного сценария и сохранить один каталог definitions. | Unsupported generalization должна fail-closed; пересмотр — отдельный ADR и migration. Точные runtime pool operations фазируются. | Часть compiler rules реализована; operational extensibility — фаза 7/после PoC. |
| **KB-D-044** | Exclusive draft сохраняется с explicit preemption; named resolver policies компилируются; withdrawal необратим обычной promotion; projection имеет отдельный lifecycle. [ADR-044](../../12-decisions.md) | Закрыть concurrency и version-selection semantics без ветвления/merge и скрытых rule objects. | Preempt abandons весь Changeset с actor/reason; `projection` не бизнес-свидетельство и rebuild получает новый ID. | Policy/reason срез — фаза 6; projection — фаза 8; Changeset/preemption — после PoC. |
| **KB-D-045** | Единственная query surface — descriptor algebra; LINQ отвергнут; multi-step named query и test/explain tooling определены ядром. [ADR-045](../../12-decisions.md) | Сделать ResolutionContext обязательным типово и позволить compiler выбирать CTE/materialization. | Нельзя подмешать arbitrary SQL; generated API и plan tests становятся частью compatibility surface. | Контракт; G3/G4/tooling — фазы 4–6 и далее. |
| **KB-D-046** | Authorization всегда принадлежит host; Interlink применяет host predicates и проверяет CommitPermit hash. [ADR-046](../../12-decisions.md) | Не создавать конфликтующую вторую модель прав и не путать approval artifact с полномочием. | AccessFingerprint входит во все caches; permit opaque provenance не даёт право сам по себе; snapshot authorization — whole artifact. | Host seam принят; execution — фазы 3+ и Alvatune I1. |
| **KB-D-047** | Relation phrase/inverse-phrase обязательны; определения получают product-facing texts/cardinality/no-manual-create. [ADR-047](../../12-decisions.md) | Строить понятные предложения и UI/validation contracts из модели, а не угадывать грамматику по technical name. | `label` не заменяет predicate phrase; cardinality и manual-create flag не означают готовый UI. | Compiler/G1/G2 реализованы; enforcement/runtime — по фазам 3–4. |
| **KB-D-048** | Normalized IR описывает model shape; seeds вынесены; grammar/diagnostics/canonical expressions versioned и fail-closed. [ADR-048](../../12-decisions.md) | Стабилизировать hash и diagnostic contract до реализации lexer/parser. | Seed value change не bump-ит model version; named query меняет shape/hash; invalid compile не пишет partial output. | Реализовано в phase 1 compiler. |
| **KB-D-049** | Identity errors не позволяют построить MetamodelSnapshot. [ADR-049](../../12-decisions.md) | Не выполнять semantic validation над неоднозначным stable ID/name graph. | Ошибки identity short-circuit snapshot/IR; diagnostics формы не обещаются для неоднозначной модели. | Реализовано. |
| **KB-D-050** | Каждое occurrence имеет immutable `occurrence_thread_id`, переживающий revisions позиции. [ADR-050](../../12-decisions.md) | Сравнивать структуру и привязывать внешний факт к тому же месту, а не к target/position number. | Thread уникальна в source revision и копируется при draft/content copy; новое место получает новую thread; одной thread недостаточно для global path address. | IR/G2 carrier реализован; registry/FK/writer — фаза 3; diff use — фаза 5+. |
| **KB-D-051** | Digital-twin foundation вводится объявленными axis/historized/signal contracts, но instance twin остаётся отдельным потребителем. [ADR-051](../../12-decisions.md) | Не закрыть архитектуру для as-designed/as-built/as-maintained, не расширяя текущий PoC. | Coordinates — model-declared canonical map; Unit/Lot/signals и bitemporal execution — post-PoC tracks, не скрытая функция ядра сегодня. | Carrier contracts частично реализованы; основная функция после PoC. |
| **KB-D-052** | Configuration contracts заморожены: threadref carve-out, три snapshot forms, Order как enterprise model, configurator contract, ThreadPath address, versioned language. [ADR-052](../../12-decisions.md) | Не выпустить G2 API, непригодный для заказов, вариантов и reused subassemblies. | Live address — `(StructureDefinitionId, ExactRootRevisionId, ThreadPath)`; Order не core primitive и может иметь много handoff snapshots; OC-A/OC-B не входят в PoC. | Carriers/codegen частично реализованы; registry — фаза 3; configuration/order/diff — после PoC. |
| **KB-D-053** | Versioned module — delivery unit с prefix/requires/namespace/manifest membership; host pins versions, core только validates. [ADR-053](../../12-decisions.md) | Независимо поставлять domain modules и избегать collision `Task`/`task` без schema-per-module. | `requires` — minimum within MAJOR; нет resolver в core; stable IDs принадлежат prefix; model сохраняет одну migration chain. | Phase 1.6 compiler/manifest и G1/G2 namespaces реализованы; `meta.module`/deployment — фаза 3. |
| **KB-D-054** | Definition names case-insensitive в generation scope; inherited attached binding хранится в declaring-type container; оставшиеся compiler decisions назначены фазам. [ADR-054](../../12-decisions.md) | Не получать CLS/Windows collisions и не переносить данные при появлении joined descendant. | `Root`/`root` — ошибка; descendant container хранит только собственные bindings; некоторые literal/query checks сознательно ждут first consumer phase. | Case/name и relevant G2 rules реализованы; назначенные query/DDL checks — фазы 3–7. |

## 4. Решения, наиболее важные для разговора с IPS

Из 54 ADR продуктовый разговор обычно упирается в десять обязательств:

| Обязательство | Решения | Что меняется относительно IPS |
|---|---|---|
| Состав сохраняет асимметрию `revision parent → identity child` | KB-D-005…007 | Базовая инженерная физика IPS сохранена. |
| Подбор всегда имеет explicit context и reason | KB-D-012, 029, 042, 044 | Нет ambient context окна и молчаливого fallback. |
| Hard concretion становится Pin с DB-инвариантом | KB-D-021, 029 | Exact choice проверяем и сильнее policy. |
| Editing context/soft concretion становятся Changeset overlay | KB-D-029, 036, 044 | Один путь изменения вместо пересекающихся session mechanisms. |
| Applicability становится typed Effectivity над axes | KB-D-029, 042, 051 | Строковые диапазоны IPS заменяются нормализованными range assertions. |
| Position получает identity между revisions | KB-D-050, 052 | Сопоставление не зависит от target, номера или sort order. |
| Точная конфигурация публикуется full Snapshot | KB-D-025, 044, 052 | История не пересчитывается по текущим rules/context. |
| Metadata компилируется, customer delta ограничена | KB-D-001, 016, 026, 027 | Свободная runtime-метамодель заменяется тремя управляемыми скоростями изменения. |
| Все записи идут через typed commands | KB-D-022, 031, 036, 041 | Нет scripts/triggers/procedures как параллельного языка бизнес-правил. |
| Workflow, access, UI и agents принадлежат host | KB-D-036, 046, 051, 052 | Interlink остаётся object/version kernel, а не монолитной PLM-системой. |

## 5. Отвергнутые и заменённые альтернативы

### 5.1. Отвергнуты как целевая архитектура

| Альтернатива | Почему отвергнута | Действующая замена |
|---|---|---|
| Универсальная instance-table/EAV | Слабые FK/типы, дорогие запросы, поздняя проверка. | Type-owned storage + generated schema (KB-D-002…004). |
| Table-per-concrete-type | Дублирование базовых колонок и тяжёлый polymorphic union. | Joined или TPH (KB-D-004). |
| Скрытый current/editing context | Разный ответ между окнами/background jobs; невидимый cache input. | Explicit ResolutionContext (KB-D-012). |
| Пара local bigint + global UUID | Двойные indexes и постоянная translation/ошибки identity. | Один UUID PK (KB-D-017). |
| `identity.current_revision_id` | Циклическая FK identity↔revision. | `revision.is_current` + partial unique (KB-D-019). |
| Business logic в triggers/Methods/stored procedures | Второй язык и путь version/deployment/write. | Typed commands + compiled query algebra (KB-D-022/023). |
| Automatic result cache | Скрытая stale engineering data. | Explicit opt-in + dependency vector/fence (KB-D-024). |
| CLUSTER marker на базовых rows состава | DAG и context-dependent expansion не имеют одного физического порядка. | Materialized Snapshot rows (KB-D-025). |
| Transparent lazy-loading properties | Hidden I/O, N+1, lost ResolutionContext. | Awaited navigation/async streaming (KB-D-030). |
| C# attributes как source of truth | Не выражают model-as-data, seed и deterministic semantic normalization. | DSL → normalized IR; adapters могут быть дополнительными (KB-D-035). |
| Полная метацикличность «тип есть редактируемый Item» | Schema drift и два versioning mechanisms типов. | Read-only metatype facet + typeref (KB-D-037). |
| Revision tags или tag-as-classification | Теневая шкала состояния/параллельные свойства. | Identity tags; Maturity; отдельная classification composition (KB-D-039/040). |
| Отдельный xProperty universe | Дублирует definitions/domain/validation. | Общий AttributeDefinition pool (KB-D-040). |
| Relation reification и anonymous tuple value | Неясные cascades/version semantics и невыразимый domain. | Attributes на relation либо named TupleType (KB-D-028/043). |
| Runtime-interpreted selection rule objects | Снова несколько языков правил и mutable per-window behaviour. | Compiled named policies + explicit fallback (KB-D-042/044). |
| Changeset queue или branch+automatic merge | Для инженерного состава нет безопасной общей merge semantics. | Exclusive draft + explicit audited preemption (KB-D-044). |
| LINQ как основная query surface | Нельзя типово обязать ResolutionContext; ошибки дают правдоподобный неверный ответ. | Descriptor algebra (KB-D-045). |
| Собственная модель прав Interlink | Конфликт с host authorization и двойной источник истины. | Host predicates + AccessFingerprint (KB-D-046). |
| Schema-per-module, per-module migration chains, dependency solver в core | Конфликт с tenant schemas; два порядка migrations; package-management не domain kernel. | Prefix/namespace, одна model chain, host-pinned module set (KB-D-053). |

### 5.2. Пересмотрены или отложены, но не отвергнуты навсегда

- SQL Server-first заменён PostgreSQL 18.6 для PoC; dialect-neutral IR сохраняет возможность другого renderer при отдельном обосновании (KB-D-013).
- Source generator перестал быть PoC-gate; Roslyn adapter остаётся допустимым дополнительным frontend (KB-D-015/035).
- Schema-per-tenant не входит в PoC, но является целевым host profile Alvatune (KB-D-014).
- Pessimistic checkout может появиться как UX поверх CAS/Changeset, но не как второй writer (KB-D-018).
- Generated SQL functions допустимы только после измеренного случая; arbitrary stored procedures не возвращаются (KB-D-023).
- Polymorphic endpoint registry, compact combined storage, online migrations и multi-node cache registry остаются открытыми измеримыми развилками, а не скрытыми roadmap promises.

## 6. Расхождения и устаревшие формулировки источников

Здесь зафиксированы не «ошибки, которые можно мысленно исправить», а места, где читатель обязан применить приоритет источников.

### KBD-C-001. README отстаёт от локальной фазы 2

[README](../../../README.md) всё ещё описывает `Interlink.CodeGen` как каркас, а `interlink generate` и CI regeneration check — как будущую работу. Текущее рабочее дерево содержит реализацию этапа 8, но она не закоммичена. Поэтому:

- README устарел относительно локальной работы;
- локальная работа не позволяет переписать историю `main` как «этап 8 принят»;
- нормативная продуктовая формулировка: G1/G2 в `main` реализованы до этапа 7, этап 8 — local WIP, final phase-2 acceptance открыта.

### KBD-C-002. «Собственная таблица каждого concrete type» — слишком сильный лозунг

[01 — Видение и принципы](../../01-vision-and-principles.md) говорит, что каждый concrete type отображается на собственную таблицу. [ADR-004](../../12-decisions.md) допускает TPH, где несколько типов делят таблицу, а joined использует несколько таблиц по иерархии. Канонический смысл: **никакой универсальной таблицы всех экземпляров; storage mapping принадлежит конкретному типу и явно объявлен**, но отношение «один type — ровно одна уникальная table» не является общим инвариантом.

### KBD-C-003. Детерминированность требует больше, чем «одинаковый normalized DSL»

Тот же [документ принципов](../../01-vision-and-principles.md) формулирует «одинаковый нормализованный DSL → побайтно одинаковый C# и SQL». Для product lock нужны как минимум exact compiler/format/CodegenVersion, module delivery manifest и deployment renderer profile; membership module намеренно не входит в normalized domain IR (ADR-053), а SQL dialect является deployment property (ADR-013). Значит, строгий reproducibility claim должен звучать как **одинаковые exact source/module lock + toolchain/artifact versions + deployment profile дают одинаковые артефакты**.

### KBD-C-004. Многошаговый named query уже решён ADR, но codegen-текст называет его open design

В конце [07 — Кодогенерация](../../07-codegen.md) многошаговый процедурный запрос ещё назван отдельным post-PoC/open design. [ADR-045](../../12-decisions.md) позднее принял `step <name> = <algebra expression>` внутри named query и оставил compiler выбор CTE/materialization. Приоритет у ADR-045; документация codegen требует синхронизации перед реализацией G3/G4.

### KBD-C-005. «Снимок заказа» в обзорном тексте не означает один Snapshot на весь жизненный цикл

[Product overview](../../../overview/README.md) в единственном числе описывает точный snapshot, на который ссылается заказ. Нормативный [09 — PLM-семантика](../../09-plm-semantics.md) и ADR-052 уточняют: Order — живой versioned domain object, а immutable `order` Snapshot создаётся **на каждую передачу**; у одного заказа может быть много snapshots. Один `SnapshotId` — частный случай, не cardinality contract.

### KBD-C-006. Общий язык «зафиксировать любой результат снимком» ограничен snapshot-safe graph

Обзорные материалы справедливо объясняют Snapshot как способ сохранить результат, но нормативный [09 — PLM-семантика](../../09-plm-semantics.md) ограничивает materialization: только полный outgoing occurrence traversal `source revision → versioned target identity → exact revision`, `cycles reject`. Incoming, identity→identity, revision-target, nonversioned и mixed traversals остаются live-only до нового ADR. Это функциональная граница, а не деталь оптимизации.

### KBD-C-007. Cookbook показывает будущие business snapshots рядом с PoC-кодом

[13 — Query cookbook](../../13-query-cookbook.md) содержит пример `SnapshotKind.Baseline`, но текст рядом отмечает, что PoC исполняет только `Projection`. Пример — frozen target API, не свидетельство runtime-функции. Тот же принцип относится к generated signatures команд и query classes будущих фаз.

## 7. Решения, которые нельзя считать закрытыми продуктово

ADR закрывает архитектурную форму, но не подтверждает product-market semantics. До разговора о паритете IPS нужны данные по следующим пунктам:

1. Какой точный порядок resolver использует каждая целевая установка IPS: современные и исторические источники расходятся.
2. В каких процессах fallback допустим только для диагностики, а где — для производства.
3. Достаточен ли exclusive affected-object Changeset для реальной параллельной работы.
4. Какие customer metadata обязаны оставаться live-editable и кто согласует promotion в DSL.
5. Какие order/baseline artifacts должны быть полностью автономными, включая значения и файлы, а не только ссылки на immutable revisions.
6. Как технологические структуры, маршруты, КТД, нормы и межцеховая применяемость отображаются на domain modules поверх object kernel.
7. Как host package manager выбирает module versions и показывает conflicts/locks оператору.
8. Какая production drift policy обязательна: block, quarantine либо warning.

Эти пункты относятся к [12 — Допущения, риски и открытые вопросы](12-open-questions-and-risks.md); их нельзя закрывать расширительным толкованием ADR.

## 8. Правило изменения реестра

Новое решение добавляется сюда только после одного из событий:

1. принят новый ADR либо пересмотрен существующий;
2. implementation evidence меняет только колонку статуса, не смысл решения;
3. product discovery обнаруживает, что принятый механизм не воспроизводит обязательный IPS outcome — тогда сначала фиксируется конфликт/вопрос, а не переписывается решение задним числом;
4. устаревшая формулировка нормативного документа синхронизирована — запись расхождения сохраняется как history note до следующего полного аудита базы.

## 9. Вывод

Главное решение Interlink — не отдельный DSL, PostgreSQL или code generator. Это связка: **compiled model, typed storage, one writer, explicit resolver и exact Snapshot при host-owned workflow/authorization**. Она сознательно сохраняет инженерные результаты IPS, но отвергает скрытый контекст, несколько языков правил и несколько путей записи. Почти все самые ценные продуктовые обещания этой связки пока находятся на уровне контрактов; PoC должен доказать их end-to-end, а не только компилируемость отдельных типов.
