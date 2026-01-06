# Technical Summary: Dynamic Price Extraction in PriceBuddy

## Overview

PriceBuddy is a Laravel-based web application that dynamically extracts product prices from URLs across different online retailers without requiring code changes for each new store. This document provides a technical deep-dive into the architecture and mechanisms that enable this flexible price extraction system.

## Core Architecture

### High-Level Flow

```
User adds URL → Store Detection → HTML Fetching → Data Extraction → Price Storage → Price History
```

### Key Components

1. **Store Configuration** - Defines extraction rules per domain
2. **Scraping Service** - Fetches HTML content (HTTP or Browser-based)
3. **Extraction Strategies** - Multiple methods to extract data from HTML
4. **Price Management** - Stores and tracks price history
5. **Auto-Discovery** - Automatically creates stores for new domains

## 1. Store Model & Configuration

**File**: `app/Models/Store.php`

A `Store` represents a retailer/website and contains all the metadata needed to extract product information from that site.

### Key Store Attributes

```php
[
    'name' => 'Amazon',
    'domains' => [
        ['domain' => 'amazon.com'],
        ['domain' => 'www.amazon.com']
    ],
    'scrape_strategy' => [
        'title' => ['type' => 'selector', 'value' => 'meta[property="og:title"]|content'],
        'price' => ['type' => 'selector', 'value' => 'span.a-price'],
        'image' => ['type' => 'selector', 'value' => 'meta[property="og:image"]|content']
    ],
    'settings' => [
        'scraper_service' => 'http',  // or 'api' for JavaScript-rendered sites
        'locale_settings' => [
            'locale' => 'en_US',
            'currency' => 'USD'
        ]
    ]
]
```

### Store Domain Matching

When a URL is added:
1. Extract hostname from URL using `Illuminate\Support\Uri::of($url)->host()`
2. Query stores with matching domain: `Store::query()->domainFilter($host)->first()`
3. Use the store's `scrape_strategy` to extract data

**Code**: `app/Services/ScrapeUrl.php:206-210`

## 2. Extraction Strategies

PriceBuddy supports **four different extraction methods**, allowing it to adapt to various website structures:

### 2.1 CSS Selectors (Most Common)

Uses standard CSS selectors to find elements in the DOM.

**Examples**:
- `span.a-price` - Select by class
- `#product-price` - Select by ID
- `meta[property="og:price:amount"]|content` - Extract attribute value

**Special Syntax**:
- `selector|attribute` - Extract an attribute value (e.g., `img|src`)
- `!selector` - Return HTML instead of text content
- `selector` (alone) - Extract text content (default)

**Implementation**: `app/Services/ScrapeUrl.php:250-271`

```php
public static function parseSelector(string $selector): array
{
    // If starts with !, return HTML
    if (str_starts_with($selector, '!')) {
        return [substr($selector, 1), 'html'];
    }
    
    // If contains |, extract attribute
    if (str_contains($selector, '|')) {
        $parts = explode('|', $selector);
        $attr = array_pop($parts);
        return [implode('|', $parts), 'attr', [$attr]];
    }
    
    // Default: extract text
    return [$selector, 'text'];
}
```

### 2.2 Regular Expressions (Regex)

Powerful pattern matching for complex extraction scenarios.

**Examples**:
- `~"price":\s?"(.*?)"~` - Extract price from JSON-like string
- `~>\$(\d+(\.\d{2})?)<~` - Extract price from HTML tag
- `~"hiRes":"(.+?)"~` - Extract high-res image URL (Amazon)

**Use Cases**:
- Extracting data embedded in JavaScript objects
- Finding patterns in unstructured text
- Handling dynamically generated content

### 2.3 XPath Expressions

XML/HTML path queries for precise element targeting.

**Examples**:
- `//div[@class="price"]/text()` - Direct path to price element
- `//meta[@property="og:title"]/@content` - Attribute extraction via XPath

### 2.4 JSON Path

Dot notation for extracting data from JSON responses or embedded JSON.

**Example**:
```
product.price
data.items[0].pricing.amount
```

**Use Cases**:
- API responses
- JavaScript configuration objects
- Embedded JSON-LD data

## 3. Scraping Service Layer

**File**: `app/Services/ScrapeUrl.php`

### 3.1 HTTP Scraper (Default)

