# Laravel Testing Pitfalls — Decision Table

## Resource Transformer Assertions

| Scenario | Wrong | Correct |
|---|---|---|
| Check `whenLoaded` not loaded | `expect($array['unit'])->toBeNull()` | `expect($array['unit'])->not->toBeInstanceOf(Unit::class)` or `array_key_exists` check |
| Validate ISO 8601 dates | `DateTimeImmutable::createFromFormat(ATOM, $str)` | `strtotime($str)` — flexible, handles `.000000Z` suffix |
| Assert factory nullable field | `'icon' => null` on NOT NULL column | Use column default: `'icon' => 'o-bell'` |
| Generate optional JSON data | `fake()->passthrough()` | `fake()->words(3)` — passthrough requires a value argument |

## PHPStan Patterns for Laravel/Livewire

| Scenario | Wrong | Correct |
|---|---|---|
| Access auth ID in blade component | `auth()->id()` | `Auth::id()` with `use Illuminate\Support\Facades\Auth;` |
| HasFactory on model | `use HasFactory;` alone | `/** @use HasFactory<\Database\Factories\XFactory> */` above `use HasFactory;` |
| JsonResource magic property | `$this->field` in `toArray()` | `@property-read` annotations + `$model = $this->resource;` with `@var Model $model` |
| After fixing PHPStan errors | Run `composer phpstan` once | Regenerate baseline: `vendor/bin/phpstan analyse --generate-baseline`, then verify |
| Multiple middleware args | `->middleware('a', 'b')` | `->middleware(['a', 'b'])` — single string or array, never two args |

## Sanctum Token Testing

**Never use `Sanctum::actingAs()` with ability middleware.** The Guard resolves
the user from the session guard first and wraps with `TransientToken` (no
abilities). The `ability:` middleware always throws `MissingAbilityException`.

**Correct pattern — real tokens via Bearer header:**

```php
// Create token with specific abilities
$token = $user->createToken('test', ['tickets:read', 'tickets:write'])->plainTextToken;

// Authenticate via header
$response = $this->withHeaders([
    'Authorization' => 'Bearer ' . $token,
    'Accept' => 'application/json',
])->getJson('/api/tickets');
```

**Test helper pattern:**

```php
private function createTokenWithAbilities(array $abilities, array $permissions = []): string
{
    $args = [];
    if ($permissions) {
        $args['permissions'] = $permissions;
    }
    ['user' => $user] = $this->createUserWithUnit(...$args);
    return $user->createToken('test-token', $abilities)->plainTextToken;
}
```

**Guard caching across requests:** When a test makes multiple HTTP requests
with different Bearer tokens (e.g. testing as user A then user B), the
Sanctum guard caches the first token's user. Call `Auth::forgetGuards()`
before each request when switching tokens within one test.

**Dual middleware trap:** Routes with both `ability:X:read` and
`role_or_permission:Y` require the test user to have BOTH the token ability
AND the Spatie permission. Pass permissions as the second arg:

```php
$token = $this->createTokenWithAbilities(['tickets:read'], ['view_all_tickets']);
```

**Zabbix / parameterized endpoints:** When testing ability middleware on
endpoints that require query parameters (e.g. Zabbix), a 422 validation
error proves the ability check passed (middleware ran in order). Accept
`assertContains($response->getStatusCode(), [200, 422])` as proof.

## Factory Creation Checklist

1. Check the migration for NOT NULL columns — factory must not pass null.
2. Check if model has `HasFactory` trait — add it if missing.
3. UUID models: manual `Str::uuid()` in `boot()` + `HasFactory` is fine;
   do NOT add `HasUuids` trait (conflicts with boot logic).
4. After seeding explicit IDs in tests, resync Postgres sequence:
   `SELECT setval('table_id_seq', (SELECT MAX(id) FROM table))`.

## E2E Test Structure

```
tests/e2e/<feature>/<feature>.spec.ts
```

Imports from `../shared/fixtures`:
- `login(page, nCode?, password?)` — fills login form, waits for redirect
- `waitForLivewire(page)` — waits for `.wire-loading` to disappear
- `waitForToast(page, text?)` — waits for toast notification

Pattern:
1. `test.beforeEach` — login + navigate to page
2. Test: page loads (assert header, key elements visible)
3. Test: CRUD operations via Livewire modals
4. Use `page.locator('input[wire\\:model="field"]')` for Livewire inputs
5. Use `page.getByRole('button', { name: '...' })` for buttons
6. Always `waitForLivewire(page)` after actions that trigger updates