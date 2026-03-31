---
title: Use Database Migrations
impact: HIGH
impactDescription: Manual schema changes cause deployment failures
tags: [database, migrations, schema, versioning]
---

## Use Database Migrations

Always use migrations for every schema change. Migrations are version-controlled, reproducible, and run automatically during deployments. Never modify the database directly through a GUI tool or raw SQL scripts outside the migration system -- manual changes are invisible to your team, cannot be rolled back reliably, and will cause deployment failures when environments drift apart.

**Incorrect**

```php
// Running raw SQL directly against the database
DB::statement('ALTER TABLE users ADD COLUMN phone VARCHAR(20)');

// Or worse: manually running SQL in a database GUI tool
// "I'll just add the column in phpMyAdmin real quick..."

// Migrations without a proper down() method
public function up(): void
{
    Schema::table('users', function (Blueprint $table) {
        $table->string('phone', 20)->nullable();
    });
}

public function down(): void
{
    // Empty - can't roll back
}

// Modifying an existing migration that has already been run
// This does nothing because Laravel tracks which migrations have run.
// Other developers and environments will never see the change.

// Putting too many unrelated changes in a single migration
public function up(): void
{
    Schema::create('orders', function (Blueprint $table) { /* ... */ });
    Schema::create('invoices', function (Blueprint $table) { /* ... */ });
    Schema::create('shipping_labels', function (Blueprint $table) { /* ... */ });
    Schema::table('users', function (Blueprint $table) {
        $table->string('stripe_id')->nullable();
    });
    // If any of these fail, you can't roll back the ones that succeeded
}
```

**Correct**

```php
// Generate a migration with artisan
// php artisan make:migration add_phone_to_users_table --table=users

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->string('phone', 20)->nullable()->after('email');
            $table->index('phone');
        });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropIndex(['phone']);
            $table->dropColumn('phone');
        });
    }
};

// Creating a table with proper foreign keys and indexes
return new class extends Migration
{
    public function up(): void
    {
        Schema::create('orders', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            $table->foreignId('product_id')->constrained()->restrictOnDelete();
            $table->string('status', 30)->default('pending');
            $table->decimal('total', 10, 2);
            $table->timestamp('shipped_at')->nullable();
            $table->timestamps();
            $table->softDeletes();

            $table->index('status');
            $table->index(['user_id', 'status']); // Composite index for common queries
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('orders');
    }
};

// Zero-downtime migration strategy for adding a required column
// Step 1: Add column as nullable (safe, no locks on large tables)
return new class extends Migration
{
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->string('timezone', 50)->nullable()->after('email');
        });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropColumn('timezone');
        });
    }
};

// Step 2: Backfill existing rows (separate migration or artisan command)
return new class extends Migration
{
    public function up(): void
    {
        DB::table('users')
            ->whereNull('timezone')
            ->update(['timezone' => 'UTC']);
    }

    public function down(): void
    {
        // No rollback needed for data backfill
    }
};

// Step 3: Add the NOT NULL constraint once all rows are populated
return new class extends Migration
{
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->string('timezone', 50)->default('UTC')->change();
        });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->string('timezone', 50)->nullable()->change();
        });
    }
};

// Renaming columns safely (requires doctrine/dbal)
return new class extends Migration
{
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->renameColumn('name', 'full_name');
        });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->renameColumn('full_name', 'name');
        });
    }
};

// Squash migrations once your app is mature to speed up fresh installs
// php artisan schema:dump
// php artisan schema:dump --prune  (dumps then deletes old migration files)
// This creates a database/schema directory with a SQL dump that runs before
// any remaining migrations, dramatically speeding up migrate:fresh.
```

**Reference:** [Laravel Migrations](https://laravel.com/docs/migrations)