- **Technology**: cURL-based HTTP requests
- **Package**: `jez500/web-scraper-for-laravel`
- **Best For**: Static HTML websites
- **Advantages**: Fast, lightweight, low resource usage
- **Limitations**: Cannot execute JavaScript

### 3.2 API Scraper (Browser-based)

- **Technology**: Headless browser (SeleniumBase)
- **Docker Service**: `scraper:3000` (SeleniumBase Scrapper)
- **Best For**: Single-page applications (SPAs), JavaScript-rendered content
- **Advantages**: Full browser rendering, handles dynamic content
- **Limitations**: Slower, higher resource usage

**Configuration**: `config/price_buddy.php:23`

```php
'scraper_api_url' => env('SCRAPER_BASE_URL', 'http://scraper:3000'),
```

### 3.3 Scraping Process

**File**: `app/Services/ScrapeUrl.php:106-146`

```php
public function scrape(array $options = []): array
{
    $attempt = 0;
    $output = [];
    
    // Retry logic (default: 3 attempts)
    while ($attempt < $this->maxAttempts) {
        $attempt++;
        
        // Disable cache on retry
        if ($attempt > 1) {
            $options['use_cache'] = false;
        }
        
        $output = $this->scrapeUrl($options);
        
        // Break if title found
        if (!empty($output['title'])) {
            break;
        }
    }
    
    // Validate required fields
    foreach (['price', 'title'] as $required) {
        if (empty($output[$required])) {
            $this->errorLog('Missing required field: ' . $required);
            return $output;
        }
    }
    
    return $output;
}
```

### 3.4 Caching

- **Cache TTL**: Configurable via `AppSettings::scrape_cache_ttl`
- **Purpose**: Reduce load on target websites and improve performance
- **Behavior**: Cache disabled on retry attempts

## 4. Auto-Discovery System

**File**: `app/Services/AutoCreateStore.php`

One of PriceBuddy's most powerful features is the ability to automatically create store configurations for new domains.

### Auto-Discovery Process

1. **URL Submitted** → Fetch HTML with HTTP scraper
2. **Try Multiple Strategies** → Attempt predefined selectors and regex patterns
3. **Validate Results** → Ensure price can be parsed as numeric value
4. **Create Store** → Save successful strategy for future use

### Predefined Strategies

**Configuration**: `config/price_buddy.php:34-74`

#### Title Extraction Strategies
```php
'title' => [
    'selector' => [
        'meta[property="og:title"]|content',  // OpenGraph meta tag
        'title',                                // Page title
        'h1',                                   // Main heading
    ],
]
```

#### Price Extraction Strategies
```php
'price' => [
    'selector' => [
        'meta[property="product:price:amount"]|content',
        'meta[property="og:price:amount"]|content',
        '.a-price .a-offscreen',                    // Amazon-specific
        '[itemProp="price"]|content',               // Schema.org markup
        '.price',                                    // Common class name
        '[class^="price"]',                          // Classes starting with "price"
        '[class*="price"]',                          // Classes containing "price"
    ],
    'regex' => [
        '~"price":\s?"(.*?)"~',                     // JSON object
        '~>\$(\d+(\.\d{2})?)<~',                    // Price in HTML tag
        '~\$(\d+(\.\d{2})?)~',                      // Dollar amount anywhere
    ],
]
```

#### Image Extraction Strategies
```php
'image' => [
    'selector' => [
        'meta[property="og:image"]|content',
        'meta[property="og:image:secure_url"]|content',
    ],
    'regex' => [
        '~"hiRes":"(.+?)"~',                        // Amazon high-res
        '~"image":\s?"(.*?\.jpg)"~',               // JSON image URL
        '~"image":\s?"(.*?\.png)"~',               // JSON image URL (PNG)
    ],
]
```

### Auto-Discovery Algorithm

**File**: `app/Services/AutoCreateStore.php:175-216`

```php
protected function attemptSelectors(array $selectors, ?Closure $validateValue = null): ?array
{
    $dom = new Crawler($this->html);
    
    foreach ($selectors as $selector) {
        $selectorSettings = ScrapeUrl::parseSelector($selector);
        $realSelector = $selectorSettings[0];
        $method = $selectorSettings[1] ?? 'text';
        $args = $selectorSettings[2] ?? [];
        
        try {
            $results = $dom->filter($realSelector)
                ->each(function (Crawler $node) use ($method, $args, $validateValue) {
                    $extracted = call_user_func_array([$node, $method], $args);
                    return is_null($validateValue) 
                        ? $extracted 
                        : $validateValue($extracted);  // Validate (e.g., parse as float)
                });
            
            $value = data_get($results, '0');
            
            if ($value) {
                return ['type' => 'selector', 'value' => $selector, 'data' => $value];
            }
        } catch (DomSelectorException $e) {
            // Try next selector
        }
    }
    
    return null;
}
```

