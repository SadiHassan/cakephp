# Chapter 2 — The HTTP Layer: Tracing One Request from Wire to Response

> **What you will be able to do after this chapter**: Explain precisely what
> happens inside CakePHP from the moment an HTTP request arrives to the moment
> a response is sent back. Write, register, position, and debug your own
> PSR-15 middleware.

---

## 2.1 — PSR-7 and PSR-15: The Two Standards the Entire Layer Rests On

Before reading any CakePHP source, you must understand two small but critical
PHP standards. Everything in `src/Http/` is built on them.

### PSR-7: Immutable HTTP Messages

PSR-7 defines interfaces for HTTP request and response objects. The most
important rule it imposes is **immutability**: every object represents a fixed
snapshot of an HTTP message. Methods that modify the state return a *new*
object; they never change the existing one.

```
PSR-7 INTERFACES (defined by PHP-FIG, implemented by laminas/laminas-diactoros)

  MessageInterface   ← common base: headers, body (stream)
      │
      ├── ServerRequestInterface       ResponseInterface
      │   ──────────────────────       ────────────────
      │   getMethod(): string          getStatusCode(): int
      │   getUri(): UriInterface       getReasonPhrase(): string
      │   getQueryParams(): array      getBody(): StreamInterface
      │   getParsedBody(): mixed       withStatus(int): static
      │   getAttribute(n): mixed       withHeader(n, v): static
      │   withAttribute(n,v): static   withBody(stream): static
      │   withParsedBody(data): static
      │
      └── RequestInterface (for outgoing client requests — not used in routing)
```

**Why immutability?** Without it, one middleware could silently modify the
request object and every other middleware — before and after it in the stack —
would suddenly see a different request. That would make bugs nearly impossible
to trace. Immutability guarantees that when you receive a `$request`, it
contains exactly what you expect: no upstream middleware has secretly changed
it.

**How immutability works in practice**: `with*()` methods return a brand-new
object with the change applied, leaving the original untouched.

```php
// $request is immutable — getUri() returns the current URI object
$uri = $request->getUri();          // e.g., http://example.com/articles

// withScheme() returns a NEW UriInterface — $uri is still http://...
$httpsUri = $uri->withScheme('https');

// To "change" the request's URI, you must create a new request:
$newRequest = $request->withUri($httpsUri);

// Now $request still has the original http:// URI.
// $newRequest has the https:// URI.
// Both objects exist independently.
```

**Consequence for middleware**: when you want to pass a modified request to
the next layer, you must pass the new object explicitly:

```php
// WRONG — passes the unmodified original request forward:
$request->withAttribute('user', $user);
$handler->handle($request);

// CORRECT — passes the new request (with the attribute) forward:
$request = $request->withAttribute('user', $user);
return $handler->handle($request);
```

### PSR-15: The Middleware Contract

PSR-15 defines exactly two interfaces:

```
┌──────────────────────────────────────────────────────────────────────┐
│  MiddlewareInterface                                                  │
│  ─────────────────────                                               │
│  process(                                                            │
│    ServerRequestInterface $request,   ← the incoming request        │
│    RequestHandlerInterface $handler   ← the rest of the pipeline    │
│  ): ResponseInterface                 ← must return a response      │
│                                                                      │
│  A middleware receives BOTH the request AND the next handler.        │
│  It decides: pass forward, modify and pass, or return early.        │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│  RequestHandlerInterface                                              │
│  ──────────────────────────                                          │
│  handle(                                                             │
│    ServerRequestInterface $request                                   │
│  ): ResponseInterface                                                │
│                                                                      │
│  A request handler receives ONLY the request.                       │
│  It IS the terminus — it produces the response entirely itself.     │
└──────────────────────────────────────────────────────────────────────┘
```

The critical distinction:
- A **middleware** does some work and then calls `$handler->handle($request)`
  to continue the pipeline — OR it short-circuits and returns a response
  without calling the handler at all.
- A **request handler** is the end of the road. It always produces a response
  from its own logic.

In CakePHP: `Runner` is a `RequestHandlerInterface`, `BaseApplication` is a
`RequestHandlerInterface` (the final fallback), and every class in
`src/Http/Middleware/` plus `src/Error/Middleware/` implements
`MiddlewareInterface`.

---

## 2.2 — Server::run() — Every Line Explained

The entry point for every HTTP request in a CakePHP application is
`Server::run()`. In production, it is called from `webroot/index.php` in the
application skeleton (that file is in `cakephp/app`, a separate repository —
this repository only contains the framework library). The call looks like:

```php
// webroot/index.php (in your application, not in this repo)
$server = new Server(new Application(dirname(__DIR__) . '/config'));
$server->emit($server->run());
```

Here is the complete `run()` method with every decision annotated:

