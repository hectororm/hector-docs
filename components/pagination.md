---
breadcrumb:
  - Components
  - Pagination
summary-order: ;6
---

# Pagination

> 🆕 **Info**: *Since version 1.3*

> ℹ️ **Note**: While pagination is part of the **Hector ORM** ecosystem, it is available as a standalone package:
> [`hectororm/pagination`](https://github.com/hectororm/pagination).
> You can find it on [Packagist](https://packagist.org/packages/hectororm/pagination).
> You can use it independently of the ORM, in any PHP application. 🎉

## Pagination Types

| Type   | Class              | Use Case                                    |
|--------|--------------------|---------------------------------------------|
| Offset | `OffsetPagination` | Page-based navigation (with optional total) |
| Cursor | `CursorPagination` | Keyset pagination for large datasets        |
| Range  | `RangePagination`  | RFC 7233 style (Content-Range headers)      |

## Quick Start

Paginators are the recommended way to handle pagination. They provide a unified API for request parsing, navigation, and response preparation.

### Offset Paginator

```php
use Hector\Pagination\OffsetPagination;
use Hector\Pagination\Paginator\OffsetPaginator;

// 1. Create paginator (injectable as service)
$paginator = new OffsetPaginator(
    pageParam: 'page',
    perPageParam: 'per_page',
    defaultPerPage: 15,
    maxPerPage: 100,
);

// 2. Parse request: ?page=3&per_page=20
$request = $paginator->createRequest($serverRequest);

// 3. Query your data
$results = $repository->findAll($request->getLimit(), $request->getOffset());

// 4. Build pagination
$pagination = new OffsetPagination(
    items: $results,
    perPage: $request->perPage,
    currentPage: $request->page,
    hasMore: count($results) >= $request->perPage,
);

// 5. Prepare response with Link headers
$response = $paginator->prepareResponse($response, $serverRequest->getUri(), $pagination);
// Adds: Link: <...?page=1>; rel="first", <...?page=2>; rel="prev", <...?page=4>; rel="next"
```

### Cursor Paginator

```php
use Hector\Pagination\CursorPagination;
use Hector\Pagination\Encoder\Base64CursorEncoder;
use Hector\Pagination\Encoder\SignedCursorEncoder;
use Hector\Pagination\Paginator\CursorPaginator;

// 1. Create paginator with signed encoder
$paginator = new CursorPaginator(
    cursorParam: 'cursor',
    perPageParam: 'per_page',
    defaultPerPage: 15,
    maxPerPage: 100,
    encoder: new SignedCursorEncoder(
        inner: new Base64CursorEncoder(),
        secret: 'your-secret-key',
    ),
);

// 2. Parse request: ?cursor=eyJpZCI6NDJ9.signature&per_page=20
$request = $paginator->createRequest($serverRequest);
$position = $request->getPosition(); // ['id' => 42] or null (already decoded)

// 3. Query your data
$results = $repository->findAfter($position, $request->getLimit() + 1);
$hasMore = count($results) > $request->perPage;
$items = array_slice($results, 0, $request->perPage);

// 4. Build pagination
$pagination = new CursorPagination(
    items: $items,
    perPage: $request->perPage,
    nextPosition: $hasMore ? ['id' => end($items)->id] : null,
    previousPosition: $position,
);

// 5. Prepare response
$response = $paginator->prepareResponse($response, $serverRequest->getUri(), $pagination);
```

### Range Paginator

```php
use Hector\Pagination\Paginator\RangePaginator;
use Hector\Pagination\RangePagination;

// 1. Create paginator
$paginator = new RangePaginator(
    rangeParam: 'range',
    rangeUnit: 'items',
    defaultLimit: 20,
    maxLimit: 100,
);

// 2. Parse request: ?range=20-39 or header Range: items=20-39
$request = $paginator->createRequest($serverRequest);

// 3. Query your data
$results = $repository->findAll($request->getLimit(), $request->getOffset());
$total = $repository->count();

// 4. Build pagination
$pagination = new RangePagination(
    items: $results,
    start: $request->start,
    end: $request->end,
    total: $total,
);

// 5. Prepare response (includes Content-Range, Accept-Ranges, status 206)
$response = $paginator->prepareResponse($response, $serverRequest->getUri(), $pagination);
// Adds: Content-Range: items 20-39/1000
// Adds: Accept-Ranges: items
// Sets: HTTP 206 Partial Content
```

## Navigators

Navigators generate pagination requests and URIs for navigation links.

### Using with Paginators

```php
$navigator = $paginator->createNavigator($pagination);

// Get request objects
$firstRequest = $navigator->getFirstRequest();
$prevRequest = $navigator->getPreviousRequest();
$nextRequest = $navigator->getNextRequest();
$lastRequest = $navigator->getLastRequest();

// Get URIs directly
$baseUri = $serverRequest->getUri();
$firstUri = $navigator->getFirstUri($baseUri);  // https://example.com/api?page=1&per_page=20
$prevUri = $navigator->getPreviousUri($baseUri);
$nextUri = $navigator->getNextUri($baseUri);
$lastUri = $navigator->getLastUri($baseUri);
```

### Standalone Usage

```php
use Hector\Pagination\Navigator\OffsetPaginationNavigator;
use Hector\Pagination\UriBuilder\OffsetPaginationUriBuilder;

$uriBuilder = new OffsetPaginationUriBuilder(
    pageParam: 'p',
    perPageParam: 'limit',
);

$navigator = new OffsetPaginationNavigator($pagination, $uriBuilder);

$nextUri = $navigator->getNextUri($baseUri);
// https://example.com/api?p=4&limit=20
```

### Generic Navigator

Auto-detects pagination type:

```php
use Hector\Pagination\Navigator\PaginationNavigator;

$navigator = new PaginationNavigator($pagination);
// Works with OffsetPagination, CursorPagination, or RangePagination
```

## URI Builders

URI builders handle query parameter generation for each pagination strategy.

```php
use Hector\Pagination\UriBuilder\OffsetPaginationUriBuilder;
use Hector\Pagination\UriBuilder\CursorPaginationUriBuilder;
use Hector\Pagination\UriBuilder\RangePaginationUriBuilder;

// Offset
$builder = new OffsetPaginationUriBuilder('page', 'per_page');
$uri = $builder->buildUri($baseUri, $offsetRequest);
// ?page=3&per_page=20

// Cursor
$builder = new CursorPaginationUriBuilder('cursor', 'per_page');
$uri = $builder->buildUri($baseUri, $cursorRequest);
// ?cursor=eyJpZCI6NDJ9&per_page=20

// Range
$builder = new RangePaginationUriBuilder('range');
$uri = $builder->buildUri($baseUri, $rangeRequest);
// ?range=20-39
```

## Pagination Classes

### Offset Pagination

Page-based pagination with optional total count.

```php
use Hector\Pagination\OffsetPagination;

// Without total (simple)
$pagination = new OffsetPagination(
    items: $results,
    perPage: 20,
    currentPage: 3,
    hasMore: true,
);

// With total (full navigation support)
$pagination = new OffsetPagination(
    items: $results,
    perPage: 20,
    currentPage: 3,
    total: 500, // or fn() => $repository->count() for lazy loading
);

$pagination->getCurrentPage(); // 3
$pagination->getOffset();      // 40
$pagination->hasMore();        // true (calculated from total)
$pagination->hasPrevious();    // true
$pagination->getTotal();       // 500 or null
$pagination->getTotalPages();  // 25 or null
$pagination->getFirstItem();   // 41 or null
$pagination->getLastItem();    // 60 or null
```

Supports lazy total count with a `Closure`:

```php
$pagination = new OffsetPagination(
    items: $results,
    perPage: 20,
    currentPage: 3,
    total: fn() => $repository->count(), // Called only when needed
);
```

### Cursor Pagination

Keyset pagination for efficient navigation through large datasets, with optional total count.

```php
use Hector\Pagination\CursorPagination;

$pagination = new CursorPagination(
    items: $results,
    perPage: 20,
    nextPosition: ['id' => 42, 'created_at' => '2024-01-15'],
    previousPosition: ['id' => 22, 'created_at' => '2024-01-10'],
    cursorName: 'my-cursor', // Optional, for server-side stored cursors
    total: 1000, // Optional
);

$pagination->hasMore();            // true
$pagination->hasPrevious();        // true
$pagination->getNextPosition();    // ['id' => 42, 'created_at' => '2024-01-15']
$pagination->getPreviousPosition(); // ['id' => 22, 'created_at' => '2024-01-10']
$pagination->getCursorName();      // 'my-cursor' or null
$pagination->getTotal();           // 1000 or null
```


To encode positions as URL-safe cursor strings, use a `CursorEncoderInterface`:

```php
use Hector\Pagination\Encoder\Base64CursorEncoder;

$encoder = new Base64CursorEncoder();
$nextCursor = $encoder->encode($pagination->getNextPosition());
// "eyJpZCI6NDIsImNyZWF0ZWRfYXQiOiIyMDI0LTAxLTE1In0"
```

### Range Pagination

RFC 7233 style pagination using Content-Range headers.

```php
use Hector\Pagination\RangePagination;

$pagination = new RangePagination(
    items: $results,
    start: 0,
    end: 19,
    total: 1000,
);

$pagination->getStart();        // 0
$pagination->getEnd();          // 19
$pagination->getTotal();        // 1000
$pagination->getContentRange(); // "items 0-19/1000"
$pagination->hasMore();         // true
$pagination->hasPrevious();     // false
```

## Cursor Encoders

Cursors are encoded for safe URL transmission.

### Base64 Encoder (default)

```php
use Hector\Pagination\Encoder\Base64CursorEncoder;

$encoder = new Base64CursorEncoder();

$cursor = $encoder->encode(['id' => 42]);
$position = $encoder->decode($cursor);
```

### Signed Encoder

HMAC-signed cursors to prevent tampering. Uses the decorator pattern wrapping any encoder.

```php
use Hector\Pagination\Encoder\Base64CursorEncoder;
use Hector\Pagination\Encoder\SignedCursorEncoder;

$encoder = new SignedCursorEncoder(
    inner: new Base64CursorEncoder(),
    secret: 'your-secret-key',
    algo: 'sha256', // Optional, default is sha256
);

$cursor = $encoder->encode(['id' => 42]);
$position = $encoder->decode($cursor); // Throws InvalidArgumentException if tampered
```

Use with `CursorPaginator` for automatic encoding/decoding:

```php
$paginator = new CursorPaginator(
    encoder: $encoder,
    // ...
);
```

### Encrypted Encoder

Fully encrypted cursors using libsodium (`sodium_crypto_secretbox`). Provides confidentiality and integrity — cursor contents are hidden from clients.

> ⚠️ **Requires** the `sodium` PHP extension.

```php
use Hector\Pagination\Encoder\EncryptedCursorEncoder;

// Generate a secure key (store it safely!)
$key = EncryptedCursorEncoder::generateKey();

$encoder = new EncryptedCursorEncoder($key);

$cursor = $encoder->encode(['id' => 42, 'tenant' => 'acme']);
// "nonce+ciphertext" in URL-safe base64

$position = $encoder->decode($cursor);
// ['id' => 42, 'tenant' => 'acme']
```

Key requirements:
- Must be exactly 32 bytes (`SODIUM_CRYPTO_SECRETBOX_KEYBYTES`)
- Use `EncryptedCursorEncoder::generateKey()` to generate a valid key
- Store the key securely (environment variable, secrets manager)

Use cases:
- Hide internal IDs or sensitive data in cursors
- Prevent cursor inspection by clients
- Stronger security than signed-only cursors

## Cursor Storage

Store cursors server-side for short URLs and complex state.

### Cache Storage (PSR-16)

```php
use Hector\Pagination\Storage\CacheCursorStorage;

$storage = new CacheCursorStorage(
    cache: $psr16Cache,
    defaultTtl: 3600, // 1 hour
);

// Store cursor state
$name = $storage->store(['id' => 42, 'filters' => ['status' => 'active']]);
// Returns: "a1b2c3d4e5f6..."

// Retrieve cursor state
$state = $storage->retrieve($name);
// Returns: ['id' => 42, 'filters' => ['status' => 'active']]

// Delete cursor
$storage->delete($name);
```

### Array Storage (testing)

```php
use Hector\Pagination\Storage\ArrayCursorStorage;

$storage = new ArrayCursorStorage();

$name = $storage->store(['id' => 42]);
$state = $storage->retrieve($name);

$storage->clear(); // Clear all stored cursors
$storage->count(); // Get count of stored cursors
```

## Pagination Requests

Request objects encapsulate pagination parameters from HTTP requests (PSR-7).

### OffsetPaginationRequest

```php
use Hector\Pagination\Request\OffsetPaginationRequest;

// From PSR-7 request: ?page=3&per_page=20
$request = OffsetPaginationRequest::fromRequest(
    request: $serverRequest,
    pageParam: 'page',
    perPageParam: 'per_page',
    defaultPerPage: 15,
    maxPerPage: 100,
);

$request->page;       // 3
$request->perPage;    // 20
$request->getOffset(); // 40
$request->getLimit();  // 20
```

### CursorPaginationRequest

```php
use Hector\Pagination\Request\CursorPaginationRequest;
use Hector\Pagination\Encoder\Base64CursorEncoder;

// From PSR-7 request: ?cursor=eyJpZCI6NDJ9&per_page=20
$request = CursorPaginationRequest::fromRequest(
    request: $serverRequest,
    cursorParam: 'cursor',
    perPageParam: 'per_page',
    defaultPerPage: 15,
    maxPerPage: 100,
    encoder: new Base64CursorEncoder(), // Optional, defaults to Base64
);

$request->perPage;       // 20
$request->getPosition(); // ['id' => 42] or null (already decoded)
$request->getLimit();    // 20

// Or create directly with decoded position
$request = new CursorPaginationRequest(
    perPage: 20,
    position: ['id' => 42],
);

// Or from encoded cursor string
$request = CursorPaginationRequest::fromCursor(
    cursor: 'eyJpZCI6NDJ9',
    perPage: 20,
    encoder: $encoder,
);
```

#### Backward navigation (previous page)

Cursor pagination supports navigating to previous pages. The `CursorPaginationNavigator` handles this automatically by encoding the direction in the cursor:

```php
use Hector\Pagination\Navigator\CursorPaginationNavigator;

$navigator = new CursorPaginationNavigator($pagination);

$nextRequest = $navigator->getNextRequest();     // Forward navigation
$prevRequest = $navigator->getPreviousRequest(); // Backward navigation (direction encoded)

$prevRequest->isBackward(); // true
```

When using `CursorPaginationUriBuilder`, the direction is transparently encoded in the cursor token. No additional query parameters are needed:

```php
$prevUri = $navigator->getPreviousUri($baseUri);
// ?cursor=eyJpZCI6NiwiX19kaXJlY3Rpb24iOiJiYWNrd2FyZCJ9&per_page=20
// The direction is embedded in the cursor — the URL stays clean.
```

The paginator (both `QueryCursorPaginator` and `BuilderCursorPaginator`) automatically detects backward requests, reverses the query direction, and returns results in the correct order.

### RangePaginationRequest

```php
use Hector\Pagination\Request\RangePaginationRequest;

// From query string: ?range=0-19 or ?offset=0&limit=20
$request = RangePaginationRequest::fromRequest(
    request: $serverRequest,
    offsetParam: 'offset',
    limitParam: 'limit',
    rangeParam: 'range',
    defaultLimit: 20,
    maxLimit: 100,
);

// Or from Range header (RFC 7233): Range: items=0-19
$request = RangePaginationRequest::fromHeader(
    request: $serverRequest,
    unit: 'items',
    defaultEnd: 19,
    maxRange: 100,
);

$request->start;      // 0
$request->end;        // 19
$request->getOffset(); // 0
$request->getLimit();  // 20
```

## JSON Serialization

All pagination classes implement `JsonSerializable`.

### OffsetPagination

```json
{
    "data": ["..."],
    "per_page": 20,
    "current_page": 3,
    "has_more": true,
    "total": 500,
    "total_pages": 25,
    "first_item": 41,
    "last_item": 60
}
```

### CursorPagination

```json
{
    "data": ["..."],
    "per_page": 20,
    "has_more": true,
    "next_position": {"id": 42, "created_at": "2024-01-15"},
    "previous_position": {"id": 22, "created_at": "2024-01-10"},
    "cursor_name": "my-cursor",
    "total": 1000
}
```

### RangePagination

```json
{
    "data": ["..."],
    "start": 0,
    "end": 19,
    "per_page": 20,
    "total": 1000,
    "has_more": true
}
```

## Iteration

All pagination classes are iterable and countable.

```php
foreach ($pagination as $item) {
    // Process item
}

// Or use count
$count = count($pagination); // Items on current page
```

## Exceptions

Pagination classes validate constructor arguments and throw `InvalidArgumentException`:

| Class              | Condition          | Message                          |
|--------------------|--------------------|----------------------------------|
| `OffsetPagination` | `$perPage <= 0`    | `perPage must be greater than 0` |
| `OffsetPagination` | `$currentPage < 1` | `currentPage must be at least 1` |
| `RangePagination`  | `$start < 0`       | `start must be >= 0`             |
| `RangePagination`  | `$end < $start`    | `end must be >= start`           |
