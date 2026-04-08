# Chapter 1 — The Architecture of CakePHP: A Map Before the Journey

> **Reader level**: PHP beginner. OOP and PSR concepts are explained in context.
> **Goal**: By the end of this chapter you will be able to open any file in the
> `cakephp/cakephp` repository, know exactly why it exists, and explain how it
> connects to every other part of the system.

---

## 1.1 — What CakePHP Is (and Isn't)

Every web application does the same three things over and over: receive an HTTP
request, do something with data, send back an HTTP response. Without a
framework you write that plumbing yourself — URL parsing, database connections,
HTML rendering, security checks — for every project. A framework is a
pre-built, battle-tested set of conventions and libraries so you can skip the
plumbing and focus on your product.

**CakePHP** was created in 2005 by Michal Tatarynowicz, heavily inspired by
Ruby on Rails. The inspiration was specific: Rails showed that a framework with
strong naming conventions could eliminate enormous amounts of configuration
code. CakePHP brought that idea to PHP and has refined it for twenty years.
Unlike many PHP frameworks that are loosely stitched bundles of third-party
libraries, CakePHP has always been opinionated and cohesive: one team, one
philosophy, one place to look for answers.

**CakePHP 5.x** (the version in this repository) modernises that foundation
significantly:

| Requirement | Detail | Why it matters |
|---|---|---|
| PHP 8.2+ | Strict types everywhere | Every file begins with `declare(strict_types=1)` — PHP refuses to silently coerce wrong types, so bugs that would previously hide become immediate errors |
| PSR-7 HTTP messages | `laminas/laminas-diactoros` 3.x | Request and Response objects follow a vendor-neutral standard; any PSR-7-aware library can work with them |
| PSR-15 middleware | Industry standard pipeline | Consistent way to add cross-cutting concerns (auth, HTTPS enforcement, CSRF) without modifying the framework |
| PSR-11 DI container | `league/container` 5.x | Classes declare their dependencies; the container assembles them automatically |
| Dates | `cakephp/chronos` 3.x | Immutable date/time objects that extend PHP's built-in `DateTimeImmutable` |

**What `declare(strict_types=1)` means for beginners**: Without it, PHP
silently converts between types. `"5" + 3` becomes `8`, a function expecting
`int` silently accepts `"5"`. With `strict_types=1`, such conversions throw a
`TypeError` instead. Every file in CakePHP declares strict types, so when you
read a method signature like `function get(int $id)`, that really means only
integers are accepted — strings like `"5"` will crash loudly, which is far
easier to debug than a silent data corruption.

What CakePHP is **not**: it is not a micro-framework. It does not ask you to
assemble your own stack. You get the whole bicycle, pre-assembled, with a map.
The map is this chapter.

---

## 1.2 — The Master Architecture Diagram

Before any detail, here is the entire system at a glance. Every box corresponds
to real code you can open right now.

One key thing to understand before reading the diagram: **routing is not a
separate step that happens after the pipeline**. It happens *inside* the
pipeline, via `RoutingMiddleware`. By the time the request reaches the
Application, it already carries the controller name, action name, and URL
parameters as attributes — placed there by the routing middleware. This is
explained fully in Chapter 3; for now just note where routing sits.

```
                      ┌────────────────────────────────────────────────────┐
                      │                    THE BROWSER                     │
                      └─────────────────────────┬──────────────────────────┘
                                                │ HTTP Request
                                                ▼
┌───────────────────────────────────────────────────────────────────────────┐
│  src/Http/Server.php  (line 75  run())                                    │
│                                                                           │
│   1. app->bootstrap()    ← loads config, plugins, DB connections         │
│   2. app->middleware()   ← YOUR app fills the queue                      │
│   3. fires 'Server.buildMiddleware' event                                 │
│   4. Runner::run(queue, request, app)  ← executes the pipeline below     │
└───────────────────────────────────────┬───────────────────────────────────┘
                                        │
                                        ▼
┌───────────────────────────────────────────────────────────────────────────┐
│  MIDDLEWARE PIPELINE                                                       │
│  src/Http/MiddlewareQueue.php   src/Http/Runner.php                       │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  ErrorHandlerMiddleware  (src/Error/Middleware/)  ← outermost      │  │
│  │  catches all exceptions thrown by anything inside it               │  │
│  │  ┌───────────────────────────────────────────────────────────────┐ │  │
│  │  │  HttpsEnforcerMiddleware  (src/Http/Middleware/)              │ │  │
│  │  │  ┌─────────────────────────────────────────────────────────┐  │ │  │
│  │  │  │  BodyParserMiddleware  (src/Http/Middleware/)            │  │ │  │
│  │  │  │  ┌───────────────────────────────────────────────────┐   │  │ │  │
│  │  │  │  │  RoutingMiddleware  (src/Routing/)                 │   │  │ │  │
│  │  │  │  │  ← adds controller/action params to $request      │   │  │ │  │
│  │  │  │  │  ┌─────────────────────────────────────────────┐  │   │  │ │  │
│  │  │  │  │  │  BaseApplication::handle()                  │  │   │  │ │  │
│  │  │  │  │  │  src/Http/BaseApplication.php:343           │  │   │  │ │  │
│  │  │  │  │  │  → ControllerFactory → Controller → Action  │  │   │  │ │  │
│  │  │  │  │  └─────────────────────────────────────────────┘  │   │  │ │  │
│  │  │  │  └───────────────────────────────────────────────────┘   │  │ │  │
│  │  │  └─────────────────────────────────────────────────────────┘  │ │  │
│  │  └───────────────────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────┬───────────────────────────────────┘
                                        │ ResponseInterface travels back out
                                        ▼
┌───────────────────────────────────────────────────────────────────────────┐
│  CONTROLLER  src/Controller/                                              │
│    reads/writes Model, passes data to View via $this->set(...)           │
│                                                                           │
│  MODEL (ORM)  src/ORM/   src/Database/                                   │
│    Table classes run queries → return Entity objects                      │
│                                                                           │
│  VIEW  src/View/   templates/                                             │
│    renders PHP templates → writes HTML into the Response body            │
└───────────────────────────────────────┬───────────────────────────────────┘
                                        │ PSR-7 Response
                                        ▼
                      ┌────────────────────────────────────────────────────┐
                      │                    THE BROWSER                     │
                      └────────────────────────────────────────────────────┘
```

