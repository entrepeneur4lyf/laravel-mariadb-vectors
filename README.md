# Vector Search in Laravel with MariaDB — No Postgres Required

Laravel's built-in vector helpers (`whereVectorSimilarTo`, `orderByVectorDistance`, `selectVectorDistance`, `whereVectorDistanceLessThan`) are locked to PostgreSQL and pgvector. If you're running MariaDB 11.7+, which ships native VECTOR columns and MHNSW indexing out of the box — no extensions to install — you're out of luck. The framework will throw an exception the moment you try.

This tutorial shows how to add full MariaDB vector support to Laravel with four files and zero compromises. The same API surface, the same method signatures, the same Eloquent workflow. Just MariaDB SQL under the hood instead of pgvector.

## What You Get

- **Full query builder API**: `whereVectorSimilarTo()`, `orderByVectorDistance()`, `selectVectorDistance()`, `whereVectorDistanceLessThan()` — all working against MariaDB's `VEC_DISTANCE_COSINE()`.
- **Eloquent cast**: Read and write VECTOR columns transparently. Handles both MariaDB's raw binary format and JSON from `VEC_ToText()`.
- **Drop-in activation**: One service provider, registered once. Every MariaDB connection in your app gets vector support automatically.
- **Forward-compatible**: Method signatures match Laravel's upstream API exactly. When Laravel eventually adds native MariaDB support, migration is trivial.

## Requirements

- **MariaDB 11.7+** (VECTOR type and VEC_DISTANCE functions are built-in)
- **Laravel 11+** (tested through 13.5)
- **PHP 8.2+**

## Architecture

The approach is three layers:

1. **`MariaDbVectorConnection`** — Extends Laravel's `MariaDbConnection`, overrides `query()` to return our custom builder.
2. **`MariaDbVectorBuilder`** — Extends Laravel's `Query\Builder`, overrides the four vector methods with MariaDB-native SQL.
3. **`AsVector`** — Eloquent cast that handles the binary ↔ PHP array conversion for VECTOR columns.

A service provider wires the connection override at boot time via `Connection::resolverFor()`.

## File 1: MariaDbVectorConnection

**`app/Database/MariaDbVectorConnection.php`**

This is the entry point. It's a one-method override that swaps Laravel's default query builder for ours:

```php
<?php

declare(strict_types=1);

namespace App\Database;

use Illuminate\Database\MariaDbConnection;

/**
 * MariaDB connection that hands out a MariaDbVectorBuilder for every
 * new query. Registered via Connection::resolverFor('mariadb', ...)
 * in VectorDatabaseServiceProvider so every MariaDB-driver connection
 * in the app picks it up transparently.
 */
class MariaDbVectorConnection extends MariaDbConnection
{
    public function query(): MariaDbVectorBuilder
    {
        return new MariaDbVectorBuilder(
            $this,
            $this->getQueryGrammar(),
            $this->getPostProcessor()
        );
    }
}
```

## File 2: MariaDbVectorBuilder

**`app/Database/MariaDbVectorBuilder.php`**

This is where the SQL translation happens. Each method mirrors Laravel's upstream API but emits `VEC_DISTANCE_COSINE(column, VEC_FromText(?))` instead of pgvector's `<=>` operator. All vector values go through parameterized bindings — no SQL injection surface.

