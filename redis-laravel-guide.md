# Redis Mastery Guide for Laravel Developers

## Table of Contents
1. [Basic Setup](#basic-setup)
2. [Caching Strategies](#caching-strategies)
3. [Queues and Job Processing](#queues-and-job-processing)
4. [Pub/Sub and Event Broadcasting](#pubsub-and-event-broadcasting)
5. [Rate Limiting](#rate-limiting)
6. [Session Management](#session-management)
7. [Microservices Communication](#microservices-communication)
8. [Real-time Features](#real-time-features)
9. [Monitoring and Optimization](#monitoring-and-optimization)

## Basic Setup

### Installation and Configuration

```bash
# Install Redis Server
sudo apt update
sudo apt install redis-server

# Install PHP Redis Extension
sudo apt install php-redis

# Verify Redis is running
redis-cli ping
```

### Laravel Configuration (config/database.php)
```php
'redis' => [
    'client' => env('REDIS_CLIENT', 'phpredis'),
    'default' => [
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'password' => env('REDIS_PASSWORD', null),
        'port' => env('REDIS_PORT', 6379),
        'database' => env('REDIS_DB', 0),
    ],
    'cache' => [
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'password' => env('REDIS_PASSWORD', null),
        'port' => env('REDIS_PORT', 6379),
        'database' => env('REDIS_CACHE_DB', 1),
    ],
]
```

## Caching Strategies

### Basic Caching
```php
// Simple key-value caching
Cache::store('redis')->put('key', 'value', $seconds);
Cache::store('redis')->get('key');

// Remember pattern
$value = Cache::store('redis')->remember('key', $seconds, function () {
    return DB::table('heavy_query')->get();
});

// Tags for grouped caching
Cache::store('redis')->tags(['users', 'profiles'])->put('user:1', $user, 3600);
```

### Advanced Caching Patterns

```php
// Atomic Cache Operations
class ProductService
{
    public function incrementViews($productId)
    {
        return Cache::store('redis')->increment("product:{$productId}:views");
    }

    // Cache-aside Pattern
    public function getProduct($id)
    {
        $cacheKey = "product:{$id}";
        
        return Cache::store('redis')->remember($cacheKey, 3600, function () use ($id) {
            return Product::with('category', 'tags')->find($id);
        });
    }

    // Write-through Cache
    public function updateProduct($id, array $data)
    {
        $product = Product::find($id);
        $product->update($data);
        
        Cache::store('redis')->put("product:{$id}", $product, 3600);
        
        return $product;
    }
}
```

### Cache Invalidation Strategies
```php
// Time-based Invalidation
Cache::store('redis')->put('key', 'value', now()->addHours(24));

// Event-based Invalidation
class ProductObserver
{
    public function updated(Product $product)
    {
        Cache::store('redis')->tags(['products'])->flush();
    }
}

// Selective Invalidation
Cache::store('redis')->tags(["product:{$id}"])->flush();
```

## Queues and Job Processing

### Queue Configuration (config/queue.php)
```php
'redis' => [
    'driver' => 'redis',
    'connection' => 'default',
    'queue' => env('REDIS_QUEUE', 'default'),
    'retry_after' => 90,
    'block_for' => null,
],
```

### Queue Implementation
```php
// Dispatch a job
ProcessOrder::dispatch($order)
    ->onQueue('orders')
    ->delay(now()->addMinutes(5));

// Job class with Redis specific features
class ProcessOrder implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
    
    public $tries = 3;
    public $backoff = [60, 180, 360];
    
    public function handle()
    {
        Redis::throttle('process-orders')
            ->allow(10)
            ->every(5)
            ->then(function () {
                // Process order
            }, function () {
                return $this->release(60);
            });
    }
}
```

## Pub/Sub and Event Broadcasting

### Laravel Echo Server Configuration
```javascript
{
    "authHost": "http://localhost",
    "authEndpoint": "/broadcasting/auth",
    "clients": [],
    "database": "redis",
    "databaseConfig": {
        "redis": {
            "port": "6379",
            "host": "127.0.0.1"
        }
    },
    "devMode": true,
    "host": null,
    "port": "6001",
    "protocol": "http",
    "socketio": {},
    "sslCertPath": "",
    "sslKeyPath": ""
}
```

### Event Broadcasting
```php
// Broadcasting Event
class OrderShipped implements ShouldBroadcast
{
    public $order;
    
    public function broadcastOn()
    {
        return new PrivateChannel('orders.' . $this->order->id);
    }
    
    public function broadcastWith()
    {
        return [
            'id' => $this->order->id,
            'status' => 'shipped'
        ];
    }
}

// Frontend Listening
Echo.private(`orders.${orderId}`)
    .listen('OrderShipped', (e) => {
        console.log(e.order);
    });
```

## Microservices Communication

### Event-Driven Architecture
```php
// Event Publisher Service
class OrderService
{
    public function createOrder(array $data)
    {
        $order = Order::create($data);
        
        Redis::publish('orders', json_encode([
            'event' => 'OrderCreated',
            'data' => $order->toArray()
        ]));
        
        return $order;
    }
}

// Event Subscriber Service
class InventoryService
{
    public function subscribeToOrders()
    {
        Redis::subscribe(['orders'], function ($message) {
            $event = json_decode($message, true);
            
            if ($event['event'] === 'OrderCreated') {
                $this->updateInventory($event['data']);
            }
        });
    }
}
```

### Distributed Rate Limiting
```php
class ApiGatewayService
{
    public function rateLimit($key, $maxAttempts, $decayMinutes)
    {
        return Redis::throttle($key)
            ->allow($maxAttempts)
            ->every($decayMinutes * 60)
            ->then(function () {
                return response()->json(['status' => 'allowed']);
            }, function () {
                return response()->json(['status' => 'throttled'], 429);
            });
    }
}
```

## Real-time Features

### Real-time Analytics
```php
class AnalyticsService
{
    public function trackPageView($page)
    {
        Redis::hincrby("pageviews:daily:" . now()->format('Y-m-d'), $page, 1);
        
        // Expire after 30 days
        Redis::expire("pageviews:daily:" . now()->format('Y-m-d'), 60 * 60 * 24 * 30);
    }
    
    public function getPopularPages($date)
    {
        return Redis::hgetall("pageviews:daily:{$date}");
    }
}
```

### Real-time Chat Implementation
```php
class ChatService
{
    public function storeMessage($roomId, $message)
    {
        // Store in sorted set with timestamp as score
        Redis::zadd(
            "chat:room:{$roomId}",
            now()->timestamp,
            json_encode($message)
        );
        
        // Broadcast to room
        Redis::publish("chat:room:{$roomId}", json_encode($message));
        
        // Trim to last 100 messages
        Redis::zremrangebyrank("chat:room:{$roomId}", 0, -101);
    }
    
    public function getRecentMessages($roomId, $limit = 50)
    {
        return Redis::zrevrange("chat:room:{$roomId}", 0, $limit - 1);
    }
}
```

## Monitoring and Optimization

### Redis Info Command Implementation
```php
class RedisMonitor
{
    public function getServerStats()
    {
        return Redis::info();
    }
    
    public function getMemoryUsage()
    {
        return Redis::info('memory');
    }
    
    public function getSlowLogs()
    {
        return Redis::slowlog('get', 10);
    }
}
```

### Memory Optimization Patterns
```php
class CacheOptimizer
{
    public function optimizeKeySize()
    {
        // Use shorter key names
        Cache::store('redis')->put('u:1', $userData, 3600); // Instead of 'user:1'
        
        // Use integers instead of strings where possible
        Cache::store('redis')->put(1, $userData, 3600); // Instead of 'user:1'
    }
    
    public function implementExpiration()
    {
        // Set TTL for all cache entries
        Cache::store('redis')->put('key', 'value', now()->addDay());
        
        // Use automatic expiration for temporary data
        Redis::setex('temporary_key', 3600, 'value');
    }
}
```

### Performance Monitoring
```php
class PerformanceMonitor
{
    public function trackCommandExecutionTime()
    {
        $start = microtime(true);
        
        Redis::get('key');
        
        $executionTime = microtime(true) - $start;
        Log::info("Redis command execution time: {$executionTime}s");
    }
    
    public function monitorConnections()
    {
        return [
            'connected_clients' => Redis::info('clients')['connected_clients'],
            'blocked_clients' => Redis::info('clients')['blocked_clients']
        ];
    }
}
```

## Best Practices and Tips

1. **Key Naming Conventions**
```php
// Use consistent separators
"user:1:profile"
"user:1:preferences"
"order:5:items"

// Use prefixes for different applications
"app1:users"
"app2:users"
```

2. **Error Handling**
```php
try {
    Redis::set('key', 'value');
} catch (RedisException $e) {
    Log::error('Redis error: ' . $e->getMessage());
    // Fallback to database or alternative cache
}
```

3. **Transaction Handling**
```php
Redis::transaction(function ($redis) {
    $redis->set('key1', 'value1');
    $redis->set('key2', 'value2');
});
```

4. **Lua Scripting for Atomic Operations**
```php
$script = <<<'LUA'
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
LUA;

Redis::eval($script, 1, 'lock_key', 'lock_value');
```

Remember to always:
- Use appropriate serialization for complex data structures
- Implement proper error handling and fallback mechanisms
- Monitor Redis memory usage and implement proper expiration policies
- Use Redis transactions for atomic operations
- Implement proper security measures including authentication and encryption
- Regular backup of Redis data
- Monitor Redis performance and optimize as needed

The configurations and code examples provided should be adjusted based on your specific needs and requirements.