Every chapter of this book zooms into one of these layers. Chapter 1 is your
orientation map. The crucial thing to take away right now: **the pipeline is
the spine**. Every HTTP request passes through it. Every cross-cutting concern
(security, sessions, parsing) lives in a middleware layer. The MVC work happens
only at the very centre, after all the infrastructure layers have run.

---

## 1.3 — The #1 Philosophy: Convention Over Configuration

The single most important idea in CakePHP is this: **if you name things the
"right" way, the framework wires everything together automatically. You only
configure the exceptions.**

This is called **Convention Over Configuration** (CoC). Before it existed,
connecting a database table to a controller to a view required pages of XML or
PHP configuration. CakePHP eliminates that entirely for the common case.

A concrete example of what this saves you: to add a blog posts feature, you
create three files and CakePHP automatically connects them with zero
configuration:

```
src/Model/Table/BlogPostsTable.php   ← reads the 'blog_posts' database table
src/Model/Entity/BlogPost.php        ← represents one row
src/Controller/BlogPostsController.php← handles /blog-posts/* URLs
templates/BlogPosts/index.php        ← renders the list
```

CakePHP knows all of this by name alone. You never write a single line saying
"BlogPostsController uses BlogPostsTable". It is inferred.

**What happens when you break conventions?** Nothing catastrophic. Conventions
are defaults, not laws. You can override any of them in configuration. The
framework is simply doing a lot of work for you when you follow them, and
requiring you to be explicit when you do not. Always prefer conventions, and
override only when you have a genuine reason.

### The Inflector: The Engine Behind Conventions

The engine that performs all name transformations is
`src/Utility/Inflector.php` (524 lines, `@since 0.2.9` — almost as old as the
framework). It knows how to convert between every naming form English uses for
nouns:

```
pluralize("user")     → "users"        pluralize("person")  → "people"
singularize("people") → "person"       singularize("users") → "user"
camelize("blog_post") → "BlogPost"     underscore("BlogPost")→ "blog_post"
tableize("BlogPost")  → "blog_posts"   classify("blog_posts")→ "BlogPost"
variable("BlogPost")  → "blogPost"     humanize("blog_post") → "Blog Post"
```

These transformations are chained together to derive every file and class name
from one canonical noun — the database table name. The diagram below shows the
full chain. The arrows show the Inflector method applied at each step; each
step builds on the one before it:

```
  You name one thing:  "blog_posts"  (the database table)
                            │
               ┌────────────┼─────────────────────────────────┐
               │            │                                 │
               ▼            ▼                                 ▼
         tableize        classify                    camelize(pluralize())
        "blog_posts"    "BlogPost"                    "BlogPosts"
        (DB table)      (Entity class)                (Table class name)
                                                      → BlogPostsTable.php
               │
               │  camelize(pluralize()) + "Controller"
               ▼
         "BlogPostsController"
         → src/Controller/BlogPostsController.php
               │
               │  pluralize()
               ▼
         "BlogPosts/"
         → templates/BlogPosts/index.php
         → templates/BlogPosts/view.php
         → templates/BlogPosts/add.php
         → templates/BlogPosts/edit.php
               │
               │  variable()           variable(singularize())
               ▼                       ▼
         "$blogPosts"            "$blogPost"
         (plural view var)       (singular view var)
```

The trait that exposes these methods to every class that needs them is
`src/Core/ConventionsTrait.php` (156 lines). Here is a concrete method from it
that derives a foreign key column name:

```php
// src/Core/ConventionsTrait.php:56
protected function _modelKey(string $name): string
{
    [, $name] = pluginSplit($name);
    return Inflector::underscore(Inflector::singularize($name)) . '_id';
}
```

