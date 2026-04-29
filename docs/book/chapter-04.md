# Chapter 4: The ORM Save Pipeline — From Entity to SQL INSERT

## What This Chapter Covers

When you call `$table->save($entity)`, CakePHP does far more than run one SQL statement.
It validates the entity, fires events, handles associations, compiles parameterized SQL, and
finally calls PHP's built-in PDO layer to execute the query.

By the end of this chapter you will be able to:

- Follow the complete call chain from `save()` to `PDOStatement::execute()`
- Know exactly which file and line handles each step
- Understand how the ORM layer and the Database layer are separated
- Subscribe to any of the five save events
- Read query logs and debug a failing insert

---

## The 30,000-Foot Map

```
Your code
  $table->save($entity)
        |
        v
  [ORM layer — src/ORM/]
  Table::save()                             :1942
    └─ Table::_processSave()                :2015
         ├─ checkRules()  (events: beforeRules, afterRules)
         ├─ dispatchEvent('Model.beforeSave')   :2034
         ├─ saveParents()  (BelongsTo associations)
         ├─ Table::_insert()               :2135  <── new entity
         │    └─ insertQuery()             :1755
         │         └─ ORM\InsertQuery      (src/ORM/Query/InsertQuery.php:26)
         │              └─ Database\InsertQuery
         │                   .insert(cols)       :62
         │                   .values(data)       :106
         │                   .execute()
         └─ dispatchEvent('Model.afterSave')    :2111
        |
        v
  [Database layer — src/Database/]
  Query::execute()                          :235
    └─ Connection::run($query)              :299
         └─ Driver::run($query)             :315
              ├─ Driver::prepare($query)    :399
              │    └─ Query::sql()          :293
              │         └─ Driver::compileQuery()  :924
              │              └─ QueryCompiler::compile()  :96
              │                   ├─ _buildInsertPart()  :422
              │                   │    "INSERT INTO articles (title, body)"
              │                   └─ _buildValuesPart()  :446
              │                        "VALUES (:c0, :c1)"
              │    └─ PDO::prepare(sql) → PDOStatement
              └─ Driver::executeStatement() :331
                   └─ Statement::execute()  :142
                        └─ PDOStatement::execute()  ← PHP's own PDO
        |
        v
  MySQL / PostgreSQL / SQLite
```

---

## Section 1 — The Entry Point: `Table::save()`

**File:** `src/ORM/Table.php:1942`

```php
public function save(
    EntityInterface $entity,
    array $options = [],
): EntityInterface|false {
```

### What a "table object" is

In CakePHP, a *Table class* represents a database table plus all the rules, associations,
and behavior that belong to it. `ArticlesTable` maps to the `articles` database table.
You get one by calling `$this->fetchTable('Articles')` inside a controller, or
`TableRegistry::getTableLocator()->get('Articles')` anywhere else.

The `save()` method is the single public door through which all insert and update
operations pass.

### Default options

```php
$options = new ArrayObject($options + [
    'atomic'         => true,   // wrap the whole save in a DB transaction
    'associated'     => true,   // also save related records (HasMany etc.)
    'checkRules'     => true,   // run application-level rules (unique email, etc.)
    'checkExisting'  => true,   // if PK supplied, check whether row exists first
    '_primary'       => true,   // internal: is this the outer save() call?
    '_cleanOnSuccess'=> true,   // mark entity as clean after save
]);
```

**`atomic => true`** means the entire save — including associated records — is wrapped in
a single database transaction. If anything fails, the whole thing rolls back.

### Early exits

```php
if ($entity->hasErrors((bool)$options['associated'])) {
    return false;
}
if ($entity->isNew() === false && !$entity->isDirty()) {
    return $entity;   // nothing changed — skip the database entirely
}
```

If the entity has validation errors, `save()` returns `false` immediately without touching
the database. If the entity is not new and nothing has changed (no dirty fields), it also
returns immediately — no wasted SQL round-trip.

### The transaction wrapper

```php
$success = $this->_executeTransaction(
    fn() => $this->_processSave($entity, $options),
    $options['atomic'],
);
```

