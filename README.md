
# 🔄 Replay - Idempotency Middleware

[![Latest Version on Packagist](https://img.shields.io/packagist/v/bvtterfly/replay.svg?style=flat-square)](https://packagist.org/packages/bvtterfly/replay)
[![GitHub Tests Action Status](https://img.shields.io/github/workflow/status/bvtterfly/replay/run-tests?label=tests)](https://github.com/bvtterfly/replay/actions?query=workflow%3Arun-tests+branch%3Amain)
[![GitHub Code Style Action Status](https://img.shields.io/github/workflow/status/bvtterfly/replay/Check%20&%20fix%20styling?label=code%20style)](https://github.com/bvtterfly/replay/actions?query=workflow%3A"Check+%26+fix+styling"+branch%3Amain)
[![Total Downloads](https://img.shields.io/packagist/dt/bvtterfly/replay.svg?style=flat-square)](https://packagist.org/packages/bvtterfly/replay)

This package makes your endpoints idempotent easily.

Check out this [Stripe Blog Post](https://stripe.com/blog/idempotency) about Idempotency.

Implementation inspired by [Stripe API](https://stripe.com/docs/api/idempotent_requests).

## 💡 Features

- Adding support idempotency requests to your APIs easily by adding a middleware.
- Works only for `POST` requests. Other endpoints are ignored.
- Record and replay only successful(2xx) and server-side errors(5xx) responses, without touching your controller again.
- it's safe to retry, it doesn't record the response with client-side errors (4xx).
- To prevent accidental misuse of the cached responses, the request's signature is validated to ensure that the cached response is returned using the same combination of Idempotency-Key and Request.
- Concurrency protection using Laravel's atomic locks to prevent race conditions.

## Installation

You can install the package via composer:

```bash
composer require bvtterfly/replay
```

You can publish the config file with:

```bash
php artisan vendor:publish --tag="replay-config"
```

This is the contents of the published config file:

```php
use Bvtterfly\Replay\StripePolicy;

return [

    /*
    |--------------------------------------------------------------------------
    | Cache Store
    |--------------------------------------------------------------------------
    |
    | This option controls the cache store that gets used while Replay will store the
    | information required for it to function.
    | By default, Replay will use the default cache store.
    |
    | Please see config/cache.php for the list of all available Cache Stores.
    |
     */

    'use' => env('REPLAY_CACHE_STORE', config('cache.default')),

    /*
    |--------------------------------------------------------------------------
    | Replay Master Switch
    |--------------------------------------------------------------------------
    |
    | Replay is enabled by default,
    | Use this setting to enable/disable the Replay.
    |
    */

    'enabled' => env('REPLAY_ENABLED', true),

    /*
    |--------------------------------------------------------------------------
    | Expiration Seconds
    |--------------------------------------------------------------------------
    |
    | This value controls the number of seconds until an idempotency response
    | is considered expired.
    |
    | The default is set to 1 day.
    |
    */

    'expiration' => 60 * 60 * 24,

    /*
    |--------------------------------------------------------------------------
    | Request Header Name
    |--------------------------------------------------------------------------
    |
    | Replay will check this header name to determine
    | if a request is an Idempotency request.
    |
    */

    'header_name' => 'Idempotency-Key',

    /*
    |--------------------------------------------------------------------------
    | Response Header Name
    |--------------------------------------------------------------------------
    |
    | Replay will add this header to previously executed responses
    | that's being replayed from the server.
    |
    | Use null or empty, if you don't need to identify these responses.
    |
    */
    'replied_header_name' => 'Idempotent-Replayed',
    
    /*
    |--------------------------------------------------------------------------
    | Policy
    |--------------------------------------------------------------------------
    |
    | The policy determines whether a request is idempotent and whether the response should
    |  be recorded.
    |
    */

    'policy' => StripePolicy::class,

];
```
> **Note:** Replay needs a cache driver that supports [Cache Tags](https://laravel.com/docs/9.x/cache#cache-tags) & [Atomic Locks](https://laravel.com/docs/9.x/cache#atomic-locks) features. Refer to [Laravel's documentation](https://laravel.com/docs/9.x/cache) to see if your driver supports these features.

Optionally, you can publish the translations using

```bash
php artisan vendor:publish --tag="replay-translations"
```

## ✨ Server Usage

The `Bvtterfly\Replay\Replay`-middleware must be registered in the kernel:
```php
//app/Http/Kernel.php

protected $routeMiddleware = [
  ...
  'replay' => \Bvtterfly\Replay\Replay,
];
```
Next, For idempotent an endpoint, apply `replay` middleware to it:
```php
Route::post('/payments', function () {
    //
})->middleware('replay');
```
By default, Replay stores the idempotent key as a cache key in the cache store, So all routes with replay middleware share the same cache key with an idempotent key. It's Okay to store it this way in most cases, but in some scenarios, we just need to separate them. In these scenarios, we can add a prefix to cache keys using middleware parameters:

```php
Route::post('/payments', function () {
//
})->middleware('replay:payments');
```

### Custom Policy

Replay use Policy to determine whether a request is idempotent and whether the response should be recorded. By default, Replay includes and uses `StripePolicy` Policy.
To create your custom policy, you first need to implement the `\Bvtterfly\Replay\Contracts\Policy` contract:

```php
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

interface Policy
{
    public function isIdempotentRequest(Request $request): bool;

    public function isRecordableResponse(Response $response): bool;
}
```
If you want to view an example implementation take a look at the `StripePolicy` class.

For using this policy, We can change the `policy` in the config file.

## ✨ Client Usage

To perform an idempotent request, Client must provide an additional `Idempotency-Key : <key>` header with a unique key to the request.

it is recommended to:
- Use "V4 UUIDs" for the creation of the idempotency unique keys (e.g. `07cd2d27-e0dc-466f-8193-28453e9c3023`).
- Derive the key from a user-attached object, like the ID of a shopping cart. This provides a relatively straightforward way to protect against double submissions.

Once Replay detects a key, it'll look it up in cache store. If found, it will serve the same response without hitting your controller action again.

To identify a previously executed response that’s being replayed from the server, look for the header `Idempotent-Replayed: true`.

If Replay can't find the key, it attempts to acquire a cache lock and caches successful or server error responses. Still, if it can't acquire the lock, another request with the same key is already in progress, then it will respond with the HTTP Conflict response status code.


## Resetting the Cache

If you need to manually reset the cache for this package, you may use the following artisan command:

```bash
php artisan replay:cache-reset
```


## 🧪 Testing

```bash
composer test
```

## Changelog

Please see [CHANGELOG](CHANGELOG.md) for more information on what has changed recently.

## Security Vulnerabilities

Please review [our security policy](../../security/policy) on how to report security vulnerabilities.

## Credits

- [Ari](https://github.com/bvtterfly)
- [All Contributors](../../contributors)

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.