```php
<?php

declare(strict_types=1);

namespace App\Database;

use Illuminate\Contracts\Support\Arrayable;
use Illuminate\Database\Query\Builder;
use Illuminate\Support\Collection;
use InvalidArgumentException;

/**
 * MariaDB-flavoured query builder that replaces Laravel's Postgres-only
 * vector helpers with MariaDB-native SQL.
 *
 * Laravel's Builder ships whereVectorSimilarTo, orderByVectorDistance,
 * selectVectorDistance, and whereVectorDistanceLessThan, but they all
 * call ensureConnectionSupportsVectors() which hard-rejects anything
 * that is not a PostgresConnection and emits pgvector's <=> operator.
 *
 * This class overrides those methods with MariaDB's
 * VEC_DISTANCE_COSINE(col, VEC_FromText(?)) shape. Method signatures
 * stay identical to the upstream API so consumers — and future
 * framework updates that unlock MariaDB support — remain compatible.
 *
 * The optimizer will use the VECTOR INDEX when the expression matches
 * the index distance type (DISTANCE=cosine). Tune mhnsw_ef_search at
 * the session level if a specific query needs higher recall.
 */
class MariaDbVectorBuilder extends Builder
{
    /**
     * Filter and order by vector similarity.
     *
     * @param  string  $column
     * @param  array<int, float>|Arrayable<int, float>|Collection<int, float>|string  $vector
     * @param  float  $minSimilarity  Minimum cosine similarity (0–1). Default 0.6.
     * @param  bool  $order  Whether to also order by distance ascending.
     */
    public function whereVectorSimilarTo($column, $vector, $minSimilarity = 0.6, $order = true)
    {
        $this->whereVectorDistanceLessThan($column, $vector, 1 - $minSimilarity);

        if ($order) {
            $this->orderByVectorDistance($column, $vector);
        }

        return $this;
    }

    /**
     * Filter rows where vector distance is within a threshold.
     *
     * @param  string  $column
     * @param  array<int, float>|Arrayable<int, float>|Collection<int, float>|string  $vector
     * @param  float  $maxDistance  Maximum cosine distance.
     * @param  string  $boolean  'and' or 'or'.
     */
    public function whereVectorDistanceLessThan($column, $vector, $maxDistance, $boolean = 'and')
    {
        $wrapped = $this->getGrammar()->wrap($column);
        $json = $this->vectorToJson($vector);

        return $this->whereRaw(
            "VEC_DISTANCE_COSINE({$wrapped}, VEC_FromText(?)) <= ?",
            [$json, $maxDistance],
            $boolean
        );
    }

    /**
     * Or-variant of whereVectorDistanceLessThan.
     */
    public function orWhereVectorDistanceLessThan($column, $vector, $maxDistance)
    {
        return $this->whereVectorDistanceLessThan($column, $vector, $maxDistance, 'or');
    }

    /**
     * Order results by vector distance.
     *
     * @param  string  $column
     * @param  array<int, float>|Arrayable<int, float>|Collection<int, float>|string  $vector
     * @param  string  $direction  'asc' (nearest first) or 'desc'.
     */
    public function orderByVectorDistance($column, $vector, $direction = 'asc')
    {
        $wrapped = $this->getGrammar()->wrap($column);
        $json = $this->vectorToJson($vector);
        $direction = strtolower($direction) === 'desc' ? 'desc' : 'asc';

        return $this->orderByRaw(
            "VEC_DISTANCE_COSINE({$wrapped}, VEC_FromText(?)) {$direction}",
            [$json]
        );
    }

    /**
     * Add the vector distance as a computed column in the SELECT.
     *
     * @param  string  $column
     * @param  array<int, float>|Arrayable<int, float>|Collection<int, float>|string  $vector
     * @param  string  $as  Alias for the distance column.
     */
    public function selectVectorDistance($column, $vector, $as = 'distance')
    {
        $wrapped = $this->getGrammar()->wrap($column);
        $asWrapped = $this->getGrammar()->wrap($as);
        $json = $this->vectorToJson($vector);

        return $this->selectRaw(
            "VEC_DISTANCE_COSINE({$wrapped}, VEC_FromText(?)) AS {$asWrapped}",
            [$json]
        );
    }

    /**
     * Override the Postgres-only guard — MariaDB 11.7+ supports vectors.
     */
    protected function ensureConnectionSupportsVectors()
    {
        // no-op: MariaDB handles vectors natively
    }

    /**
     * Normalize any supported vector input to a JSON string for binding.
     *
     * @param  array<int, float>|Arrayable<int, float>|Collection<int, float>|string  $vector
     */
    private function vectorToJson(mixed $vector): string
    {
        if (is_string($vector)) {
            return $vector;
        }

        if ($vector instanceof Collection) {
            $vector = $vector->all();
        } elseif ($vector instanceof Arrayable) {
            $vector = $vector->toArray();
        }

        if (! is_array($vector)) {
            throw new InvalidArgumentException(
                'Vector must be an array of floats, Arrayable, Collection, or JSON string; got '
                .get_debug_type($vector)
            );
        }

        return json_encode(
            array_values(array_map('floatval', $vector)),
            JSON_THROW_ON_ERROR
        );
    }
}
```

## File 3: AsVector Eloquent Cast

**`app/Database/Casts/AsVector.php`**