`_executeTransaction` calls `$connection->transactional(fn)`. It runs the closure inside
a `BEGIN` / `COMMIT` block. If the closure throws, it issues a `ROLLBACK`.

### afterSaveCommit

```php
if ($success) {
    if ($this->_transactionCommitted($options['atomic'], $options['_primary'])) {
        $this->dispatchEvent('Model.afterSaveCommit', compact('entity', 'options'));  // :1970
    }
}
```

`Model.afterSaveCommit` fires *after* the transaction is fully committed, making it safe
to send emails or push to a queue — things you must not do inside an uncommitted transaction.

---

## Section 2 — The Save Orchestrator: `Table::_processSave()`

**File:** `src/ORM/Table.php:2015`

This protected method contains the core save logic. It runs every time `save()` is called
on any table. Here is its simplified skeleton:

```
_processSave()
  1. check if PK exists → decide new vs existing          :2019
  2. checkRules()  (fires beforeRules, afterRules events)  :2029
  3. dispatchEvent('Model.beforeSave')                     :2034
  4. _associations->saveParents()  (BelongsTo)             :2056
  5. extract dirty fields from entity                      :2067
  6. _insert()  OR  _update()                              :2070
  7. _onSaveSuccess() → saveChildren() + afterSave event   :2077
```

### Step 2 — Application rules

```php
$mode = $entity->isNew() ? RulesChecker::CREATE : RulesChecker::UPDATE;
if ($options['checkRules'] && !$this->checkRules($entity, $mode, $options)) {
    return false;
}
```

Rules are things like "email must be unique across the users table" or "user_id must
reference a real row in users". They run *after* basic field validation, *before* the SQL
fires. The events `Model.beforeRules` and `Model.afterRules` fire inside
`checkRules()` — see `src/Datasource/RulesAwareTrait.php:61` and `:73`.

### Step 3 — `Model.beforeSave`

```php
$event = $this->dispatchEvent('Model.beforeSave', compact('entity', 'options'));  // :2034
if ($event->isStopped()) {
    return $event->getResult();
}
```

A listener can stop the event (preventing the save) by calling `$event->stopPropagation()`.
It can also return a modified entity or `false`. This is where behaviors like `TimestampBehavior`
set `created` and `modified` fields.

### Step 5 — Extracting dirty fields

```php
$data = $entity->extract($this->getSchema()->columns(), true);  // :2067
```

`extract(columns, onlyDirty: true)` returns only the fields that have changed since the
entity was last cleaned. For a new entity that means every non-null field. This is the
`$data` array that gets passed to `_insert()`.

### Step 6 — The new/update branch

```php
if ($isNew) {
    $success = $this->_insert($entity, $data);   // :2071
} else {
    $success = $this->_update($entity, $data);
}
```

---

## Section 3 — Building the Insert: `Table::_insert()`

**File:** `src/ORM/Table.php:2135`

```php
protected function _insert(EntityInterface $entity, array $data): EntityInterface|false
```

This method does two things:

1. Resolves the primary key value (auto-increment ID, UUID, or composite key)
2. Builds and executes the INSERT query via the fluent interface

### The critical three lines

```php
$statement = $this->insertQuery()->insert(array_keys($data))  // :2176
    ->values($data)
    ->execute();
```

Read this left to right:

| Call | Returns | Does |
|------|---------|------|
| `insertQuery()` | `ORM\InsertQuery` | creates a blank insert query for this table |
| `->insert(array_keys($data))` | same query | registers the column names |
| `->values($data)` | same query | attaches the row data |
| `->execute()` | `StatementInterface` | compiles SQL, sends it to PDO |

### `insertQuery()` — the factory

```php
public function insertQuery(): InsertQuery   // :1755
{
    return $this->queryFactory->insert($this);
}
```

`queryFactory` is a `QueryFactory` object injected into the Table. It creates an
`ORM\InsertQuery` (not `Database\InsertQuery` directly) and passes `$this` (the Table) so
the query knows which table name and type bindings to use.

---

## Section 4 — The ORM/Database Bridge: `ORM\InsertQuery`

**File:** `src/ORM/Query/InsertQuery.php:26`

