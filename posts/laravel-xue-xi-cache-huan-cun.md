---
title: 'Laravel学习-cache缓存'
date: 2020-06-20 21:27:10
tags: [laravel,cache]
published: true
hideInList: false
feature: 
isTop: false
---
[Illuminate/Cache/Repository.php][1]
[Illuminate/Cache/CacheManager.php][2]
#### 配置
> Laravel 支持多种缓存系统, 并提供了统一的api接口，支持以下几种缓存模式，默认为file。默认的缓存配置文件在 config/cache.php
```PHP
 'stores' => [
       //APC缓存，APC是PHP的一个扩展，可选PHP缓存。它提供了缓存和优化PHP的中间代码的框架。 APC的缓存分两部分:系统缓存和用户数据缓存 https://www.php.net/manual/zh/book.apc.php
        'apc' => [
            'driver' => 'apc',
        ],
        //数组缓存驱动（array）往往仅仅用于测试，好处是不会持久化，只会在一次PHP脚本执行的生命周期内有效
        'array' => [
            'driver' => 'array',
            'serialize' => false,
        ],
        // 数据库缓存驱动（database）将缓存数据存储到数据库中，使用之前需要在数据库中新建一张表用于存放缓存项，该表表结构可定义如下
        'database' => [
            'driver' => 'database',
            'table' => 'cache',
            'connection' => null,
        ],
        // 文件缓存驱动（file）往往只用于本地开发测试，因为文件缓存将缓存存储到文件中，读取时从硬盘读取，性能自然不及基于内存的缓存系统如APC或Memcached以及Redis。
        'file' => [
            'driver' => 'file',
            'path' => storage_path('framework/cache/data'),
        ],
        // （memcached）缓存驱动基于Memcached，是基于内存的分布式缓存系统；读写性能优异，特别是高并发时和文件缓存比有明显优势；支持集群，并且是自动管理负载均衡。
        'memcached' => [
            'driver' => 'memcached',
            'persistent_id' => env('MEMCACHED_PERSISTENT_ID'),
            'sasl' => [
                env('MEMCACHED_USERNAME'),
                env('MEMCACHED_PASSWORD'),
            ],
            'options' => [
                // Memcached::OPT_CONNECT_TIMEOUT => 2000,
            ],
            'servers' => [
                [
                    'host' => env('MEMCACHED_HOST', '127.0.0.1'),
                    'port' => env('MEMCACHED_PORT', 11211),
                    'weight' => 100,
                ],
            ],
        ],
        //Redis 是一个开源的，高级键值对存储数据库。由于它包含 字符串，哈希，列表，集合，和 有序集合 这些数据类型
        'redis' => [
            'driver' => 'redis',
            'connection' => 'cache',
        ],
        // 云数据库 Amazon DynamoDB 是一项快速灵活的 NoSQL云数据库服务,它是完全托管的数据库
        'dynamodb' => [
            'driver' => 'dynamodb',
            'key' => env('AWS_ACCESS_KEY_ID'),
            'secret' => env('AWS_SECRET_ACCESS_KEY'),
            'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
            'table' => env('DYNAMODB_CACHE_TABLE', 'cache'),
            'endpoint' => env('DYNAMODB_ENDPOINT'),
        ],
```


#### 常用方法
##### has()
> has($key) 判断缓存中是否存在该键对应的缓存，missing()相反
```PHP
>>> use Illuminate\Support\Facades\Cache;
>>> Cache::put('key',1);
>>> Cache::has('key');
=> true
>>> Cache::has('key2');
=> false
```

##### get()
> get($key, $default = null) 获取缓存key对应的值，$default可以传默认值，也可以使用函数,
many(array $keys)，getMultiple($keys, $default = null)类似
```PHP
>>> Cache::get('key1');
=> null
>>> Cache::get('key1', 2);//默认值
=> 2
>>> Cache::get(['key','key1']);//可一次获取多个指定key
=> [
     "key" => 1,
     "key1" => null,
   ]
>>> Cache::get('key1',function(){return 3;});
=> 3
```

##### pull()
> pull($key, $default = null) 取回緩存key對應的值並移除
```PHP
>>> Cache::pull('key')
=> 1
>>> Cache::get('key')
=> null
```

##### put()&add()
> put($key, $value, $ttl = null);add($key, $value, $ttl = null) 存儲緩存 $ttl未过期时间，不传的forever
```PHP
>>> Cache::put('key',[1,2,3,4],100)
=> true
```

##### increment()&&decrement()
> 增加/减少緩存的值
```PHP
>>> Cache::add('key', 1)
=> true
>>> Cache::increment('key')
=> 2
>>> Cache::increment('key', 3)
=> 5
>>> Cache::decrement('key', 3)
=> 2
```

##### remember()
> remember($key, $ttl, Closure $callback) 獲取指定key的緩存值，如果不存在，則存儲并返回
```PHP
Cache::remember('key', $minutes, function(){ return 'value' });
Cache::rememberForever('key', function(){ return 'value' });
```

##### forget()

#### cache源码解读
Laravel 中常用 Cache Facade 来操作缓存, 对应的实际类是 Illuminate\Cache\CacheManager 缓存管理类(工厂).
>Cache::xxx()

我们通过 CacheManager 类获取持有不同存储驱动的 Illuminate\Cache\Repository 类
>CacheManager::store($name = null)

Repository 仓库类代理了实现存储驱动接口 Illuminate\Contracts\Cache\Store 的类实例.

