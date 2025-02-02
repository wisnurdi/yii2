Caching Data
============

Caching data yang dimaksud di sini adalah menyimpan beberapa variabel PHP dalam cache dan memanggilnya lagi kemudian dari cache. Pemahaman tentang topik caching ini tentu akan menjadi dasar untuk penggunaan fitur caching tingkat lanjut, seperti [caching kueri](#query-caching) dan [caching laman/page](caching-page.md).

Sintaks berikut pola penggunaan yang khusus untuk caching, di mana `$cache` mengacu kepada sebuah [komponen cache](#cache-components):

```php
// memanggil $data dari cache
$data = $cache->get($key);

if ($data === false) {
    // $data tidak ada di cache, menghitung dari awal 
    $data = $this->calculateSomething();

    // menyimpan $data di cache agar bisa dipanggil lain kali
    $cache->set($key, $data);
}

// variabel $data sudah ada di sini 
```

Sejak versi 2.0.11, [komponen cache](#cache-components) menyediakan metode [[yii\caching\Cache::getOrSet()|getOrSet()]] yang menyederhanakan sintaks untuk mendapatkan (get) data, melakukan kalkulasi, dan menyimpannya. Sintaks berikut sama persis hasilnya dengan sintaks sebelumnya:

```php
$data = $cache->getOrSet($key, function () {
    return $this->calculateSomething();
});
```

Ketika pada cache ada data yang terasosiasi dengan variabel `$key`, nilai yang sudah disimpan di cache akan diberikan. Sebaliknya fungsi anonim akan diekseksusi untuk mengkalkulasi nilai yang akan disimpan di cache dan diberikan ketika dipanggil. 

Jika fungsi anonim tadi memerlukan data lain dari skup luar, Anda bisa memakai peryataan `use`. Berikut contoh sintaksnya:

```php
$user_id = 42;
$data = $cache->getOrSet($key, function () use ($user_id) {
    return $this->calculateSomething($user_id);
});
```

> Note: [[yii\caching\Cache::getOrSet()|getOrSet()]] method supports duration and dependencies as well. 
  See [Cache Expiration](#cache-expiration) and [Cache Dependencies](#cache-dependencies) to know more.
  

## Komponen Cache <span id="cache-components"></span>

Caching data bergantung pada *komponen cache* yang merupakan representasi penyimpanan cache di komputer, seperti memori, file, dan database.

Komponen cache biasanya terdaftar sebagai [komponen aplikasi ](structure-application-components.md) sehingga bisa diakses dan dikonfigurasi secara global. Sintaks berikut menunjukkan bagaimana untuk melakukan konfigurasi komponen aplikasi `cache` dengan menggunakan [memcached](http://memcached.org/) melalui dua server cache:

```php
'components' => [
    'cache' => [
        'class' => 'yii\caching\MemCache',
        'servers' => [
            [
                'host' => 'server1',
                'port' => 11211,
                'weight' => 100,
            ],
            [
                'host' => 'server2',
                'port' => 11211,
                'weight' => 50,
            ],
        ],
    ],
],
```

Selanjutnya Anda bisa mengakses komponen cache di atas menggunakan kode  `Yii::$app->cache`.

Karena semua komponen cache mendukung set API yang sama, Anda bisa melakukan swap (menukar) komponen cache satu dengan yang lain melalui cara rekonfigurasi pada konfigurasi aplikasi tanpa harus memodifikasi kode yang menggunakan cache. 

Sebagai contoh, Anda dapat menguhab konfigurasi di atas menggunakan [[yii\caching\ApcCache|APC cache]]:

```php
'components' => [
    'cache' => [
        'class' => 'yii\caching\ApcCache',
    ],
],
```

> Tip: Anda dapat mendaftarkan banyak komponen aplikasi cache. Nama komponen `cache` dipakai secara default oleh banyak kelas yang cache-dependent atau kelas yang bergantung pada cache (seperti [[yii\web\UrlManager]]).

### Supported Cache Storage <span id="supported-cache-storage"></span>

Yii supports a wide range of cache storage. The following is a summary:

* [[yii\caching\ApcCache]]: uses PHP [APC](https://www.php.net/manual/en/book.apcu.php) extension. This option can be
  considered as the fastest one when dealing with cache for a centralized thick application (e.g. one
  server, no dedicated load balancers, etc.).
* [[yii\caching\DbCache]]: uses a database table to store cached data. To use this cache, you must
  create a table as specified in [[yii\caching\DbCache::cacheTable]].
* [[yii\caching\ArrayCache]]: provides caching for the current request only by storing the values in an array.
  For enhanced performance of ArrayCache, you can disable serialization of the stored data by setting
  [[yii\caching\ArrayCache::$serializer]] to `false`.
* [[yii\caching\DummyCache]]: serves as a cache placeholder which does no real caching.
  The purpose of this component is to simplify the code that needs to check the availability of cache.
  For example, during development or if the server doesn't have actual cache support, you may configure
  a cache component to use this cache. When an actual cache support is enabled, you can switch to use
  the corresponding cache component. In both cases, you may use the same code
  `Yii::$app->cache->get($key)` to attempt retrieving data from the cache without worrying that
  `Yii::$app->cache` might be `null`.
* [[yii\caching\FileCache]]: uses standard files to store cached data. This is particularly suitable
  to cache large chunk of data, such as page content.
* [[yii\caching\MemCache]]: uses PHP [memcache](https://www.php.net/manual/en/book.memcache.php)
  and [memcached](https://www.php.net/manual/en/book.memcached.php) extensions. This option can be considered as
  the fastest one when dealing with cache in a distributed applications (e.g. with several servers, load
  balancers, etc.)
* [[yii\redis\Cache]]: implements a cache component based on [Redis](http://redis.io/) key-value store
  (redis version 2.6.12 or higher is required).
* [[yii\caching\WinCache]]: uses PHP [WinCache](http://iis.net/downloads/microsoft/wincache-extension)
  ([see also](https://www.php.net/manual/en/book.wincache.php)) extension.
* [[yii\caching\XCache]] _(deprecated)_: uses PHP [XCache](https://en.wikipedia.org/wiki/List_of_PHP_accelerators#XCache) extension.
* [[yii\caching\ZendDataCache]] _(deprecated)_: uses
  [Zend Data Cache](http://files.zend.com/help/Zend-Server-6/zend-server.htm#data_cache_component.htm)
  as the underlying caching medium.


> Tip: You may use different cache storage in the same application. A common strategy is to use memory-based
  cache storage to store data that is small but constantly used (e.g. statistical data), and use file-based
  or database-based cache storage to store data that is big and less frequently used (e.g. page content).


## Cache APIs <span id="cache-apis"></span>

All cache components have the same base class [[yii\caching\Cache]] and thus support the following APIs:

* [[yii\caching\Cache::get()|get()]]: retrieves a data item from cache with a specified key. A `false`
  value will be returned if the data item is not found in the cache or is expired/invalidated.
* [[yii\caching\Cache::set()|set()]]: stores a data item identified by a key in cache.
* [[yii\caching\Cache::add()|add()]]: stores a data item identified by a key in cache if the key is not found in the cache.
* [[yii\caching\Cache::getOrSet()|getOrSet()]]: retrieves a data item from cache with a specified key or executes passed
  callback, stores return of the callback in a cache by a key and returns that data.  
* [[yii\caching\Cache::multiGet()|multiGet()]]: retrieves multiple data items from cache with the specified keys.
* [[yii\caching\Cache::multiSet()|multiSet()]]: stores multiple data items in cache. Each item is identified by a key.
* [[yii\caching\Cache::multiAdd()|multiAdd()]]: stores multiple data items in cache. Each item is identified by a key.
  If a key already exists in the cache, the data item will be skipped.
* [[yii\caching\Cache::exists()|exists()]]: returns a value indicating whether the specified key is found in the cache.
* [[yii\caching\Cache::delete()|delete()]]: removes a data item identified by a key from the cache.
* [[yii\caching\Cache::flush()|flush()]]: removes all data items from the cache.

> Note: Do not cache a `false` boolean value directly because the [[yii\caching\Cache::get()|get()]] method uses
`false` return value to indicate the data item is not found in the cache. You may put `false` in an array and cache
this array instead to avoid this problem. 

Some cache storage, such as MemCache, APC, support retrieving multiple cached values in a batch mode,
which may reduce the overhead involved in retrieving cached data. The APIs [[yii\caching\Cache::multiGet()|multiGet()]]
and [[yii\caching\Cache::multiAdd()|multiAdd()]] are provided to exploit this feature. In case the underlying cache storage
does not support this feature, it will be simulated.

Because [[yii\caching\Cache]] implements `ArrayAccess`, a cache component can be used like an array. The following
are some examples:

```php
$cache['var1'] = $value1;  // equivalent to: $cache->set('var1', $value1);
$value2 = $cache['var2'];  // equivalent to: $value2 = $cache->get('var2');
```


### Cache Keys <span id="cache-keys"></span>

Each data item stored in cache is uniquely identified by a key. When you store a data item in cache,
you have to specify a key for it. Later when you retrieve the data item from cache, you should provide
the corresponding key.

You may use a string or an arbitrary value as a cache key. When a key is not a string, it will be automatically
serialized into a string.

A common strategy of defining a cache key is to include all determining factors in terms of an array.
For example, [[yii\db\Schema]] uses the following key to cache schema information about a database table:

```php
[
    __CLASS__,              // schema class name
    $this->db->dsn,         // DB connection data source name
    $this->db->username,    // DB connection login user
    $name,                  // table name
];
```

As you can see, the key includes all necessary information needed to uniquely specify a database table.

> Note: Values stored in cache via [[yii\caching\Cache::multiSet()|multiSet()]] or [[yii\caching\Cache::multiAdd()|multiAdd()]] can
have only string or integer keys. If you need to set more complex key store the value separately via 
[[yii\caching\Cache::set()|set()]] or [[yii\caching\Cache::add()|add()]].

When the same cache storage is used by different applications, you should specify a unique cache key prefix
for each application to avoid conflicts of cache keys. This can be done by configuring the [[yii\caching\Cache::keyPrefix]]
property. For example, in the application configuration you can write the following code:

```php
'components' => [
    'cache' => [
        'class' => 'yii\caching\ApcCache',
        'keyPrefix' => 'myapp',       // a unique cache key prefix
    ],
],
```

To ensure interoperability, only alphanumeric characters should be used.


### Cache Expiration <span id="cache-expiration"></span>

A data item stored in a cache will remain there forever unless it is removed because of some caching policy
enforcement (e.g. caching space is full and the oldest data are removed). To change this behavior, you can provide
an expiration parameter when calling [[yii\caching\Cache::set()|set()]] to store a data item. The parameter
indicates for how many seconds the data item can remain valid in the cache. When you call
[[yii\caching\Cache::get()|get()]] to retrieve the data item, if it has passed the expiration time, the method
will return `false`, indicating the data item is not found in the cache. For example,

```php
// keep the data in cache for at most 45 seconds
$cache->set($key, $data, 45);

sleep(50);

$data = $cache->get($key);
if ($data === false) {
    // $data is expired or is not found in the cache
}
```

Since 2.0.11 you may set [[yii\caching\Cache::$defaultDuration|defaultDuration]] value in your cache component configuration if you prefer a custom cache duration
over the default unlimited duration.
This will allow you not to pass custom `duration` parameter to [[yii\caching\Cache::set()|set()]] each time.


### Cache Dependencies <span id="cache-dependencies"></span>

Besides expiration setting, cached data item may also be invalidated by changes of the so-called *cache dependencies*.
For example, [[yii\caching\FileDependency]] represents the dependency of a file's modification time.
When this dependency changes, it means the corresponding file is modified. As a result, any outdated
file content found in the cache should be invalidated and the [[yii\caching\Cache::get()|get()]] call
should return `false`.

Cache dependencies are represented as objects of [[yii\caching\Dependency]] descendant classes. When you call
[[yii\caching\Cache::set()|set()]] to store a data item in the cache, you can pass along an associated cache
dependency object. For example,

```php
// Create a dependency on the modification time of file example.txt.
$dependency = new \yii\caching\FileDependency(['fileName' => 'example.txt']);

// The data will expire in 30 seconds.
// It may also be invalidated earlier if example.txt is modified.
$cache->set($key, $data, 30, $dependency);

// The cache will check if the data has expired.
// It will also check if the associated dependency was changed.
// It will return false if any of these conditions are met.
$data = $cache->get($key);
```

Below is a summary of the available cache dependencies:

- [[yii\caching\ChainedDependency]]: the dependency is changed if any of the dependencies on the chain is changed.
- [[yii\caching\DbDependency]]: the dependency is changed if the query result of the specified SQL statement is changed.
- [[yii\caching\ExpressionDependency]]: the dependency is changed if the result of the specified PHP expression is changed.
- [[yii\caching\FileDependency]]: the dependency is changed if the file's last modification time is changed.
- [[yii\caching\TagDependency]]: associates a cached data item with one or multiple tags. You may invalidate
  the cached data items with the specified tag(s) by calling [[yii\caching\TagDependency::invalidate()]].

> Note: Avoid using [[yii\caching\Cache::exists()|exists()]] method along with dependencies. It does not check whether
  the dependency associated with the cached data, if there is any, has changed. So a call to
  [[yii\caching\Cache::get()|get()]] may return `false` while [[yii\caching\Cache::exists()|exists()]] returns `true`.


## Query Caching <span id="query-caching"></span>

Query caching is a special caching feature built on top of data caching. It is provided to cache the result
of database queries.

Query caching requires a [[yii\db\Connection|DB connection]] and a valid `cache` [application component](#cache-components).
The basic usage of query caching is as follows, assuming `$db` is a [[yii\db\Connection]] instance:

```php
$result = $db->cache(function ($db) {

    // the result of the SQL query will be served from the cache
    // if query caching is enabled and the query result is found in the cache
    return $db->createCommand('SELECT * FROM customer WHERE id=1')->queryOne();

});
```

Query caching can be used for [DAO](db-dao.md) as well as [ActiveRecord](db-active-record.md):

```php
$result = Customer::getDb()->cache(function ($db) {
    return Customer::find()->where(['id' => 1])->one();
});
```

> Info: Some DBMS (e.g. [MySQL](https://dev.mysql.com/doc/refman/5.6/en/query-cache.html))
  also support query caching on the DB server-side. You may choose to use either query caching mechanism.
  The query caching described above has the advantage that you may specify flexible cache dependencies
  and are potentially more efficient.

Since 2.0.14 you can use the following shortcuts:

```php
(new Query())->cache(7200)->all();
// and
User::find()->cache(7200)->all();
```


### Configurations <span id="query-caching-configs"></span>

Query caching has three global configurable options through [[yii\db\Connection]]:

* [[yii\db\Connection::enableQueryCache|enableQueryCache]]: whether to turn on or off query caching.
  It defaults to `true`. Note that to effectively turn on query caching, you also need to have a valid
  cache, as specified by [[yii\db\Connection::queryCache|queryCache]].
* [[yii\db\Connection::queryCacheDuration|queryCacheDuration]]: this represents the number of seconds
  that a query result can remain valid in the cache. You can use 0 to indicate a query result should
  remain in the cache forever. This property is the default value used when [[yii\db\Connection::cache()]]
  is called without specifying a duration.
* [[yii\db\Connection::queryCache|queryCache]]: this represents the ID of the cache application component.
  It defaults to `'cache'`. Query caching is enabled only if there is a valid cache application component.


### Usages <span id="query-caching-usages"></span>

You can use [[yii\db\Connection::cache()]] if you have multiple SQL queries that need to take advantage of
query caching. The usage is as follows,

```php
$duration = 60;     // cache query results for 60 seconds.
$dependency = ...;  // optional dependency

$result = $db->cache(function ($db) {

    // ... perform SQL queries here ...

    return $result;

}, $duration, $dependency);
```

Any SQL queries in the anonymous function will be cached for the specified duration with the specified dependency.
If the result of a query is found valid in the cache, the query will be skipped and the result will be served
from the cache instead. If you do not specify the `$duration` parameter, the value of
[[yii\db\Connection::queryCacheDuration|queryCacheDuration]] will be used instead.

Sometimes within `cache()`, you may want to disable query caching for some particular queries. You can use
[[yii\db\Connection::noCache()]] in this case.

```php
$result = $db->cache(function ($db) {

    // SQL queries that use query caching

    $db->noCache(function ($db) {

        // SQL queries that do not use query caching

    });

    // ...

    return $result;
});
```

If you just want to use query caching for a single query, you can call [[yii\db\Command::cache()]] when building
the command. For example,

```php
// use query caching and set query cache duration to be 60 seconds
$customer = $db->createCommand('SELECT * FROM customer WHERE id=1')->cache(60)->queryOne();
```

You can also use [[yii\db\Command::noCache()]] to disable query caching for a single command. For example,

```php
$result = $db->cache(function ($db) {

    // SQL queries that use query caching

    // do not use query caching for this command
    $customer = $db->createCommand('SELECT * FROM customer WHERE id=1')->noCache()->queryOne();

    // ...

    return $result;
});
```


### Limitations <span id="query-caching-limitations"></span>

Query caching does not work with query results that contain resource handlers. For example,
when using the `BLOB` column type in some DBMS, the query result will return a resource
handler for the column data.

Some caching storage has size limitation. For example, memcache limits the maximum size
of each entry to be 1MB. Therefore, if the size of a query result exceeds this limit,
the caching will fail.


## Cache Flushing <span id="cache-flushing"></span>

When you need to invalidate all the stored cache data, you can call [[yii\caching\Cache::flush()]].

You can flush the cache from the console by calling `yii cache/flush` as well.
 - `yii cache`: lists the available caches in application
 - `yii cache/flush cache1 cache2`: flushes the cache components `cache1`, `cache2` (you can pass multiple component
 names separated with space)
 - `yii cache/flush-all`: flushes all cache components in the application
 - `yii cache/flush-schema db`: clears DB schema cache for a given connection component

> Info: Console application uses a separate configuration file by default. Ensure, that you have the same caching
components in your web and console application configs to reach the proper effect.