### Price Validation

**File**: `app/Services/AutoCreateStore.php:147-161`

```php
protected function parsePrice(): ?array
{
    $validateCallback = function ($value) {
        return CurrencyHelper::toFloat($value);  // Must parse to valid float
    };
    
    // Try selectors first
    if ($match = $this->attemptSelectors($this->getStrategy('price', 'selector'), $validateCallback)) {
        return $match;
    }
    
    // Fallback to regex
    if ($match = $this->attemptRegex($this->getStrategy('price', 'regex'), $validateCallback)) {
        return $match;
    }
    
    return [];
}
```

## 5. Currency & Locale Handling

**File**: `app/Services/Helpers/CurrencyHelper.php`

PriceBuddy handles international currencies and locales intelligently.

### Currency Parsing

The `CurrencyHelper::toFloat()` method normalizes price strings across different formats:

```php
public static function toFloat(mixed $value, ?string $locale = null, ?string $iso = null): float
{
    $iso = $iso ?? self::getCurrency();    // e.g., 'USD', 'EUR'
    $locale = $locale ?? self::getLocale();  // e.g., 'en_US', 'fr_FR'
    
    // Strip non-numeric characters except . and ,
    $value = preg_replace('/[^\d\.\,]/', '', (string) $value);
    
    // Use PHP's NumberFormatter with locale-aware parsing
    $currencies = new ISOCurrencies;
    $numberFormatter = new NumberFormatter($locale, NumberFormatter::DECIMAL);
    $moneyParser = new IntlLocalizedDecimalParser($numberFormatter, $currencies);
    $moneyFormatter = new DecimalMoneyFormatter($currencies);
    
    $money = $moneyParser->parse($value, new Currency($iso));
    
    return (float) $moneyFormatter->format($money);
}
```

### Examples

| Input | Locale | Currency | Output |
|-------|--------|----------|--------|
| "$1,234.56" | en_US | USD | 1234.56 |
| "€1.234,56" | de_DE | EUR | 1234.56 |
| "¥123,456" | ja_JP | JPY | 123456 |
| "£1 234.56" | en_GB | GBP | 1234.56 |

### Store-Specific Locale

Each store can override global locale settings:

```php
'settings' => [
    'locale_settings' => [
        'locale' => 'en_AU',
        'currency' => 'AUD'
    ]
]
```

## 6. Data Models & Relationships

### Entity Relationship

```
User
  └── Product (1:N)
        ├── Url (1:N)
        │     ├── Store (N:1)
        │     └── Price (1:N)
        └── Tag (N:N)
```

### Product Model

**File**: `app/Models/Product.php`

- Tracks product information across multiple stores
- Aggregates prices from all associated URLs
- Maintains price history and trends

**Key Attributes**:
```php
[
    'title' => 'Product Name',
    'image' => 'https://...',
    'current_price' => 99.99,           // Lowest current price
    'price_cache' => [...],             // Cached price data per URL
    'notify_price' => 89.99,            // Price alert threshold
    'notify_percent' => 10,             // Percentage drop alert
    'user_id' => 1,
]
```

### URL Model

**File**: `app/Models/Url.php`

Represents a specific product page on a store.

```php
[
    'url' => 'https://amazon.com/product/...',
    'product_id' => 123,
    'store_id' => 45,
]
```

**Key Method**:
```php
public function updatePrice(int|float|string|null $price = null): Price|Model|null
{
    // If price not provided, scrape it
    if (is_null($price) || $price === '') {
        $price = data_get($this->scrape(), 'price');
    }
    
    // Parse and normalize currency
    return $this->prices()->create([
        'price' => CurrencyHelper::toFloat(
            $price, 
            locale: $this->store?->locale, 
            iso: $this->store?->currency
        ),
        'store_id' => $this->store_id,
    ]);
}
```

### Price Model

**File**: `app/Models/Price.php`

Stores individual price points with timestamps.