```php
class InsertQuery extends DbInsertQuery
{
    use CommonQueryTrait;

    public function __construct(Table $table)
    {
        parent::__construct($table->getConnection());   // :37
        $this->setRepository($table);
        $this->addDefaultTypes($table);
    }
```

The ORM `InsertQuery` extends `Database\InsertQuery`. Its constructor takes a `Table`
object and:

- Passes the Table's database connection up to the parent
- Stores the Table as the "repository" (so it can read the table name)
- Calls `addDefaultTypes($table)` to register PHP→SQL type mappings (e.g. `DateTime` → a
  MySQL `DATETIME` string)

### Why does it override `sql()`?

```php
public function sql(?ValueBinder $binder = null): string   // :46
{
    if (empty($this->_parts['into'])) {
        $repository = $this->getRepository();
        $this->into($repository->getTable());   // e.g. "articles"
    }
    return parent::sql($binder);
}
```

The `into()` call is *lazy* — it reads the table name from the repository just before SQL
is compiled, not at construction time. This means you can rename the table object's logical
name at any point before `execute()` is called.

---

## Section 5 — The Query Builder: `Database\InsertQuery`

**File:** `src/Database/Query/InsertQuery.php:27`

This is a pure SQL builder. It knows nothing about CakePHP ORM entities or PHP types.

### `insert(columns)` — registering column names

```php
public function insert(array $columns, array $types = [])   // :62
{
    $this->_parts['insert'][1] = $columns;
    if (!$this->_parts['values']) {
        $this->_parts['values'] = new ValuesExpression($columns, $this->getTypeMap()->setTypes($types));
    }
    return $this;
}
```

When you call `.insert(['title', 'body'])`:

1. The column list is stored in `$this->_parts['insert'][1]`
2. A `ValuesExpression` object is created to hold future row data

### `ValuesExpression` — the placeholder factory

**File:** `src/Database/Expression/ValuesExpression.php:34`

`ValuesExpression` accumulates rows. When it is time to generate SQL it loops over each
row, asks a `ValueBinder` to generate a unique placeholder name (`:c0`, `:c1`, …), and
records the actual value alongside its type.

```php
public function sql(ValueBinder $binder): string   // :214
{
    foreach ($this->_values as $row) {
        foreach ($columns as $column) {
            $placeholder = $binder->placeholder('c');   // ":c0", ":c1", …
            $rowPlaceholders[] = $placeholder;
            $binder->bind($placeholder, $value, $types[$column]);
        }
    }
    return sprintf(' VALUES (%s)', implode('), (', $placeholders));
}
```

This is how SQL injection is prevented: the actual values **never** appear in the SQL
string — only named placeholders do. PDO binds the real values at execution time.

### `values(data)` — adding a row

```php
public function values(ValuesExpression|Query|array $data)   // :106
{
    /** @var \Cake\Database\Expression\ValuesExpression $valuesExpr */
    $valuesExpr = $this->_parts['values'];
    $valuesExpr->add($data);
    return $this;
}
```

Adds the row array to the `ValuesExpression`. You can call `values()` multiple times to
build a multi-row insert.

---

## Section 6 — Triggering Execution: `Query::execute()`

**File:** `src/Database/Query.php:235`

```php
public function execute(): StatementInterface
{
    $this->_statement = null;
    $this->_statement = $this->_connection->run($this);
    $this->_dirty = false;
    return $this->_statement;
}
```

`execute()` hands the query object to the connection. The query object carries:

- `$this->_parts` — the structured pieces (insert columns, values, etc.)
- `$this->getValueBinder()` — the binder that holds placeholder→value mappings

Nothing SQL-like has been generated yet at this point. The string compilation happens
inside `run()`.

---

## Section 7 — The Connection Layer: `Connection::run()`

**File:** `src/Database/Connection.php:299`

```php
public function run(Query $query): StatementInterface
{
    return $this->getDisconnectRetry()->run(
        fn() => $this->getDriver($query->getConnectionRole())->run($query)
    );
}
```

Two things happen here:

1. **Disconnect retry**: if the database connection was dropped (e.g. MySQL idle timeout),
   `getDisconnectRetry()` transparently reconnects and retries once before surfacing the
   error to your application.

