---
layout: post
title: "Incompatibility between Redis Namespace and Sidekiq Alive"
subtitle: "And how we dealt with it"
date: 2021-01-10 00:30:00 -0400
background: '/img/posts/09.jpg'
---

## Introduction
This post discusses why Sidekiq Alive and Redis Namespace are incompatible, and it also discusses how to remove Redis Namespace from a system. If you're only interested in removing Redis Namespace, then skip ahead to the `Removing Redis Namespace` section.

### Background information: Redis Namespace
[Redis Namespace](https://github.com/resque/redis-namespace) is a gem used to add a prefix to all Ruby-created keys that share the same Redis Namespace configuration. I'm not sure on its intended use cases, but I guess that someone somewhere uses it as a logical separation of keys, in a situation where they don't want to use separate Redis databases inside the same instance. I'm still not sure why, but our product was configured to use Redis Namespace to prefix our keys with `<product>-<environment>`. In this article, that prefix is `pa-sandbox`.

## Background information: Sidekiq Alive
Our team recently worked on adding healthchecks to our asynchronous processor, [Sidekiq](https://github.com/mperham/sidekiq), using [Sidekiq Alive](https://github.com/arturictus/sidekiq_alive). Typically, a Rails application writes asynchronous jobs to a Redis instance, and Sidekiq reads jobs from Redis, marking them as complete once they are done.

Each Sidekiq instance that uses Sidekiq Alive creates a Sidekiq-instance-specific key in Redis, which is programmed to time out after a customisable length of time (e.g 120 seconds). 
```
# In Redis CLI
TTL sidekiq-instance-1
=> (integer) 113

# Wait 5 seconds
TTL sidekiq-instance-1
=> (integer) 108
```
To ensure that the key doesn't time out, Sidekiq Alive updates the TTL value of the key before the key times out. This would be observed in the Redis CLI as shown:
```
# In Redis CLI
TTL sidekiq-instance-1
=> (integer) 61

# Wait 5 seconds
TTL sidekiq-instance-1
=> (integer) 120
```

Sidekiq Alive also creates an HTTP server, which returns a positive response if the key has not timed out (i.e. if `TTL <key>` returns a number greater than 0. This lets users determine whether a Sidekiq instance is successfully maintaining a healthy connection to Redis.

## Incompatibility issue
During development, we saw that healthchecks were not working as intended, returning negative responses. Running `TTL <key>` returned -2, [which means that the key does not exist](https://redis.io/commands/ttl). 
```
# In Redis CLI
TTL sidekiq-instance-1
=> (integer) -2
```

Upon further inspection, we saw that while Sidekiq Alive's key did not exist in Redis, a Redis Namespaced equivalent key did exist.
```
# In Redis CLI
keys *
=> 1) "pa-sandbox:stat:processed"
   2) "pa-sandbox:stat:failed"
   3) "pa-sandbox:stat:processed:2021-01-09"
   4) "pa-sandbox:stat:failed:2021-01-09"
   5) "processes"
   6) "queues"
   7) "pa-sandbox:sidekiq-instance-1"
```

The 7th key in this list looks like the key we are looking for, but it is prefixed with `pa-sandbox:`. It seemed that Sidekiq Alive did not realise that the key it had created was getting prefixed by `pa-sandbox:`, so although it was searching for `sidekiq-instance-1`, the key was called `pa-sandbox:sidekiq-instance-1`. 


## Removing Redis Namespace
Mike Perham (creator of Sidekiq) [does not recommend using Redis Namespace](https://www.mikeperham.com/2015/09/24/storing-data-with-redis/), and it didn't seem to be adding any value to me. After checking with our developers, I confirmed that Redis Namespace is not required in our project anymore, so I learnt how to remove it.

Mike Perham also [wrote a blog post on how to remove Redis Namespace](https://www.mikeperham.com/2017/04/10/migrating-from-redis-namespace/), in which he uses a script to rename each Redis key to remove the prefix. However, I wasn't convinced on this method, because if I remove Redis Namespace, Sidekiq will instantly create new keys in Redis, which will contain new data as the system gets used. If I run his script at that point, it would overwrite the new data with outdated data from the namespaced keys. Instead of overwriting the new data with old data, I would like to keep my new data, and remove the old data entirely. Note that my approach therefore places lower emphasis on the preservation of Sidekiq statistical data, which may or may not be acceptable to you.

First, I removed Redis Namespace from my codebase. I removed the gem, and erased traces of it inside my source code. E.g. in `config/initializers/sidekiq.rb`, I edited `Sidekiq.configure_client` and `Sidekiq.configure_server`, to no longer have
```
namespace: ENV['REDIS_NAMESPACE']
```

On pushing this change to my test environment, I confirmed that Sidekiq had created new Redis keys, without deleting the old ones.
```
# In Redis CLI
keys *
=> 1) "pa-sandbox:stat:processed"
   2) "pa-sandbox:stat:failed"
   3) "pa-sandbox:stat:processed:2021-01-09"
   4) "pa-sandbox:stat:failed:2021-01-09"
   5) "processes"
   6) "queues"
   7) "pa-sandbox:sidekiq-instance-1"
   8) "stat:processed"
   9) "stat:failed"
  10) "stat:processed:2021-01-09"
  11) "stat:failed:2021-01-09"
  13) "sidekiq-instance-1"
```

At this point, Sidekiq Alive started working again, because Sidekiq Alive's healthcheck endpoint runs `TTL sidekiq-instance-1`, which is now an existing key.

However, the work isn't finished yet, because we now have 5 "hanging" keys: the prefixed keys are no longer managed by Sidekiq, and may exist indefinitely in Redis. This is where Mike Perham's approach differs from mine: his approach would replace the contents of `stat:failed` with `pa-sandbox:stat:failed`, whereas I would rather remove the outdated key and leave the new key untouched.

To remove these "hanging keys", I ran 
```
redis-cli -h $REDIS_HOST --scan --pattern pa-sandbox:* | xargs redis-cli  -h $REDIS_HOST del
```

After this, running `keys *` showed that only desired Redis keys remained, and the cleanup process was complete.

```
# In Redis CLI
keys *
=> 1) "processes"
   2) "queues"
   3) "stat:processed"
   4) "stat:failed"
   5) "stat:processed:2021-01-09"
   6) "stat:failed:2021-01-09"
   7) "sidekiq-instance-1"
```
