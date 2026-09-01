# 03. Понятийная модель платформы

## 1. Карта уровней

```text
Поставка модели
  organizational module / versioned delivery unit → DSL definitions → composed model → normalized IR → manifest/hash

Предметная модель
  current: ObjectType / RelationType / TupleType / AttributeDefinition / ValueList / Traversal / Query / Maturity
  target post-PoC: Axis / named Policy и другие объявленные definitions

Предметные данные
  Identity → Revision/Draft
  Revision ─Occurrence→ Identity
  Relation / Tuple / Attribute values / Tags

Работа с данными
  generated descriptors + records + queries + commands
  ResolutionContext → resolved revisions + reasons
  Changeset → atomic publication
  Snapshot → immutable or rebuildable exact graph

Хост
  authorization / workflow / agents / files / UI / integrations / tenant control
```

Каждый уровень имеет отдельную ответственность. Смешение уровней — основной источник неверных ожиданий: например, наличие `ObjectType` не означает готовую карточку объекта, а наличие `SnapshotKind.Order` в контракте не означает работающий заказ.

## 2. Model и Module

### Model

Скомпонованная предметная модель — проверенный набор определений с единой maturity scale и нормализованным смыслом. Её hash не зависит от несущественного порядка исходных файлов, но зависит от семантики IR.

### Module

Любой модуль — единица организации исходников и порядка `after`, а не предметный объект. Только модуль с `version` становится delivery unit:

- versioned delivery unit имеет обязательный physical-name `prefix`;
- может объявлять minimum dependency `requires` в пределах совместимого MAJOR;
- владеет stable IDs под своим префиксом;
- получает собственное пространство generated API;
- присутствует в `modules[]` manifest;
- не встраивает принадлежность к поставке в normalized domain IR.

Versionless module не является поставляемым пакетом: у него нет `prefix`/`requires`/собственной записи `modules[]`, и он остаётся организационной частью составленной модели.

Ядро проверяет уже выбранный pinned-набор модулей, но не решает зависимости как package manager. Выбор exact versions принадлежит хосту/сборке продукта. [IL-DSL §2, «Модуль как единица поставки»], [IL-ADR ADR-053], [IL-PLAN-2].

## 3. Definition

Definition — стабильное описание элемента модели. Основные виды:

- `ObjectTypeDefinition`;
- `RelationTypeDefinition`;
- `TupleTypeDefinition`;
- `AttributeDefinition` и `AttributeBinding`;
- `ValueListDefinition`;
- `TraversalDefinition`;
- maturity scale/stage;
- `QueryDefinition` и maturity scale/stage;
- axis/named policy и другие объявленные контракты — только в целевых post-PoC фазах.

Общий минимум definition — machine name и stable ID. Человекочитаемые тексты, ownership и другие свойства принадлежат только тем видам definitions, где они объявлены; например, Traversal и Query не имеют общего `DefinitionTexts`/ownership-контракта. Stable ID отвечает за идентичность при rename и между deployment-ами; имя — за читаемость и generated API. [IL-META §1], [IL-DSL §5], [IL-ADR ADR-017, ADR-047, ADR-054].

## 4. ObjectType

Тип объекта задаёт:

- versioned или nonversioned семантику;
- abstract/concrete иерархию;
- стратегию хранения `joined` либо `table-per-hierarchy`;
- физические имена под module prefix;
- атрибуты identity/revision/nonversioned scope;
- Caption, icons, extensibility и taggability;
- допустимые связи и generated forms.

Interlink не использует одну универсальную таблицу экземпляров. Concrete type получает собственное type-owned отображение; при TPH несколько типов разделяют объявленную таблицу и discriminator. Поэтому принцип «типизированная физика» точнее, чем упрощённое «ровно одна таблица на каждый тип». [IL-META §2], [IL-DB §3], [IL-ADR ADR-002–004].

## 5. Identity, Revision и Draft

### Identity

Устойчивый предмет: «что это». Один UUID служит PK и переносимой идентичностью. Человекочитаемое обозначение остаётся атрибутом.

### Revision

Зафиксированное инженерное состояние identity: «каково это состояние». Вне mutable-zone содержимое неизменяемо. Новое исправление создаёт следующую revision, а не переписывает историю.