2. **Driver selection**: `getConnectionRole()` returns either `'write'` or `'read'`. For an
   INSERT it is always `'write'`. CakePHP supports read/write replica setups out of the box.

---

## Section 8 — The Driver: Compile, Prepare, Execute

**File:** `src/Database/Driver.php`

The `Driver` class is the PDO wrapper. One concrete subclass exists per database engine
(`Mysql`, `Postgres`, `Sqlite`, etc.).

### Step 1 — `Driver::run()` at line 315

```php
public function run(Query $query): StatementInterface
{
    $statement = $this->prepare($query);         // :317
    $query->getValueBinder()->attachTo($statement);   // bind values
    $this->executeStatement($statement);         // :319
    return $statement;
}
```

Three sub-steps: prepare the SQL, attach bound parameters, execute.

### Step 2 — `Driver::prepare()` at line 399

```php
public function prepare(Query|string $query): StatementInterface
{
    $statement = $this->getPdo()->prepare(
        $query instanceof Query ? $query->sql() : $query   // :402
    );
    return new (static::STATEMENT_CLASS)($statement, $this, …);
}
```

`$query->sql()` is called here for the first time. Tracing it:

```
Query::sql()                      src/Database/Query.php:293
  └─ getDriver()->compileQuery()  src/Database/Driver.php:924
       └─ QueryCompiler::compile()  src/Database/QueryCompiler.php:96
```

### Step 3 — `Driver::compileQuery()` at line 924

```php
public function compileQuery(Query $query, ValueBinder $binder): string
{
    $processor = $this->newCompiler();
    $query = $this->transformQuery($query);   // dialect-specific tweaks
    return $processor->compile($query, $binder);
}
```

`transformQuery()` is where driver-specific SQL dialects are applied. The MySQL driver,
for example, may rewrite certain expressions to match MySQL syntax.

---

## Section 9 — SQL String Assembly: `QueryCompiler`

**File:** `src/Database/QueryCompiler.php`

`QueryCompiler` is responsible for turning the structured `_parts` array into a SQL string.

### The INSERT parts list (line 79)

```php
protected array $_insertParts = ['comment', 'with', 'insert', 'values', 'epilog'];
```

`compile()` walks this list in order, calling a `_build*Part()` method for each non-empty
entry. For a typical insert only `insert` and `values` are populated.

### `compile()` at line 96

```php
public function compile(Query $query, ValueBinder $binder): string
{
    $sql = '';
    $type = $query->type();    // "insert"
    $query->traverseParts(
        $this->_sqlCompiler($sql, $query, $binder),
        $this->{"_{$type}Parts"},                // $_insertParts
    );
    return $sql;
}
```

`traverseParts` iterates `$_insertParts` and for each part name calls
`$this->_sqlCompiler(…)`. The compiler closure calls `_buildInsertPart()` or
`_buildValuesPart()` depending on the part name.

### `_buildInsertPart()` at line 422

```php
protected function _buildInsertPart(array $parts, …): string
{
    $table   = $parts[0];    // "articles"
    $columns = $this->_stringifyExpressions($parts[1], $binder);  // ['title', 'body']
    return sprintf('INSERT%s%s INTO %s (%s)', $hint, $modifiers, $table, implode(', ', $columns));
}
```

Output: `INSERT INTO articles (title, body)`

### `_buildValuesPart()` at line 446

```php
protected function _buildValuesPart(array $parts, …): string
{
    return implode('', $this->_stringifyExpressions($parts, $binder));
}
```

This calls `ValuesExpression::sql($binder)` which produces ` VALUES (:c0, :c1)`.

Combined result: `INSERT INTO articles (title, body) VALUES (:c0, :c1)`

---

## Section 10 — Binding Values and Firing PDO

Back in `Driver::run()` (line 315), after `prepare()` returns the wrapped `PDOStatement`:

```php
$query->getValueBinder()->attachTo($statement);
```

`ValueBinder::attachTo()` loops over every `placeholder → value → type` triplet registered
during `ValuesExpression::sql()` and calls `$statement->bindValue(…)` for each one.
CakePHP's type system converts PHP values (e.g. a `DateTime` object) into the SQL string
format the driver expects before binding.

