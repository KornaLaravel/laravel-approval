# Upgrade Guide

## Upgrading from v2.0 to v2.1

v2.1 is a minor release that drops support for Laravel 11. There are no public API changes, no schema changes, and no required code changes for applications running Laravel 12 or 13.

### Requirements

| Requirement | v2.0           | v2.1           |
|-------------|----------------|----------------|
| PHP         | ^8.3           | ^8.3           |
| Laravel     | ^11, ^12, ^13  | ^12, ^13       |

### Why was Laravel 11 dropped?

Laravel 11 reached end-of-life on **March 12, 2026** and no longer receives bug or security fixes from the Laravel team. Continuing to support an unmaintained framework version is a security risk.

### What if I'm still on Laravel 11?

Stay on `^2.0` until you've upgraded your application:

```bash
composer require cjmellor/approval:"^2.0"
```

Then follow the [Laravel upgrade guide](https://laravel.com/docs/upgrade) to move to Laravel 12 or 13 before upgrading this package to `^2.1`.

### Upgrade Process (Laravel 12 / 13)

```bash
composer require cjmellor/approval:"^2.1"
```

That's it — no migrations, no config republish, no code changes required.

### Internal Notes

- The `requestedBy` query scope on `Cjmellor\Approval\Models\Approval` has been migrated from the `scopeRequestedBy` method-name convention to Laravel 12's `#[Scope]` attribute. **This is an internal refactor** — calling `Approval::requestedBy($user)` continues to work exactly as before.

## Upgrading from v1 to v2

This guide will help you safely upgrade from v1.x to v2.x of the Approval package.

### Breaking Changes

The following changes in v2 may require updates to your application code:

| Change | Before (v1) | After (v2) | Action |
|--------|-------------|------------|--------|
| Custom expiration method | `thenDo(callable $callback)` | `thenCustom()` | Rename calls; move callback logic to an `ApprovalExpired` event listener |
| Rollback event class | `ModelRolledBackEvent` | `ModelRolledBack` | Update event listener type-hints and references |
| Facade removed | `Cjmellor\Approval\Facades\Approval` | *(removed)* | Use `Cjmellor\Approval\Models\Approval` directly |
| Config keys flattened | `config('approval.approval.approval_pivot')` | `config('approval.approval_pivot')` | Re-publish config: `php artisan vendor:publish --tag="approval-config" --force` |
| Event `$approval` property | Typed as `Illuminate\Database\Eloquent\Model` | Typed as `Cjmellor\Approval\Models\Approval` | Update any event listener type-hints |
| `pending()` scope | Returns all approvals with `state=pending` | Excludes approvals with a `custom_state` set | Use `whereState('pending')` if you need the old behaviour |
| Expiration actions | Stored as raw strings (`'reject'`, `'postpone'`) | Stored via `ExpirationAction` enum | Update any code that reads `expiration_action` directly |
| Mass assignment | `$guarded = []` | Explicit `$fillable` array | If you were mass-assigning unusual columns, use `forceFill()` |

### Before You Begin

1. **Create a backup of your database** - This is critical as schema changes will be made
2. **Ensure your application has no pending migrations** - Run `php artisan migrate` to apply any outstanding migrations

### Upgrade Process

#### Step 1: Update the Package

```bash
composer require cjmellor/approval:"^2.0"
```

#### Step 2: Publish New Migrations

The v2 package includes new migrations that need to be published:

```bash
php artisan vendor:publish --tag="approval-migrations"
```

#### Step 3: Run the Automated Upgrade Command

```bash
php artisan approval:upgrade-to-v2
```

This command will:
- Verify your database backup (with a confirmation prompt)
- Add the necessary `custom_state` column for configurable states
- Preserve all your existing approval data
- Validate the data integrity after the migration

#### Step 4: Apply Additional Migrations

```bash
php artisan migrate
```

This will apply any remaining migrations included with v2, such as those for expiration and requestor tracking.

#### Step 5: Re-Publish Configuration

> **Required if you published the config in v1.** The config structure changed — keys were flattened from `approval.approval.*` to `approval.*`, and new options were added (`users_table`, `states`).

```bash
php artisan vendor:publish --tag="approval-config" --force
```

The v1 config looked like this:

```php
return [
    'approval' => [
        'approval_pivot' => 'approvalable',
    ],
];
```

The v2 config is now:

```php
return [
    'approval_pivot' => 'approvalable',
    'users_table' => 'users',
    'states' => [
        'approved' => ['name' => 'Approved'],
        'pending' => ['name' => 'Pending', 'default' => true],
        'rejected' => ['name' => 'Rejected'],
    ],
];
```

If you had customised the `approval_pivot` value, re-apply your customisation after re-publishing. If you reference config values anywhere in your app, update the paths:

```php
// Before (v1)
config('approval.approval.approval_pivot');

// After (v2)
config('approval.approval_pivot');
```

### Verifying Your Upgrade

After upgrading, verify that:

1. Your existing approvals are still accessible
2. Standard approval operations (approve, reject) work correctly
3. The new features like custom states are available

### Potential Issues

- If you encounter a "table approvals has no column named custom_state" error after upgrading, run `php artisan approval:upgrade-to-v2`
- If you see errors about missing creator columns, ensure you've completed step 4 (Apply Additional Migrations)