```php
[
    'url_id' => 789,
    'store_id' => 45,
    'price' => 99.99,
    'notified' => false,
    'created_at' => '2025-01-06 10:30:00',
]
```

## 7. Price Fetching Pipeline

### Scheduled Updates

**Cron Job** (configured in Docker): Daily price updates

**Entry Point**: `app/Services/PriceFetcherService.php`

```php
public function updateAllPrices(): void
{
    Product::select('id')
        ->published()
        ->chunk(10, function (EloquentCollection $productIds) {
            UpdateAllPricesJob::dispatch($productIds->pluck('id')->toArray());
        });
}
```

### Job Processing

**File**: `app/Jobs/UpdateProductPricesJob.php`

```php
public function handle(): void
{
    logger()->info("Starting price fetch for: '{$this->product->title}'");
    
    $successful = $this->product->updatePrices();  // Update all URLs
    
    if (!$successful) {
        $this->product->user?->notify(new ScrapeFailNotification($this->product));
    }
    
    // Rate limiting between scrapes
    Sleep::for(AppSettings::new()->sleep_seconds_between_scrape)->seconds();
}
```

### Product Price Update

**File**: `app/Models/Product.php:464-473`

```php
public function updatePrices(): bool
{
    $successful = $this->urls
        ->map(fn (Url $url) => $url->updatePrice())
        ->filter();  // Filter out null results
    
    $this->updatePriceCache();  // Rebuild cached aggregate data
    
    return $successful->count() === $this->urls->count();
}
```

## 8. Price History & Caching

### Price Cache Structure

**File**: `app/Models/Product.php:370-416`

Products maintain a denormalized cache for performance:

```php
'price_cache' => [
    [
        'store_id' => 45,
        'store_name' => 'Amazon',
        'url_id' => 789,
        'url' => 'https://amazon.com/...',
        'trend' => 'down',              // up/down/stable/none
        'price' => 99.99,                // Current price
        'history' => [
            '2025-01-01' => 105.00,
            '2025-01-02' => 102.50,
            '2025-01-03' => 99.99,
        ],
        'last_scrape' => '2025-01-06 10:30:00',
        'locale' => 'en_US',
        'currency' => 'USD',
    ],
    // ... more URLs
]
```

### Cache Building

```php
public function buildPriceCache(): Collection
{
    $history = $this->getPriceHistory();  // Query from prices table
    $urls = Url::findMany($history->keys());
    
    return $urls
        ->filter(fn($url) => $url->store !== null)
        ->map(function ($url) use ($history): array {
            $urlHistory = $history->get($url->getKey());
            $lastScrapedPrice = $url->prices()->latest('id')->first();
            
            // Calculate trend: current vs average vs min
            $trend = Trend::calculateTrend(
                $urlHistory->last(),
                $urlHistory->values()->avg(),
                $urlHistory->values()->min(),
            );
            
            return [
                'store_id' => $url->store->getKey(),
                'store_name' => $url->store->name,
                'url_id' => $url->getKey(),
                'url' => $url->buy_url,
                'trend' => $trend,
                'price' => $urlHistory->last(),
                'history' => $urlHistory->toArray(),
                'last_scrape' => $lastScrapedPrice?->created_at?->toDateTimeString(),
                'locale' => $url->store->locale,
                'currency' => $url->store->currency,
            ];
        })
        ->sortBy('price')  // Lowest price first
        ->values();
}
```

### Trend Calculation

**File**: `app/Enums/Trend.php`

```php
public static function calculateTrend(float $current, float $avg, float $min): string
{
    if ($current === 0 || $avg === 0) {
        return self::None->value;
    }
    
    if ($current <= $min) {
        return self::Down->value;    // At historic low
    }
    
    if ($current < $avg) {
        return self::Down->value;    // Below average
    }
    
    if ($current > $avg) {
        return self::Up->value;      // Above average
    }
    
    return self::Stable->value;
}
```

## 9. Error Handling & Retry Logic

### Scraping Retries

**File**: `app/Services/ScrapeUrl.php:106-146`

- **Default attempts**: 3 (configurable via `max_attempts_to_scrape` setting)
- **Cache disabled** on retry attempts
- **Logs errors** with context (URL, store, HTML body, errors)

### Error Logging

**Database Logging**: `app/Services/ScrapeUrl.php:286-293`

