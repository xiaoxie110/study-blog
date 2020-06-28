---
title: 'Laravel学习-Facade(门面)'
date: 2020-06-28 22:27:08
tags: [laravel,facade]
published: false
hideInList: false
feature: 
isTop: false
---

#### 基本定义
>Facades 为应用的 IoC 服务容器 的类提供了一个静态的接口。Laravel 里面自带了一些 Facades，如Cache,Route等。Laravel的门面作为服务容器中底层类的“静态代理”，相比于传统静态方法，在维护时能够提供更加易于测试、更加灵活、简明优雅的语法。通常在项目开发中，通过 ServiceProvider 注入容器的服务类构建一个门面，以便可以非常方便地调用这些类接口

#### 工作原理
>在 Laravel 应用中，门面就是一个为容器中对象提供访问方式的类。该机制原理由 Facade 类实现。Laravel 自带的门面，以及我们创建的自定义门面，都会继承自 Illuminate\Support\Facades\Facade 基类。
```PHP
// cache的门面
class Cache extends Facade {
    /**
     * Get the registered name of the component.
     *
     * @return string
     */
    protected static function getFacadeAccessor() { return 'cache'; }
}

// 具体调用
Cache::get($key) 实际上执行  $app->make('cache')->get('key');
```
>门面类只需要实现一个方法：getFacadeAccessor。正是 getFacadeAccessor 方法定义了从容器中解析什么。然后 Facade 基类使用魔术方法 __callStatic() 代理门面上静态方法的调用，并将其交给通过 getFacadeAccessor,方法定义的从容器中解析出来的服务类来执行。

#### 源码解读
>通过 getFacadeAccessor 标识门面对应的服务类，通过核心魔术方法__callStatic($method, $args) 来动态绑定门面上的静态方法调用，将其绑定到门面对应的服务对象上来调用
```PHP
   /**
     * Handle dynamic, static calls to the object.
     * 动态绑定，将门面的静态方法调用绑定到门面对应的服务对象实例来执行
     * @param  string  $method
     * @param  array   $args
     * @return mixed
     *
     * @throws \RuntimeException
     */
    public static function __callStatic($method, $args)
    {
        $instance = static::getFacadeRoot();//解析出实例
        if (! $instance) {
            throw new RuntimeException('A facade root has not been set.');
        }
        return $instance->$method(...$args);//調出实例對應方法
    }
    
     /**
     * Get the root object behind the facade.
     * 返回当前门面对应服务对象的实例
     * @return mixed
     */
    public static function getFacadeRoot()
    {
        return static::resolveFacadeInstance(static::getFacadeAccessor());
    }
    
    /**
     * Get the registered name of the component.
     * 返回此门面具体对应的服务类名，
    * 一般为绑定在 Application 中的类对应的 $abstract，
     * 或者带有完整命名空间的类名。由子类实现此方法
     * @return string
     *
     * @throws \RuntimeException
     */
    protected static function getFacadeAccessor()
    {
        throw new RuntimeException('Facade does not implement getFacadeAccessor method.');
    }
    
    /**
     * Resolve the facade root instance from the container.
     * 獲取或创建并返回 $name 对应的对象实例
     *
     * @param  object|string  $name
     * @return mixed
     */
    protected static function resolveFacadeInstance($name)
    {
        if (is_object($name)) {
            return $name;
        }
        if (isset(static::$resolvedInstance[$name])) {
            return static::$resolvedInstance[$name];
        }
        //具体的创建操作由 Application 类对象的来进行,所以 $name 需要在 Application 对象中进行过绑定,具体可见app.php中aliases
        if (static::$app) {
            return static::$resolvedInstance[$name] = static::$app[$name];
        }
    }
   
    
```

#### 门面集合