This cast handles the read/write conversion between PHP arrays and MariaDB's VECTOR column type. The read path handles two formats: the raw little-endian float32 binary that MariaDB returns by default, and JSON strings from `VEC_ToText()`. The write path emits a raw SQL expression so `VEC_FromText()` is called server-side.

```php
<?php

declare(strict_types=1);

namespace App\Database\Casts;

use Illuminate\Contracts\Database\Eloquent\CastsAttributes;
use Illuminate\Contracts\Support\Arrayable;
use Illuminate\Database\Query\Expression;
use InvalidArgumentException;

/**
 * Cast between a PHP array<int, float> and a MariaDB VECTOR column.
 *
 * - Read path: MariaDB returns the column as little-endian IEEE 754
 *   float32 bytes; we unpack them. If VEC_ToText() has been applied
 *   upstream we receive a JSON string and json_decode.
 * - Write path: we emit VEC_FromText('[...]') as a raw Expression so
 *   it is injected verbatim into the INSERT/UPDATE statement. JSON
 *   encoding of a float array produces only ASCII digits, comma,
 *   period, bracket, minus, and 'e' for scientific notation — none
 *   of which need SQL escaping — so single-quote wrapping is safe.
 */
final class AsVector implements CastsAttributes
{
    /**
     * Decode a VECTOR column into a PHP float array.
     *
     * @return array<int, float>|null
     */
    public function get($model, string $key, $value, array $attributes): ?array
    {
        if ($value === null) {
            return null;
        }

        if (is_array($value)) {
            return array_values(array_map('floatval', $value));
        }

        if (! is_string($value)) {
            throw new InvalidArgumentException(
                "Cannot decode vector column {$key}: unexpected value type "
                .get_debug_type($value)
            );
        }

        // VEC_ToText() or JSON fed directly from the app.
        if ($this->looksLikeJson($value)) {
            $decoded = json_decode($value, true);
            if (! is_array($decoded)) {
                throw new InvalidArgumentException(
                    "Cannot decode vector column {$key}: invalid JSON payload"
                );
            }

            return array_values(array_map('floatval', $decoded));
        }

        // Raw binary VECTOR — four bytes per float, little-endian.
        $floats = unpack('g*', $value);
        if ($floats === false) {
            throw new InvalidArgumentException(
                "Cannot decode vector column {$key}: unpack failed"
            );
        }

        return array_values($floats);
    }

    /**
     * Encode a PHP float array into a VEC_FromText() expression.
     */
    public function set($model, string $key, $value, array $attributes): array
    {
        if ($value === null) {
            return [$key => null];
        }

        if ($value instanceof Arrayable) {
            $value = $value->toArray();
        }

        if (! is_array($value)) {
            throw new InvalidArgumentException(
                "Vector attribute {$key} must be an array of floats, got "
                .get_debug_type($value)
            );
        }

        $floats = array_values(array_map('floatval', $value));
        $json = json_encode($floats, JSON_THROW_ON_ERROR);

        return [$key => new Expression("VEC_FromText('{$json}')")];
    }

    private function looksLikeJson(string $value): bool
    {
        return $value !== '' && $value[0] === '[';
    }
}
```

## File 4: Service Provider

**`app/Providers/VectorDatabaseServiceProvider.php`**

This wires everything together. It tells Laravel to use our custom connection class for every MariaDB connection. Register it in `bootstrap/providers.php` — ideally as the first entry so it runs before any database queries.

```php
<?php

declare(strict_types=1);

namespace App\Providers;

use App\Database\MariaDbVectorConnection;
use Illuminate\Database\Connection;
use Illuminate\Support\ServiceProvider;

/**
 * Swaps Laravel's default MariaDB connection for one that hands out
 * MariaDbVectorBuilder — so the four vector helpers Laravel ships
 * (whereVectorSimilarTo, whereVectorDistanceLessThan,
 * orderByVectorDistance, selectVectorDistance) work on MariaDB 11.7+
 * with native VEC_DISTANCE_COSINE SQL instead of pgvector's <=>
 * operator.
 */
class VectorDatabaseServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        Connection::resolverFor('mariadb', function ($connection, $database, $prefix, $config) {
            return new MariaDbVectorConnection($connection, $database, $prefix, $config);
        });
    }
}
```

## Installation

1. Create the directory structure:

```
app/Database/
├── Casts/
│   └── AsVector.php
├── MariaDbVectorBuilder.php
└── MariaDbVectorConnection.php

app/Providers/
└── VectorDatabaseServiceProvider.php
```