```php
// src/Http/Server.php:75
public function run(
    ?ServerRequestInterface $request = null,
    ?MiddlewareQueue $middlewareQueue = null,
): ResponseInterface {

    // ── STEP 1: Bootstrap ────────────────────────────────────────────
    $this->bootstrap();
    // Protected method at line 115. Calls:
    //   $this->app->bootstrap()        → require_once config/bootstrap.php
    //   $this->app->pluginBootstrap()  → each plugin's bootstrap() method
    // This is where database connections, cache configurations, and
    // global settings are established — before any request is processed.
    // See: src/Http/BaseApplication.php:184

    // ── STEP 2: Build the PSR-7 request object ───────────────────────
    $request = $request ?: ServerRequestFactory::fromGlobals();
    // In production, $request is null and is built here from PHP's superglobals:
    //   $_SERVER  → method, URI, headers, server variables
    //   $_GET     → query string parameters
    //   $_POST    → form data
    //   $_FILES   → uploaded files
    //   $_COOKIE  → cookies
    // The result is a PSR-7 ServerRequest object — all those superglobals
    // collected into one clean, typed, immutable object.
    // The optional $request parameter exists for testing: your test passes
    // a pre-built request instead of relying on superglobals.

    // ── STEP 3: Build the MiddlewareQueue ────────────────────────────
    if ($middlewareQueue === null) {
        if ($this->app instanceof ContainerApplicationInterface) {
            $middlewareQueue = new MiddlewareQueue([], $this->app->getContainer());
        } else {
            $middlewareQueue = new MiddlewareQueue();
        }
    }
    // An empty queue is created. If the app has a DI container,
    // the queue receives a reference to it. The container is used
    // later to instantiate middleware that has constructor dependencies.

    // ── STEP 4: Populate the queue ───────────────────────────────────
    $middleware = $this->app->middleware($middlewareQueue);
    // YOUR Application class's middleware() method adds layers to the queue.
    // After this call, the queue might contain:
    //   [0] => ErrorHandlerMiddleware (object)
    //   [1] => 'HttpsEnforcer'        (string — resolved lazily later)
    //   [2] => Closure               (anonymous function)
    //   [3] => RoutingMiddleware      (object)

    if ($this->app instanceof PluginApplicationInterface) {
        $middleware = $this->app->pluginMiddleware($middleware);
    }
    // Each plugin that declares middleware gets to add its own entries.
    // src/Http/BaseApplication.php:127 iterates plugins flagged with 'middleware'.

    // ── STEP 5: Last-chance event hook ───────────────────────────────
    $this->dispatchEvent('Server.buildMiddleware', ['middleware' => $middleware]);
    // Any event listener can modify the queue here — add, remove, or reorder.
    // Useful for conditional middleware (e.g., add profiling middleware
    // only when a debug flag is on), or for plugins that need to inject
    // themselves at a precise position.

    // ── STEP 6: Execute the pipeline ─────────────────────────────────
    $response = $this->runner->run($middleware, $request, $this->app);
    // Runner iterates the queue. $this->app is passed as the "fallback handler":
    // when all middleware have run, BaseApplication::handle() is called,
    // which routes to a controller and builds the response.

    // ── STEP 7: Clean up the session ─────────────────────────────────
    if ($request instanceof ServerRequest) {
        $request->getSession()->close();
    }
    // Writing buffered session data to the session store.

    return $response;
}
```

After `run()` returns a response, `Server::emit()` (line 139) takes over:

```php
// src/Http/Server.php:139
public function emit(ResponseInterface $response, ?ResponseEmitter $emitter = null): void
{
    $emitter ??= new ResponseEmitter();
    $emitter->emit($response);  // writes HTTP headers + body to PHP output buffer

    // ... (retrieve request from container or Router)

    $this->dispatchEvent('Server.terminate', compact('request', 'response'));
    // Server.terminate fires AFTER the response is sent.
    // Use it for work that is safe to do after the client has received their data:
    // sending confirmation emails, writing audit logs, updating caches, etc.
    // On PHP-FPM this genuinely runs after flush; on other SAPIs it runs before.
}
```

---

## 2.3 — MiddlewareQueue: Building and Resolving the Pipeline

The queue is a simple ordered list that stores middleware in the order they
should execute. It implements PHP's `SeekableIterator` and `Countable`
interfaces so it can be iterated by the `Runner`.

```
  ┌────────────────────────────────────────────────────────────────────┐
  │  MiddlewareQueue  src/Http/MiddlewareQueue.php:36                  │
  │                                                                    │
  │  Internal state after app->middleware() has run:                  │
  │    $queue = [                                                      │
  │        0 => ErrorHandlerMiddleware (object — already instantiated) │
  │        1 => 'HttpsEnforcer'        (string — not yet instantiated) │
  │        2 => Closure { ... }        (anonymous function)            │
  │        3 => RoutingMiddleware      (object)                        │
  │    ]                                                               │
  │    $position  = 0      ← iterator cursor, starts at 0             │
  │    $container = ...    ← DI container reference (or null)         │
  │                                                                    │
  │  Building API (all return $this for chaining):                    │
  │    add($mw)                → append to end                        │
  │    prepend($mw)            → insert at front                      │
  │    insertAt(int, $mw)      → insert at specific index             │
  │    insertBefore(cls, $mw)  → insert before first class match      │
  │    insertAfter(cls, $mw)   → insert after first class match       │
  └────────────────────────────────────────────────────────────────────┘
```