Tracing through this for `"BlogPosts"`:
1. `pluginSplit("BlogPosts")` → `[null, "BlogPosts"]` (strips plugin prefix if any)
2. `Inflector::singularize("BlogPosts")` → `"BlogPost"`
3. `Inflector::underscore("BlogPost")` → `"blog_post"`
4. `."_id"` → `"blog_post_id"`

So any table that has a `BlogPosts` relationship will automatically look for a
`blog_post_id` foreign key column. Zero configuration.

---

## 1.4 — The 6 Core Design Patterns

CakePHP is built on six design patterns. You do not need to have studied design
patterns before — each one is explained from scratch here. Understanding them
is understanding CakePHP itself, because every major namespace maps to one or
more of these patterns.

---

### Pattern 1: MVC (Model–View–Controller)

**What it is**: MVC splits an application into three distinct responsibilities
so that each part can change independently. The classic benefit: if your
designer wants to change how articles look (View), they do not need to touch
the database code (Model). If you want to switch from MySQL to PostgreSQL
(Model), the HTML templates (View) are not affected.

```
┌───────────────────────────────────────────────────────────────────────┐
│                           MVC SEPARATION                              │
│                                                                       │
│  ┌───────────────┐  SQL queries  ┌───────────────────────────────┐   │
│  │  MODEL (M)    │◀────────────▶│  DATABASE (MySQL/Postgres/...) │   │
│  │               │               └───────────────────────────────┘   │
│  │  src/ORM/     │                                                    │
│  │  src/Database/│  returns Entity objects (plain PHP objects         │
│  │  src/Datasource│ representing one database row)                    │
│  └───────┬───────┘                                                    │
│          │ Entity/Collection of Entities                              │
│          ▼                                                            │
│  ┌──────────────────┐  $this->set('articles', $data)  ┌──────────┐  │
│  │  CONTROLLER (C)  │──────────────────────────────▶  │ VIEW (V) │  │
│  │                  │                                  │          │  │
│  │  src/Controller/ │  ← receives HTTP request         │templates/│  │
│  │  (14 PHP files)  │    from Router                   │src/View/ │  │
│  └──────────────────┘                                  └────┬─────┘  │
│          ▲                                                   │ HTML   │
│          │ HTTP Request                                      ▼        │
│       Browser  ◀──────────────────────────────────────── Response    │
└───────────────────────────────────────────────────────────────────────┘
```

`$this->set('articles', $articles)` is the hand-off from Controller to View.
It stores a variable in the View's scope so the template can access it as
`$articles`. The Controller never renders HTML; the View never touches the
database. That clean boundary is the entire point of MVC.

**In the files**:
- Model: `src/ORM/` (58 files), `src/Database/` (104 files), `src/Datasource/` (35 files)
- View: `src/View/` (53 files), templates live in the application's `templates/` directory
- Controller: `src/Controller/` (14 files)

---

### Pattern 2: Middleware Pipeline (PSR-15)

**What PSR means**: PSR stands for *PHP Standards Recommendation*, published by
PHP-FIG (Framework Interoperability Group) — a consortium of major PHP
framework authors. PSRs are vendor-neutral interfaces that allow any framework
or library that implements them to interoperate. CakePHP 5.x implements PSR-7
(HTTP messages), PSR-11 (DI container), PSR-15 (middleware), and PSR-3
(logging).

**What PSR-15 defines**: Two interfaces — `MiddlewareInterface` (a class that
processes a request and either passes to the next layer or returns a response
itself) and `RequestHandlerInterface` (a class that always produces a response).

**The onion model**: Middleware layers wrap around the application like skins
of an onion. The request travels inward through every layer; the response
travels back outward through every layer. This means every middleware can
inspect and modify **both** the incoming request and the outgoing response.

```
  Incoming Request
        │
        ▼
  ┌────────────────────────────────────────────────────────┐
  │  ErrorHandlerMiddleware  (outermost — MUST be first)   │
  │  Why outermost: it wraps everything in try/catch.      │
  │  If RoutingMiddleware throws a 404, ErrorHandler       │
  │  catches it. If it were inner, it couldn't catch       │
  │  errors from outer layers.                             │
  │  ┌──────────────────────────────────────────────────┐  │
  │  │  HttpsEnforcerMiddleware                         │  │
  │  │  ┌────────────────────────────────────────────┐  │  │
  │  │  │  BodyParserMiddleware                      │  │  │
  │  │  │  ┌──────────────────────────────────────┐  │  │  │
  │  │  │  │  RoutingMiddleware                   │  │  │  │
  │  │  │  │  ┌────────────────────────────────┐  │  │  │  │
  │  │  │  │  │  Application (innermost)        │  │  │  │  │
  │  │  │  │  │  Controller + Action + View     │  │  │  │  │
  │  │  │  │  └────────────────────────────────┘  │  │  │  │
  │  │  │  └──────────────────────────────────────┘  │  │  │
  │  │  └────────────────────────────────────────────┘  │  │
  │  └──────────────────────────────────────────────────┘  │
  └────────────────────────────────────────────────────────┘
        │
        ▼
  Outgoing Response
  (passes back through every layer — each can modify it)
```