2. Register the service provider in `bootstrap/providers.php`:

```php
return [
    App\Providers\VectorDatabaseServiceProvider::class, // first
    App\Providers\AppServiceProvider::class,
    // ...
];
```

3. Verify the connection is active:

```bash
php artisan tinker
>>> DB::connection()::class
# Should output: App\Database\MariaDbVectorConnection
```

## Migration

Create a VECTOR column and index in your migration. Laravel's schema grammar doesn't yet support MariaDB's `VECTOR` type natively, so the index creation uses a raw statement:

```php
Schema::create('documents', function (Blueprint $table) {
    $table->id();
    $table->string('title');
    $table->text('content');
    $table->vector('embedding', 768); // 768-dimension vector
    $table->timestamps();
});

// Add the MHNSW vector index (raw SQL until Laravel grammar lands)
DB::statement('ALTER TABLE documents ADD VECTOR INDEX vec_embedding (embedding) DISTANCE=cosine');
```

## Model Setup

Apply the `AsVector` cast to your embedding column:

```php
namespace App\Models;

use App\Database\Casts\AsVector;
use Illuminate\Database\Eloquent\Model;

class Document extends Model
{
    protected $fillable = ['title', 'content', 'embedding'];

    protected $casts = [
        'embedding' => AsVector::class,
    ];
}
```

## Usage

### Store a vector

```php
$embedding = $aiSdk->embed('How to train a neural network');

Document::create([
    'title'     => 'ML Basics',
    'content'   => 'Neural networks learn by...',
    'embedding' => $embedding, // array of 768 floats
]);
```

### Find similar documents

```php
$query = $aiSdk->embed('deep learning tutorial');

$results = Document::query()
    ->whereVectorSimilarTo('embedding', $query, minSimilarity: 0.7)
    ->limit(10)
    ->get();
```

### Order by distance

```php
$nearest = Document::query()
    ->orderByVectorDistance('embedding', $query)
    ->limit(5)
    ->get();
```

### Get distance values

```php
$withScores = Document::query()
    ->selectVectorDistance('embedding', $query, as: 'distance')
    ->addSelect(['id', 'title'])
    ->orderByVectorDistance('embedding', $query)
    ->limit(10)
    ->get();

foreach ($withScores as $doc) {
    echo "{$doc->title}: {$doc->distance}\n";
}
```

### Filter by maximum distance

```php
$close = Document::query()
    ->whereVectorDistanceLessThan('embedding', $query, maxDistance: 0.3)
    ->get();
```

## Generated SQL

For reference, here's what each method produces:

| Method | SQL |
|---|---|
| `whereVectorSimilarTo('embedding', $vec, 0.7)` | `WHERE VEC_DISTANCE_COSINE(\`embedding\`, VEC_FromText(?)) <= 0.3 ORDER BY VEC_DISTANCE_COSINE(\`embedding\`, VEC_FromText(?)) ASC` |
| `orderByVectorDistance('embedding', $vec)` | `ORDER BY VEC_DISTANCE_COSINE(\`embedding\`, VEC_FromText(?)) ASC` |
| `selectVectorDistance('embedding', $vec, 'dist')` | `VEC_DISTANCE_COSINE(\`embedding\`, VEC_FromText(?)) AS \`dist\`` |
| `whereVectorDistanceLessThan('embedding', $vec, 0.4)` | `WHERE VEC_DISTANCE_COSINE(\`embedding\`, VEC_FromText(?)) <= 0.4` |

All vector values are passed as parameterized bindings — no raw interpolation, no injection surface.

## Why MariaDB Over Postgres for Vectors?

- **Zero extensions**: VECTOR is a native column type in MariaDB 11.7+. No `CREATE EXTENSION pgvector`, no shared library management, no version compatibility checks.
- **MHNSW built-in**: The approximate nearest neighbor index ships with the server. `ALTER TABLE ... ADD VECTOR INDEX ... DISTANCE=cosine` and you're done.
- **Performance**: For datasets under ~10M rows (most applications), MariaDB's vector search is competitive with pgvector while giving you all of MariaDB's other advantages — Galera clustering, simpler replication, lower memory overhead.
- **Existing stack**: If you're already running MariaDB (or MySQL), you don't need to add a second database server just for vector search.

## License

MIT. Use it, ship it, package it.