All five methods accept three forms of middleware:
- **Object** `new ErrorHandlerMiddleware()` — already a `MiddlewareInterface`, stored as-is
- **String** `'HttpsEnforcer'` or `HttpsEnforcerMiddleware::class` — stored as a string, resolved to an object lazily on first use
- **Closure** `function($req, $handler) {...}` — stored as-is, wrapped in `ClosureDecoratorMiddleware` on first use

### Lazy Resolution: How a String Becomes a Middleware Object

The queue stores strings and closures raw. Resolution happens in `current()`
(line 278), which is called by the `Runner` just before it executes each layer:

```php
// src/Http/MiddlewareQueue.php:278
public function current(): MiddlewareInterface
{
    // If already a resolved MiddlewareInterface object — return it directly.
    if ($this->queue[$this->position] instanceof MiddlewareInterface) {
        return $this->queue[$this->position];
    }
    // Resolve the string or Closure, replace it in $queue (cache it),
    // and return the resolved MiddlewareInterface.
    return $this->queue[$this->position] = $this->resolve($this->queue[$this->position]);
}
```

The first time a position is visited, the string or Closure is resolved and
replaced in the array. Every subsequent request reuses the same object.

`resolve()` (line 76) works as follows:

```
Input: string class name (e.g. 'HttpsEnforcer' or HttpsEnforcerMiddleware::class)
  │
  ├── Does $container->has($className)?
  │     YES → $container->get($className)
  │           (full DI resolution — dependencies injected from container)
  │
  └── NO container, or container doesn't know it:
        → App::className($name, 'Middleware', 'Middleware')
          This applies CakePHP's class resolution convention:
            1. Look for App\Middleware\{Name}Middleware  (your app)
            2. Look for {Plugin}\Middleware\{Name}Middleware  (plugins)
            3. Look for Cake\Http\Middleware\{Name}Middleware (framework)
          → new $className()  (no DI — constructor must have no required args)

Input: Closure
  └── new ClosureDecoratorMiddleware($closure)
      src/Http/Middleware/ClosureDecoratorMiddleware.php:39
      Its process() method simply calls ($this->callable)($request, $handler)

Input: already a MiddlewareInterface object
  └── Returned as-is
```

The convention `App::className('HttpsEnforcer', 'Middleware', 'Middleware')`
means: look for a class named `HttpsEnforcerMiddleware` (name + 'Middleware'
suffix) in a directory named `Middleware/` (the second argument). This is why
you can write `$queue->add('HttpsEnforcer')` and it resolves to
`Cake\Http\Middleware\HttpsEnforcerMiddleware`.

---

## 2.4 — Runner::handle() — How the Pipeline Actually Executes

This is the most important method in the HTTP layer. The entire pipeline
mechanism fits in 26 lines.

```php
// src/Http/Runner.php:51
public function run(
    MiddlewareQueue $queue,
    ServerRequestInterface $request,
    ?RequestHandlerInterface $fallbackHandler = null,
): ResponseInterface {
    $this->queue = $queue;
    $this->queue->rewind();           // reset iterator to position 0
    $this->fallbackHandler = $fallbackHandler;
    return $this->handle($request);   // begin the recursive descent
}

// src/Http/Runner.php:69
public function handle(ServerRequestInterface $request): ResponseInterface
{
    if (
        $this->fallbackHandler instanceof RoutingApplicationInterface &&
        $request instanceof ServerRequest
    ) {
        Router::setRequest($request);  // keep the global Router in sync
    }

    if ($this->queue->valid()) {               // is there a next middleware?
        $middleware = $this->queue->current(); // get it (resolves lazily)
        $this->queue->next();                  // advance the cursor BEFORE calling
        return $middleware->process($request, $this);
        //                                     ^^^^
        //   $this IS the Runner — a RequestHandlerInterface.
        //   When middleware calls $handler->handle($request),
        //   it calls Runner::handle() again.
    }

    if ($this->fallbackHandler) {
        return $this->fallbackHandler->handle($request);
        // All middleware have run. BaseApplication::handle() takes over.
    }

    return new Response(['status' => 500, ...]);  // safety net — should never fire
}
```

### The Critical Detail: `next()` Before `process()`

Notice that `$this->queue->next()` advances the cursor *before* calling
`$middleware->process()`. This is deliberate. When the middleware eventually
calls `$handler->handle($request)`, `Runner::handle()` runs again. At that
point, the cursor is already pointing at the *next* middleware. Without this
pre-advance, every call to `handle()` would re-execute the same middleware in
an infinite loop.

### This IS Real Recursion on the PHP Call Stack

A common misconception is that the cursor mechanism replaces recursion. It does
not. The PHP call stack grows with each middleware layer. Here is the stack for
a three-middleware queue at the deepest point (just before BaseApplication runs):

```
PHP Call Stack (bottom to top)
────────────────────────────────────────────────────────────────
Frame 1: Runner::handle()  [position was 0 when this was called]
  → called ErrorHandlerMiddleware::process($req, $runner)

Frame 2: ErrorHandlerMiddleware::process()
  → called $handler->handle($req)  = Runner::handle()

Frame 3: Runner::handle()  [position was 1]
  → called RoutingMiddleware::process($req, $runner)

Frame 4: RoutingMiddleware::process()
  → called $handler->handle($req)  = Runner::handle()

Frame 5: Runner::handle()  [position was 2]
  → called BodyParserMiddleware::process($req, $runner)

Frame 6: BodyParserMiddleware::process()
  → called $handler->handle($req)  = Runner::handle()

Frame 7: Runner::handle()  [position=3, queue->valid()==false]
  → called fallbackHandler->handle($req)

Frame 8: BaseApplication::handle()  ← executing NOW
────────────────────────────────────────────────────────────────
```

