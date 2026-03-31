---
title: Use Transactions for Multi-Step Operations
impact: HIGH
impactDescription: Partial writes cause data inconsistency
tags: [database, transactions, consistency, atomicity]
---

## Use Transactions for Multi-Step Operations

When a business operation requires multiple database writes that must succeed or fail together, wrap them in a transaction. Without a transaction, a failure partway through leaves your data in an inconsistent state -- for example, an order is created but the inventory is never decremented, or money is deducted from one account but never credited to another.

**Incorrect**

```php
// Multiple writes without a transaction - if any step fails,
// previous steps are already committed and cannot be undone
public function placeOrder(Request $request): Order
{
    $order = Order::create([
        'user_id' => $request->user()->id,
        'total' => $request->total,
    ]);

    foreach ($request->items as $item) {
        $order->items()->create($item); // What if this fails on item 3 of 5?
    }

    $order->user->decrement('balance', $request->total); // What if this fails?

    Payment::create([
        'order_id' => $order->id,
        'amount' => $request->total,
        'status' => 'completed',
    ]);

    return $order;
}

// Catching exceptions but still having partial writes
public function transferFunds(Account $from, Account $to, float $amount): void
{
    try {
        $from->decrement('balance', $amount); // This commits immediately
        $to->increment('balance', $amount);   // If this fails, money vanishes
    } catch (\Exception $e) {
        // Too late - the decrement is already committed
        Log::error('Transfer failed: ' . $e->getMessage());
    }
}
```

**Correct**

```php
use Illuminate\Support\Facades\DB;

// DB::transaction() closure - the cleanest approach
// Automatically commits on success, rolls back on any exception
public function placeOrder(Request $request): Order
{
    return DB::transaction(function () use ($request) {
        $order = Order::create([
            'user_id' => $request->user()->id,
            'total' => $request->total,
        ]);

        foreach ($request->items as $item) {
            $order->items()->create($item);
        }

        $order->user->decrement('balance', $request->total);

        Payment::create([
            'order_id' => $order->id,
            'amount' => $request->total,
            'status' => 'completed',
        ]);

        return $order;
    });
}

// Handling deadlocks with automatic retries (second argument = attempts)
public function transferFunds(Account $from, Account $to, float $amount): void
{
    DB::transaction(function () use ($from, $to, $amount) {
        // Lock the rows to prevent concurrent modification
        $from = Account::lockForUpdate()->find($from->id);
        $to = Account::lockForUpdate()->find($to->id);

        if ($from->balance < $amount) {
            throw new InsufficientFundsException('Insufficient balance.');
        }

        $from->decrement('balance', $amount);
        $to->increment('balance', $amount);

        TransferLog::create([
            'from_account_id' => $from->id,
            'to_account_id' => $to->id,
            'amount' => $amount,
        ]);
    }, attempts: 3); // Retry up to 3 times on deadlock
}

// Manual transaction control for complex flows where you need
// fine-grained control over when to commit or roll back
public function importUsers(array $records): ImportResult
{
    $result = new ImportResult();

    DB::beginTransaction();

    try {
        foreach ($records as $record) {
            $user = User::create($record);
            $user->assignRole('member');
            $result->addSuccess($user);
        }

        DB::commit();
    } catch (\Exception $e) {
        DB::rollBack();
        $result->setError($e->getMessage());
    }

    return $result;
}

// Nested transactions with savepoints
// Laravel automatically uses savepoints for nested transaction calls
public function processTeamSignup(array $data): Team
{
    return DB::transaction(function () use ($data) {
        $team = Team::create(['name' => $data['team_name']]);

        // This creates a savepoint - if it fails, only this inner
        // transaction rolls back, not the outer one
        try {
            DB::transaction(function () use ($team, $data) {
                foreach ($data['members'] as $memberData) {
                    $user = User::create($memberData);
                    $team->members()->attach($user);
                }
            });
        } catch (\Exception $e) {
            // Inner transaction rolled back to savepoint.
            // The team is still created; we can handle members later.
            Log::warning("Member import failed for team {$team->id}: {$e->getMessage()}");
            $team->update(['member_import_status' => 'failed']);
        }

        return $team;
    });
}

// Transaction events for cache invalidation and side effects
// These ensure side effects only run when the transaction actually commits
use Illuminate\Support\Facades\DB;

public function updateProduct(Product $product, array $data): Product
{
    return DB::transaction(function () use ($product, $data) {
        $product->update($data);
        $product->variants()->upsert($data['variants'], ['sku'], ['price', 'stock']);

        // This callback only fires if the transaction commits successfully.
        // If the transaction rolls back, the cache is never cleared,
        // which is exactly what you want.
        DB::afterCommit(function () use ($product) {
            Cache::tags(['products'])->forget("product:{$product->id}");
            Cache::tags(['products'])->forget('product:listing');
            event(new ProductUpdated($product));
        });

        return $product;
    });
}

// Combining transactions with queue dispatches
// Use afterCommit on jobs to ensure they only dispatch after the transaction commits
public function createInvoice(Order $order): Invoice
{
    return DB::transaction(function () use ($order) {
        $invoice = Invoice::create([
            'order_id' => $order->id,
            'amount' => $order->total,
            'status' => 'pending',
        ]);

        $order->update(['invoiced_at' => now()]);

        // afterCommit() ensures the job is only dispatched after the
        // transaction commits. Without this, the job might run before
        // the invoice row is visible to other database connections.
        SendInvoiceEmail::dispatch($invoice)->afterCommit();

        return $invoice;
    });
}
```

**Reference:** [Laravel Database Transactions](https://laravel.com/docs/database#database-transactions)