**Key files**:
- `src/Http/MiddlewareQueue.php` (323 lines) — the ordered list of middleware layers
- `src/Http/Runner.php` (95 lines) — iterates the queue, recursively calling each layer

The `Runner` is itself a `RequestHandlerInterface`:
```php
// src/Http/Runner.php:29
class Runner implements RequestHandlerInterface
```

When a middleware calls `$handler->handle($request)`, it is calling
`Runner::handle()`, which calls the *next* middleware in the queue. This is
how the pipeline "passes" the request inward through the layers.

---

### Pattern 3: Registry Pattern

**What it is**: A central place to store and retrieve named object instances,
so the same object is never created twice. Think of it as a named cache for
collaborating objects of the same family.

**Why CakePHP needs it**: A controller needs Components (like Auth, Flash,
Paginator) by *name*. At the time you write `$this->Auth->...`, CakePHP needs
to resolve the string `"Auth"` to a live `AuthComponent` instance, wire it up
to the event system, and remember it for reuse.

```
  Controller requests "Auth" component
            │
            ▼
  ┌────────────────────────────────────────────────────────────┐
  │  ComponentRegistry  (extends ObjectRegistry)               │
  │                                                            │
  │  Internal cache ($_loaded):                               │
  │    "Auth"      → AuthComponent instance (already created) │
  │    "Flash"     → FlashComponent instance                   │
  │    "Paginator" → PaginatorComponent instance               │
  │                                                            │
  │  load("Auth"):                                             │
  │    1. Already in $_loaded?  → return cached instance       │
  │    2. Resolve class name: "Auth" → AuthComponent::class    │
  │    3. Instantiate, passing config                          │
  │    4. Attach event listeners (beforeFilter, startup, etc.) │
  │    5. Store in $_loaded["Auth"]                            │
  │    6. Return instance                                      │
  └────────────────────────────────────────────────────────────┘
```

**The abstract base**: `src/Core/ObjectRegistry.php` (406 lines):
```php
// src/Core/ObjectRegistry.php:45
abstract class ObjectRegistry implements Countable, IteratorAggregate
{
    protected array $_loaded = [];  // line 52: the instance cache

    // The "load" algorithm is provided here (check cache → resolve → create → store).
    // The HOW is delegated to subclasses via these abstract methods:
    abstract protected function _resolveClassName(string $class): string|false;
    abstract protected function _throwMissingClassError(string $class, ?string $plugin): void;
    abstract protected function _create(mixed $class, string $alias, array $config): object;
}
```

Concrete registries in the codebase:
- `src/Controller/ComponentRegistry.php` — manages Components for a controller
- `src/View/HelperRegistry.php` — manages Helpers for a view
- `src/Console/TaskRegistry.php` — manages Tasks for a shell command

**How Registry differs from DI Container**: The Registry manages a *family* of
identically-typed objects addressed by name (all things are Components, or all
things are Helpers). The DI Container (Pattern 6) resolves *any* class from any
namespace by its constructor dependencies. Registry is for "give me the Auth
Component"; the Container is for "build me an ArticlesController and resolve
everything it needs to work".

---

### Pattern 4: Observer / Events

**What it is**: Objects can broadcast named events without knowing who is
listening. Other objects subscribe to those event names and are called when the
event fires. This decouples the broadcaster entirely from the listener — the
ORM's `Table` class fires `Model.afterSave` without knowing or caring whether a
listener sends an email, updates a cache, or does nothing.

```
  PUBLISHER (any object using EventDispatcherTrait)
       │
       │  $this->dispatchEvent('Model.afterSave', ['entity' => $entity])
       │
       ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │  EventManager  src/Event/EventManager.php (531 lines)            │
  │                                                                  │
  │  Storage structure (listeners sorted by priority):               │
  │    "Model.afterSave" → [                                         │
  │        priority 1  → [AuditListener::afterSave],  ← runs first  │
  │        priority 10 → [CacheListener::afterSave],  ← runs second │
  │        priority 20 → [EmailListener::afterSave],  ← runs last   │
  │    ]                                                             │
  │                                                                  │
  │  Dispatch algorithm:                                             │
  │    asort(priorities)     ← lower number = runs earlier          │
  │    foreach priority:                                             │
  │        call each listener                                        │
  │        if event->isStopped() → break  ← "halted" means this    │
  └──────────────────────────────────────────────────────────────────┘
       │
       ▼
  LISTENERS (objects implementing EventListenerInterface,
             or any Closure passed to $eventManager->on())
```

**Priority**: Lower numbers run first. Default priority is 10
(`EventManager::$defaultPriority`, line 38). A listener at priority 1 will
always run before one at priority 10. Use lower numbers for listeners that
*must* execute first (e.g., a security check), higher numbers for things that
are safe to run last (e.g., sending a confirmation email).