The cursor tracks *which* middleware to call next; the PHP call stack is the
execution depth. They are two separate concepts working together. When
`BaseApplication::handle()` returns a response, the call stack unwinds in
reverse: `BodyParserMiddleware` can modify the response, then
`RoutingMiddleware`, then `ErrorHandlerMiddleware`. This is the onion model
playing out on the real call stack.

### Full Execution Trace

```
Runner::handle()  [cursor=0]
  cursor → 1
  ErrorHandlerMiddleware::process($req, $runner)
    try {
      Runner::handle()  [cursor=1]
        cursor → 2
        RoutingMiddleware::process($req, $runner)
          $req = $req->withAttribute('controller', 'Articles')  [NEW $req object]
          Runner::handle()  [cursor=2]
            cursor → 3
            BodyParserMiddleware::process($req, $runner)
              if Content-Type is application/json:
                $req = $req->withParsedBody(json_decode(...))  [NEW $req object]
              Runner::handle()  [cursor=3, queue exhausted]
                BaseApplication::handle($req)
                  ControllerFactory::create($req)
                  ControllerFactory::invoke($ctrl)
                  ← ResponseInterface
              ← BodyParserMiddleware returns response (no modification here)
            ← RoutingMiddleware returns response (no modification here)
          ← response travels back up
        ← ErrorHandlerMiddleware returns response
    } catch (Throwable $e) {
      ← ErrorHandlerMiddleware renders error page
    }
```

---

## 2.5 — The Built-in Middleware Catalogue

CakePHP ships ten middleware classes. Nine live in `src/Http/Middleware/` and
one lives in `src/Error/Middleware/`. This distinction matters when importing:

```
src/Http/Middleware/
├── BodyParserMiddleware.php
├── ClosureDecoratorMiddleware.php    ← internal, not used directly
├── CspMiddleware.php
├── CsrfProtectionMiddleware.php
├── EncryptedCookieMiddleware.php
├── HttpsEnforcerMiddleware.php
├── RateLimitMiddleware.php
├── SecurityHeadersMiddleware.php
└── SessionCsrfProtectionMiddleware.php

src/Error/Middleware/
└── ErrorHandlerMiddleware.php        ← NOTE: different namespace!
```

| Middleware | Namespace | What it does in `process()` | Short-circuits? |
|---|---|---|---|
| `ErrorHandlerMiddleware` | `Cake\Error\Middleware` | Wraps the entire rest of the pipeline in `try/catch`; converts any `Throwable` into an error response via `ExceptionTrap` | Yes (on exception) |
| `HttpsEnforcerMiddleware` | `Cake\Http\Middleware` | Returns 301 redirect (GET) or throws 400 if scheme is not `https`; disabled in debug mode by default | Yes (if HTTP) |
| `SecurityHeadersMiddleware` | `Cake\Http\Middleware` | Calls the handler; adds `X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy` headers to the *response* on the way back | No |
| `BodyParserMiddleware` | `Cake\Http\Middleware` | Reads `Content-Type`; if `application/json`, decodes the body and sets it as parsed body on a *new* request object | No |
| `CsrfProtectionMiddleware` | `Cake\Http\Middleware` | On GET: sets a CSRF cookie. On POST/PUT/DELETE: verifies the token sent in the request matches the cookie | Yes (on mismatch) |
| `SessionCsrfProtectionMiddleware` | `Cake\Http\Middleware` | Same as above but stores the token in the session instead of a cookie — safer against XSS | Yes (on mismatch) |
| `EncryptedCookieMiddleware` | `Cake\Http\Middleware` | Decrypts named cookies inbound (on the request); encrypts them outbound (on the response) | No |
| `CspMiddleware` | `Cake\Http\Middleware` | Builds a `Content-Security-Policy` header from configuration and adds it to the response | No |
| `RateLimitMiddleware` | `Cake\Http\Middleware` | Enforces request rate limits; returns 429 Too Many Requests if the limit is exceeded | Yes (on limit hit) |
| `ClosureDecoratorMiddleware` | `Cake\Http\Middleware` | Wraps any `Closure` so it satisfies `MiddlewareInterface`; used internally by `MiddlewareQueue` | Depends on closure |

### Reading ErrorHandlerMiddleware in Full

This is the most important middleware to understand because it is always
outermost. Its `process()` method is only 7 lines:

```php
// src/Error/Middleware/ErrorHandlerMiddleware.php:112
public function process(
    ServerRequestInterface $request,
    RequestHandlerInterface $handler
): ResponseInterface {
    try {
        return $handler->handle($request);  // run everything inside it
    } catch (RedirectException $exception) {
        return $this->handleRedirect($exception);  // turn redirect exceptions into 301/302
    } catch (Throwable $exception) {
        return $this->handleException($exception, Router::getRequest() ?? $request);
        // Throwable catches EVERYTHING: exceptions AND PHP errors (TypeError, etc.)
        // handleException() logs the error and renders an HTML or JSON error page
    }
}
```