### Draft

Изменяемое незавершённое состояние внутри Changeset. Draft существующей identity строится от unique fixed head; историческая revision может быть источником копирования содержания, но не создаёт параллельную ancestry-ветвь.

### Maturity

Упорядоченная стадия инженерной зрелости revision. Process-state вроде «на согласовании» не является maturity и принадлежит хостовому workflow.

[IL-PLM §1, §6], [IL-VERS-CONCEPTS §1–§2], [IL-VERS-SPEC §2–§3].

## 6. Relation и Occurrence

### Relation

First-class типизированная запись с identity, атрибутами и endpoint constraints. Пример: документ описывает деталь.

### Occurrence

Специализированная структурная relation, которая означает место использования:

```text
точная Revision родителя ──Occurrence──> Identity ребёнка
```

Occurrence хранит свойства места:

- quantity и unit;
- position/find number;
- sort order;
- exact pin, если есть;
- произвольные объявленные relation attributes уже входят в текущий IR/G2 (например, `Quantity`);
- effectivity и variant condition добавляются в будущих фазах.

Количество и позиция не принадлежат карточке ребёнка. Выпуск новой revision ребёнка не переписывает родителей: exact revision выбирается при чтении. [IL-PLM §2–§3], [IL-META §3], [IPS-A03].

### Occurrence thread

`occurrence_thread_id` — identity позиции через revisions родительского состава:

- новая позиция получает новую нить;
- скопированная в следующий состав сохраняет нить;
- внешние факты о месте могут ссылаться на реестр нитей;
- полный live-address использует не одну нить, а `StructureDefinitionId + ExactRootRevisionId + ThreadPath`.

Это предотвращает ошибочное сопоставление позиций по target, номеру или порядку. [IL-PLM §2], [IL-ADR ADR-050, ADR-052].

## 7. Endpoint scope

Каждый конец relation явно адресует identity либо exact revision. Различие влияет на семантику:

- identity → identity: устойчивая предметная связь, policy не нужна;
- revision → identity: occurrence, где ребёнок должен быть разрешён;
- revision → revision: точная историческая связь;
- endpoints на tuple допускают структурно идентифицируемый контекст.

Не всякая ссылка на versioned object должна автоматически выбирать Current. Разрешение возникает только там, где контракт требует revision. [IL-META §3.2–§3.3], [IL-QUERY §2], [IL-PLM §4].

## 8. AttributeDefinition и Binding

Определение атрибута задаёт тип и домен. Binding прикрепляет определение к типу и задаёт scope/storage/required semantics.

Три режима прикрепления:

- `static` — обычная колонка/вычисляемое поле для каждого экземпляра;
- `on-demand` — определение известно типу, но прикрепляется конкретному экземпляру явно;
- `open` — экземпляр extensible-типа может получить определение из разрешённого пула.

Даже open-атрибут не является строковым key/value: значение всегда управляется typed definition. Популярное расширение может быть promoted из JSONB-container в колонку migration-ой. [IL-META §5, §6], [IL-ADR ADR-008, ADR-026–027].

## 9. Reference и Relation — разные вещи

Reference attribute подходит, когда ссылка:

- функционально является значением поля;
- не требует собственной identity;
- не несёт значимых атрибутов, жизненного цикла и навигационной фразы;
- бывает scalar или ordered list.

Relation нужна, когда связь:

- сама имеет identity/атрибуты;
- участвует в графе и where-used;
- имеет endpoint constraints, cardinality, phrase/inverse phrase;
- требует отдельного ownership/audit/commands.

Смешение этих механизмов создаёт неявные графы и разные правила удаления. [IL-META §3, §5.3], [IL-ADR ADR-005, ADR-038, ADR-047].

## 10. TupleType

Tuple — nonversioned тип со структурной идентичностью по набору typed slots. Пример: `PartAtPlant`.

Назначение:

- моделировать контекст комбинации объектов без искусственного «объекта-связки»;
- гарантировать uniqueness комбинации;
- быть endpoint relation;
- получать собственные attributes, Caption, RowVersion и typed queries/commands.

`GetOrCreateTuple` не является generic upsert: generated facade знает точные slot types и валидирует структуру. [IL-META §4], [IL-ADR ADR-028], [IL-POC §4, сценарий 13].