### `Driver::executeStatement()` at line 331

```php
protected function executeStatement(StatementInterface $statement, ?array $params = null): void
{
    if ($this->logger === null) {
        $statement->execute($params);   // :335
        return;
    }
    // logger path: measure time, log query, handle exception
    $start = microtime(true);
    $statement->execute($params);       // :348
    $took = (float)number_format((microtime(true) - $start) * 1000, 1);
    // … log $took ms
}
```

When a logger is attached (via `$connection->setLogger(…)`), the method times the
execution and logs the final SQL with parameter values.

### `Statement::execute()` at line 142 — the finish line

**File:** `src/Database/Statement/Statement.php:142`

```php
public function execute(?array $params = null): bool
{
    return $this->statement->execute($params);
}
```

`$this->statement` is PHP's own `PDOStatement` object. This single line is where the SQL
leaves PHP and enters the database engine. From here MySQL/Postgres/SQLite handles the
write and returns a row count.

---

## Section 11 — Events Fired Along the Way

All five events pass `compact('entity', 'options')` as data.

```
Order of events for a NEW entity
─────────────────────────────────────────────────────────────────────
Event                  Fired in                                 Line
─────────────────────────────────────────────────────────────────────
Model.beforeRules      src/Datasource/RulesAwareTrait.php        :61
Model.afterRules       src/Datasource/RulesAwareTrait.php        :73
Model.beforeSave       src/ORM/Table.php                        :2034
  — INSERT executes here —
Model.afterSave        src/ORM/Table.php                        :2111
  — transaction COMMITS here —
Model.afterSaveCommit  src/ORM/Table.php                        :1970
─────────────────────────────────────────────────────────────────────
```

### Subscribing to an event (practical example)

```php
// In your Table class
public function initialize(array $config): void
{
    $this->getEventManager()->on('Model.beforeSave', function ($event, $entity, $options) {
        $entity->set('slug', Text::slug($entity->title));
    });
}
```

Or use `implementedEvents()` in a Behavior:

```php
public function implementedEvents(): array
{
    return ['Model.beforeSave' => 'beforeSave'];
}

public function beforeSave(EventInterface $event, EntityInterface $entity, ArrayObject $options): void
{
    // runs before every save on any table that loads this behavior
}
```

### When to use which event

| Event | Use for |
|-------|---------|
| `Model.beforeRules` | inject context data into the rules checker |
| `Model.beforeSave` | stamp timestamps, slugs, hashed passwords |
| `Model.afterSave` | update denormalized counts, trigger cache clear |
| `Model.afterSaveCommit` | send emails, push to queues, external API calls |

---

## Section 12 — Debugging an INSERT

### Enable query logging

```php
// In config/app.php or bootstrap.php
use Cake\Database\Log\QueryLogger;
$connection = \Cake\Datasource\ConnectionManager::get('default');
$connection->getDriver()->setLogger(new QueryLogger());
```

Every executed query is now written to the configured log (usually `logs/queries.log`).

### Read back the SQL before executing

```php
$query = $table->insertQuery()
    ->insert(['title', 'body'])
    ->values(['title' => 'Hello', 'body' => 'World']);

debug($query->sql());   // INSERT INTO articles (title, body) VALUES (:c0, :c1)
debug($query->getValueBinder()->bindings());  // [':c0' => 'Hello', ':c1' => 'World']
```

### Check the statement's row count

```php
$statement = $query->execute();
echo $statement->rowCount();  // 1 on success, 0 on failure
```

### Common failure points

| Symptom | Where to look |
|---------|--------------|
| `save()` returns `false`, no exception | `$entity->getErrors()` — validator rejected a field |
| `save()` returns `false`, no errors | `Model.beforeSave` listener returned `false` or stopped the event |
| `DatabaseException: no primary key` | `src/ORM/Table.php:2143` — table has no PK defined in schema |
| `PDOException: SQLSTATE[23000]` duplicate entry | unique constraint violation — check your Rules |
| Missing columns in INSERT | field not in `$this->getSchema()->columns()` — not in `_columns` of schema |