Because it catches `Throwable` (not just `Exception`), `ErrorHandlerMiddleware`
catches PHP 8.x `TypeError`, `ValueError`, `Error`, and all exceptions. Any
unhandled problem anywhere in the pipeline — routing, database, controller,
view — surfaces here as a proper HTTP error response instead of a blank page or
a raw PHP error.

**Why it must be outermost**: If it were not the first (outermost) layer, errors
thrown by outer middleware (like `HttpsEnforcerMiddleware`) would bypass it.
Error handling must wrap everything.

### Reading HttpsEnforcerMiddleware: Short-Circuit vs Pass-Through

This middleware illustrates both paths a middleware can take:

```php
// src/Http/Middleware/HttpsEnforcerMiddleware.php:84
public function process(
    ServerRequestInterface $request,
    RequestHandlerInterface $handler
): ResponseInterface {

    // PATH A: Already HTTPS (or debug mode bypasses it) → pass through
    if ($request->getUri()->getScheme() === 'https'
        || ($this->config['disableOnDebug'] && Configure::read('debug'))
    ) {
        $response = $handler->handle($request);  // pass to next middleware
        if ($this->config['hsts']) {
            // Add Strict-Transport-Security header to the response on the way OUT.
            // This header tells the browser to always use HTTPS for this domain.
            return $this->addHsts($response);
        }
        return $response;
    }

    // PATH B: HTTP + GET → redirect to HTTPS (short-circuit: no handler call)
    if ($this->config['redirect'] && $request->getMethod() === 'GET') {
        // PSR-7 immutability: withScheme() returns a NEW UriInterface object.
        // $request->getUri() is unchanged.
        $uri = $request->getUri()->withScheme('https');
        return new RedirectResponse($uri, 301);
        // Nothing after this line executes. The rest of the pipeline is skipped.
    }

    // PATH C: HTTP + non-GET → throw a 400 (also short-circuits)
    throw new BadRequestException('Requests to this URL must be made with HTTPS.');
    // ErrorHandlerMiddleware (the outermost layer) catches this and renders
    // a 400 Bad Request error page.
}
```

Trace the PSR-7 immutability on line 104 carefully: `$request->getUri()` returns
the existing `UriInterface`. `.withScheme('https')` returns a **new**
`UriInterface`. The original `$request` is never modified. This is the rule:
every `with*()` call produces a new object; always assign the result.

---

## 2.6 — BaseApplication::handle() — The Bridge from HTTP to MVC

When the middleware queue is exhausted, the `Runner` calls `$this->fallbackHandler->handle($request)`. That fallback handler is `BaseApplication`. Its `handle()` method is where the HTTP pipeline ends and the MVC layer begins:

```php
// src/Http/BaseApplication.php:343
public function handle(ServerRequestInterface $request): ResponseInterface
{
    $container = $this->getContainer();

    // Register the current request in the container.
    // Any service or controller that type-hints ServerRequest
    // in its constructor will now receive this exact request object via DI.
    $container->add(ServerRequest::class, $request);
    $container->add(ContainerInterface::class, $container);

    // Run event listener registration for the app and all plugins.
    // app->events() and plugin->events() register listeners on the
    // global EventManager. This runs here (at request time, not bootstrap)
    // so that any application-level event listeners are registered
    // before Controller.startup fires.
    $eventManager = $this->events($this->getEventManager());
    $this->setEventManager($this->pluginEvents($eventManager));

    // Build the ControllerFactory if this is the first request.
    // The ??= operator means "assign only if currently null".
    $this->controllerFactory ??= new ControllerFactory($container);

    // Keep the global Router's "current request" reference in sync.
    if (Router::getRequest() !== $request) {
        assert($request instanceof ServerRequest);
        Router::setRequest($request);
    }

    // Resolve and instantiate the controller for this request.
    $controller = $this->controllerFactory->create($request);

    // Run the controller action and return the response.
    return $this->controllerFactory->invoke($controller);
}
```

Two design decisions are worth examining:

1. **Request in the container** (lines 347–348): The request is added to the
   container on every request. This means any class registered in the
   container — services, repositories, or even the controller itself — can
   declare `ServerRequest $request` as a constructor parameter and receive the
   current request automatically, without it being passed explicitly.

2. **`BaseApplication` implements `RequestHandlerInterface`**: It has a
   `handle()` method that accepts a request and returns a response. This is
   what allows it to be passed as the `fallbackHandler` to `Runner::run()` at
   `Server.php:98`. The same interface that your middleware implements is the
   same interface `BaseApplication` implements — they are interchangeable at
   the `Runner` level.

---

## 2.7 — ControllerFactory: From Route Parameters to a Running Action

At this point in the flow, the `$request` object already carries routing
parameters attached by `RoutingMiddleware`. Those parameters look like:

```
$request->getParam('controller') = 'Articles'
$request->getParam('action')     = 'view'
$request->getParam('pass')       = ['5']   ← URL segments after the action
$request->getParam('plugin')     = null
$request->getParam('prefix')     = null
```