## 11. Descriptor, Record, Link и Catalog

### Descriptor (G1)

Immutable typed metadata для model-aware кода. Descriptor знает definition, effective attribute bindings, endpoints и query primitives; он не выполняет I/O.

### Record (G2)

Generated typed DTO результата: identity, revision, canonical projection, nonversioned object, relation или tuple. Это не универсальный domain primitive `Record(kind,json)`.

### Link/Pair/Node (G2)

Generated формы relation/occurrence и результата навигации. Occurrence node может нести exact resolved target и `ResolutionReason`; identity-only link не обязан разрешать revision.

### Catalog

Immutable model catalog объединяет descriptors и позволяет находить definitions по stable ID. PostgreSQL `meta` catalog — развёрнутая deployment projection для validation/migrations/runtime extension, а не второй авторитет sealed-модели.

[IL-CODEGEN §2–§4], [IL-ARCH §3–§4].

## 12. Query

Потребитель описывает запрос через generated roots/descriptors и object algebra:

```text
G3/G4 fluent or named query
  → Query AST
  → binding / normalization / polymorphism / context injection / optimization
  → SQL IR
  → PostgreSQL SQL + ordinal materialization
```

G3 отвечает за точечное и обычное чтение. G4 — за Children/Parents/Descendants/Ancestors/Rollup/Traversal, streaming и snapshots.

Имена таблиц поступают только из проверенного manifest, значения — только параметры. Raw SQL и stored procedures не являются extension point. [IL-QUERY §1–§6, §10, §14], [IL-ADR ADR-010–011, ADR-023].

## 13. ResolutionContext и Resolved result

`ResolutionContext` — явный набор фактических входов выбора revision:

- общая policy;
- per-edge policy map;
- snapshot selection;
- coordinates;
- changeset id;
- fallback.

Целевой порядок:

```text
Snapshot → Exact Pin → Changeset overlay/override → Effectivity → Policy → explicit Fallback → NoMatch
```

Результат содержит selected revision и reason. Plain DTO может хранить reason в коррелированной `QueryExecutionTrace`, но исход `NoMatch` не исчезает. Один и тот же context не обещает вечный результат при изменившихся данных; для межвременной воспроизводимости нужен exact graph или Snapshot. [IL-PLM §4], [IL-QUERY §7, §12].

## 14. Command и Changeset

### Generated command facade

Typed API конкретного object/relation type. Он принимает typed values/patch/options и `expectedRowVersion`, а не generic JSON.

Это целевая generated surface. Сейчас sample `Commands.cs` — рукописный frozen specimen с phase gates; текущий `ModelGenerator` команды ещё не выпускает.

### Минимальный writer PoC

Низкоуровневые операции фаз 3:

- create/update/commit/drop draft;
- create/update/delete nonversioned;
- replace ordered list atomically;
- insert/update/delete relation;
- tuple, tag и maturity primitives.

### Full Changeset

Единственный контейнер незавершённого изменения после PoC:

- affected-object locks;
- Draft и typed pending operations;
- target maturity/effectivity;
- preview/validate/approval hash;
- atomic commit/abandon/preempt;
- read-overlay для собственного context.

Хостовый ChangeCase/Workflow координирует Changeset, но не создаёт второй object-write mechanism. [IL-VERS-SPEC §3, §7, §10], [IL-ARCH §6], [ALV-ADR068].

## 15. Snapshot

Три kind имеют разную продуктовую роль:

| Kind | Роль | Жизненный цикл | Статус |
|---|---|---|---|
| `projection` | Техническое ускорение exact full graph | Новый snapshot при перестроении; старый может быть удалён privileged maintenance | Минимальный PoC-срез, фаза 8 |
| `baseline` | Бессрочное бизнес-свидетельство конфигурации | Append-only, не обновляется/не удаляется | После PoC |
| `order` | Точный состав одной передачи заказа | Append-only; у живого заказа может быть много публикаций | После PoC |