---

## Complete Call Chain Reference

| Step | File | Line | What happens |
|------|------|------|-------------|
| 1 | `src/ORM/Table.php` | 1942 | `save()` validates entity, wraps in transaction |
| 2 | `src/ORM/Table.php` | 2015 | `_processSave()` fires events, checks rules, branches |
| 3 | `src/ORM/Table.php` | 2067 | extracts dirty fields from entity |
| 4 | `src/ORM/Table.php` | 2135 | `_insert()` resolves PK, calls fluent query chain |
| 5 | `src/ORM/Table.php` | 1755 | `insertQuery()` creates `ORM\InsertQuery` via QueryFactory |
| 6 | `src/ORM/Query/InsertQuery.php` | 26 | ORM bridge: sets table name, adds default types |
| 7 | `src/Database/Query/InsertQuery.php` | 62 | `insert(cols)` stores column list, creates `ValuesExpression` |
| 8 | `src/Database/Query/InsertQuery.php` | 106 | `values(data)` adds row data to ValuesExpression |
| 9 | `src/Database/Query.php` | 235 | `execute()` calls `$connection->run($this)` |
| 10 | `src/Database/Connection.php` | 299 | `run()` retry wrapper, selects write driver |
| 11 | `src/Database/Driver.php` | 315 | `run()` calls `prepare()` then `executeStatement()` |
| 12 | `src/Database/Query.php` | 293 | `sql()` calls `driver->compileQuery()` |
| 13 | `src/Database/Driver.php` | 924 | `compileQuery()` creates QueryCompiler, transforms query |
| 14 | `src/Database/QueryCompiler.php` | 96 | `compile()` walks `$_insertParts` list |
| 15 | `src/Database/QueryCompiler.php` | 422 | `_buildInsertPart()` → `INSERT INTO articles (title, body)` |
| 16 | `src/Database/Expression/ValuesExpression.php` | 214 | `sql()` → ` VALUES (:c0, :c1)` + binds values |
| 17 | `src/Database/QueryCompiler.php` | 446 | `_buildValuesPart()` delegates to ValuesExpression::sql() |
| 18 | `src/Database/Driver.php` | 399 | `prepare()` calls `PDO::prepare(sql)` → PDOStatement |
| 19 | `src/Database/Driver.php` | 331 | `executeStatement()` calls `$statement->execute()` |
| 20 | `src/Database/Statement/Statement.php` | 142 | `execute()` calls `PDOStatement::execute()` |

---

## Key Concepts Introduced in This Chapter

**Dirty tracking** — An entity knows which fields have changed since it was loaded or
last cleaned. `entity->isDirty()` checks the whole entity; `entity->isDirty('title')`
checks one field. Only dirty fields are included in INSERT/UPDATE SQL.

**Fluent interface** — Methods return `$this` so you can chain calls:
`$query->insert(…)->values(…)->execute()`. Each call mutates internal state and returns
the same object.

**Prepared statements** — SQL is compiled with named placeholders (`:c0`, `:c1`).
The actual values are bound separately via `PDO::bindValue()`. The database engine
treats placeholder values as data, never as SQL syntax — this is the core defense
against SQL injection.

**Type mapping** — CakePHP's type system (classes in `src/Database/Type/`) converts PHP
values to the format the database expects. A PHP `DateTime` becomes `'2024-01-15 09:30:00'`.
An `array` can become a JSON string. Types are registered per-column in the schema.

**Two InsertQuery classes** — `src/ORM/Query/InsertQuery.php` and
`src/Database/Query/InsertQuery.php` have the same name in different namespaces.
The ORM one extends the Database one and adds table-name resolution and type bindings.
This layering lets the Database layer work independently of the ORM.

---

## What's Coming in Chapter 5

This chapter traced one entity going **into** the database. Chapter 5 will trace data
coming **out**: how a `SelectQuery` becomes a list of entity objects. You will see how
`QueryCompiler` builds a SELECT statement, how `ResultSet` lazily hydrates rows into
entities as you iterate, and how associations (BelongsTo, HasMany) are loaded — either
through JOIN or through separate queries with the IN strategy.