`ControllerFactory` takes those parameters and produces a fully-executed
controller response in three phases:

```
ControllerFactory::create($request)    src/Controller/ControllerFactory.php:72
│
│  1. getControllerClass($request)
│     Reads $request->getParam('controller') = 'Articles'
│     Applies convention: 'Articles' → 'App\Controller\ArticlesController'
│     (or plugin-namespaced if $request->getParam('plugin') is set)
│
│  2. new ReflectionClass($className)
│     PHP reflection lets us inspect the class without instantiating it.
│     If the class is abstract → throw MissingControllerException.
│
│  3. Build the controller:
│     a. $container->has($className)?
│        YES → $container->get($className)  ← full DI, dependencies injected
│        NO  → $reflection->newInstance($request)  ← minimal, request only
│
└──▶ Returns Controller instance

ControllerFactory::invoke($controller)  line 128
│
│  Does the controller have per-action middleware?
│  (set with $this->middleware() inside the controller)
│    YES → build a NEW MiddlewareQueue + Runner for this action only
│          the pipeline executes again, but scoped to this one action
│    NO  → call handle($request) directly
│
└──▶ ControllerFactory::handle($request)  line 150

ControllerFactory::handle($request)  line 150
│
│  1. controller->startupProcess()     Controller.php:600
│     Fires 'Controller.initialize' event → components' beforeFilter() runs
│     Fires 'Controller.startup' event   → components' startup() runs
│     Either event can return a ResponseInterface to short-circuit
│     (e.g. an Auth component that redirects unauthenticated users)
│
│  2. controller->getAction()
│     Returns a Closure that wraps the matched action method.
│     e.g. for 'view', it returns Closure for ArticlesController::view()
│
│  3. getActionArgs($action, $passedParams)   line 183
│     Uses PHP ReflectionFunction to inspect the action's parameters.
│     For each parameter:
│       - Non-built-in type (class/interface) → try $container->get(type)
│       - Built-in type (int, string, etc.)   → take next from $passedParams
│     See section 2.7a below for a full example.
│
│  4. controller->invokeAction($action, $args)
│     Calls the action closure with the resolved arguments.
│     The action runs: queries the database, sets view variables, etc.
│
│  5. controller->shutdownProcess()     Controller.php:624
│     Fires 'Controller.shutdown' event → components' afterFilter() runs
│     Fires view rendering (which writes HTML into the Response body)
│
└──▶ Returns ResponseInterface back up through the middleware call stack
```

### 2.7a — Action-Level Dependency Injection

CakePHP 5.x introduced parameter-by-parameter DI for controller actions. Given:

```php
// In ArticlesController:
public function view(int $id, ArticlesTable $articles): void
{
    $article = $articles->get($id);   // $articles injected, $id from URL
    $this->set(compact('article'));
}
```

`getActionArgs()` uses `ReflectionFunction` to inspect each parameter:

```
Parameter 1: int $id
  → type 'int' is a built-in → take first value from $passedParams
  → $passedParams[0] = '5' (the URL segment) → cast to int → 5
  → resolved: 5

Parameter 2: ArticlesTable $articles
  → type 'ArticlesTable' is a class → check $container->has(ArticlesTable::class)
  → if registered: $container->get(ArticlesTable::class) → injected from DI
  → if not registered: throw InvalidParameterException (must be registered)
  → resolved: ArticlesTable instance
```

This eliminates `$this->loadModel('Articles')` in every action. Register the
table once in `Application::services()` and every action that needs it gets it
automatically.

---

## 2.8 — Writing, Registering, and Debugging Your Own Middleware

Now that you understand the pipeline completely, you can write middleware with
confidence.

### Writing Middleware

Every middleware must implement `MiddlewareInterface` and its single method,
`process()`. Here is a complete, production-quality example:

```php
<?php
declare(strict_types=1);

namespace App\Middleware;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;

/**
 * Adds an X-Response-Time header to every response.
 */
class RequestTimingMiddleware implements MiddlewareInterface
{
    public function process(
        ServerRequestInterface $request,
        RequestHandlerInterface $handler,
    ): ResponseInterface {
        $start = hrtime(true);  // nanoseconds via monotonic clock (not wall clock)

        // Pass the request to the rest of the pipeline.
        // Everything deeper than this middleware runs during this call.
        $response = $handler->handle($request);

        // $response is now back. Calculate elapsed time.
        $ms = (hrtime(true) - $start) / 1_000_000;

        // PSR-7 RULE: withHeader() returns a NEW response object.
        // Assign it — do not discard the return value.
        return $response->withHeader('X-Response-Time', round($ms, 2) . 'ms');
    }
}
```

### Registering Middleware

```php
// In your Application class (extends BaseApplication):
use App\Middleware\RequestTimingMiddleware;
use Cake\Error\Middleware\ErrorHandlerMiddleware;  // note: Cake\Error, not Cake\Http
use Cake\Http\Middleware\BodyParserMiddleware;
use Cake\Routing\Middleware\RoutingMiddleware;

public function middleware(MiddlewareQueue $middlewareQueue): MiddlewareQueue
{
    $middlewareQueue
        ->add(new ErrorHandlerMiddleware())     // ALWAYS first — wraps everything
        ->add(new RequestTimingMiddleware())    // your middleware
        ->add(new BodyParserMiddleware())       // parse JSON/XML bodies
        ->add(new RoutingMiddleware($this));    // resolves route parameters

    return $middlewareQueue;
}
```