```PHP
//在配置文件 config\app.php 中定义了 Cache 服务提供者
'providers' => [
        // ......
        Illuminate\Cache\CacheServiceProvider::class,
        // ......
    ],
    
//Illuminate\Cache\CacheServiceProvider 文件
    public function register()
    {
        $this->app->singleton('cache', function ($app) {
            return new CacheManager($app);//实例化CacheManager
        });
        $this->app->singleton('cache.store', function ($app) {
            return $app['cache']->driver();
        });
        $this->app->singleton('memcached.connector', function () {
            return new MemcachedConnector;
        });
    }
```
##### CacheManager
>CacheManager 实现了 Illuminate\Contracts\Cache\Factory 接口, 实现了一个简单工厂, 传入存储驱动名, 返回对应的驱动实例.
```PHP
// Cache\Factory 工厂接口
namespace Illuminate\Contracts\Cache;
interface Factory
{
    /**
     * Get a cache store instance by name.
     *
     * @param  string|null  $name
     * @return \Illuminate\Contracts\Cache\Repository
     */
    public function store($name = null);
}

//CacheManager实现的简单工厂接口方法
use Aws\DynamoDb\DynamoDbClient;
use Closure;
use Illuminate\Contracts\Cache\Factory as FactoryContract;
use Illuminate\Contracts\Cache\Store;
use Illuminate\Contracts\Events\Dispatcher as DispatcherContract;
use Illuminate\Support\Arr;
use InvalidArgumentException;

/**
 * @mixin \Illuminate\Contracts\Cache\Repository
 */
class CacheManager implements FactoryContract
{
    /**
     * Get a cache store instance by name, wrapped in a repository.
     * 实现接口方法
     * @param  string|null  $name
     * @return \Illuminate\Contracts\Cache\Repository
     */
    public function store($name = null)
    {
        $name = $name ?: $this->getDefaultDriver();

        return $this->stores[$name] = $this->get($name);
    }
    ...
    
    /**
     * Resolve the given store.
     * 解析过程
     * 自定义驱动: 查看是否有通过 CacheManager::extend(...)自定义的驱动
     * Laravel提供的驱动: 查看是否存在 CacheManager::createXxxDriver(...)方法
     * 这些方法都是实现了 Illuminate\Contracts\Cache\Repository 接口 
     *
     * @param  string  $name
     * @return \Illuminate\Contracts\Cache\Repository
     *
     * @throws \InvalidArgumentException
     */
    protected function resolve($name)
    {
        $config = $this->getConfig($name);

        if (is_null($config)) {
            throw new InvalidArgumentException("Cache store [{$name}] is not defined.");
        }

        if (isset($this->customCreators[$config['driver']])) {
            return $this->callCustomCreator($config);
        } else {
            $driverMethod = 'create'.ucfirst($config['driver']).'Driver';

            if (method_exists($this, $driverMethod)) {
                return $this->{$driverMethod}($config);
            } else {
                throw new InvalidArgumentException("Driver [{$config['driver']}] is not supported.");
            }
        }
    }
    

    /**
     * Register a custom driver creator Closure.
     * 自定义一个驱动程序，$driver对应 config/cache.php 配置文件中的 driver 选项。 第二个参数是返回 Illuminate\Cache\Repository 实例的闭包,相当于一个服务容器的实例
     * @param  string  $driver
     * @param  \Closure  $callback
     * @return $this
     */
    public function extend($driver, Closure $callback)
    {
        $this->customCreators[$driver] = $callback->bindTo($this, $this);
        return $this;
    }

    /**
     * Dynamically call the default driver instance.
     * 魔术方法 以便快速调用默认缓存驱动
     *
     * @param  string  $method
     * @param  array  $parameters
     * @return mixed
     */
    public function __call($method, $parameters)
    {
        return $this->store()->$method(...$parameters);
    }
```
##### Repository
>Illuminate\Contracts\Cache\Repository 接口，Repository 是一个符合 PSR-16: Common Interface for Caching Libraries 规范的缓存仓库类, 其在Laravel相应的实现类: Illuminate\Cache\Repository,其实现了代理模式, 具体的实现是交由 Illuminate\Contracts\Cache\Store 来处理（具体的store实现）, Repository 主要作用是
提供一些便捷操作 读取等操作
Event 事件触发, 包括缓存命中/未命中、写入/删除键值

##### Store
>Illuminate\Contracts\Cache 缓存驱动是实际处理缓存如何写入/读取/删除的类，具体的实现类有：

    - ApcStore 
    - ArrayStore 
    - NullStore 
    - DatabaseStore 
    - FileStore 
    - MemcachedStore
    - RedisStore 
    - DynamoDbStore
```PHP
namespace Illuminate\Contracts\Cache;
interface Store
{
    public function get($key);
    public function many(array $keys);
    public function put($key, $value, $minutes);
    public function putMany(array $values, $minutes);
    public function increment($key, $value = 1);
    public function decrement($key, $value = 1);
    public function forever($key, $value);
    public function forget($key);
    public function flush();
    public function getPrefix();
}
```
  [1]: https://github.com/xiaoxie110/laravel/blob/master/vendor/laravel/framework/src/Illuminate/Cache/Repository.php
  [2]: https://github.com/xiaoxie110/laravel/blob/master/vendor/laravel/framework/src/Illuminate/Cache/CacheManager.php