| 门面(Facade)        | 类(class)   |  服务容器绑定  |
| --------   | :----- | :----:  |
|App | \Illuminate\Contracts\Foundation\Application | app |
|Artisan | Illuminate\Contracts\Console\Kernel | artisan |
|Auth | \Illuminate\Auth\AuthManager | auth |
|Blade | \Illuminate\View\Compilers\BladeCompiler | blade.compiler |
|Broadcast | \Illuminate\Contracts\Broadcasting\Factory | 
|Bus | \Illuminate\Contracts\Bus\Dispatcher |
|Cache | \Illuminate\Cache\CacheManager | cache |
|Config | \Illuminate\Config\Repository | config |
|Cookie | \Illuminate\Cookie\CookieJar | cookie |
|Crypt | \Illuminate\Encryption\Encrypter | encrypter |
|DB | \Illuminate\Database\DatabaseManager | db |
|Event | \Illuminate\Events\Dispatcher | events |
|File | \Illuminate\Filesystem\Filesystem | files |
|Gate | \Illuminate\Contracts\Auth\Access\Gate |
|Hash | \Illuminate\Hashing\HashManager | hash |
|Http | \Illuminate\Http\Client\Factory |
|Lang | \Illuminate\Translation\Translator | translator |
|Log | \Illuminate\Log\Logger | log |
|Mail | \Illuminate\Mail\Mailer | mail.manager | 
|Notification | \Illuminate\Notifications\ChannelManager |
|Password | \Illuminate\Auth\Passwords\PasswordBroker | auth.password |
|Queue | \Illuminate\Queue\QueueManager | queue |
|Redirect | \Illuminate\Routing\Redirector | redirect |
|Redis | \Illuminate\Redis\RedisManager | redis |
|Request | \Illuminate\Http\Request | request |
|Response | \Illuminate\Contracts\Routing\ResponseFactory |
|Route | \Illuminate\Routing\Router | router
|Schema | \Illuminate\Database\Schema\Builder |
|Session | \Illuminate\Session\SessionManager | session |
|Storage | \Illuminate\Filesystem\FilesystemManager | filesystem |
|URL | \Illuminate\Routing\UrlGenerator | url |
|Validator | \Illuminate\Validation\Factory | validator |
|View | \Illuminate\View\Factory | view

>app.aliases 配置文件，是门面类的别名与门面类的对应关系可以用顶层命名空间调用门面,在框架启动过程中，会调用 Illuminate\Foundation\Bootstrap\RegisterFacades 类的 bootstrap 方法
```PHP
class RegisterFacades
{
    /**
     * Bootstrap the given application.
     *  启动框架的门面服务
     * @param  \Illuminate\Contracts\Foundation\Application  $app
     * @return void
     */
    public function bootstrap(Application $app)
    {
        Facade::clearResolvedInstances();

        Facade::setFacadeApplication($app);

        AliasLoader::getInstance(array_merge(
            $app->make('config')->get('app.aliases', []),
            $app->make(PackageManifest::class)->aliases()
        ))->register();
    }
}
```
>  AliasLoader 类注册 app.aliases 配置下的别名
```PHP
  /**
     * Get or create the singleton alias loader instance.
     * 返回或创建针对 $aliases 的 AliasLoader 类
     *
     * @param  array  $aliases
     * @return \Illuminate\Foundation\AliasLoader
     */
    public static function getInstance(array $aliases = [])
    {
        if (is_null(static::$instance)) {
            return static::$instance = new static($aliases);
        }

        $aliases = array_merge(static::$instance->getAliases(), $aliases);

        static::$instance->setAliases($aliases);

        return static::$instance;
    }
```
> 调用了 AliasLoader->register 方法
```PHP
    /**
     * Register the loader on the auto-loader stack.
     * 注册当前的加载器到程序的自动加载栈
     *
     * @return void
     */
    public function register()
    {
        if (! $this->registered) {
            $this->prependToLoaderStack();

            $this->registered = true;
        }
    }
    
       /**
     * Prepend the load method to the auto-loader stack.
     * 注册当前对象的 load 方法到程序的自动加载栈
     *
     * @return void
     */
    protected function prependToLoaderStack()
    {
        spl_autoload_register([$this, 'load'], true, true);
    }
    
    /**
     * Load a class alias if it is registered.
     *
     * @param  string  $alias
     * @return bool|null
     */
    public function load($alias)
    {
        if (static::$facadeNamespace && strpos($alias, static::$facadeNamespace) === 0) {
            $this->loadFacade($alias);
            return true;
        }

        if (isset($this->aliases[$alias])) {//实例化一个对象
            return class_alias($this->aliases[$alias], $alias);//核心函数
        }
    }
    
    // class_alias ( string $original , string $alias [, bool $autoload = TRUE ] ) : bool;基于用户定义的类 original 创建别名 alias
```