# CakePHP Deep-Dive Book Project

## Project Identity
- **Repo**: `cakephp/cakephp`, branch `5.x`
- **Working dir**: `/Users/mallikimtiaz.hassan/Desktop/Sadi/cakephp`
- **Book output dir**: `docs/book/` (created by us, not part of original repo)

## What We Are Building
A book-style technical guide to CakePHP 5.x internals.
- **Reader level**: PHP beginner — OOP/PSR concepts must be explained in context
- **Goal**: beginner → contributor to core and ecosystem plugins
- **Style**: ASCII diagram first, then narrative, then real file:line references
- **Trigger for next chapter**: user says "next"

## Progress
| Chapter | Title | File | Status |
|---|---|---|---|
| 1 | The Architecture of CakePHP: A Map Before the Journey | `docs/book/chapter-01.md` | DONE |
| 2 | The HTTP Layer: Tracing One Request from Wire to Response | `docs/book/chapter-02.md` | DONE |
| 3+ | TBD | — | not started |

## Chapter 1 Key Decisions (don't repeat in later chapters)
- Defined all 6 design patterns (MVC, PSR-15 middleware, Registry, Observer/Events, Traits, DI)
- Mapped all 25 namespaces in a 4-tier table with file counts
- Established the namespace→directory lookup rule (used throughout)
- Listed the 6 entry-point files every contributor must know
- Explained PSR-4 autoloading via composer.json:91

## Conventions for Every Chapter
1. Start every section with an ASCII diagram
2. Every concept gets a real `file:line` reference
3. PHP concepts explained in plain English before code
4. No content repeated from earlier chapters
5. End with "What's Coming in Chapter N+1" bridge

## Key File Reference (verified line counts, CakePHP 5.x branch)
| File | Lines | Key line(s) |
|---|---|---|
| `src/Utility/Inflector.php` | 524 | :27 class, :34 plural rules |
| `src/Core/ConventionsTrait.php` | 156 | :56 _modelKey() |
| `src/Core/ObjectRegistry.php` | 406 | :45 abstract class |
| `src/Event/EventManager.php` | 531 | :31 class, :45 $_generalManager |
| `src/Core/Container.php` | 28 | :26 extends LeagueContainer |
| `src/Http/Server.php` | 200 | :36 class, :75 run() |
| `src/Http/Runner.php` | 95 | :29 implements RequestHandlerInterface |
| `src/Http/MiddlewareQueue.php` | 323 | :36 class |
| `src/Http/BaseApplication.php` | 364 | :59 abstract class |
| `src/Core/InstanceConfigTrait.php` | 326 | :28 trait, :35 $_config |
| `src/Core/StaticConfigTrait.php` | — | :30 trait, adapter façade pattern |
| `composer.json` | 149 | :91 PSR-4 autoload, :24 require |

## Namespace File Counts (from actual repo)
Cache:26, Collection:21, Command:29, Console:35, Controller:14, Core:34,
Database:104, Datasource:35, Error:31, Event:11, Form:3, Http:79, I18n:28,
Log:11, Mailer:12, Network:2, ORM:58, Routing:18, TestSuite:66, Utility:11,
Validation:8, View:53

## Chapter 2 Key Decisions (don't repeat in later chapters)
- Explained PSR-7 immutability ("with*() returns a new object")
- Explained PSR-15 MiddlewareInterface vs RequestHandlerInterface distinction
- Walked through Server::run() line by line (lines 75-105)
- Explained lazy resolution in MiddlewareQueue::current() (line 278)
- Traced Runner::handle() cursor-advance-before-call pattern (line 69)
- Catalogued all 9 built-in middleware classes with short-circuit analysis
- Covered BaseApplication::handle() as the MVC bridge (line 343)
- Covered ControllerFactory::create() + invoke() + getActionArgs() (lines 72-183)
- Taught writing/registering/debugging custom middleware (3 rules)

## Chapter 3 Scope (planned, not started)
Topic: Routing — how URL strings become controller/action/pass parameter arrays.
Reader outcome: able to define any route pattern and debug routing mismatches.
Key files: src/Routing/Router.php, src/Routing/RouteBuilder.php, src/Routing/Route/Route.php

## User Preferences
- No emojis
- Concise communication
- Output format: Markdown files in docs/book/
- No code changes to the framework source — documentation only