**Order is position in the onion — not execution priority.** The first
`add()` becomes the outermost layer: it runs first inbound and last outbound.
Always keep `ErrorHandlerMiddleware` first so it can catch errors from all
other layers.

### Positioning Middleware Precisely

When writing a plugin, you need to insert middleware at a specific position
relative to existing middleware rather than just appending:

```php
// Insert immediately after ErrorHandlerMiddleware:
$queue->insertAfter(
    ErrorHandlerMiddleware::class,
    new RequestTimingMiddleware(),
);
// src/Http/MiddlewareQueue.php:210

// Insert before RoutingMiddleware (e.g. auth must happen before routing resolves):
$queue->insertBefore(
    RoutingMiddleware::class,
    new AuthMiddleware(),
);
// src/Http/MiddlewareQueue.php:177

// Insert at a known index position:
$queue->insertAt(2, new CorsMiddleware());
// src/Http/MiddlewareQueue.php:159
```

`insertBefore` and `insertAfter` search for the first matching class in the
queue. If `insertAfter` does not find the target class, it falls back to `add()`
(appends to the end). `insertBefore` throws a `LogicException` if not found.

### Debugging Middleware

**Technique 1 — Inspect the queue at the last possible moment**

Hook into `Server.buildMiddleware` (fired at `Server.php:96`), which fires
*after* all middleware have been added but *before* the first request is
processed:

```php
// In config/bootstrap.php:
use Cake\Event\EventManager;

EventManager::instance()->on('Server.buildMiddleware', function ($event) {
    $queue = $event->getData('middleware');
    foreach ($queue as $i => $mw) {
        // During the event, items may still be strings/closures.
        // get_class() / gettype() shows what is in the queue.
        echo $i . ': ' . (is_object($mw) ? get_class($mw) : gettype($mw)) . PHP_EOL;
    }
});
```

**Technique 2 — Add an inline logging closure**

Insert a Closure at any position. `MiddlewareQueue` wraps it in
`ClosureDecoratorMiddleware` automatically:

```php
$middlewareQueue->add(function (ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface {
    error_log('[MW] before: ' . $request->getUri()->getPath());
    $response = $handler->handle($request);
    error_log('[MW] after:  status=' . $response->getStatusCode());
    return $response;
});
```

**Technique 3 — Count and iterate the queue**

`MiddlewareQueue` implements `Countable` (line 240) and `SeekableIterator`:

```php
$count = count($queue);                 // total layers
foreach ($queue as $index => $layer) {  // iterates, resolving each entry
    echo $index . ': ' . get_class($layer) . PHP_EOL;
}
```

Note that iterating resolves all strings/closures (lazy instantiation
triggers). This is fine for debugging but avoid doing it in production code
outside the pipeline execution.

### The Three Rules of Middleware

These rules are short. Breaking any of them causes bugs that are difficult to
trace because they show up far from the source.

```
RULE 1 — Always return a ResponseInterface.
  Never return null. Never call echo or print. Only the ResponseEmitter
  (in Server::emit()) writes output. Middleware must pass a response
  object back up the call stack.

RULE 2 — PSR-7 objects are immutable. Assign every with*() result.
  $request->withAttribute('user', $user);   // WRONG: result discarded
  $request = $request->withAttribute('user', $user);  // CORRECT
  $response->withHeader('X-Foo', 'bar');    // WRONG
  return $response->withHeader('X-Foo', 'bar');       // CORRECT

RULE 3 — Call $handler->handle($request) exactly once, or not at all.
  Calling it twice runs the entire rest of the pipeline twice — a subtle
  but severe bug.
  Not calling it (without returning a response) is a short-circuit —
  intentional (like a CSRF check that rejects the request) or a bug
  (you forgot to call it). Be deliberate about which case you intend.
```

---

## 2.9 — The Complete Request-to-Response Call Graph

Here is the entire journey of a normal GET request in one diagram, with file
and line references at every step:

```
webroot/index.php  (in your app, not in this repo)
  new Server($app)               Server.php:54
  $server->run()                 Server.php:75
    │
    ├─ Server::bootstrap()                      Server.php:115
    │    app->bootstrap()                       BaseApplication.php:184
    │      require_once config/bootstrap.php
    │    app->pluginBootstrap()                 BaseApplication.php:198
    │
    ├─ ServerRequestFactory::fromGlobals()
    │    builds PSR-7 ServerRequest from $_SERVER, $_GET, $_POST, etc.
    │
    ├─ new MiddlewareQueue([], $container)      MiddlewareQueue.php:63
    │
    ├─ app->middleware($queue)                  (your Application class)
    │    $queue->add(ErrorHandlerMiddleware)
    │    $queue->add(HttpsEnforcerMiddleware)
    │    $queue->add(BodyParserMiddleware)
    │    $queue->add(RoutingMiddleware)
    │
    ├─ app->pluginMiddleware($queue)            BaseApplication.php:127
    │
    ├─ dispatchEvent('Server.buildMiddleware')  Server.php:96
    │
    └─ Runner::run($queue, $req, $app)          Runner.php:51
         queue->rewind()
         Runner::handle($req)                  Runner.php:69   [cursor=0]
           │
           ├─ ErrorHandlerMiddleware::process()  Error/Middleware/ErrorHandlerMiddleware.php:112
           │    try {
           │      Runner::handle($req)          [cursor=1]
           │      │
           │      ├─ HttpsEnforcerMiddleware::process()  Http/Middleware/HttpsEnforcerMiddleware.php:84
           │      │    (HTTPS → pass through; HTTP GET → 301 redirect short-circuit)
           │      │    Runner::handle($req)     [cursor=2]
           │      │    │
           │      │    ├─ BodyParserMiddleware::process()  Http/Middleware/BodyParserMiddleware.php
           │      │    │    if Content-Type is JSON:
           │      │    │      $req = $req->withParsedBody(...)  ← NEW $req object (immutable)
           │      │    │    Runner::handle($req)  [cursor=3]   ← passes NEW $req forward
           │      │    │    │
           │      │    │    ├─ RoutingMiddleware::process()
           │      │    │    │    matches URL → attaches controller/action/pass params
           │      │    │    │    $req = $req->withAttribute(...)  ← NEW $req object
           │      │    │    │    Runner::handle($req)  [cursor=4, queue exhausted]
           │      │    │    │    │
           │      │    │    │    └─ BaseApplication::handle($req)   BaseApplication.php:343
           │      │    │    │         container->add(ServerRequest, $req)
           │      │    │    │         ControllerFactory::create($req)   CF.php:72
           │      │    │    │           resolve 'Articles' → ArticlesController class
           │      │    │    │           instantiate controller (via container or reflection)
           │      │    │    │         ControllerFactory::invoke($ctrl)  CF.php:128
           │      │    │    │           startupProcess()  → Controller.initialize event
           │      │    │    │                             → Controller.startup event
           │      │    │    │           getActionArgs()   → resolve action parameters
           │      │    │    │           invokeAction()    → run action (e.g. view())
           │      │    │    │           shutdownProcess() → Controller.shutdown event
           │      │    │    │                             → render view → build Response
           │      │    │    │         ← ResponseInterface
           │      │    │    │
           │      │    │    ← RoutingMiddleware returns response unchanged
           │      │    │
           │      │    ← BodyParserMiddleware returns response unchanged
           │      │
           │      ← HttpsEnforcerMiddleware may add HSTS header to response
           │
           │    } catch (Throwable $e) {
           │      ← ErrorHandlerMiddleware::handleException() renders error page
           │    }
           │
           ← ErrorHandlerMiddleware returns final response

    Server::emit($response)                    Server.php:139
      ResponseEmitter::emit()
        write HTTP headers to PHP output
        write body to PHP output
      dispatchEvent('Server.terminate')
        ← any post-response cleanup runs here
```

---

## Summary: Chapter 2 Reference Card

```
CONCEPT                       FILE:LINE
─────────────────────────────────────────────────────────────────────────
PSR-7 immutability rule       Demonstrated in HttpsEnforcerMiddleware.php:104
Server entry point            src/Http/Server.php:75       run()
Server response emission      src/Http/Server.php:139      emit()
Build empty queue             src/Http/MiddlewareQueue.php:63  __construct()
Add middleware to queue       src/Http/MiddlewareQueue.php:107 add()
Lazy resolution of strings    src/Http/MiddlewareQueue.php:278 current()
Resolution algorithm          src/Http/MiddlewareQueue.php:76  resolve()
Closure wrapping              src/Http/Middleware/ClosureDecoratorMiddleware.php:39
Pipeline execution            src/Http/Runner.php:51       run()
Per-layer execution (recursive) src/Http/Runner.php:69   handle()
App bootstrap                 src/Http/BaseApplication.php:184 bootstrap()
HTTP-to-MVC bridge            src/Http/BaseApplication.php:343 handle()
Controller instantiation      src/Controller/ControllerFactory.php:72  create()
Controller invocation         src/Controller/ControllerFactory.php:128 invoke()
Action argument DI            src/Controller/ControllerFactory.php:183 getActionArgs()
Controller lifecycle events   src/Controller/Controller.php:600  startupProcess()
Error middleware              src/Error/Middleware/ErrorHandlerMiddleware.php:112
```

---

## What's Coming in Chapter 3

You now understand how a request travels through the pipeline and arrives at
the controller. But one step in that journey was treated as a black box:
**Routing**. We said "RoutingMiddleware attaches controller/action parameters
to the request" without showing how a string like `/articles/view/5` becomes
the structured map `['controller' => 'Articles', 'action' => 'view', 'pass' =>
['5']]`.

Chapter 3 opens that box completely. We will read `src/Routing/Router.php`,
`src/Routing/RouteBuilder.php`, and several `Route` subclasses to understand
how routes are compiled, matched, and reversed. We will cover route scopes,
prefixes, resource routes, named routes, and the reverse-URL-generation system
that makes `Router::url()` work.

By the end of Chapter 3 you will be able to define any route pattern you need,
understand every routing option available, and debug routing mismatches without
guessing.