```php
protected function errorLog(string $message, array $data = []): void
{
    if (!$this->logErrors) {
        return;
    }
    
    // Log to database with URL context
    $this->logger->error($message, $data);
}
```

### Notification System

**File**: `app/Jobs/UpdateProductPricesJob.php:41-43`

```php
if (!$successful) {
    $this->product->user?->notify(new ScrapeFailNotification($this->product));
}
```

Users receive notifications when:
- Scraping fails for a product
- Required fields (title, price) are missing
- Price drops below threshold

## 10. Advanced Features

### Affiliate Link Support

**File**: `app/Services/Helpers/AffiliateHelper.php`

PriceBuddy can inject affiliate codes into product URLs:

- Configured per store in `config/affiliates.php`
- Can be disabled globally with `AFFILIATE_ENABLED=false`

### Product Sources (Search Integration)

**File**: `app/Services/ProductSourceSearchService.php`

Beyond scraping individual URLs, PriceBuddy can search across:
- Deal aggregator sites (OzBargain, Slickdeals)
- Online store search pages (Amazon, eBay)

**Configuration**:
```php
[
    'search_url' => 'https://amazon.com/s?k=:search_term',
    'extraction_strategy' => [
        'list_container' => ['type' => 'selector', 'value' => '.s-result-item'],
        'product_title' => ['type' => 'selector', 'value' => 'h2 a span'],
        'product_url' => ['type' => 'selector', 'value' => 'h2 a|href'],
    ]
]
```

### API Access

**Package**: `rupadana/filament-api-service`

RESTful API endpoints for:
- Listing stores
- Creating/updating products
- Fetching price history
- Managing URLs

## 11. Key Design Decisions

### 1. Store-Based Configuration (Not Site-Wide)

**Why**: Different domains require different extraction strategies. By tying strategies to stores, PriceBuddy can handle multiple variations of the same retailer (e.g., amazon.com vs amazon.co.uk).

### 2. Multiple Extraction Methods

**Why**: Websites use diverse structures. Supporting CSS selectors, regex, XPath, and JSON Path ensures broad compatibility.

### 3. Auto-Discovery with Fallbacks

**Why**: Reduces manual configuration burden. Common patterns (OpenGraph tags, schema.org markup) cover 70%+ of e-commerce sites.

### 4. Price Caching

**Why**: Aggregating prices across stores is expensive. Caching denormalized data enables fast dashboard loads and trend visualization.

### 5. Retry Logic with Cache Busting

**Why**: Network issues and temporary site changes are common. Retrying without cache increases success rate.

### 6. Locale-Aware Currency Parsing

**Why**: International support requires handling different number formats (1,234.56 vs 1.234,56) and currencies.

### 7. Database Logging

**Why**: Debugging scraping issues requires context. Storing logs in the database (with URL, HTML, errors) enables troubleshooting without SSH access.

## 12. Performance Optimizations

### Chunked Processing

```php
Product::select('id')
    ->published()
    ->chunk(10, function (EloquentCollection $productIds) {
        UpdateAllPricesJob::dispatch($productIds->pluck('id')->toArray());
    });
```

**Why**: Prevents memory exhaustion on large product catalogs.

### Rate Limiting

```php
Sleep::for(AppSettings::new()->sleep_seconds_between_scrape)->seconds();
```

**Why**: Respects target sites, avoids IP blocks.

### Selective Field Loading

```php
Product::select('id')->published()->chunk(...)
```

**Why**: Reduces database I/O by loading only needed columns.

### Price History Window

```php
whereDate('prices.created_at', '>=', now()->subYear())
```

**Why**: Limits history queries to 1 year, keeps charts performant.

## 13. Security Considerations

### Input Validation

- URLs validated and sanitized before scraping
- Price values validated as numeric
- Store domains checked for duplicates

### User Isolation

- Products scoped to users: `$query->where('user_id', auth()->id())`
- Stores shared globally, but products are private

### Rate Limiting

- Configurable delays between scrapes
- Prevents abuse of external sites

### Error Disclosure

- Errors logged to database, not exposed to UI
- Scraping failures don't reveal internal details

## 14. Technology Stack

### Core Framework

- **Laravel 12** - PHP web application framework
- **Filament 3** - Admin panel and UI components

### Scraping Libraries

