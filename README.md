# Hibla Database

The official documentation for the Hibla PHP database ecosystem: a fully asynchronous, framework-agnostic database layer built on fibers/promises and non-blocking I/O.

## Table of Contents

- [Overview](#overview)
- [Quick Start](#quick-start)
- [Packages](#packages)
- [Requirements](#requirements)
- [Supported Drivers](#supported-drivers)
- [Installation](#installation)
  - [Query Builder Only](#query-builder-only)
  - [With Schema Manager](#with-schema-manager)
- [Configuration](#configuration)
  - [The Config File](#the-config-file)
  - [Environment Variables](#environment-variables)
  - [Custom Config Locations](#custom-config-locations)
- [Getting Started](#getting-started)
  - [Bootstrapping the Connection](#bootstrapping-the-connection)
  - [Using the DB Facade](#using-the-db-facade)
    - [Closing Connections](#closing-connections)
    - [Testing with setSqlClient](#testing-with-setsqlclient)
    - [Dynamic Connection Management](#dynamic-connection-management)
    - [Raw Queries](#raw-queries)
  - [Dependency Injection Mode](#dependency-injection-mode)
    - [Choosing an Interface](#choosing-an-interface)
    - [Constructor Injection](#constructor-injection)
    - [Registering with a DI Container](#registering-with-a-di-container)
- [Query Builder](#query-builder)
  - [Reading Data](#reading-data)
    - [Fetching Data](#fetching-data)
    - [Selecting Columns](#selecting-columns)
    - [Where Conditions](#where-conditions)
    - [Advanced Conditions](#advanced-conditions)
    - [Raw Expressions](#raw-expressions)
    - [Conditional Query Building](#conditional-query-building)
    - [JSON Conditions](#json-conditions)
    - [Joins](#joins)
    - [Grouping and Ordering](#grouping-and-ordering)
    - [Aggregates and Existence](#aggregates-and-existence)
    - [Pagination](#pagination)
  - [Writing Data](#writing-data)
    - [Inserting Data](#inserting-data)
    - [Updating Data](#updating-data)
    - [Deleting Data](#deleting-data)
  - [Advanced](#advanced)
    - [Processing Large Datasets (Streams vs Chunks)](#processing-large-datasets-streams-vs-chunks)
    - [Transactions](#transactions)
    - [Common Table Expressions](#common-table-expressions)
    - [Pessimistic Locking](#pessimistic-locking)
    - [Debugging](#debugging)
- [Schema Manager](#schema-manager)
  - [Installing the Schema Manager](#installing-the-schema-manager)
  - [Initializing](#initializing)
  - [Custom CLI Entry Point](#custom-cli-entry-point)
  - [CLI Reference](#cli-reference)
  - [Migrations](#migrations)
    - [Creating Migration Files](#creating-migration-files)
    - [Writing Migrations](#writing-migrations)
    - [Blueprint Column Types](#blueprint-column-types)
    - [Running Migrations](#running-migrations)
    - [Rolling Back](#rolling-back)
    - [Migration Status](#migration-status)
    - [Schema Dump](#schema-dump)
  - [Seeders](#seeders)
    - [Creating Seeders](#creating-seeders)
    - [Writing Seeders](#writing-seeders)
    - [Running Seeders](#running-seeders)
  - [Multiple Connections](#multiple-connections)
  - [Safe Mode](#safe-mode)
- [License](#license)
---

## Overview

Hibla Database is an asynchronous PHP database layer designed for long-running processes, high-concurrency workloads, and fiber-based PHP applications. It is built on the Hibla async runtime and provides a fluent, immutable query builder with an optional schema management layer.

Every execution method returns a `PromiseInterface` and must be awaited using `await()` or chain it with `then()`.  This is not a synchronous ORM; it is a high-level, non-blocking database abstraction layer designed to be composed into whatever architecture your application needs.

---

## Quick Start

```bash
composer require hiblaphp/database
```

Add your credentials to a `.env` file:

```dotenv
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=myapp
DB_USERNAME=root
DB_PASSWORD=secret
```

Copy the default config to your project root:

```bash
./vendor/bin/hibla-db init
```

Then run your first query:

```php
<?php

require 'vendor/autoload.php';

use Hibla\QueryBuilder\DB;
use function Hibla\await;

$users = await(DB::table('users')->where('active', true)->get());

foreach ($users as $user) {
    echo $user->name . PHP_EOL;
}

DB::close();
```

The sections below cover configuration, the full query builder API, and the schema manager in detail.

---

## Packages

| Package | Description | Standalone |
| :--- | :--- | :---: |
| `hiblaphp/database` | Meta package that installs both packages below | ✅ Yes |
| `hiblaphp/query-builder` | Async query builder, connection pooling, transactions, pagination | ✅ Yes |
| `hiblaphp/schema-manager` | Migrations, seeders, CLI tooling | Requires query-builder |

> `hiblaphp/database` is a convenience meta package that pulls in both `hiblaphp/query-builder` and `hiblaphp/schema-manager` together. If you want the full stack, install that instead. The two packages are documented here under a single unified reference because of it.

---

## Requirements

- PHP 8.4 or higher
- Fiber support (PHP 8.1+, enabled by default in 8.4)
- MySQL 8.0+ or PostgreSQL 15+

---

## Supported Drivers

| Driver | Status | Notes |
| :--- | :---: | :--- |
| MySQL / MariaDB | ✅ | Full support including locking, JSON, CTEs |
| PostgreSQL | ✅ | Full support including JSONB, vector columns, schema dump |
| SQLite | ⚠️ | No async support yet; schema manager has limited ALTER support |

SQLite currently has no async driver support. An async SQLite client is in development and will be integrated into the query builder once ready, bringing SQLite to full parity with the MySQL and PostgreSQL drivers.

---

## Installation

### Query Builder Only

If you only need the query builder without schema management:

```bash
composer require hiblaphp/query-builder
```

Then copy the default config file to your project root:

```bash
cp vendor/hiblaphp/query-builder/hibla-database.php hibla-database.php
```

### With Schema Manager

The schema manager includes the query builder as a dependency, so you only need one command:

```bash
composer require hiblaphp/schema-manager
```

Then run the init command to scaffold all config files at once:

```bash
./vendor/bin/hibla-db init
```

---

## Configuration

### The Config File

The query builder looks for `hibla-database.php` in your project root or in a `config/` subdirectory. This file returns a plain PHP array:

```php
<?php
// hibla-database.php

use function Rcalicdan\ConfigLoader\env;

require 'vendor/autoload.php';

return [
    /*
    |--------------------------------------------------------------------------
    | Default Connection
    |--------------------------------------------------------------------------
    */
    'default' => env('DB_CONNECTION', 'mysql'),

    /*
    |--------------------------------------------------------------------------
    | Database Connections
    |--------------------------------------------------------------------------
    */
    'connections' => [
        'mysql' => [
            'driver'   => 'mysql',
            'host'     => env('DB_HOST', '127.0.0.1'),
            'port'     => env('DB_PORT', 3306, convertNumeric: true),
            'database' => env('DB_DATABASE', 'myapp'),
            'username' => env('DB_USERNAME', 'root'),
            'password' => env('DB_PASSWORD', ''),
            'charset'  => 'utf8mb4',

            // Connection pool
            'max_connections'  => env('DB_MAX_CONNECTIONS', 10, convertNumeric: true),
            'min_connections'  => env('DB_MIN_CONNECTIONS', 0,  convertNumeric: true),
            'idle_timeout'     => 60,
            'max_lifetime'     => 3600,
            'acquire_timeout'  => 10.0,
        ],

        'pgsql' => [
            'driver'   => 'pgsql',
            'host'     => env('DB_HOST', '127.0.0.1'),
            'port'     => env('DB_PORT', 5432, convertNumeric: true),
            'database' => env('DB_DATABASE', 'myapp'),
            'username' => env('DB_USERNAME', 'postgres'),
            'password' => env('DB_PASSWORD', ''),

            'max_connections' => env('DB_MAX_CONNECTIONS', 10, convertNumeric: true),
            'min_connections' => env('DB_MIN_CONNECTIONS', 0,  convertNumeric: true),
        ],
    ],

    /*
    |--------------------------------------------------------------------------
    | Pagination
    |--------------------------------------------------------------------------
    |
    | templates_path — null uses the built-in templates. Set to an absolute
    | directory path after running `publish:templates` to use custom templates.
    |
    | default_template        — template used by paginate()
    | default_cursor_template — template used by cursorPaginate()
    |
    | Available built-in names: tailwind, bootstrap, simple,
    |                            cursor-tailwind, cursor-bootstrap, cursor-simple
    */
    'pagination' => [
        'templates_path'          => null,
        'default_template'        => 'tailwind',
        'default_cursor_template' => 'cursor-tailwind',
    ],
];
```

### Environment Variables

Create a `.env` file in your project root:

```dotenv
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=myapp
DB_USERNAME=root
DB_PASSWORD=secret
```

### Custom Config Locations

If your config files live somewhere other than the project root or `config/`, point to them with environment variables:

```dotenv
# Path relative to project root, without .php extension
HIBLA_DB_CONFIG=app/config/database
HIBLA_MIGRATIONS_CONFIG=app/config/migrations
HIBLA_SEEDERS_CONFIG=app/config/seeders
```

The `init` command handles this automatically when you use the `--dir` or custom name options, and it prints the exact variables you need to add:

```bash
# Place all configs in a config/ directory
./vendor/bin/hibla-db init --dir=config

# Custom file names
./vendor/bin/hibla-db init \
  --db-config=database \
  --migrations-config=migrations \
  --seeders-config=seeders
```

---

## Getting Started

### Bootstrapping the Connection

Connections are lazy and nothing opens until the first query. The `DB` facade initializes the `DatabaseManager` on first use, which reads your config and sets up the connection pool.

```php
<?php

require 'vendor/autoload.php';

use Hibla\QueryBuilder\DB;
use function Hibla\await;
use function Hibla\run;


$users = await(DB::table('users')->where('active', true)->get());

foreach ($users as $user) {
    echo $user->name . PHP_EOL;
}

DB::close(); // close the pool when done
```

### Using the DB Facade

`DB` is a static facade over a singleton `DatabaseManager` instance. The manager is created on the first call to any `DB` method and reused for the lifetime of the process. Being a singleton does not mean being limited to one database and the manager maintains a registry of named connection pools, and you can register as many as your application needs. Each pool is lazy: nothing opens until the first query hits that connection.

```php
use Hibla\QueryBuilder\DB;
use function Hibla\await;

// Default connection — resolves from DB_CONNECTION in your config
$user = await(DB::table('users')->find(1));

// A second named connection — its pool is opened independently on first use
$stats = await(DB::connection('pgsql')->table('events')->count());

// A third connection registered at runtime
$orders = await(DB::connection('tenant_acme')->table('orders')->get());
```

Each named connection has its own isolated pool with its own `min_connections`, `max_connections`, and lifecycle settings. Queries on `mysql` never share a pool slot with queries on `pgsql` or any other registered connection — they are fully independent.

Because `DB` is a singleton, all code in the same process shares the same pool registry. For testing, `DB::reset()` destroys the singleton entirely so the next call starts fresh.

---

#### Closing Connections

```php
// Close one specific pool
DB::close('pgsql');

// Close all registered pools
DB::close();

// Async variants (returns PromiseInterface<void>)
await(DB::closeAsync('pgsql'));
await(DB::closeAsync());
```

Calling `DB::close()` without a name iterates every registered connection and closes its pool. The async variant does the same but waits for all pools to drain cleanly before resolving.

Calling `close()` or `closeAsync()` is not strictly required. If you do not call it, the underlying connection pool will be closed automatically when the `DatabaseManager` instance is garbage collected. You should close explicitly when you want to control the shutdown order and for example, at the end of a long-running worker loop, or in a test teardown to ensure pools do not bleed between test cases.

---

#### Testing with setSqlClient

`DB::setSqlClient()` is the intended escape hatch for injecting a mock or in-memory client without touching any config file. It registers the client as the default connection and sets it as the active default:

```php
use Hibla\QueryBuilder\DB;
use Hibla\QueryBuilder\Enums\DatabaseDriver;

// Inject a mock client before the test runs
DB::setSqlClient($mockSqlClient, DatabaseDriver::Mysql);

// All DB::table() calls now run against the mock
$user = await(DB::table('users')->find(1));

// Tear down after the test
DB::reset();
```

For services that use constructor injection via `DatabaseConnectionInterface`, you can swap the binding in the container instead and never touch `DB` directly and see [Dependency Injection Mode](#dependency-injection-mode).

---

#### Dynamic Connection Management

`addConnection()` and `removeConnection()` let you register and deregister named connections at runtime without touching the config file. This is most useful when building custom schema builders, multi-tenant systems, or tooling that resolves connections dynamically:

```php
use Hibla\QueryBuilder\DB;
use Hibla\QueryBuilder\Internals\DatabaseConnection;

// Register a connection built from an arbitrary config array
$client = DB::resolveClientFromConfig([
    'driver'          => 'pgsql',
    'host'            => $tenantHost,
    'database'        => $tenantDatabase,
    'username'        => $tenantUser,
    'password'        => $tenantPassword,
    'max_connections' => 5,
]);

$connection = new DatabaseConnection($client, 'pgsql');
DB::addConnection('tenant_' . $tenantId, $connection);

// Use it by name
$orders = await(DB::connection('tenant_' . $tenantId)->table('orders')->get());

// Remove and close the pool when done
DB::removeConnection('tenant_' . $tenantId);
```

`removeConnection()` closes the pool synchronously before deregistering the name, so the connection is fully cleaned up in one call. If you need to close it asynchronously first, call `closeAsync()` on the connection before removing it:

```php
$conn = DB::connection('tenant_' . $tenantId);
await($conn->closeAsync());
DB::removeConnection('tenant_' . $tenantId);
```

---

#### Raw Queries

All raw methods bypass the query builder entirely and execute SQL directly against the active connection. They are useful for complex queries that the builder cannot express, database-specific syntax, or bulk operations where you want full control.

```php
// Execute and return all rows
$results = await(DB::raw('SELECT * FROM users WHERE active = ?', [true]));
// Returns array<int, array<string, mixed>>

// Return only the first row, or null
$user = await(DB::rawFirst('SELECT * FROM users WHERE id = ?', [1]));

// Return a single scalar value from the first column of the first row
$count = await(DB::rawValue('SELECT COUNT(*) FROM users WHERE active = ?', [true]));

// Execute a write statement — returns number of affected rows
$affected = await(DB::rawExecute(
    'UPDATE users SET last_seen_at = NOW() WHERE id IN (?, ?, ?)',
    [1, 2, 3]
));

// Execute and return an unbuffered row stream (for large result sets)
$stream = await(DB::rawStream(
    'SELECT * FROM events WHERE created_at > ?',
    ['2024-01-01'],
    bufferSize: 200
));

foreach ($stream as $row) {
    handle($row);
}

$stream->cancel(); // stop early if needed
```

All five raw methods are also available directly on any `DatabaseConnectionInterface` instance:

```php
$conn = DB::connection('pgsql');

$results  = await($conn->raw($sql, $bindings));
$first    = await($conn->rawFirst($sql, $bindings));
$value    = await($conn->rawValue($sql, $bindings));
$affected = await($conn->rawExecute($sql, $bindings));
$stream   = await($conn->rawStream($sql, $bindings, bufferSize: 100));
```

> Raw methods pass bindings as a flat positional array (`[true]`, `[1, 2, 3]`). They do not go through the query builder's binding compiler, so the caller is responsible for ordering values correctly relative to the `?` placeholders.

### Dependency Injection Mode

For testable architectures, the query builder exposes clean interfaces. Prefer injecting these over depending on the static `DB` facade in your services.

#### Choosing an Interface

| Interface | Use when |
| :--- | :--- |
| `DatabaseConnectionInterface` | Your service needs transactions, raw queries, or dynamic table access |
| `QueryBuilderInterface` | Your service is scoped to one table and may need transactions |
| `BaseQueryBuilderInterface` | Your service only reads data and never needs to start transactions |

#### Constructor Injection

```php
<?php

namespace App\Repositories;

use Hibla\Promise\Interfaces\PromiseInterface;
use Hibla\QueryBuilder\Interfaces\DatabaseConnectionInterface;

class UserRepository
{
    public function __construct(
        private readonly DatabaseConnectionInterface $db
    ) {}

    /** @return PromiseInterface<list<array<string, mixed>>> */
    public function getActive(): PromiseInterface
    {
        return $this->db->table('users')
            ->where('active', true)
            ->latest()
            ->get();
    }

    /** @return PromiseInterface<array<string, mixed>|null> */
    public function findById(int $id): PromiseInterface
    {
        return $this->db->table('users')->find($id);
    }

    /** @return PromiseInterface<int> */
    public function create(array $data): PromiseInterface
    {
        return $this->db->table('users')->insertGetId($data);
    }
}
```

When your service is scoped to a single table and may need transactions, use `QueryBuilderInterface`:

```php
use Hibla\QueryBuilder\Interfaces\QueryBuilderInterface;

class PostRepository
{
    public function __construct(
        private readonly QueryBuilderInterface $query
    ) {}

    public function published(): PromiseInterface
    {
        return $this->query->where('published', true)->latest()->get();
    }
}
```

When your service only reads data and never needs to start transactions, use `BaseQueryBuilderInterface` as the narrowest possible contract.

#### Registering with a DI Container

Bind the interfaces to concrete instances in your container bootstrap. The pattern applies to any PSR-11-compatible container:

```php
<?php
// bootstrap/container.php

use Hibla\QueryBuilder\DB;
use Hibla\QueryBuilder\Interfaces\DatabaseConnectionInterface;
use Hibla\QueryBuilder\Interfaces\QueryBuilderInterface;

// Bind the default connection
$container->bind(DatabaseConnectionInterface::class, fn() => DB::connection());

// Bind a named connection
$container->bind('db.analytics', fn() => DB::connection('pgsql'));

// Bind a pre-scoped query builder for a specific service
$container->bind(QueryBuilderInterface::class, fn() => DB::connection()->table('users'));
```

Usage in a service:

```php
<?php

namespace App\Services;

use App\Repositories\UserRepository;
use Hibla\QueryBuilder\Interfaces\DatabaseConnectionInterface;
use function Hibla\await;

class UserService
{
    public function __construct(
        private readonly UserRepository $users,
        private readonly DatabaseConnectionInterface $db
    ) {}

    public function register(array $data): int
    {
        return await(
            $this->db->transaction(function ($tx) use ($data) {
                $userId = await($tx->table('users')->insertGetId($data));
                await($tx->table('user_profiles')->insert(['user_id' => $userId]));
                return $userId;
            })
        );
    }
}
```

> For testing, swap the `DatabaseConnectionInterface` binding with a mock or an in-memory connection without changing any service code.

---

## Query Builder

All query builder methods return a new immutable instance and calling a method never modifies the original. See the [query builder primitives documentation](https://github.com/rcalicdan/query-builder-primitives) for full details on immutability. The sections below document the execution side.

> **Default result shape: `stdClass` objects.** Execution methods that return rows (`get()`, `first()`, `find()`, `findOrFail()`, `pluck()`, and the streaming methods) return `stdClass` objects by default. If you prefer associative arrays, call `toArray()` anywhere in the chain before executing. To be explicit about objects, use `toObject()`. Both methods return a new immutable instance and do not affect the original query.
>
> ```php
> // stdClass — default, no call needed
> $users = await(DB::table('users')->get());
> echo $users[0]->name;
>
> // Associative array
> $users = await(DB::table('users')->toArray()->get());
> echo $users[0]['name'];
>
> // Explicit object (same as default, useful for clarity in DI contexts)
> $users = await(DB::table('users')->toObject()->get());
> echo $users[0]->name;
> ```
>
> The `toArray()` / `toObject()` setting is scoped to the query instance it is called on. It does not affect other queries built from the same base or any global state.

---

## Reading Data

### Fetching Data

```php
use Hibla\QueryBuilder\DB;
use function Hibla\await;

// All rows
$users = await(DB::table('users')->get());

// First row, or null
$user = await(DB::table('users')->where('email', 'alice@example.com')->first());

// First row or throw RecordNotFoundException
$user = await(DB::table('users')->where('status', 'active')->firstOrFail());

// Find by primary key
$user = await(DB::table('users')->find(42));

// Find or throw RecordNotFoundException
$user = await(DB::table('users')->findOrFail(42));
```

The difference between `firstOrFail()` and `findOrFail()` is intent. `findOrFail()` is for looking up a specific record by a known ID and it adds a `WHERE id = ?` clause for you. `firstOrFail()` is for when you have already built your conditions and simply want to assert that at least one matching record exists, throwing if none do.

```php
// findOrFail — shorthand for a primary key lookup
$user = await(DB::table('users')->findOrFail(42));

// firstOrFail — asserts a match exists for your own conditions
$user = await(
    DB::table('users')
        ->where('email', $email)
        ->where('active', true)
        ->firstOrFail()
);

// Single column value from first row
$email = await(DB::table('users')->where('id', 1)->value('email'));

// Array of values from one column
$emails = await(DB::table('users')->where('active', true)->pluck('email'));
// ['alice@example.com', 'bob@example.com']

// key => value pairs
$names = await(DB::table('users')->pluck('name', 'id'));
// [1 => 'Alice', 2 => 'Bob']

// Results as objects (default)
$users = await(DB::table('users')->toObject()->get());
// $users[0]->name

// Results as associative arrays
$users = await(DB::table('users')->toArray()->get());
// $users[0]['name']
```

---

### Selecting Columns

```php
// Specific columns
$users = await(DB::table('users')->select('id', 'name', 'email')->get());

// Add columns to an existing select
$query = DB::table('users')->select('id', 'name');
$query = $query->addSelect('email', 'created_at');

// DISTINCT
$countries = await(DB::table('users')->selectDistinct('country')->get());
```

---

### Where Conditions

```php
// Equality (two-argument shorthand)
$active = await(DB::table('users')->where('status', 'active')->get());

// With explicit operator
$seniors = await(DB::table('users')->where('age', '>=', 65)->get());

// OR WHERE
$results = await(
    DB::table('users')
        ->where('status', 'active')
        ->orWhere('status', 'pending')
        ->get()
);

// WHERE IN / NOT IN
$selected = await(DB::table('users')->whereIn('id', [1, 2, 3])->get());
$excluded = await(DB::table('users')->whereNotIn('role', ['guest', 'banned'])->get());

// BETWEEN
$range = await(DB::table('orders')->whereBetween('total', [100, 500])->get());

// NULL checks
$unverified = await(DB::table('users')->whereNull('verified_at')->get());
$verified   = await(DB::table('users')->whereNotNull('verified_at')->get());

// LIKE — side: 'both' (default), 'before', 'after'
$search = await(DB::table('users')->like('name', 'john')->get());
// name LIKE '%john%'

$starts = await(DB::table('users')->like('username', 'admin', 'after')->get());
// username LIKE 'admin%'

// Column comparison — no binding, no injection risk
$mismatch = await(DB::table('audit_log')->whereColumn('expected_hash', '!=', 'actual_hash')->get());

// Reset all WHERE conditions
$base  = DB::table('users')->where('active', true);
$clean = $base->resetWhere();
```

---

### Advanced Conditions

```php
// Grouped conditions — (status = ? AND role = ?) OR (status = ? AND invited = ?)
$results = await(
    DB::table('users')
        ->whereGroup(fn($q) => $q->where('status', 'active')->where('role', 'admin'))
        ->orWhereNested(fn($q) => $q->where('status', 'pending')->where('invited', true))
        ->get()
);

// EXISTS subquery
$usersWithOrders = await(
    DB::table('users')
        ->whereExists(function ($q) {
            return $q->from('orders')
                     ->whereRaw('orders.user_id = users.id')
                     ->where('total', '>', 1000);
        })
        ->get()
);

// NOT EXISTS
$usersWithoutOrders = await(
    DB::table('users')
        ->whereNotExists(function ($q) {
            return $q->from('orders')->whereRaw('orders.user_id = users.id');
        })
        ->get()
);

// Subquery in WHERE
$results = await(
    DB::table('products')
        ->whereSub('price', '>', function ($q) {
            return $q->from('products')->selectRaw('AVG(price)');
        })
        ->get()
);
```

---

### Raw Expressions

Whenever you need to bypass the query builder's abstraction and write raw SQL conditions or projections, use the `*Raw` methods.

> **Security Note:** Always use parameter bindings (`?`) for any user-provided input inside raw expressions to prevent SQL injection. Do not concatenate strings directly into raw SQL.

```php
// selectRaw — easily inject functions or complex aliases
$users = await(DB::table('users')
    ->selectRaw('COUNT(*) as total, YEAR(created_at) as year')
    ->groupByRaw('YEAR(created_at)')
    ->get());

// whereRaw & orWhereRaw
$users = await(DB::table('users')
    ->whereRaw('LENGTH(name) > ?', [10])
    ->orWhereRaw('score > ? AND status = ?', [1000, 'active'])
    ->get());

// havingRaw & orHavingRaw
$stats = await(DB::table('orders')
    ->selectRaw('user_id, SUM(total) as revenue')
    ->groupBy('user_id')
    ->havingRaw('SUM(total) - SUM(tax) > ?', [5000])
    ->orHavingRaw('MAX(total) > ?', [1000])
    ->get());

// orderByRaw — useful for functions or specific ordering mechanisms
$customSort = await(DB::table('tasks')
    ->orderByRaw('FIELD(status, ?, ?, ?)', ['todo', 'in_progress', 'done'])
    ->get());

// groupByRaw
$grouped = await(DB::table('users')
    ->selectRaw('DATE(created_at) as date, COUNT(*) as count')
    ->groupByRaw('DATE(created_at)')
    ->get());
```

---

### Conditional Query Building

Apply constraints only when a value is present, with no `if`/`else` branches needed:

```php
$search  = $request->input('search');
$status  = $request->input('status');
$isAdmin = $user->isAdmin();

$users = await(
    DB::table('users')
        ->when($search,    fn($q, $v) => $q->like('name', $v))
        ->when($status,    fn($q, $v) => $q->where('status', $v))
        ->unless($isAdmin, fn($q, $v) => $q->where('active', true))
        ->latest()
        ->get()
);

// With a default branch (runs when value is falsy)
$posts = await(
    DB::table('posts')
        ->when(
            $sortColumn,
            fn($q, $v) => $q->orderBy($v, $sortDirection ?? 'ASC'),
            fn($q, $v) => $q->latest()   // default if no sort given
        )
        ->get()
);

// Invokable class as condition
class IsSubscribed
{
    public function __invoke(mixed $builder): bool
    {
        return auth()->user()?->hasActiveSubscription() ?? false;
    }
}

$features = await(
    DB::table('features')
        ->when(new IsSubscribed(), fn($q) => $q->where('tier', 'premium'))
        ->get()
);
```

---

### JSON Conditions

JSON conditions compile to the correct driver dialect automatically (MySQL, PostgreSQL, SQLite):

```php
// Scalar field comparison — two-argument shorthand defaults to '='
$dark    = await(DB::table('users')->whereJson('options->theme', 'dark')->get());
$enabled = await(DB::table('users')->whereJson('options->notifications->email', true)->get());

// With explicit operator
$highLevel = await(DB::table('users')->whereJson('settings->level', '>', 5)->get());

// OR JSON condition
$admins = await(
    DB::table('users')
        ->whereJson('settings->role', 'admin')
        ->orWhereJson('settings->role', 'moderator')
        ->get()
);

// Array contains
$english = await(DB::table('users')->whereJsonContains('options->languages', 'en')->get());

// Array does NOT contain
$noBlock = await(DB::table('users')->whereJsonDoesntContain('options->blocked', 'PH')->get());

// Array length
$skilled = await(DB::table('users')->whereJsonLength('options->skills', '>', 3)->get());
$exact   = await(DB::table('posts')->whereJsonLength('metadata->tags', '=', 0)->get());

// Combined with regular conditions
$results = await(
    DB::table('users')
        ->where('active', true)
        ->whereJson('options->theme', 'dark')
        ->whereJsonContains('options->languages', 'en')
        ->whereJsonLength('options->skills', '>=', 2)
        ->get()
);
```

---

### Joins

```php
// Simple string condition
$orders = await(
    DB::table('orders')
        ->leftJoin('users', 'users.id = orders.user_id')
        ->select('orders.*', 'users.name as customer_name')
        ->get()
);

// Closure — multi-condition join with value filters
$orders = await(
    DB::table('orders')
        ->leftJoin('users', function ($join) {
            return $join
                ->on('users.id', '=', 'orders.user_id')
                ->where('users.active', true)
                ->whereNull('users.deleted_at');
        })
        ->get()
);

// OR ON condition
$contacts = await(
    DB::table('contacts')
        ->leftJoin('users', function ($join) {
            return $join
                ->on('users.email', '=', 'contacts.primary_email')
                ->orOn('users.email', '=', 'contacts.secondary_email');
        })
        ->get()
);

// Multiple joins
$report = await(
    DB::table('orders')
        ->leftJoin('users',    'users.id = orders.user_id')
        ->leftJoin('products', 'products.id = orders.product_id')
        ->leftJoin('payments', 'payments.order_id = orders.id')
        ->select('orders.id', 'users.name', 'products.title', 'payments.status')
        ->get()
);

// CROSS JOIN
$grid = await(DB::table('colors')->crossJoin('sizes')->get());
```

---

### Grouping and Ordering

```php
// GROUP BY — string, array, or comma-separated string
$stats = await(
    DB::table('orders')
        ->select('user_id')
        ->selectRaw('COUNT(*) as total, SUM(total) as revenue')
        ->groupBy('user_id')
        ->having('total', '>', 5)
        ->get()
);

$multi = await(DB::table('sales')->groupBy(['region', 'month'])->get());

// Standard ORDER BY
$sorted = await(DB::table('posts')->orderBy('created_at', 'DESC')->orderBy('title', 'ASC')->get());
$desc   = await(DB::table('posts')->orderByDesc('created_at')->get());
$asc    = await(DB::table('posts')->orderByAsc('title')->get());

// Semantic aliases (default to created_at)
$recent  = await(DB::table('posts')->latest()->get());           // ORDER BY created_at DESC
$oldest  = await(DB::table('posts')->oldest()->get());           // ORDER BY created_at ASC
$pubDesc = await(DB::table('posts')->latest('published_at')->get());

// Random order — adapts to driver (RAND() / RANDOM())
$sample = await(DB::table('products')->inRandomOrder()->limit(5)->get());

// reorder() — clear existing ORDER BY and optionally replace
$base    = DB::table('users')->orderByDesc('created_at');
$fresh   = $base->reorder();                     // clears all ORDER BY
$renewed = $base->reorder('name', 'ASC');        // clears then sets a new one
$raw     = $base->reorder()->orderByRaw('RAND()');

// LIMIT and OFFSET
$page    = await(DB::table('users')->limit(10)->offset(20)->get());
$combo   = await(DB::table('users')->limit(10, 20)->get()); // combined shorthand
$pagined = await(DB::table('users')->forPage(3, 15)->get()); // page 3, 15 per page
```

---

### Aggregates and Existence

```php
// COUNT — always returns int
$total    = await(DB::table('users')->count());
$active   = await(DB::table('users')->where('active', true)->count());
$distinct = await(DB::table('orders')->count('DISTINCT user_id'));

// SUM, AVG, MIN, MAX
$revenue  = await(DB::table('orders')->where('status', 'completed')->sum('total'));
$average  = await(DB::table('orders')->avg('total'));
$cheapest = await(DB::table('products')->min('price'));
$priciest = await(DB::table('products')->max('price'));

// Scoped aggregates
$avgPremium = await(
    DB::table('orders')
        ->where('tier', 'premium')
        ->whereBetween('created_at', ['2024-01-01', '2024-12-31'])
        ->avg('total')
);

// EXISTS — always returns bool
$emailTaken = await(DB::table('users')->where('email', $email)->exists());
if ($emailTaken) {
    throw new \RuntimeException('Email already registered.');
}

// DOESN'T EXIST — always returns bool
$available = await(DB::table('users')->where('username', $username)->doesntExist());

// Existence check with JOIN
$isEnrolled = await(
    DB::table('enrollments')
        ->where('user_id', $userId)
        ->where('course_id', $courseId)
        ->whereNull('cancelled_at')
        ->exists()
);
```

> **Return types for `sum()`, `avg()`, `min()`, and `max()`**
>
> These four methods return `mixed`, the raw value as the database driver delivers it. In practice this is almost always a `string`, even when the underlying column is `DECIMAL`, `FLOAT`, or `INT`. Database drivers return numeric aggregates as strings to preserve precision; PHP has no native arbitrary-precision numeric type, so casting to `int` or `float` silently drops precision on large or high-precision values.
>
> ```php
> $total = await(DB::table('orders')->sum('total'));
> // $total is '9999999.99' (string) — not 9999999.99 (float)
> ```
>
> If you need to do arithmetic on the result, use `bcmath` or `brick/math` rather than casting:
>
> ```php
> $revenue = await(DB::table('orders')->sum('total'));
> $tax     = await(DB::table('orders')->sum('tax'));
>
> // Safe — no precision loss
> $gross = bcadd((string) $revenue, (string) $tax, 2);
>
> // Unsafe — float precision loss on large or high-precision values
> $gross = (float) $revenue + (float) $tax;
> ```
>
> If you know the column is a small integer and precision is not a concern, a straight cast is fine:
>
> ```php
> $count = (int) await(DB::table('products')->sum('stock'));
> ```
>
> `count()` is the one exception: it always returns `int` because the query builder casts the result explicitly after fetching it.

---

### Pagination

#### Offset Pagination

Standard page-based pagination. The current page is read automatically from `$_GET['page']`:

```php
// Basic — 15 results per page
$paginator = await(DB::table('users')->where('active', true)->paginate(15));

// With a custom base path for link generation
$paginator = await(DB::table('users')->paginate(15, path: 'https://example.com/users'));
```

**Available properties:**

| Property | Type | Description |
| :--- | :--- | :--- |
| `$paginator->items` | `array` | The rows for the current page |
| `$paginator->total` | `int` | Total record count across all pages |
| `$paginator->perPage` | `int` | Records per page |
| `$paginator->currentPage` | `int` | Current page number |
| `$paginator->lastPage` | `int` | Last page number |
| `$paginator->from` | `int` | Starting record number on this page |
| `$paginator->to` | `int` | Ending record number on this page |
| `$paginator->hasMore` | `bool` | Whether a next page exists |
| `$paginator->hasPages` | `bool` | Whether there is more than one page |
| `$paginator->isFirstPage` | `bool` | Whether this is the first page |
| `$paginator->isLastPage` | `bool` | Whether this is the last page |
| `$paginator->path` | `string\|null` | Base URL for link generation |

**Available methods:**

```php
$paginator->url(3);              // URL for a specific page
$paginator->nextPageUrl();       // URL for the next page, or null
$paginator->previousPageUrl();   // URL for the previous page, or null
$paginator->getUrlRange(1, 5);   // [1 => '...', 2 => '...', ...]
$paginator->render();            // render HTML pagination links
$paginator->render('bootstrap'); // render with a specific template
$paginator->links();             // alias for render()
$paginator->toArray();           // convert to array (for APIs)
$paginator->toJson();            // convert to JSON string
```

#### Cursor Pagination

Cursor pagination is more efficient than offset pagination on large tables because it avoids `COUNT(*)` and deep `OFFSET` scans. It reads the cursor token from `$_GET['cursor']` automatically:

```php
// Single column cursor
$paginator = await(DB::table('posts')->cursorPaginate(20, 'id'));

// Multi-column cursor with directions
$paginator = await(
    DB::table('posts')
        ->where('published', true)
        ->cursorPaginate(20, ['created_at' => 'desc', 'id' => 'asc'])
);
```

**Available properties:**

| Property | Type | Description |
| :--- | :--- | :--- |
| `$paginator->items` | `array` | The rows for the current window |
| `$paginator->perPage` | `int` | Records per window |
| `$paginator->nextCursor` | `string\|null` | Encoded cursor token for the next page |
| `$paginator->hasMore` | `bool` | Whether more records exist |
| `$paginator->cursorColumns` | `array` | Columns and directions used for cursor sorting |
| `$paginator->path` | `string\|null` | Base URL for link generation |

**Available methods:**

```php
$paginator->nextPageUrl();               // URL with ?cursor=... appended, or null
$paginator->render();                    // render HTML next-page link
$paginator->render('cursor-bootstrap');
$paginator->links();                     // alias for render()
$paginator->toArray();
$paginator->toJson();
```

#### Pagination Templates

Six templates are built in. Configure the default in `hibla-database.php`:

| Template name | Style | Paginator type |
| :--- | :--- | :--- |
| `tailwind` | Tailwind CSS | Offset |
| `bootstrap` | Bootstrap 5 | Offset |
| `simple` | Plain HTML | Offset |
| `cursor-tailwind` | Tailwind CSS | Cursor |
| `cursor-bootstrap` | Bootstrap 5 | Cursor |
| `cursor-simple` | Plain HTML | Cursor |

To use a specific template for a single render:

```php
echo $paginator->render('bootstrap');
echo $paginator->render('cursor-simple');
```

**Publishing and customizing templates:**

Set `templates_path` in your `hibla-database.php` before publishing:

```php
'pagination' => [
    'templates_path'          => __DIR__ . '/resources/views/pagination',
    'default_template'        => 'tailwind',
    'default_cursor_template' => 'cursor-tailwind',
],
```

Then run the publish command (requires `hiblaphp/schema-manager` for CLI access):

```bash
./vendor/bin/hibla-db publish:templates

# Publish to a custom path
./vendor/bin/hibla-db publish:templates --path=resources/views/pagination

# Overwrite existing published templates
./vendor/bin/hibla-db publish:templates --force
```

The six `.php` template files will be copied to your configured directory. They are standard PHP templates with access to a `$paginator` variable. Published templates take priority over the built-in ones automatically.

#### Pagination for APIs

When building JSON APIs, use `toArray()` or `toJson()` instead of rendering HTML:

```php
// Offset paginator
$paginator = await(DB::table('users')->paginate(15));
return response()->json($paginator->toArray());
// {
//   "data": [...],
//   "meta": { "total": 100, "per_page": 15, "current_page": 1, "last_page": 7, "from": 1, "to": 15 },
//   "links": { "first": "...", "last": "...", "prev": null, "next": "..." }
// }

// Cursor paginator
$paginator = await(DB::table('posts')->cursorPaginate(20, 'id'));
return response()->json($paginator->toArray());
// {
//   "data": [...],
//   "meta": { "per_page": 20, "has_more": true },
//   "links": { "next": "...?cursor=eyJpZCI6MjB9" }
// }

// Exclude items (return only meta/links)
return response()->json($paginator->toArray(includeItems: false));

// Override path for link generation
return response()->json($paginator->toArray(basePath: 'https://api.example.com/v1/users'));
```

---

## Writing Data

### Inserting Data

```php
// Single row — returns number of affected rows (usually 1)
$affected = await(DB::table('users')->insert([
    'name'  => 'Alice',
    'email' => 'alice@example.com',
]));

// Insert Ignore — skip inserting if a unique constraint is violated (returns 0)
$affected = await(DB::table('users')->insertIgnore([
    'name'  => 'Alice Clone',
    'email' => 'alice@example.com',
]));

// Single row — inserts data and returns the new auto-increment ID.
// By default, this method retrieves and returns the value of the 'id' column.
$id = await(DB::table('users')->insertGetId([
    'name'  => 'Bob',
    'email' => 'bob@example.com',
]));

// If your database table uses a custom primary key name (particularly important 
// for PostgreSQL sequences), you can override the default 'id' by passing your 
// custom column name as the second parameter:
$adminId = await(DB::table('admins')->insertGetId($adminData, 'admin_id'));

// PostgreSQL requires the sequence/primary key name when it differs from 'id'
$id = await(DB::table('admins')->insertGetId($data, 'admin_id'));

// Batch insert — one SQL statement for all rows
$affected = await(DB::table('tags')->insertBatch([
    ['name' => 'php',        'slug' => 'php'],
    ['name' => 'javascript', 'slug' => 'javascript'],
    ['name' => 'python',     'slug' => 'python'],
]));

$affected = await(DB::table('tags')->insertIgnoreBatch([
    ['name' => 'php',        'slug' => 'php'],        // Skipped
    ['name' => 'python',     'slug' => 'python'],     // Inserted
]));

// Upsert — insert or update on conflict (single row)
// Compiles to ON DUPLICATE KEY UPDATE (MySQL) / ON CONFLICT DO UPDATE (PgSQL/SQLite)
$affected = await(DB::table('user_settings')->upsert(
    ['user_id' => 1, 'theme' => 'dark', 'language' => 'en'],
    uniqueColumns: 'user_id',
    updateColumns: ['theme', 'language']   // null = update all non-unique columns
));

// Batch upsert
$affected = await(DB::table('products')->upsertBatch(
    [
        ['sku' => 'ABC', 'stock' => 10, 'price' => 9.99],
        ['sku' => 'DEF', 'stock' => 5,  'price' => 14.99],
    ],
    uniqueColumns: ['sku'],
    updateColumns: ['stock', 'price']
));

// Upsert with composite unique key
$affected = await(DB::table('product_prices')->upsert(
    ['product_id' => 1, 'currency' => 'USD', 'amount' => 9.99],
    uniqueColumns: ['product_id', 'currency'],
    updateColumns: ['amount']
));
```

---

### Updating Data

```php
// Basic update — returns number of affected rows
$affected = await(
    DB::table('users')
        ->where('id', 1)
        ->update(['name' => 'Alice Updated', 'updated_at' => now()])
);

// Atomic increment — UPDATE users SET views = views + 1
$affected = await(DB::table('posts')->where('id', $postId)->increment('views'));

// Increment by a custom amount
$affected = await(DB::table('products')->where('id', $id)->increment('stock', 10));

// Increment with additional column updates in the same statement
$affected = await(
    DB::table('events')
        ->where('id', $eventId)
        ->increment('attendees', 1, ['updated_at' => now()])
);

// Atomic decrement
$affected = await(DB::table('products')->where('id', $id)->decrement('stock', 5));

// Decrement with guard condition (won't go below zero in a race)
$affected = await(
    DB::table('products')
        ->where('id', $id)
        ->where('stock', '>=', $quantity)
        ->decrement('stock', $quantity, ['reserved_at' => now()])
);
```

---

### Deleting Data

```php
// Delete matching rows — returns number of affected rows
$deleted = await(DB::table('users')->where('id', 42)->delete());

// Bulk delete
$deleted = await(DB::table('sessions')->where('expires_at', '<', now())->delete());

// Soft delete pattern (no native soft-delete support — handle at the application level)
await(DB::table('posts')->where('id', $postId)->update(['deleted_at' => now()]));

// Scope soft-deleted records out of queries
$active = await(DB::table('posts')->whereNull('deleted_at')->get());
```

---

## Advanced

### Processing Large Datasets (Streams vs. Chunks)

When retrieving massive amounts of data, a simple `get()` will exhaust your server's memory. Hibla offers two distinct architectural approaches to process large datasets: **Socket Streaming** and **Query Chunking**.

#### 1. Socket Streaming (`stream`, `each`, `chunkStream`)
Streaming keeps a **single query** open and reads rows incrementally off the database network socket. 
*   **Pros:** Incredible performance. It executes exactly one query and maintains a near-zero memory footprint (~2MB) regardless of table size.
*   **Cons:** It holds the active database connection open until iteration completes. If your processing logic is slow (e.g. sending 100k emails), it can monopolize the connection pool.

```php
// each() - Process row by row. Best for fast operations.
await(DB::table('orders')->each(function (object $order) {
    processOrder($order);
    // Return false to stop streaming early
}));

// chunkStream() - Retrieve rows in small arrays. Best for bulk inserts/dispatches.
await(DB::table('users')->chunkStream(500, function (array $chunk) {
    sendBatchEmail($chunk);
}));

// stream() - Raw iterator access for maximum control
$stream = await(DB::table('events')->stream(bufferSize: 200));
foreach ($stream as $row) {
    handle($row);
}
$stream->cancel(); // Important: Aborts the query on the database server!
```

#### 2. Query Chunking (`chunk`, `chunkById`)
Chunking executes **multiple, independent queries**. Between each query, the database connection is returned to the pool.
*   **Pros:** Safe for long-running, slow processing tasks because it doesn't hold a connection open. 
*   **Cons:** `chunk()` relies on SQL `OFFSET`, which gets progressively slower on massive tables.

```php
// chunk() - Uses LIMIT and OFFSET under the hood.
await(DB::table('products')->chunk(100, function (array $products) {
    reindexProducts($products);
    // Return false to stop early
}));

// chunkById() - AVOID OFFSET PENALTY. Uses `WHERE id > ? LIMIT`. 
// This is the fastest, safest way to chunk large tables.
await(DB::table('users')->chunkById(100, function (array $users) {
    processUsers($users);
}));

// chunkById() overriding the ID column and using an alias (critical for JOINs)
await(DB::table('users')
    ->join('orders', 'users.id', '=', 'orders.user_id')
    ->chunkById(100, function (array $users) {
        // ...
    }, column: 'users.id', alias: 'id'));
```

---

### Transactions

Transactions safely wrap multiple queries. In Hibla, transactions utilize **SQL Savepoints** automatically, allowing infinite nesting without breaking your database state.

```php
// Auto-managed — commits on success, rolls back automatically on any exception
$userId = await(
    DB::transaction(function ($tx) use ($data) {
        $id = await($tx->table('users')->insertGetId($data['user']));
        await($tx->table('profiles')->insert(['user_id' => $id, ...$data['profile']]));
        await($tx->table('settings')->insert(['user_id' => $id]));
        return $id;
    })
);

// Manual — you control commit and rollback
$tx = await(DB::beginTransaction());

try {
    await($tx->table('accounts')->where('id', $from)->decrement('balance', $amount));
    await($tx->table('accounts')->where('id', $to)->increment('balance', $amount));
    await($tx->commit());
} catch (\Throwable $e) {
    await($tx->rollback());
    throw $e;
}

// Nested transactions via savepoints
await(
    DB::transaction(function ($outer) use ($orderData) {
        await($outer->table('orders')->insert($orderData));

        // Inner failure rolls back only the inner work, not the outer transaction
        await($outer->transaction(function ($inner) use ($orderData) {
            await($inner->table('inventory')->decrement('stock'));
            await($inner->table('audit_log')->insert($auditData));
        }));
    })
);

// Savepoints manually
$tx = await(DB::beginTransaction());
await($tx->savepoint('before_email'));
try {
    await($tx->table('email_queue')->insert($emailData));
} catch (\Throwable $e) {
    await($tx->rollbackTo('before_email'));
}
await($tx->commit());

// Commit and rollback hooks
$tx = await(DB::beginTransaction());
$tx->onCommit(fn()   => Cache::flush('orders'));
$tx->onRollback(fn() => logger()->warning('Order transaction rolled back'));

// Bind an existing query builder to an active transaction
// Returns a cloned builder that executes on $tx — does not mutate the original
$txQuery = DB::table('users')->transacting($tx);
await($txQuery->where('id', 1)->update(['status' => 'processing']));
```

---

### Common Table Expressions

```php
// Simple CTE — pre-filter before the main query
$active = await(
    DB::table('active_users')
        ->with('active_users', fn($q) => $q->from('users')->where('status', 'active'))
        ->select('id', 'name', 'email')
        ->get()
);
// WITH active_users AS (SELECT * FROM users WHERE status = ?)
// SELECT id, name, email FROM active_users

// Chained CTEs — each builds on the previous
$report = await(
    DB::table('final')
        ->with('raw',    fn($q) => $q->from('orders')->where('status', 'completed'))
        ->with('totals', fn($q) => $q->from('raw')->select('user_id')->selectRaw('SUM(total) as spent')->groupBy('user_id'))
        ->with('final',  fn($q) => $q->from('totals')->where('spent', '>', 500))
        ->select('user_id', 'spent')
        ->orderByDesc('spent')
        ->get()
);

// Recursive CTE — withRecursive() is a readable shorthand for with(..., recursive: true).
// Both compile identically; use whichever reads more clearly at the call site.
$tree = await(
    DB::table('category_tree')
        ->withRecursive('category_tree', function ($q) {
            return $q
                ->from('categories')
                ->select('id', 'name', 'parent_id')
                ->whereNull('parent_id')          // anchor: root nodes
                ->unionAll(fn($u) => $u
                    ->from('categories')
                    ->select('categories.id', 'categories.name', 'categories.parent_id')
                    ->join('category_tree', 'category_tree.id = categories.parent_id'));
        })
        ->get()
);
```

---

### Pessimistic Locking

Lock clauses are only meaningful inside a database transaction.

```php
// Exclusive lock — no other transaction can read or lock the rows
$pdo->beginTransaction();
$order = await(
    DB::table('orders')
        ->where('id', $orderId)
        ->where('status', 'pending')
        ->lockForUpdate()
        ->first()
);
// ... process order
$pdo->commit();

// Skip locked rows — essential for job queue workers so they don't block each other
$job = await(
    DB::table('jobs')
        ->where('status', 'available')
        ->orderByDesc('priority')
        ->oldest()
        ->limit(1)
        ->lockForUpdate()
        ->skipLocked()
        ->first()
);

// Fail immediately if locked
$item = await(
    DB::table('inventory')
        ->where('id', $itemId)
        ->lockForUpdate()
        ->noWait()
        ->first()
);

// Shared lock — others can read but not modify
$balance = await(
    DB::table('accounts')
        ->where('id', $accountId)
        ->lockForShare()
        ->first()
);
```

---

### Debugging

```php
// Get the compiled SQL string (with ? placeholders)
$sql = DB::table('users')->where('active', true)->latest()->toSql();
// SELECT * FROM users WHERE active = ? ORDER BY created_at DESC

// Get the positional bindings in order
$bindings = DB::table('users')->where('active', true)->getBindings();
// [true]

// Get interpolated SQL — for human reading only, NEVER execute this
$raw = DB::table('users')->where('active', true)->toRawSql();
// SELECT * FROM users WHERE active = '1' ORDER BY created_at DESC

// Dump query at any point in the chain and continue building
$users = await(
    DB::table('users')
        ->where('active', true)
        ->debug()          // prints the query here
        ->latest()
        ->limit(10)
        ->dump()          // prints again after the additions
        ->get()
);

// Dump and die — stops execution
DB::table('users')->where('active', true)->halt();
```

---

## Schema Manager

The schema manager is an **optional** package. Install it only when you need migrations, seeders, or the CLI tooling. The query builder works independently without it.

### Installing the Schema Manager

```bash
composer require hiblaphp/schema-manager
```

### Initializing

Run once after installation:

```bash
./vendor/bin/hibla-db init
```

This creates three config files in your project root:

```
hibla-database.php    ← Database connections (shared with the query builder)
hibla-migrations.php  ← Migration settings
hibla-seeders.php     ← Seeder settings
```

Check the current config resolution status at any time:

```bash
./vendor/bin/hibla-db status
```

For custom config locations, see [Custom Config Locations](#custom-config-locations).

### Custom Entry Point

By default, all CLI commands are run through `./vendor/bin/hibla-db`. If you prefer a shorter command or a project-specific name, you can create your own entry point — a plain file with no extension placed in your project root.

For example, create a file named `database` (no `.php` extension):

```php
#!/usr/bin/env php
<?php

require_once __DIR__ . '/vendor/autoload.php';

use Hibla\SchemaManager\Console\DbSeedCommand;
use Symfony\Component\Console\Application;
use Hibla\SchemaManager\Console\InitCommand;
use Hibla\SchemaManager\Console\PublishTemplatesCommand;
use Hibla\SchemaManager\Console\MakeMigrationCommand;
use Hibla\SchemaManager\Console\MakeSeederCommand;
use Hibla\SchemaManager\Console\MigrateCommand;
use Hibla\SchemaManager\Console\MigrateRollbackCommand;
use Hibla\SchemaManager\Console\MigrateResetCommand;
use Hibla\SchemaManager\Console\MigrateRefreshCommand;
use Hibla\SchemaManager\Console\MigrateFreshCommand;
use Hibla\SchemaManager\Console\MigrateStatusCommand;
use Hibla\SchemaManager\Console\SchemaDumpCommand;
use Hibla\SchemaManager\Console\StatusCommand;

$application = new Application('Hibla Database', '1.0.0');

$application->addCommand(new InitCommand());
$application->addCommand(new PublishTemplatesCommand());
$application->addCommand(new MakeMigrationCommand());
$application->addCommand(new MakeSeederCommand());
$application->addCommand(new DbSeedCommand());
$application->addCommand(new MigrateCommand());
$application->addCommand(new MigrateRollbackCommand());
$application->addCommand(new MigrateResetCommand());
$application->addCommand(new MigrateRefreshCommand());
$application->addCommand(new MigrateFreshCommand());
$application->addCommand(new MigrateStatusCommand());
$application->addCommand(new SchemaDumpCommand());
$application->addCommand(new StatusCommand());

$application->run();
```

Despite having no `.php` extension, this file contains PHP code. The `#!/usr/bin/env php` line at the top tells your shell to run it with PHP. The file can be named anything you like — `database`, `db`, `artisan`, or whatever fits your project.

Then invoke it with `php` followed by the filename:

```bash
php database migrate
php database make:migration create_users_table
php database db:seed --class=UserSeeder
```

On Linux and macOS, you can make the file executable to drop the `php` prefix entirely:

```bash
chmod +x database
./database migrate
```

> The entry point file registers exactly the same commands as the built-in binary. Any flag, option, or `--connection` argument documented in the [CLI Reference](#cli-reference) works identically.

### CLI Reference

Full command reference — jump to the sections below for usage examples and options.

| Command | Description |
| :--- | :--- |
| `init` | Scaffold config files |
| `status` | Show config resolution status |
| `make:migration <name>` | Generate a migration file |
| `make:seeder <name>` | Generate a seeder file |
| `migrate` | Run pending migrations |
| `migrate:rollback` | Roll back the last batch |
| `migrate:reset` | Roll back all migrations |
| `migrate:refresh` | Reset and re-run all migrations |
| `migrate:fresh` | Drop all tables and re-run from scratch |
| `migrate:status` | Show run status of each migration |
| `db:seed` | Run database seeders |
| `schema:dump` | Dump the current schema to a SQL file |
| `publish:templates` | Publish pagination templates for customization |

All commands that touch the database accept `--connection=<name>` to target a specific named connection from `hibla-database.php`.

---

### Migrations

#### Creating Migration Files

The migration name is used to auto-detect the operation (`create_*_table`, `add_*_to_*`, `drop_*`):

```bash
# Auto-detects CREATE TABLE
./vendor/bin/hibla-db make:migration create_users_table

# Auto-detects ALTER TABLE
./vendor/bin/hibla-db make:migration add_avatar_to_users_table

# Explicit table or alter
./vendor/bin/hibla-db make:migration create_posts_table --table=posts
./vendor/bin/hibla-db make:migration add_bio_to_users --alter=users

# Organize into subdirectories
./vendor/bin/hibla-db make:migration create_orders_table --path=ecommerce
# Creates: database/migrations/ecommerce/2024_01_01_000000_create_orders_table.php

# Target a specific named connection
./vendor/bin/hibla-db make:migration create_events_table --connection=pgsql
```

Naming conventions are configured in `hibla-migrations.php`:

```php
'naming_convention' => 'timestamp',   // 2024_01_01_000000_create_users_table.php
// or
'naming_convention' => 'sequential',  // 0001_create_users_table.php
```

#### Writing Migrations

Migrations return an anonymous class extending `Migration`. All schema methods return promises and must be awaited:

```php
<?php

use Hibla\Migrations\Schema\Blueprint;
use Hibla\Migrations\Schema\Migration;
use function Hibla\await;

return new class extends Migration
{
    public function up(): void
    {
        await($this->create('users', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('email')->unique();
            $table->string('password');
            $table->timestamp('email_verified_at')->nullable();
            $table->boolean('active')->default(true);
            $table->timestamps();
            
            // Table Options (MySQL/MariaDB)
            $table->engine('InnoDB');
            $table->charset('utf8mb4');
            $table->collation('utf8mb4_unicode_ci');
        }));
    }

    public function down(): void
    {
        await($this->dropIfExists('users'));
    }
};
```

Altering a table and modifying columns:

```php
return new class extends Migration
{
    public function up(): void
    {
        await($this->table('users', function (Blueprint $table) {
            // Add new columns
            $table->string('avatar_url')->nullable();
            $table->string('phone', 20)->nullable()->after('email');
            
            // Modify existing columns
            $table->modifyString('name', 100)->nullable();
            $table->modifyInteger('views', unsigned: true)->default(0);
            // (Available: modifyString, modifyInteger, modifyBigInteger, modifySmallInteger, 
            // modifyTinyInteger, modifyText, modifyDecimal, modifyBoolean)
            
            $table->index('phone');
            $table->foreign('role_id')->references('id')->on('roles')->cascadeOnDelete();
        }));
    }

    public function down(): void
    {
        await($this->table('users', function (Blueprint $table) {
            $table->dropForeign(['users_role_id_foreign']);
            $table->dropIndex(['users_phone_index']);
            $table->dropColumn(['avatar_url', 'phone']);
            
            // Revert modified columns
            $table->modifyString('name', 255)->nullable(false);
        }));
    }
};
```

Pinning a migration to a specific connection:

```php
return new class extends Migration
{
    protected ?string $connection = 'pgsql';

    public function up(): void
    {
        await($this->create('analytics_events', function (Blueprint $table) {
            $table->id();
            $table->string('event_name');
            $table->json('payload');
            $table->timestamps();
        }));
    }

    public function down(): void
    {
        await($this->dropIfExists('analytics_events'));
    }
};
```

Disabling transactions (e.g. for `CREATE INDEX CONCURRENTLY` on PostgreSQL):

```php
return new class extends Migration
{
    protected bool $withinTransaction = false;

    public function up(): void
    {
        await($this->rawExecute(
            'CREATE INDEX CONCURRENTLY idx_users_email ON users (email)'
        ));
    }

    public function down(): void
    {
        await($this->rawExecute(
            'DROP INDEX CONCURRENTLY IF EXISTS idx_users_email'
        ));
    }
};
```

Ad-hoc queries inside a migration. `$this->db()` returns a full `BaseQueryBuilderInterface` instance bound to the migration's active transaction:

```php
public function up(): void
{
    await($this->create('roles', function (Blueprint $table) {
        $table->id();
        $table->string('name')->unique();
    }));

    // Seed default data as part of the migration
    await($this->db('roles')->insertBatch([
        ['name' => 'admin'],
        ['name' => 'editor'],
        ['name' => 'viewer'],
    ]));
}
```

#### Blueprint Column Types

```php
// Auto-increment primary keys
$table->id();                            // BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY
$table->bigIncrements('id');
$table->increments('id');                // INT AUTO_INCREMENT
$table->mediumIncrements('id');
$table->smallIncrements('id');
$table->tinyIncrements('id');

// Integers
$table->bigInteger('score');
$table->integer('count');
$table->mediumInteger('views');
$table->smallInteger('priority');
$table->tinyInteger('flag');
$table->unsignedBigInteger('user_id');   // common for FK columns
$table->unsignedInteger('order_number');

// Strings
$table->string('name');                  // VARCHAR(255)
$table->string('code', 10);             // VARCHAR(10)
$table->text('body');
$table->mediumText('content');
$table->longText('article');

// Numerics
$table->decimal('price', 10, 2);        // DECIMAL(10,2)
$table->float('ratio', 8, 2);
$table->double('score', 8, 4);
$table->unsignedDecimal('amount', 10, 2);

// Date and time
$table->date('birth_date');
$table->dateTime('scheduled_at');
$table->timestamp('verified_at')->nullable();
$table->timestamp('login_at')->timezone('America/New_York'); // Timezone aware
$table->timestamps();                    // created_at + updated_at, nullable
$table->softDeletes();                   // deleted_at nullable timestamp
$table->softDeletes('removed_at');       // custom column name

// Other
$table->boolean('active')->default(false);
$table->json('options');
$table->enum('status', ['draft', 'published', 'archived']);

// PostgreSQL only
$table->vector('embedding', 1536);       // pgvector column

// Geometry
$table->point('location');
$table->polygon('boundary');
$table->geometry('shape');

// Column modifiers (chainable)
$table->string('bio')->nullable();
$table->integer('order')->default(0);
$table->string('country')->after('city');
$table->timestamp('published_at')->useCurrent();
$table->string('slug')->unique();
$table->text('search_body')->fullText();
$table->string('notes')->comment('Internal use only');

// Indexes
$table->index('email');
$table->index(['last_name', 'first_name']);       // composite
$table->unique('email');
$table->unique(['email', 'tenant_id']);
$table->fullText('body');
$table->spatialIndex('location');

// PostgreSQL Vector Index (Supports COSINE, L2, or INNER_PRODUCT)
$table->vectorIndex('embedding', 'vector_idx', 'COSINE');

// Raw/Custom index expressions
$table->rawIndex('CONSTRAINT idx_score UNIQUE (score)', 'idx_score'); 

// Foreign keys
$table->foreignId('user_id')->constrained();              // references users.id
$table->foreignId('user_id')->constrained()->cascadeOnDelete();
$table->foreignId('category_id')->constrained('categories')->nullOnDelete();

$table->foreign('user_id')->references('id')->on('users')->cascadeOnDelete();
$table->foreign('tenant_id')->references('id')->on('tenants')->restrictOnDelete();

// Drop operations (for alter migrations)
$table->dropColumn('avatar_url');
$table->dropColumn(['avatar_url', 'phone']);
$table->renameColumn('bio', 'biography');
$table->dropIndex(['users_phone_index']);
$table->dropUnique(['users_email_unique']);
$table->dropForeign(['users_role_id_foreign']);
$table->dropPrimary();
```

#### Running Migrations

```bash
# Run all pending migrations
./vendor/bin/hibla-db migrate

# Run migrations across ALL configured connection pools automatically
./vendor/bin/hibla-db migrate --all

# Run only the next N migrations
./vendor/bin/hibla-db migrate --step=3

# Target a specific connection
./vendor/bin/hibla-db migrate --connection=pgsql

# Run only migrations in a subdirectory
./vendor/bin/hibla-db migrate --path=ecommerce

# Create the database if it doesn't exist yet
./vendor/bin/hibla-db migrate --force
```

#### Rolling Back

```bash
# Roll back the last batch
./vendor/bin/hibla-db migrate:rollback

# Roll back a specific number of migrations
./vendor/bin/hibla-db migrate:rollback --step=5

# Roll back all migrations (prompts for confirmation)
./vendor/bin/hibla-db migrate:reset

# Roll back all then re-run all
./vendor/bin/hibla-db migrate:refresh

# Drop all tables and re-run from scratch
./vendor/bin/hibla-db migrate:fresh

# Skip confirmation
./vendor/bin/hibla-db migrate:fresh --force
./vendor/bin/hibla-db migrate:reset --force
```

> All destructive commands (`migrate:fresh`, `migrate:reset`, `migrate:refresh`, `migrate:rollback`) are blocked entirely when [Safe Mode](#safe-mode) is enabled.

#### Migration Status

```bash
# Show all migrations with run status
./vendor/bin/hibla-db migrate:status

# Filter to pending only
./vendor/bin/hibla-db migrate:status --pending

# Filter to completed only
./vendor/bin/hibla-db migrate:status --ran

# Show status for all configured connections
./vendor/bin/hibla-db migrate:status --all
```

Output shows each migration path, its status (`✓ Ran` / `Pending` / `✓ Ran (Pruned)`), the batch number it ran in, and the connection it targets.

#### Schema Dump

For mature projects with many migrations, you can squash all ran migrations into a single SQL file. Fresh environments load from this file directly, skipping potentially hundreds of migration files:

```bash
# Dump current schema to database/schema/mysql-schema.sql
./vendor/bin/hibla-db schema:dump

# Dump and delete all migration files that have already run
./vendor/bin/hibla-db schema:dump --prune

# Dump a specific connection's schema
./vendor/bin/hibla-db schema:dump --connection=pgsql
# → database/schema/pgsql-schema.sql
```

On the next `migrate` run against a fresh database, the migration system detects the schema file and loads it before running any new migrations. This requires the `mysql`, `psql`, or `sqlite3` CLI tools to be available in the system PATH; if they are not found, a pure PHP fallback is used automatically.

---

### Seeders

#### Creating Seeders

```bash
./vendor/bin/hibla-db make:seeder UserSeeder

# In a subdirectory
./vendor/bin/hibla-db make:seeder testing/UserSeeder

# For a specific connection
./vendor/bin/hibla-db make:seeder AnalyticsSeeder --connection=pgsql
```

#### Writing Seeders

Seeders return an anonymous class extending `Seeder`. Use `await()` for all database calls:

```php
<?php

use Hibla\Migrations\Schema\Seeder;
use function Hibla\await;

return new class extends Seeder
{
    public function run(): void
    {
        await($this->db('users')->insertBatch([
            ['name' => 'Alice', 'email' => 'alice@example.com', 'active' => true],
            ['name' => 'Bob',   'email' => 'bob@example.com',   'active' => true],
        ]));
    }
};
```

Orchestrating from a root seeder:

You can call other seeders from a root seeder (like `DatabaseSeeder.php`) using the `call()` method. This method is highly polymorphic and accepts three types of arguments:

1. **By File Name:** Passes a relative path (great for anonymous classes without autoloading).
2. **By Class String:** Passes the fully qualified class name of an autoloaded seeder.
3. **By Object Instance:** Passes an instantiated seeder object, which is perfect for injecting custom dependencies.

```php
<?php

use Hibla\Migrations\Schema\Seeder;
use function Hibla\await;

return new class extends Seeder
{
    public function run(): void
    {
        // 1. By relative file path
        await($this->call('UserSeeder'));
        await($this->call('testing/DummyDataSeeder'));

        // 2. By class-string
        await($this->call(\Database\Seeders\OrderSeeder::class));

        // 3. By object instance (Injecting dependencies!)
        $apiClient = new CustomApiClient();
        await($this->call(new SettingsSeeder($apiClient)));
    }
};
```

The called seeder inherits the parent connection unless it declares its own. You can use any query builder feature inside a seeder:

```php
public function run(): void
{
    // Idempotent seeding — skip if already seeded
    $exists = await($this->db('roles')->where('name', 'admin')->exists());
    if (! $exists) {
        await($this->db('roles')->insertBatch([
            ['name' => 'admin'],
            ['name' => 'editor'],
        ]));
    }
}
```

#### Running Seeders

```bash
# Run DatabaseSeeder if it exists, otherwise runs all discovered seeders alphabetically
./vendor/bin/hibla-db db:seed

# Run a specific seeder by name
./vendor/bin/hibla-db db:seed --class=UserSeeder

# Target a named connection
./vendor/bin/hibla-db db:seed --connection=pgsql

# Force in safe-mode environments
./vendor/bin/hibla-db db:seed --force
```

---

### Multiple Connections

Every CLI command accepts `--connection` to target a named connection. Migration files can also declare their own connection via the `$connection` property (see [Writing Migrations](#writing-migrations)).

To organize migration and seeder files by connection, configure `connection_paths` in the respective config files:

```php
// hibla-migrations.php
'connection_paths' => [
    'mysql' => 'mysql',
    'pgsql' => 'postgres',
],
// make:migration --connection=pgsql → database/migrations/postgres/...

// hibla-seeders.php
'connection_paths' => [
    'mysql' => 'mysql',
    'pgsql' => 'postgres',
],
```

You can also override the migrations table name and naming convention per connection:

```php
// hibla-migrations.php
'connections' => [
    'pgsql' => [
        'migrations_table'  => 'schema_versions',
        'naming_convention' => 'sequential',
        'timezone'          => 'America/New_York',
    ],
],
```

---

### Safe Mode

Safe mode prevents destructive commands (`migrate:fresh`, `migrate:reset`, `migrate:refresh`, and `migrate:rollback`) from running, and also blocks `db:seed` unless forced. It is configured in `hibla-migrations.php` and is strongly recommended for production environments:

```php
// hibla-migrations.php
'safe_mode' => env('DB_SAFE_MODE', false),
```

```dotenv
# .env (production)
DB_SAFE_MODE=true
```

With safe mode enabled:

```bash
# HARD BLOCKED — prints an error and exits immediately.
./vendor/bin/hibla-db migrate:fresh
./vendor/bin/hibla-db migrate:reset
./vendor/bin/hibla-db migrate:refresh
./vendor/bin/hibla-db migrate:rollback

# Blocked by default, but CAN be forced
./vendor/bin/hibla-db db:seed
./vendor/bin/hibla-db db:seed --force

# ALLOWED — Standard forward migrations are not affected
./vendor/bin/hibla-db migrate
```

> **NOTE: `--force` does NOT override Safe Mode for schema commands.** 
> If `DB_SAFE_MODE=true` is set, passing the `--force` flag to `migrate:fresh`, `reset`, `refresh`, or `rollback` **will not work**. The command will still immediately abort. This is a deliberate design choice to prevent automated CI/CD pipelines or sleepy developers from accidentally wiping a production database. The only command where `--force` bypasses Safe Mode is `db:seed`.

---

Here is the **Inspiration** section, written to be professional, appreciative, and perfectly positioned to go right before your **License** section:

---

## Inspiration

The design, API, and developer experience of the Hibla Database Ecosystem are heavily inspired by two outstanding open-source projects:

*   **[Laravel's Database Tooling](https://laravel.com):** The fluent query builder syntax, migration blueprints, and CLI output structures are modeled after Laravel's world-class developer experience.
*   **[Knex.js](https://knexjs.org):** The asynchronous, promise-driven execution model, native connection pooling paradigms, and standalone CLI flow are inspired by the standard-bearer of JavaScript database tools.

Hibla aims to bring the structural elegance of Laravel and the non-blocking, event-driven power of Knex.js together natively into the modern PHP Fiber runtime.

---

## License

MIT