**"Stopped" / "Halted"**: Any listener can call `$event->stopPropagation()`.
After that, no further listeners for that event are called — the same concept
as `stopPropagation()` in browser JavaScript events. This is how an Auth
component can fire `Controller.startup`, and if authentication fails, stop all
subsequent listeners (including the controller's own `beforeFilter`) from
running.

**Two scopes**: The global `EventManager::instance()` receives events from all
objects in the application. Each object that uses `EventDispatcherTrait` also
carries its own local EventManager for events that should only notify that
object's own listeners. Events are dispatched to both.

---

### Pattern 5: Traits as Mixins

**What a PHP trait is**: PHP allows a class to extend only one parent class.
But you often want to share a small, focused capability across many unrelated
classes. PHP **traits** solve this: they are reusable code blocks you include
into any class with `use TraitName;`. The trait's methods become part of the
class as if you had typed them there. Unlike inheritance, a trait does not
create an "is-a" relationship — it just shares code.

CakePHP uses traits everywhere to share capabilities without building deep,
fragile inheritance hierarchies.

```
  Without traits: "deep hierarchy" problem
  ─────────────────────────────────────────
  Component → BaseObject → EventAware → Configurable → ConventionAware
    (all classes must extend in the right order; any change breaks the chain)

  With traits: "mix what you need" solution
  ──────────────────────────────────────────
  class AuthComponent extends Component
  {
      use InstanceConfigTrait;    ← brings in getConfig()/setConfig()
      use EventDispatcherTrait;   ← brings in dispatchEvent()
      // No ConventionsTrait needed; AuthComponent doesn't need naming conventions
  }
```

**Traits in use across the codebase**:

```
InstanceConfigTrait  (src/Core/InstanceConfigTrait.php, 326 lines)
  Used by: every Component, Helper, Behavior, Cell, FormHelper, etc.
  Provides: getConfig(), setConfig(), configShallow()
  Rule: any class that accepts configuration options from user code uses this trait

EventDispatcherTrait (src/Event/EventDispatcherTrait.php)
  Used by: Server, BaseApplication, Table, Controller, etc.
  Provides: dispatchEvent(), getEventManager(), setEventManager()
  Rule: any class that can broadcast events uses this trait

ConventionsTrait     (src/Core/ConventionsTrait.php, 156 lines)
  Used by: bake generators, test helpers
  Provides: _entityName(), _modelKey(), _variableName(), _singularName(), etc.
  Rule: any class that needs to derive CakePHP naming conventions uses this
```

Here is a concrete example showing how `InstanceConfigTrait` saves work:

```php
// src/Core/InstanceConfigTrait.php:35
// These two properties are declared by the trait — you never write them:
protected array $_config = [];              // merged runtime config
protected bool $_configInitialized = false; // tracks whether defaults were applied

// A Component that uses the trait:
class PaginatorComponent extends Component  // Component already uses InstanceConfigTrait
{
    protected array $_defaultConfig = [
        'maxLimit' => 100,
        'limit'    => 20,
    ];
    // getConfig('maxLimit') returns 100.
    // User can override: new PaginatorComponent([], ['maxLimit' => 200])
    // Still no need to write getConfig()/setConfig() — the trait provides them.
}
```

---

### Pattern 6: Dependency Injection / Service Container

**What dependency injection is**: Instead of a class creating its own
dependencies with `new Foo()` internally, those dependencies are *provided from
outside* when the class is constructed. This has two benefits: the class is
easier to test (you can pass a fake/mock), and it is decoupled (it works with
any implementation that satisfies the type hint).

```
  Without DI (tightly coupled, hard to test):
  ────────────────────────────────────────────
  class ArticlesController {
      public function __construct() {
          $this->table = new ArticlesTable();  // hard-wired — cannot mock in tests
          $this->mailer = new Mailer();
      }
  }

  With DI (loosely coupled, easy to test):
  ─────────────────────────────────────────
  class ArticlesController {
      public function __construct(
          private ArticlesTable $table,   // provided from outside
          private Mailer $mailer,         // provided from outside
      ) {}
  }
  // In tests: pass a FakeArticlesTable and a FakeMailer.
  // In production: the Container builds and injects the real ones.
```

**A Service Container** automates the "provide from outside" step. You register
what each class needs; the container builds the whole dependency chain for you.

```
  app->services($container):
    $container->add(ArticlesTable::class)
              ->addArgument(ConnectionManager::get('default'));

  Then at request time:
  ControllerFactory asks container for ArticlesController
      │
      ▼
  ┌──────────────────────────────────────────────────────────┐
  │  Container  src/Core/Container.php                       │
  │  (28 lines — extends league/container)                   │
  │                                                          │
  │  get(ArticlesController::class):                         │
  │    1. Inspect ArticlesController constructor             │
  │    2. Parameter: ArticlesTable → already registered      │
  │    3. ArticlesTable needs: Connection → get from config  │
  │    4. Build ArticlesTable with Connection                │
  │    5. Build ArticlesController with ArticlesTable        │
  │    6. Return fully-assembled controller                  │
  └──────────────────────────────────────────────────────────┘
```

CakePHP's `Container` (line 26) is a 28-line wrapper around `league/container`:

```php
// src/Core/Container.php:26
class Container extends LeagueContainer implements ContainerInterface
{
    // Intentionally empty — the value is in implementing ContainerInterface,
    // which means any PSR-11 compliant container could replace this.
}
```

The empty body is not laziness — it is intentional design. By implementing
`ContainerInterface` (PSR-11), CakePHP guarantees that the rest of the
framework never depends on League's concrete implementation, only on the
interface. Any compatible container could be swapped in.

---

## 1.5 — The 22 Namespaces: A Map of the City

Every class in CakePHP lives under `Cake\<Namespace>`, which maps to
`src/<Namespace>/` on disk. There are 22 namespace directories in `src/`
(plus `functions.php` helper files which are covered in section 1.7).

### Tier 1 — Foundation (the framework cannot start without these)

| Namespace | Purpose | Key File | PHP Files |
|---|---|---|---|
| `Cake\Core` | App config, plugin system, DI container, base interfaces, the `App` class that resolves class names | `src/Core/App.php` | 34 |
| `Cake\Http` | HTTP server, middleware, request/response objects, application base class | `src/Http/Server.php` | 79 |
| `Cake\Routing` | URL parsing (string → params) and URL generation (params → string) | `src/Routing/Router.php` | 18 |

### Tier 2 — MVC (the application layer)

| Namespace | Purpose | Key File | PHP Files |
|---|---|---|---|
| `Cake\Controller` | Request handling, component registry, controller base class | `src/Controller/Controller.php` | 14 |
| `Cake\View` | Template rendering, helper registry, cell rendering, content types | `src/View/View.php` | 53 |
| `Cake\ORM` | Object-relational mapper: Table, Entity, Query, associations (hasMany, belongsTo, etc.) | `src/ORM/Table.php` | 58 |
| `Cake\Database` | SQL abstraction, dialect adapters (MySQL/Postgres/SQLite), type casting system, query builder | `src/Database/Connection.php` | 104 |
| `Cake\Datasource` | Abstract contracts for any data source — implemented by ORM but also non-SQL sources | `src/Datasource/QueryInterface.php` | 35 |

### Tier 3 — Services (cross-cutting concerns)

| Namespace | Purpose | Key File | PHP Files |
|---|---|---|---|
| `Cake\Validation` | Rule-based data validation with built-in rules for strings, numbers, dates, etc. | `src/Validation/Validator.php` | 8 |
| `Cake\Cache` | Multi-backend caching with a unified API (file, APCu, Redis, Memcached, …) | `src/Cache/Cache.php` | 26 |
| `Cake\Log` | PSR-3 logging with multiple configurable backends (file, syslog, …) | `src/Log/Log.php` | 11 |
| `Cake\Error` | Exception trapping, error rendering, `ErrorHandlerMiddleware` | `src/Error/ExceptionTrap.php` | 31 |
| `Cake\Event` | Observer/event system — EventManager, EventDispatcherTrait, EventListenerInterface | `src/Event/EventManager.php` | 11 |
| `Cake\I18n` | Internationalisation: translation functions, date/time/number formatting | `src/I18n/I18n.php` | 28 |
| `Cake\Mailer` | Email composition and sending with pluggable transport adapters | `src/Mailer/Mailer.php` | 12 |

### Tier 4 — Tools (developer productivity)

| Namespace | Purpose | Key File | PHP Files |
|---|---|---|---|
| `Cake\Console` | CLI application framework: argument parsing, I/O helpers, shell runner | `src/Console/ConsoleIo.php` | 35 |
| `Cake\Command` | Framework-provided CLI commands: `cache clear`, `routes`, `i18n extract`, etc. (Note: `bake` and `migrations` are *separate plugins* — `cakephp/bake` and `cakephp/migrations`) | `src/Command/CacheClearCommand.php` | 29 |
| `Cake\Collection` | Lazy functional collection library: map, filter, reduce, groupBy, sortBy, … | `src/Collection/Collection.php` | 21 |
| `Cake\Utility` | Word inflection (Inflector), array access (Hash), string manipulation (Text), XML, Security | `src/Utility/Inflector.php` | 11 |
| `Cake\Form` | Standalone form objects for validation and processing without an ORM table | `src/Form/Form.php` | 3 |
| `Cake\Network` | Low-level socket utilities | `src/Network/Socket.php` | 2 |
| `Cake\TestSuite` | Testing utilities: `TestCase`, `IntegrationTestTrait`, fixture loading, HTTP test client | `src/TestSuite/TestCase.php` | 66 |

**Size perspective**: `Cake\Database` is the largest namespace (104 files)
because supporting multiple SQL dialects, a complete type-casting system, and a
fluent query builder is genuinely complex work. `Cake\Network` is the smallest
(2 files) because modern HTTP is handled by `Cake\Http` and PSR-7 libraries,
leaving only raw socket work.

---

## 1.6 — Essential PHP Concepts (Only What You Need)

You do not need a complete PHP OOP tutorial before you can read this codebase.
You need five specific concepts that CakePHP uses in almost every file.

---

### 1. `declare(strict_types=1)`

This declaration appears as line 2 of **every single PHP file** in the
repository. Without it, PHP silently coerces types: passing the string `"5"`
to a function expecting `int` would work silently. With `strict_types=1`, the
same call throws a `TypeError` immediately.

```php
// Every CakePHP file starts with this:
<?php
declare(strict_types=1);
```

As a contributor, you must add this line to every new PHP file you create.
Without it, PHP will silently accept wrong types in your function calls, which
masks bugs.

---

### 2. Namespaces and PSR-4 Autoloading

**The problem**: With thousands of classes across dozens of files, requiring
them manually (`require_once 'other/file.php'`) is error-prone and fragile.

**The solution**: PSR-4 autoloading. Composer registers a rule in
`composer.json`:

```json
// composer.json:91
"autoload": {
    "psr-4": {
        "Cake\\": "src/"
    }
}
```

This tells PHP's autoloader: when you need a class whose name starts with
`Cake\`, find it by:
1. Replacing `Cake\` with `src/`
2. Replacing every remaining `\` with `/`
3. Appending `.php`

```
Class needed:    Cake\Http\Server
                 ────  ────────────
Step 1:          src/ + Http\Server
Step 2 + 3:      src/Http/Server.php   ✓
```

`use Cake\Http\Server;` at the top of a file tells PHP "I will use this class".
Composer's autoloader finds the file automatically when the class is first
referenced. No `require` statement needed.

**Important exception**: the `src/Core/functions.php`,
`src/Collection/functions.php`, and similar files contain standalone PHP
functions (not classes) and do not follow the class lookup rule. They are
loaded via the `"files"` key in `composer.json:94` and are available globally
without any `use` statement.

---

### 3. Interfaces

**What an interface is**: A named contract. It declares a set of method
signatures (names, parameters, return types) but contains no implementation.
Any class that `implements` an interface *must* provide those methods.

CakePHP uses PSR interfaces as the contracts between its layers, so the
framework code never depends on a specific implementation — only on the
contract.

```php
// PSR-7: the contract for an HTTP request object
interface ServerRequestInterface {
    public function getMethod(): string;
    public function getUri(): UriInterface;
    public function getBody(): StreamInterface;
    public function getAttribute(string $name, mixed $default = null): mixed;
    // ... 20+ more methods
}

// CakePHP's implementation satisfies the contract:
// src/Http/ServerRequest.php
class ServerRequest implements ServerRequestInterface { ... }

// Framework code accepts ANY PSR-7 request — not just CakePHP's:
// src/Http/Runner.php:69
public function handle(ServerRequestInterface $request): ResponseInterface
//                     ^^^^^^^^^^^^^^^^^^^^^^^^
// "I work with any object that satisfies the PSR-7 ServerRequest contract"
```

Interfaces you will see in almost every file:
| Interface | PSR | Purpose |
|---|---|---|
| `ServerRequestInterface` / `ResponseInterface` | PSR-7 | HTTP message contracts |
| `MiddlewareInterface` / `RequestHandlerInterface` | PSR-15 | Pipeline layer contracts |
| `ContainerInterface` | PSR-11 | DI container contract |
| `LoggerInterface` | PSR-3 | Logging contract |

---

### 4. Traits

Covered fully in section 1.4 Pattern 5. The one additional point worth
repeating: when you see `use SomeTrait;` inside a class body, it is not an
import statement (those go at the top of the file). It is an instruction to
*include the trait's methods into this class*. The result is as if you had
typed those methods directly into the class.

```php
// src/Http/Server.php:36–41
class Server implements EventDispatcherInterface
{
    use EventDispatcherTrait;  // ← includes dispatchEvent(), getEventManager(), etc.
                               //   into Server. Not inherited. Not imported.
                               //   Literally merged in at compile time.
}
```

---

### 5. Abstract Classes

**What an abstract class is**: A class that provides a partial implementation
but cannot be instantiated directly. It can declare `abstract` methods — method
signatures with no body — that concrete subclasses must implement.

This is different from an interface: an abstract class *can* provide working
method implementations; it just forces subclasses to fill in the blanks.

```php
// src/Core/ObjectRegistry.php:45
abstract class ObjectRegistry implements Countable, IteratorAggregate
{
    // CONCRETE — provided, ready to use:
    public function load(string $name, array $config = []): object
    {
        // check $_loaded cache → call _resolveClassName → call _create → store
        // The ALGORITHM is written here once.
    }

    // ABSTRACT — the concrete "how" is left to subclasses:
    abstract protected function _resolveClassName(string $class): string|false;
    abstract protected function _create(mixed $class, string $alias, array $config): object;
}

// ComponentRegistry fills in the blanks for Components:
class ComponentRegistry extends ObjectRegistry
{
    protected function _resolveClassName(string $class): string|false
    {
        return App::className($class, 'Controller/Component', 'Component');
    }
    // ...
}
```

`ObjectRegistry` owns the `load()` algorithm. `ComponentRegistry` owns the
knowledge of what a "Component" class looks like and where to find it. Neither
class could work alone.

---

## 1.7 — How to Read This Codebase

### The Namespace → Directory Rule

```
Cake\<Namespace>\<ClassName>
  │        │         │
  │        │         └──  <ClassName>.php
  │        └──────────── src/<Namespace>/
  └───────────────────── (always "Cake\")

Examples:
  Cake\Http\Server          → src/Http/Server.php
  Cake\ORM\Table            → src/ORM/Table.php
  Cake\Core\ObjectRegistry  → src/Core/ObjectRegistry.php
  Cake\Event\EventManager   → src/Event/EventManager.php
  Cake\Error\Middleware\ErrorHandlerMiddleware
                            → src/Error/Middleware/ErrorHandlerMiddleware.php
```

Sub-namespaces work exactly the same way: `Cake\Error\Middleware\` maps to
`src/Error/Middleware/`. Follow the backslashes.

### The 6 Entry Points Every Contributor Must Know Cold

```
1. src/Http/Server.php  (200 lines)
   The first class called for every HTTP request.
   run() bootstraps the app, builds middleware, fires the pipeline.
   Start here for: "how does a request become a response?"

2. src/Http/BaseApplication.php  (364 lines)
   The base all CakePHP applications extend.
   Defines bootstrap(), middleware(), routes(), services().
   Start here for: "where does the app configure itself?"

3. src/Core/ObjectRegistry.php  (406 lines)
   The registry pattern used by all Component/Helper/Behavior loading.
   Start here for: "how does CakePHP load named objects on demand?"

4. src/Core/ConventionsTrait.php  (156 lines)
   The naming-convention translation layer.
   Start here for: "how does CakePHP derive class names from table names?"

5. src/Utility/Inflector.php  (524 lines)
   The word-transformation engine behind all conventions.
   Start here for: "why does 'person' become 'people'?"

6. composer.json  (149 lines)
   All dependencies, autoloading rules, and dev scripts.
   Start here for: "what external libraries does CakePHP use, and why?"
```

### Running the Developer Tools

All commands are defined in `composer.json` under `"scripts"`:

```bash
# Run the full test suite (PHPUnit)
composer test

# Run a specific test class only
composer test -- --filter TableTest

# Run tests in a specific file
composer test -- tests/TestCase/ORM/TableTest.php

# Check code style (PHP_CodeSniffer, CakePHP ruleset)
composer cs-check

# Fix code style automatically
composer cs-fix

# Run static analysis (PHPStan)
composer stan

# Run style check + tests together
composer check
```

The test suite uses `Cake\TestSuite` (66 PHP files) which provides fixture
loading (test database data), `IntegrationTestTrait` for testing full HTTP
requests in memory, and automatic database transaction rollback between tests
so each test starts with a clean state.

### Finding Any Class in Under 10 Seconds

1. You see `Cake\Routing\Router` mentioned in a stack trace.
2. Drop `Cake\` → you have `Routing\Router`.
3. Replace `\` with `/` → `Routing/Router`.
4. Add `src/` prefix and `.php` suffix → `src/Routing/Router.php`.
5. Open it.

**For sub-namespaces**: `Cake\Error\Middleware\ErrorHandlerMiddleware` →
drop `Cake\` → `Error\Middleware\ErrorHandlerMiddleware` → replace `\` →
`Error/Middleware/ErrorHandlerMiddleware` → `src/Error/Middleware/ErrorHandlerMiddleware.php`.

This rule works for every class in the `Cake\` namespace. The only exceptions
are the `functions.php` files (global helper functions, not classes) which are
loaded automatically by Composer.

---

## Summary: Chapter 1 Reference Card

```
CONCEPT                     WHERE IN THE CODE
──────────────────────────────────────────────────────────────────────
strict_types declaration    Line 2 of every .php file in src/
PSR-4 autoloading rule      composer.json:91  "Cake\\" → "src/"
HTTP entry point            src/Http/Server.php:75  run()
App bootstrap + middleware  src/Http/BaseApplication.php:59 (abstract)
Middleware queue            src/Http/MiddlewareQueue.php:36
MW execution / pipeline     src/Http/Runner.php:29
Convention engine           src/Utility/Inflector.php:27
Convention helper methods   src/Core/ConventionsTrait.php:24
Registry pattern            src/Core/ObjectRegistry.php:45 (abstract)
Event system                src/Event/EventManager.php:31
Event priority ordering     src/Event/EventManager.php:376  asort()
Config mixin (trait)        src/Core/InstanceConfigTrait.php:28
DI container                src/Core/Container.php:26 (wraps League)
Error handling middleware   src/Error/Middleware/ErrorHandlerMiddleware.php:44
```

---

## What's Coming in Chapter 2

You now have the map. Chapter 2 begins the journey into the first box on that
map: the **HTTP Layer**. We will trace the exact path of a single HTTP request
from the moment it arrives at `Server::run()` through every middleware, the
application handle method, and the controller factory, and back out as a
response. We will read `src/Http/Server.php`, `src/Http/Runner.php`,
`src/Http/MiddlewareQueue.php`, and several middleware classes line by line —
not to memorise them, but to build the mental model of how PSR-15 pipelines
actually execute on the PHP call stack.

By the end of Chapter 2 you will be able to write, register, position, and
debug your own middleware.