- **jez500/web-scraper-for-laravel** - Wrapper for HTTP and browser scraping
- **SeleniumBase** - Headless browser automation (via Docker)
- **Symfony DomCrawler** - DOM traversal and manipulation

### Currency Handling

- **moneyphp/money** - Currency parsing and formatting
- **symfony/intl** - Internationalization (locale/currency data)

### Job Queue

- **Laravel Queue** - Background job processing
- **Database** - Default queue driver

## 15. Deployment Architecture

### Docker Services

```yaml
services:
  app:
    # PHP 8.4 + Laravel
    # Runs web server and queue workers
    # Includes cron for scheduled tasks
    
  scraper:
    # SeleniumBase Scraper
    # Headless browser service
    # Accessed via HTTP API at :3000
    
  db:
    # MySQL/PostgreSQL
    # Stores products, prices, stores
    
  redis:
    # Cache and queue backend
```

### Cron Configuration

**File**: `docker/crontab`

```cron
0 0 * * * php /var/www/html/artisan schedule:run
```

**Scheduled Tasks**:
- Daily price updates
- Notification checks
- Cache cleanup

## 16. Extensibility Points

### Adding New Extraction Methods

To support a new extraction strategy:

1. Add method to `app/Services/ScrapeUrl.php:getMethodFromType()`
2. Implement extraction in web-scraper package
3. Add UI option in Store form

### Custom Scraper Services

To integrate a different scraping backend:

1. Implement `WebScraperInterface`
2. Register in `jez500/web-scraper-for-laravel`
3. Add to Store settings dropdown

### Additional Data Fields

To extract more fields (e.g., SKU, ratings):

1. Add to `$keys` array in `ScrapeUrl.php:43-48`
2. Add database column to `urls` or `products` table
3. Add to scrape strategy configuration
4. Update UI forms

## 17. Limitations & Trade-offs

### Limitations

1. **JavaScript-Heavy Sites**: Requires API scraper (slower)
2. **Anti-Bot Protection**: May require proxy rotation (not built-in)
3. **Dynamic Selectors**: Sites with obfuscated class names may break
4. **Currency Mixing**: Comparing prices in different currencies is unreliable

### Trade-offs

1. **Flexibility vs Performance**: Multiple strategies add overhead
2. **Auto-Discovery vs Accuracy**: Automatic store creation may produce suboptimal strategies
3. **Caching vs Freshness**: Cached scrapes reduce load but may show stale data
4. **Simplicity vs Power**: UI abstracts complexity but limits advanced configurations

## 18. Debugging & Troubleshooting

### Testing Store Configuration

**UI**: Store detail page → "Test" button
- Fetches test URL
- Shows extracted title, price, image
- Displays HTML and errors

### Logs

**Database Table**: `laravel_logs`
```sql
SELECT * FROM laravel_logs 
WHERE context LIKE '%url%' 
ORDER BY created_at DESC;
```

**Log Context**:
- URL being scraped
- Store ID
- Errors from scraper
- Raw HTML (truncated)

### Manual Price Fetch

**UI**: Product detail page → "Fetch Prices" button
- Triggers immediate scrape
- Shows results in real-time
- Useful for testing changes

## 19. Future Enhancements

Based on the architecture, potential improvements include:

1. **Machine Learning**: Auto-detect optimal selectors using ML
2. **Proxy Support**: Built-in proxy rotation for bot protection
3. **Screenshot Storage**: Store product images locally
4. **Price Prediction**: ML-based price drop forecasting
5. **Browser Extension**: Add products directly from browsing
6. **Webhook Notifications**: Real-time price alerts via webhooks

## 20. Conclusion

PriceBuddy's dynamic price extraction system is built on several key architectural principles:

1. **Store-Based Configuration** - Extraction rules tied to domains
2. **Multiple Extraction Strategies** - CSS, Regex, XPath, JSON Path
3. **Auto-Discovery** - Automatic store creation with fallback strategies
4. **Locale-Aware Parsing** - International currency support
5. **Flexible Scraping** - HTTP for static sites, browser for SPAs
6. **Performance Optimization** - Caching, chunking, selective loading
7. **Robust Error Handling** - Retries, logging, notifications

This architecture enables PriceBuddy to scrape prices from virtually any e-commerce site without code changes, making it a truly flexible price tracking solution.

---

**Document Version**: 1.0  
**Last Updated**: January 6, 2026  
**PriceBuddy Version**: Based on latest main branch
