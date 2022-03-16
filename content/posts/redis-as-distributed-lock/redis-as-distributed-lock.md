---
title: "Distributed Locking with AWS Redis"
date: 2021-08-09T21:08:21+02:00
draft: false
tags: ["blog","golang","aws","redis","back end"]
---

# Intro
In this post we're going to explore one of the many ways we can use Redis and Go to build something cool.
Specifically, we'll be building a distributed lock to prevent race condition between micro-services.

# Redis
Let's start by looking at Redis. If you google it, the first result states that:
> Redis is an open source (BSD licensed), in-memory data structure store, used as a database, cache, and message broker. Redis provides data structures such as strings, hashes, lists, sets, sorted sets with range queries, bitmaps, hyperloglogs, geospatial indexes, and streams. Redis has built-in replication, Lua scripting, LRU eviction, transactions, and different levels of on-disk persistence, and provides high availability via Redis Sentinel and automatic partitioning with Redis Cluster

Cool stuff right? What interests us are these two things:
- _"in-memory data structure store"_: meaning it's extremely fast to retrieve and store values
- _"Redis has built-in [...] Lua scripting": it will come into play later on. For now, let's just say it gives great flexibility
  

# The need for a distributed lock
In the era of micro-services, where everything is cloud based, the need for a distributed lock is more common than you might think.

For the sake of this example, we're going to use two micro-services.
For simplicity they both share the same codebase, but an environment variable determines which service is started:

```golang
type Service interface {
	Start(context.Context) error
}

type ServiceName string

const (
	FirstServiceName  ServiceName = "service.first"
	SecondServiceName ServiceName = "service.second"
)

func initService() (Service, error) {
	cfg := configFromFile("./.env")

	services := map[ServiceName]Service{
		FirstServiceName:  NewFirstService(cfg),
		SecondServiceName: NewSecondService(cfg),
	}

	service, ok := services[cfg.Runtime]
	if !ok {
		return nil, fmt.Errorf("unknown runtime: [%s]", cfg.Runtime)
	}

	return service, nil
}

func main() {
	service, err := initServices()
	if err != nil {
		panic(err)
	}

	log.Fatal(service.Start(context.Background()))
}
```

# Let's code!
Now, pretend both micro-services, for some reasons, needs to access the same database.
Additionally, let's pretend we want to run the database's migrations inside our `initService()` function:

```golang
func initService() (Service, error) {
	cfg := configFromFile("./.env")

	db := NewDbConnection(cfg.DbSettings)
	db.RunMigrations("./migrations/")

	//...
	return service, nil
}
```
Easy right? But wait! Both services are going to run `db.RunMigrations` and we have no way to know which service is going to reach this line first

Like any race condition, we solve this by putting a lock on the shared resource.
Since we are dealing with micro-services running on two separate instances, we need to use a distributed lock!

Let's start by defining what a lock should look like:
```golang
// UnlockFunc should be used to release a lock,
// its behavior must be kept internal to the implementation
type UnlockFunc func() error

// Locker offers the option to lock a key for a given amount of time
// after that time has passed, the lock should be released (to prevent deadlocks)
type Locker interface {
	Lock(key string, timeout time.Duration) (UnlockFunc, error)
}
```

Nice! Now, for the actual implementation... A lock should perform an atomic operation, and here's where Lua scripting comes into play!

```golang
const redisLockScript = `return redis.call('SET', KEYS[1], ARGV[1], 'NX', 'PX', ARGV[2])`
```

If it looks scary, here's a break-down of the script:
- `redis.call` calls a Redis action
- `SET` is the Redis action we want to perform
- `KEYS[1]` takes the first (and only) key in the given list
- `ARGV[1]` takes the first argument, and maps it to the previously passed key
- `NX` only creates the Key-Argv mapping if it doesn't already exists
- `PX` sets a lifespan on the mapping. Takes one parameter: the lock duration
- `ARGV[2]` takes the second argument and uses it as the parameter for `PX`
  
In short: we are telling redis to set a key-value pair with an expiration date, if it doesn't already exists (and it's atomic)

