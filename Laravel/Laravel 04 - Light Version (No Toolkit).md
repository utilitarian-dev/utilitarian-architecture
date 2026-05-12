# Laravel 04 - Light Version (No Toolkit)

The toolkit is optional. Utilitarian Architecture is a set of conventions, not a runtime dependency. This document describes how to implement the same architecture without installing `utilitarian-laravel-toolkit`.

The light version is not a simplified variant. It is the same architecture — Commands handle writes, Queries handle reads, Actions orchestrate — with manual wiring instead of automatic infrastructure.

Two scenarios where this applies:

- Starting a new project without the toolkit (the natural default for many teams)
- Adopting the architecture on an existing codebase before introducing new infrastructure

The light version intentionally accepts a small downgrade in wiring purity. Calling `boot()` from the constructor is a transitional Laravel-friendly compromise, not the strictest form of Two-Phase Initialization. It keeps vanilla Laravel usage simple while preserving a mechanical migration path to the toolkit.

## How it differs from the full version

| Aspect | Light version | Full version (with toolkit) |
| --- | --- | --- |
| `boot()` signature | `public function boot(Dep $dep): void` | `public function boot(Dep $dep): void` — **identical** |
| Who calls `boot()` | Constructor: `app()->call([$this, 'boot'])` | Bus (automatically, before `execute()`) |
| Invocation | `(new Foo(...))->execute()` | `Foo::make(...)->dispatch()` |
| Base / abstract classes | None | Provided by toolkit |
| Middleware support | Not available | Via `middleware()` method |
| `execute()` body | **Identical** | **Identical** |

The key design constraint: `boot()` has the same signature in both versions. The only structural difference is where it is called from. This makes migration mechanical.

## Commands

A Command handles one write operation. The constructor receives business parameters only. When a Command needs external dependencies, `boot()` declares typed dependencies and the container resolves them via `app()->call()`.

```php
class CreatePostCommand
{
    public function __construct(
        public readonly int $authorId,
        public readonly string $title,
        public readonly string $body,
    ) {}

    public function execute(): Post
    {
        return Post::query()->create([
            'author_id' => $this->authorId,
            'title'     => $this->title,
            'body'      => $this->body,
        ]);
    }
}
```

Commands that use Eloquent or the Query Builder directly do not need `boot()`. Use `boot()` only when the operation depends on an external client, adapter, or configuration object.

### Usage

```php
$post = (new CreatePostCommand(
    authorId: $request->user()->id,
    title: $request->string('title'),
    body: $request->string('body'),
))->execute();
```

## Queries

A Query handles one read operation with zero side effects.

### Without boot

When a Query uses Eloquent directly, there is nothing to inject. No `boot()` needed.

```php
class GetPublishedPostsQuery
{
    public function __construct(
        public readonly int $limit = 20,
    ) {}

    public function execute(): Collection
    {
        return Post::query()
            ->where('status', 'published')
            ->latest()
            ->limit($this->limit)
            ->get();
    }
}
```

Only introduce `boot()` when there is an external dependency to resolve.

### With boot

```php
class FindPostsByTagQuery
{
    private SearchIndexClient $search;

    public function __construct(
        public readonly string $tag,
    ) {
        app()->call([$this, 'boot']);
    }

    public function boot(SearchIndexClient $search): void
    {
        $this->search = $search;
    }

    public function execute(): Collection
    {
        return $this->search->findByTag($this->tag);
    }
}
```

### Usage

```php
$posts  = (new GetPublishedPostsQuery(limit: 10))->execute();
$tagged = (new FindPostsByTagQuery(tag: 'laravel'))->execute();
```

## Actions

An Action encapsulates business logic and orchestration. It coordinates Commands and Queries, applies rules, and produces an outcome.

```php
class RegisterUserAction
{
    private WelcomeEmailSender $welcomeEmail;

    public function __construct(
        public readonly string $name,
        public readonly string $email,
        public readonly string $password,
    ) {
        app()->call([$this, 'boot']);
    }

    public function boot(WelcomeEmailSender $welcomeEmail): void
    {
        $this->welcomeEmail = $welcomeEmail;
    }

    public function execute(): User
    {
        $user = (new CreateUserCommand(
            name: $this->name,
            email: $this->email,
            password: $this->password,
        ))->execute();

        $this->welcomeEmail->send($user);

        return $user;
    }
}
```

The Action resolves only its own dependencies in `boot()`. Downstream Commands and Queries wire themselves independently through their own constructors.

### Usage

```php
$user = (new RegisterUserAction(
    name: $request->string('name'),
    email: $request->string('email'),
    password: $request->string('password'),
))->execute();
```

## Combining Commands and Queries

A realistic multi-step Action that reads, writes, and notifies:

```php
class CreateAndPublishPostAction
{
    private NotificationClient $notifications;

    public function __construct(
        public readonly int $authorId,
        public readonly string $title,
        public readonly string $body,
    ) {
        app()->call([$this, 'boot']);
    }

    public function boot(NotificationClient $notifications): void
    {
        $this->notifications = $notifications;
    }

    public function execute(): Post
    {
        $author = (new GetAuthorQuery(id: $this->authorId))->execute();

        $post = (new CreatePostCommand(
            authorId: $this->authorId,
            title: $this->title,
            body: $this->body,
        ))->execute();

        (new PublishPostCommand(postId: $post->id))->execute();

        $this->notifications->notifyFollowers($author, $post);

        return $post;
    }
}
```

