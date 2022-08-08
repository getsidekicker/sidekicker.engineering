---
layout: post
title:  "Do you need OPCache"
author: wesley
categories: [ php ]
tags: [performance]
image: "assets/posts/2022-08-08-do-you-need-opcache/php-execution-time.png"
description: ""
comments: false
---

From [PHP official documentation](https://www.php.net/manual/en/book.opcache.php), we know that:

> OPCache improves PHP performance by storing precompiled script bytecode in shared memory, thereby removing the need for PHP to load and parse scripts on each request.

In other words, after OPCache is configured, the PHP script will be parsed and compiled to OPCode, and saved to the shared memory. So subsequent requests can read the OPCode from memory to reduce the execution time of PHP.

To make it more clear, the below flowchart describes its life cycle:

![OPcache Life Cycle](/assets/posts/2022-08-08-do-you-need-opcache/opcache-life-cycle.png)

## What happened after I enabled OPCache?
You often see OPCache performance tests online, claiming that it can make your PHP application 2x/3x faster, or the reduction of response time can vary from 14% to 74% as described by AppDynamics.

You might wonder will this be true for me? I want to show you some results from a real word application first:

![PHP Execution Time](/assets/posts/2022-08-08-do-you-need-opcache/php-execution-time.png)

As you can see from the above graph, the PHP execution time was reduced from about 150 ms to 50 ms. Yup, that's 67% faster. But is that the only metric that can prove the benefits of OPCache? How about CPU and memory usage?

![CPU Utilisation](/assets/posts/2022-08-08-do-you-need-opcache/cpu-utilisation.png)
![Memory Utilisation](/assets/posts/2022-08-08-do-you-need-opcache/memory-utilisation.png)

Surprisingly, the average CPU and memory utilisation of our app containers in AWS ECS was decreased from 10% to 3%, and from 23% to 10% respectively. We didn't anticipate this before we decided to enable OPCache in the docker containers, so this was a nice bonus.

## Back to configuration
There are plenty of articles showing you how to install and configure OPCache. An especially useful example is Steve Corona's [post](https://www.scalingphpbook.com/blog/2014/02/14/best-zend-opcache-settings.html), where he explains the most commonly used settings. But what's the best approach to figure out the settings yourself? TRY IT ON YOUR STAGING SERVER!

Give it a best guess, and then check the statistics from your staging/production site and adjust the settings over time. Below is a sample of data from the `opcache_get_status()` function:

```json
{
    "opcache_enabled": true,
    "cache_full": false,
    "restart_pending": false,
    "restart_in_progress": false,
    "memory_usage": {
        "used_memory": 41037512,
        "free_memory": 59625784,
        "wasted_memory": 0,
        "current_wasted_percentage": 0
    },
    "interned_strings_usage": {
        "buffer_size": 10485336,
        "used_memory": 7958976,
        "free_memory": 2526360,
        "number_of_strings": 99538
    },
    "opcache_statistics": {
        "num_cached_scripts": 1893,
        "num_cached_keys": 3240,
        "max_cached_keys": 32531,
        "hits": 30848167,
        "start_time": 1576813046,
        "last_restart_time": 0,
        "oom_restarts": 0,
        "hash_restarts": 0,
        "manual_restarts": 0,
        "misses": 2066,
        "blacklist_misses": 0,
        "blacklist_miss_ratio": 0,
        "opcache_hit_rate": 99.99330312999581
    }
}
```

From the example, we can see that the OPCache hit rate is 99.99%, which is pretty impressive. After leaving the OPCache running for a few days to a couple of weeks, you should be able to adjust the settings in `php.ini` based on the statistics collected on your server:

```ini
opcache.enabled=1

# remove this setting in dev environment or set it to true,
# as we don't want OPCache locally.
opcache.validate_timestamps=0

# this is irrelevant because the validate_timestamps is disabled,
# but locally OPCache will check for updates on every request.
opcache.revalidate_freq=0

# enable the opcode cache for the CLI version of PHP.
opcache.enable_cli=1

# the maximum number of keys in the OPCache hash table,
# better to use one of the prime numbers from the official manual,
# and make sure your project has fewer PHP files then the value you set here.
opcache.max_accelerated_files=16229

# adjust this value based on memory_usage -> used_memory in statistics
opcache.memory_consumption=128

# adjust this value based on interned_strings_usage -> used_memory in statistics
opcache.interned_strings_buffer=12

# it has been removed in PHP 7.2.0.
opcache.fast_shutdown=1
```

## Any concerns before using it?
YES, OPCache is a cache. As with any other cache, the potential issues are around two things:

* **Invalidation**: This is irrelevant in our use case as we don't perform in-place deployment, we use [buildkite](https://buildkite.com/) to automate our delivery pipelines, each deployment will spin up new docker containers in AWS ECS with the latest code. Where invalidation could potentially be an issue would be any dynamic PHP files where content changes at runtime but doesn't generate with a unique file.
* **Overhead of warming the cache**: This is minimal. Post-deployment, every PHP file will cache miss. So the first n requests that have new file execution paths will be moderately slower than at present. But if you look at the chart I showed you earlier, the PHP execution time didn't go up after the deployment.

## Do you need it?
As with everything, YMMV. In our case, we saw some awesome improvements (and we're kind of wondering why we didn't turn it on years ago!). However, you might not notice obvious improvements for small applications. But it is definitely worth trying out. You can always mess around with it on your staging server to compare the performance before and after the change, and tweak the configs to make it suitable for your applications.

