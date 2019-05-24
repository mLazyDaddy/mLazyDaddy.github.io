---
layout: post
title:  "Objective-C 锁性能测试"
date:   2019-05-10 12:00:00

---
（本文主要讨论`@synchronized`）

YYKit作者曾在[不再安全的 OSSpinLock](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)
中在单线程情况下对各种锁进行了简单的性能测试，结果`@synchronized`排名垫底。于是我好奇为什么苹果要设计出性能这么差的锁。根据YKit作者的测试代码我做了一个多线程的测试，代码在[这里](https://github.com/mLazyDaddy/LockTest/tree/master/LockTest)，测试环境是

|        |  |
| ---------- | --- |
|处理器|2.6 GHz Intel Core i5|
|内存|8 GB 1600 MHz DDR3|
|iPhone模拟器|iPhone XR + iOS 12.1|
|Mac||
|真机|iPhone X + iOS 12.1.3|


下面结果是对每种锁各开启10个线程运行100万次加解锁的平均耗时
<escape>
<table>
  <tr>
    <th rowspan ="2">锁</th>
    <th colspan="3"> 耗时（单位ms） </th>
  </tr>
  <tr>
    <td >iPhone XR模拟器</td>
    <td>iPhone X</td>
    <td>Mac</td>
  </tr>
  <tr>
    <td>`os_unfair_lock`</td>
    <td>745.41</td>
    <td>2335.76</td>
    <td>739.07</td>    
  </tr>
  <tr>
    <td>`@synchronized`</td>
    <td>3561.31</td>
    <td>5028.55</td>
    <td>56720.88</td>    
  </tr>
  <tr>
    <td>`dispatch_semaphore`</td>
    <td>46847.54</td>
    <td>26218.09</td>
    <td>42328.00</td>    
  </tr>
  <tr>
    <td>`pthread_mutex`</td>
    <td>49044.54</td>
    <td>5411.79</td>
    <td>42328.00</td>    
  </tr>
  <tr>
    <td>`NSLock`</td>
    <td>58718.06</td>
    <td>5407.33</td>
    <td>45350.73</td>    
  </tr>
  <tr>
    <td>`pthread_mutex(recursive)`</td>
    <td>52890.94</td>
    <td>6664.93</td>
    <td>44197.51</td>    
  </tr>
  <tr>
    <td>`NSRecursiveLock`</td>
    <td>58020.91</td>
    <td>7209.72</td>
    <td>46778.44</td>    
  </tr>
  <tr>
    <td>`NSCondition`</td>
    <td>92535.24</td>
    <td>71056.85</td>
    <td>135457.84</td>    
  </tr>
  <tr>
    <td>`NSConditionLock`</td>
    <td>236849.20</td>
    <td>73267.84</td>
    <td>146121.84</td>    
  </tr>  
</table>
</escape>


我们这里主要关注统一运行环境下锁之间的耗时差距。可以看出iPhone真机或者模拟器中多线程里`@synchronized`的性能仅次于`os_unfair_lock`，这样看来属性合成器在使用atomic修饰的情况下使用`@synchronized`来实现原子操作就显得比较合理了。但Mac中`@synchronized`的性能又很差，这让我十分不解。

为什么`@synchronized`在iOS中单线程下和多线程下的性能有那么大的差距,在苹果开源的MacOS objc4的代码里，我找到了慢的原因，但没有找到快的答案。

# `@synchronized`源码
苹果开源了MacOS的objc4的源码，其中包含了`@synchronized `的实现，下面的分析基于此源码。


`clang -rewrite-objc` Objective-C代码

```
@synchronized (self) {
}
```
会变成如下形式：

```
id _rethrow = 0; id _sync_obj = (id)self;
objc_sync_enter(_sync_obj);
try {
	struct _SYNC_EXIT {
		_SYNC_EXIT(id arg) : sync_exit(arg) {}
			~_SYNC_EXIT() {
				objc_sync_exit(sync_exit);
			}
			id sync_exit;
        }_sync_exit(_sync_obj);
} catch (id e){
	_rethrow = e;
}
{
	struct _FIN {
		_FIN(id reth) : rethrow(reth) {}
		~_FIN() {
			if (rethrow)
			objc_exception_throw(rethrow);
		}
		id rethrow;
	} _fin_force_rethow(_rethrow);
}
```

其中有两个关键函数`objc_sync_enter`和`objc_sync_exit`，从名字上可以看出前者用于加锁后者用于解锁，它们的实现如下：

```
// Begin synchronizing on 'obj'. 
// Allocates recursive mutex associated with 'obj' if needed.
// Returns OBJC_SYNC_SUCCESS once lock is acquired.  
int objc_sync_enter(id obj)
{
    int result = OBJC_SYNC_SUCCESS;

    if (obj) {
        SyncData* data = id2data(obj, ACQUIRE);
        assert(data);
        data->mutex.lock();
    } else {
        // @synchronized(nil) does nothing
        if (DebugNilSync) {
            _objc_inform("NIL SYNC DEBUG: @synchronized(nil); set a breakpoint on objc_sync_nil to debug");
        }
        objc_sync_nil();
    }

    return result;
}


// End synchronizing on 'obj'. 
// Returns OBJC_SYNC_SUCCESS or OBJC_SYNC_NOT_OWNING_THREAD_ERROR
int objc_sync_exit(id obj)
{
    int result = OBJC_SYNC_SUCCESS;
    
    if (obj) {
        SyncData* data = id2data(obj, RELEASE); 
        if (!data) {
            result = OBJC_SYNC_NOT_OWNING_THREAD_ERROR;
        } else {
            bool okay = data->mutex.tryUnlock();
            if (!okay) {
                result = OBJC_SYNC_NOT_OWNING_THREAD_ERROR;
            }
        }
    } else {
        // @synchronized(nil) does nothing
    }
	

    return result;
}
```

我们可以看到两个函数都调用了`id2data`获取了一个`SyncData`对象。

## SyncData


```
typedef struct SyncData {
    struct SyncData* nextData; //
    DisguisedPtr<objc_object> object; //同步对象，例如上面代码@synchronized (self)的self
    int32_t threadCount;  // 正在使用SyncData的线程数
    recursive_mutex_t mutex; //recursive_mutex_t使用的是pthread_mutex_t递归锁
} SyncData;
```


## id2data函数

```
static SyncData* id2data(id object, enum usage why)
{
    spinlock_t *lockp = &LOCK_FOR_OBJ(object);
    SyncData **listp = &LIST_FOR_OBJ(object);
    SyncData* result = NULL;

#if SUPPORT_DIRECT_THREAD_KEYS
    // Check per-thread single-entry fast cache for matching object
    bool fastCacheOccupied = NO;
    SyncData *data = (SyncData *)tls_get_direct(SYNC_DATA_DIRECT_KEY);
    if (data) {
        fastCacheOccupied = YES;

        if (data->object == object) {
            // Found a match in fast cache.
            uintptr_t lockCount;

            result = data;
            lockCount = (uintptr_t)tls_get_direct(SYNC_COUNT_DIRECT_KEY);
            if (result->threadCount <= 0  ||  lockCount <= 0) {
                _objc_fatal("id2data fastcache is buggy");
            }

            switch(why) {
            case ACQUIRE: {
                lockCount++;
                tls_set_direct(SYNC_COUNT_DIRECT_KEY, (void*)lockCount);
                break;
            }
            case RELEASE:
                lockCount--;
                tls_set_direct(SYNC_COUNT_DIRECT_KEY, (void*)lockCount);
                if (lockCount == 0) {
                    // remove from fast cache
                    tls_set_direct(SYNC_DATA_DIRECT_KEY, NULL);
                    // atomic because may collide with concurrent ACQUIRE
                    OSAtomicDecrement32Barrier(&result->threadCount);
                }
                break;
            case CHECK:
                // do nothing
                break;
            }

            return result;
        }
    }
#endif

    // Check per-thread cache of already-owned locks for matching object
    SyncCache *cache = fetch_cache(NO);
    if (cache) {
        unsigned int i;
        for (i = 0; i < cache->used; i++) {
            SyncCacheItem *item = &cache->list[i];
            if (item->data->object != object) continue;

            // Found a match.
            result = item->data;
            if (result->threadCount <= 0  ||  item->lockCount <= 0) {
                _objc_fatal("id2data cache is buggy");
            }
                
            switch(why) {
            case ACQUIRE:
                item->lockCount++;
                break;
            case RELEASE:
                item->lockCount--;
                if (item->lockCount == 0) {
                    // remove from per-thread cache
                    cache->list[i] = cache->list[--cache->used];
                    // atomic because may collide with concurrent ACQUIRE
                    OSAtomicDecrement32Barrier(&result->threadCount);
                }
                break;
            case CHECK:
                // do nothing
                break;
            }

            return result;
        }
    }

    // Thread cache didn't find anything.
    // Walk in-use list looking for matching object
    // Spinlock prevents multiple threads from creating multiple 
    // locks for the same new object.
    // We could keep the nodes in some hash table if we find that there are
    // more than 20 or so distinct locks active, but we don't do that now.
    
    lockp->lock();

    {
        SyncData* p;
        SyncData* firstUnused = NULL;
        for (p = *listp; p != NULL; p = p->nextData) {
            if ( p->object == object ) {
                result = p;
                // atomic because may collide with concurrent RELEASE
                OSAtomicIncrement32Barrier(&result->threadCount);
                goto done;
            }
            if ( (firstUnused == NULL) && (p->threadCount == 0) )
                firstUnused = p;
        }
    
        // no SyncData currently associated with object
        if ( (why == RELEASE) || (why == CHECK) )
            goto done;
    
        // an unused one was found, use it
        if ( firstUnused != NULL ) {
            result = firstUnused;
            result->object = (objc_object *)object;
            result->threadCount = 1;
            goto done;
        }
    }

    // malloc a new SyncData and add to list.
    // XXX calling malloc with a global lock held is bad practice,
    // might be worth releasing the lock, mallocing, and searching again.
    // But since we never free these guys we won't be stuck in malloc very often.
    result = (SyncData*)calloc(sizeof(SyncData), 1);
    result->object = (objc_object *)object;
    result->threadCount = 1;
    new (&result->mutex) recursive_mutex_t(fork_unsafe_lock);
    result->nextData = *listp;
    *listp = result;
    
 done:
    lockp->unlock();
    if (result) {
        // Only new ACQUIRE should get here.
        // All RELEASE and CHECK and recursive ACQUIRE are 
        // handled by the per-thread caches above.
        if (why == RELEASE) {
            // Probably some thread is incorrectly exiting 
            // while the object is held by another thread.
            return nil;
        }
        if (why != ACQUIRE) _objc_fatal("id2data is buggy");
        if (result->object != object) _objc_fatal("id2data is buggy");

#if SUPPORT_DIRECT_THREAD_KEYS
        if (!fastCacheOccupied) {
            // Save in fast thread cache
            tls_set_direct(SYNC_DATA_DIRECT_KEY, result);
            tls_set_direct(SYNC_COUNT_DIRECT_KEY, (void*)1);
        } else 
#endif
        {
            // Save in thread cache
            if (!cache) cache = fetch_cache(YES);
            cache->list[cache->used].data = result;
            cache->list[cache->used].lockCount = 1;
            cache->used++;
        }
    }

    return result;
}
```

整个函数我们简化成下面流程：

```
1.根据同步对象查找线程一级缓存，有则返回，无则进入步骤2；
2.查找线程二级缓存，有则返回，无则进入步骤3；
3.查找全局缓存表，有则返回，无则进入步骤4；
4.创建SyncData对象，将同步对象放入SyncData中，把SyncData加入缓存，然后返回SyncData对象。
```

* 一级缓存是key-value形式，直接通过key：`SYNC_DATA_DIRECT_KEY`获取，其方法类似于`getValueStoredInCurrentThreadByKey(SYNC_DATA_DIRECT_KEY)`（为了方便理解虚拟的函数名）；

* 二级缓存位于一个链表中，链表存放于SyncCache，SyncCache位于key是`TLS_DIRECT_KEY`的data里，获取链表的方法类似于`getValueStoredInCurrentThreadByKey(TLS_DIRECT_KEY)->syncCache->list`;

* 一二级缓存只供当前线程访问，所以获取缓存不需要加锁；全局缓存表供多线程访问，在查找时需要加锁；一二级缓存是互斥的，如果使用了一级缓存，二级缓存将不会使用，反之亦然。

* 若在一级或者二级缓存中找到匹配的SyncData对象，在返回前，还需要根据当前是加锁还是解锁做相应处理。如果是加锁，锁计数加1；如果是解锁，锁计数减1，并且在锁计数为0的时候移除缓存。

## `@synchronized`慢的原因
从上面的分析可以看出，在MacOS的实现里，在非递归使用`@synchronized`的情况下，无论是加解锁，`@synchronized`的实现都会根据同步对象从缓存中查找出（没有则新建）对应的SyncData对象，然后进行对应操作。加锁的时候缓存位于全局缓存表中，解锁时缓存位于一级缓存中，所以加锁时查找速度会低于解锁。又因为对锁的操作实际上是对`pthread_mutex_t`的操作，查找时间加上对锁的操作耗时，其性能低于`pthread_mutex`是意料之中。

iOS里多线程下`@synchronized`的表现优于`pthread_mutex`这个应该是苹果用了某种手段优化了吧。