Each building block is responsible for its own wiring. The Action is responsible for the sequence and the outcome. The body of `execute()` reads as a business narrative.

## Entry points

All routing rules from Laravel 02 and entry point rules from Laravel 01 apply unchanged. The only difference from the full version is the invocation style: `(new Foo(...))->execute()` instead of `Foo::make(...)->dispatch()`.

### Route closure

```php
Route::post('/posts', function (Request $request) {
    $post = (new CreateAndPublishPostAction(
        authorId: $request->user()->id,
        title: $request->string('title'),
        body: $request->string('body'),
    ))->execute();

    return response()->json($post, 201);
})->name('posts.create');
```

### Controller

```php
class PostController
{
    public function store(StorePostRequest $request): JsonResponse
    {
        $post = (new CreateAndPublishPostAction(
            authorId: $request->user()->id,
            title: $request->validated('title'),
            body: $request->validated('body'),
        ))->execute();

        return new JsonResponse(new PostResource($post), 201);
    }
}
```

The controller handles HTTP concerns: form requests, authorization, response formatting, status codes. The Action handles business logic. The boundary is identical to the full version.

## Migrating to the full version

When ready to add `utilitarian-laravel-toolkit`, the migration is mechanical. The architecture does not change — only the wiring mechanism does.

### Step 1: Install the package

```
composer require utilitarian/laravel-toolkit
```

### Step 2: Update Commands and Actions

For operations that define `boot()`, remove `app()->call([$this, 'boot'])` from the constructor. That is the only change to the class itself.

Before:

```php
public function __construct(
    public readonly string $email,
) {
    app()->call([$this, 'boot']); // remove this line
}

public function boot(WelcomeEmailSender $welcomeEmail): void
{
    $this->welcomeEmail = $welcomeEmail; // no changes
}
```

After:

```php
public function __construct(
    public readonly string $email,
) {}

public function boot(WelcomeEmailSender $welcomeEmail): void
{
    $this->welcomeEmail = $welcomeEmail;
}
```

`execute()` requires no changes.

### Step 3: Update Queries

Queries that use Eloquent directly (no `boot()`) require no changes at all.

Queries that have a `boot()` follow the same procedure as Commands: remove `app()->call([$this, 'boot'])` from the constructor.

### Step 4: Update invocation in Actions

Inside `execute()`, replace direct instantiation with `::make()->dispatch()`:

```php
// Before:
$user = (new CreateUserCommand(
    name: $this->name,
    email: $this->email,
    password: $this->password,
))->execute();

// After:
$user = CreateUserCommand::make(
    name: $this->name,
    email: $this->email,
    password: $this->password,
)->dispatch();
```

`::make()` returns the operation instance, allowing middleware or other configuration to be chained before `dispatch()`:

```php
$user = CreateUserCommand::make(name: $this->name, email: $this->email, password: $this->password)
    ->middleware(TransactionMiddleware::class)
    ->dispatch();
```

### Step 5: Update invocation at entry points

```php
// Before:
$post = (new CreateAndPublishPostAction(
    authorId: $request->user()->id,
    title: $request->string('title'),
    body: $request->string('body'),
))->execute();

// After:
$post = CreateAndPublishPostAction::make(
    authorId: $request->user()->id,
    title: $request->string('title'),
    body: $request->string('body'),
)->dispatch();
```

### Migration summary

| What changes | Light version | Full version |
| --- | --- | --- |
| `boot()` visibility and signature | `public boot(Dep $dep): void` | `public boot(Dep $dep): void` — no change |
| Who calls `boot()` | `app()->call([$this, 'boot'])` in constructor | Bus, automatically |
| Change required per class | Remove one line from constructor | — |
| Invocation at call sites | `(new Foo(...))->execute()` | `Foo::make(...)->dispatch()` |
| Invocation of nested ops | `(new Bar(...))->execute()` | `Bar::make(...)->dispatch()` |
| Middleware, chaining | Not available | `->middleware(...)->dispatch()` |
| `execute()` body | No changes | No changes |

If the light version was built with discipline — no logic in constructors, no business rules in `boot()`, all logic in `execute()` — migration is a find-and-replace exercise, not a refactor.

## What stays the same

Every architectural rule documented elsewhere in this series applies unchanged to the light version:

- Naming conventions (Commands, Queries, Actions)
- Directory structure and grouping by domain
- Routing rules (Laravel 02 — Routing Rules)
- Entry point rules (Laravel 01 — HTTP Layer and Entry Points)
- The decision framework: when to use Command vs Query vs Action vs Eloquent directly
- The two-phase initialization concept: constructor for business parameters, `boot()` for infrastructure, `execute()` for logic
- The rule: no business logic in `boot()`, no dependency resolution in `execute()`

The light version is not a different architecture. It is the same architecture with different plumbing.