Snapshot хранит exact root, entries, topology, resolved revisions, occurrence payload, reasons, normalized context и exact model/policy/customization provenance. Он не является произвольно отфильтрованной выгрузкой: materializer обязан пройти полную snapshot-safe структуру до листьев, а любой `NoMatch` отклоняет весь batch. [IL-PLM §5], [IL-VERS-SPEC §5, §7–§8].

## 16. Ownership модели

| Уровень | Кто владеет | Допустимое изменение |
|---|---|---|
| `sealed` | поставщик модуля | Только новая версия DSL + migration |
| `customizable` | предприятие в объявленной границе | Настройка через catalog, three-way reconcile и export patch |
| runtime extension | администратор tenant | Новый `AttributeDefinition` и при необходимости поддерживающий runtime `ValueList` в разрешённом extensible pool; audit/export/promotion |

Изменяемость выдаётся явно, а не наследуется всеми definitions. Effective runtime model может включать customer delta, но baseline и envelope его изменяемости определены DSL. [IL-ARCH §4], [IL-META §6], [IL-MIG §6].

## 17. Tenancy и StorageContext

Полная мультитенантность не входит в PoC. Однако контракты заранее требуют:

- `StorageIdentity` в plan/result/memo cache keys;
- хостом созданный `StorageContext`, а не request-derived schema name;
- прикладной query/command API не требует передавать physical schema/table names вручную; infrastructure/compiler descriptors и доверенный host `StorageContext` эти имена содержат;
- разделение session admission/authorization и object semantics;
- отдельный migration/deployment profile для каждого tenant binding.

Для Alvatune целевой профиль — `control` + `t_<opaque>_meta` + `t_<opaque>_data` + `t_<opaque>_app`, но это решение хоста, не общий metamodel primitive Interlink. [IL-HOST], [ALV-ADR069], [ALV-TRANSITION §4].

## 18. ActorContext и audit seam

Каждая команда получает actor/correlation context, достаточный для различения:

- человека;
- агента;
- system service;
- действия от имени accountable user;
- correlation с host run/case/action.

Interlink хранит предметный след изменения. Подробный agent trace, prompt/tool calls, budgets и evaluations хранит Alvatune. Два журнала связываются immutable identifiers/hashes, но не дублируют друг друга. [IL-HOST §3–§4], [ALV-AGENT §4–§6].

## 19. Инварианты, определяющие продукт

1. Содержание и ancestry fixed Revision не изменяются на месте; disposition меняется только typed audited intents.
2. Occurrence принадлежит exact source revision и target identity.
3. Quantity/position принадлежат occurrence.
4. Pin target revision принадлежит target identity.
5. Revision-selecting query не имеет implicit context.
6. Resolver объясняет выбор или `NoMatch`.
7. Один object не редактируется двумя open Changesets без явного preemption.
8. Публикация Changeset атомарна.
9. Snapshot не бывает частичным.
10. Model/code/DB drift обнаруживается.
11. Runtime writes идут через typed command boundary.
12. Host authorization не превращается в скрытую фильтрацию состава business Snapshot.
13. Stable IDs и physical names versioned module принадлежат declared delivery unit; versionless module не получает такую package-область.
14. Unsupported capability fail-closed.

## 20. Статус понятий

| Понятие | Форма | Исполнение |
|---|---|---|
| Текущие Definitions/IR/manifest и composition modules | Реализовано | Реализовано в compiler; delivery-unit semantics только для versioned modules |
| Axis/named Policy как полноценные IR definitions | Целевой контракт | После PoC; текущие constants/carriers не равны готовым IR rows/grammar |
| Descriptors/records/links/catalog | Реализовано | Generated G1/G2 в фазе 2; приёмка фазы открыта |
| Commands | Заморожено | Фаза 3 |
| G3/G4 | Заморожено/частично library contracts | Фазы 4–6 |
| ResolutionContext carrier | Реализовано | Профили по фазам 4–6; full context после PoC |
| Occurrence thread/path | Контракт принят | Registry фаза 3, path use фазы 5+ |
| Changeset | Нормативно специфицирован | После PoC/I1 |
| Effectivity/configurator | Контракт принят | OC-A/OC-B после PoC |
| `projection` Snapshot | Контракт принят | Фаза 8 |
| `baseline`/`order` | Контракт принят | После PoC |
| Tenant profile | Host seam принят | I2 |
