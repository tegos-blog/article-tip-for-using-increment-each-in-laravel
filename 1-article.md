
Laravel provides a convenient way to increment or decrement column values in a model using the `increment` and `decrement` methods.

Starting from [Laravel 9](https://laravel-news.com/laravel-9-48-0), an additional method, `incrementEach` (and `decrementEach`), allows updating multiple columns at once.

## The Problem: Incrementing and Decrementing Simultaneously

What if you need to **increment and decrement** different columns at the same time?

Consider the following scenario with a `PriceImport` model that has the columns `imported_rows` and `unknown_rows`. You might try something like this:

```php
PriceImport::query()->increment('unknown_rows')->decrement('imported_rows');
```

But, this approach won't work since `increment` and `decrement` methods can't be chained.

### Common Workaround (But Not Ideal)

Typical way to handle this is by using `DB::raw()`:

```php
PriceImport::query()->whereKey($priceImport->id)->update([
    'imported_rows' => DB::raw('imported_rows + 1'),
    'unknown_rows' => DB::raw('unknown_rows - 1'),
]);
```

Or little better:

```php
PriceImport::query()->whereKey($priceImport->id)->increment('imported_rows', [
    'unknown_rows' => DB::raw('unknown_rows - 1'),
]);
```

While this works, it's not the most elegant or laravel-friendly solution.

## The Elegant Solution: Using Negative Values with incrementEach

Laravel’s `incrementEach` method allows passing negative values, which effectively works as a decrement operation.

Here’s how you can achieve both incrementing and decrementing at the same time in a clean way:

### Full Example

```php
<?php

namespace App\Actions\PriceImport;

use App\Actions\Actionable;
use App\Models\PriceImport\PriceImport;
use App\Models\PriceImport\PriceImportProduct;
use App\QueryBuilder\Queries\PriceImport\MovePriceImportDuplicateRowQuery;

final class PriceImportHandleDuplicateAction implements Actionable
{
    public function handle(PriceImport $priceImport): void
    {
        $duplicateCount = PriceImportProduct::query()
            ->where('row_occurrences', '>', 1)
            ->where('price_import_id', $priceImport->id)
            ->count();

        PriceImport::query()->whereKey($priceImport->id)->incrementEach([
            'imported_rows' => -$duplicateCount, // Decrementing
            'unknown_rows' => $duplicateCount,   // Incrementing
        ]);

        (new MovePriceImportDuplicateRowQuery($priceImport->id))->query();
    }
}
```

### Resulting Query

```sql
UPDATE `price_imports`
SET `imported_rows` = `imported_rows` + -1,
    `unknown_rows` = `unknown_rows` + 1
WHERE `price_imports`.`id` = 5;
```

## Conclusion

With this simple trick, you can increment and decrement multiple columns simultaneously using Laravel’s `incrementEach` method, making your code cleaner and more readable.