We'll use a similar approach for the unlock script:
```golang
// if the key is present, delete it
const redisUnlockScript = `
		if redis.call("get",KEYS[1]) == ARGV[1] then
		    return redis.call("del", KEYS[1])
		else
		    return 0
		end`
```

Before we can use these scripts, we first need to do some interface magic around the package `"github.com/go-redis/redis"`:
```golang
package infrastructure 

// clientWrapper wraps the methods shared between redis.ClusterClient and redis.Client
// implements redis.scripter and is implemented by redis.Cmdble
type clientWrapper interface {
	Ping() *redis.StatusCmd
	Get(string) *redis.StringCmd
	Set(string, interface{}, time.Duration) *redis.StatusCmd
	Del(...string) *redis.IntCmd

	Eval(script string, keys []string, args ...interface{}) *redis.Cmd
	EvalSha(sha1 string, keys []string, args ...interface{}) *redis.Cmd
	ScriptExists(hashes ...string) *redis.BoolSliceCmd
	ScriptLoad(script string) *redis.StringCmd
}

// RedisRunner describes the method we want to expose from the "go-redis/redis" client
type RedisRunner interface {
	Run(script string, keys []string, args ...interface{}) (interface{}, error)
}

// RedisClient is the public implementation of RedisRunner using the underlying clientWrapper
type RedisClient struct {
	client clientWrapper
}

// Run a given script
func (rc *RedisClient) Run(script string, keys []string, args ...interface{}) (interface{}, error) {
	return redis.NewScript(script).Run(rc.client, keys, args...).Result()
}
```

We are now able to run our scripts in a nice and clean way:

```golang
package lock


// RedisLock implements the Locker interface using Redis
type RedisLock struct {
	client infrastructure.RedisRunner
}

func (r *RedisLock) Lock(key string, timeout time.Duration) (UnlockFunc, error) {
	val := uuid.NewV4().String()

	res, err := rc.client.Run(redisLockScript, []string{key}, val, int(timeout/time.Millisecond))
	if err != nil {
		return nil, err
	}

	if res != infrastructure.RedisResponseOk {
		return nil, ErrUnableToLock
	}

	return rc.unlock(key, val), nil
}

func (r *RedisLock) unlock(key, value string) UnlockFunc {
	return func() error {
		res, err := rc.client.Run(redisUnlockScript, []string{key}, value)
		if err != nil {
			return err
		}

		if i, ok := res.(int64); !ok || i != 1 {
			return ErrUnableToUnlock
		}

		return nil
	}
}
```

# Using the lock
Amazing! Finally, we can see the lock at work:

```golang
func initService() (Service, error) {
	cfg := configFromFile("./.env")

	redisLock := lock.NewRedisLock(cfg.RedisSettings)

	unlock, err := redisLock.Lock(cfg.MigrationLockKey, time.Minute*5)
	if err != nil {
		return nil, err
	}

	defer unlock() // not checking the error for brevity's sake

	db := NewDbConnection(cfg.DbSettings)
	db.RunMigrations("./migrations/")

	//...
	return service, nil
}
```

That's it: we now have a distributed lock, there's nothing that can stop us! Unless...

# Namespaces
We might want to go the extra mile and use namespaces to further prevent collisions, this change is pretty straight forward, we just need to wrap our `RedisLock` and prefix each key with the namespace:
```golang
package lock

type NamespacedRedisLock struct {
	namespace string
	rl        RedisLock
}

func (n *NamespacedRedisLock) prefix(key string) string {
	return fmt.Sprintf("%s:%s", n.namespace, key)
}

func (n *NamespacedRedisLock) Lock(key string, timeout time.Duration) (UnlockFunc, error) {
	return n.client.Lock(n.prefix(key), timeout)
}
```

# Conclusion

Today we learned about distributed locks, a very powerful and helpful tool for any developer. 
It might look like this is a bit of an overkill, but the distributed locks' use-cases don't just stop at synchronizing database's accesses.
Whether you're working with an asynchronous architecture like an event-based system, or you need to coordinate a set of operations on the same resource, distributed locks offers lots of possibilities and are extremely relevant in the current world of micro-services.