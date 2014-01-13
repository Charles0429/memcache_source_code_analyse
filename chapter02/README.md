##memcache中hash相关代码解读

###简介

memcache中有关hash的代码在`hash.c/hash.h`和`assoc.c/assoc.h`

- `hash.c/hash.h`的内容主要是一个hash算法的实现
- `assoc.c/assoc.h`则主要是hashtable的管理


###hash算法的实现

这个hash算法来自Bob Jenkins，这个算法的时间复杂度为`O(6n + 35)`，而且冲突率非常低，对这个算法感兴趣的可以点[这里](http://burtleburtle.net/bob/hash/doobs.html)

###hastable的管理

在memcache中有两个hash表，一个为主表，另外一个为原hash表。设置两个表的目的是为了在哈希表要进行冲突扩展的时候使用的，扩张时，如果数据在原有hash表，则在原有hash表读数据，如果数据表在主表，则在主表读数据。

hashtable涉及的操作主要如下：

- hash表初始化，对应函数为`assoc_init`
- 根据key查找value，对应函数为`assoc_find`
- 插入一个元素，对应函数为`assoc_insert`
- 删除一个元素，对应函数为`assoc_delete`
- hash表扩展，对应函数为`assoc_expand`
- hash表扩展过程，对应函数为`assoc_maintenance_thread`

###hash表初始化

    void assoc_init(const int hashtable_init) {
        if (hashtable_init) {
            hashpower = hashtable_init;
        }
        primary_hashtable = calloc(hashsize(hashpower), sizeof(void *));
        if (! primary_hashtable) {
            fprintf(stderr, "Failed to init hashtable.\n");
            exit(EXIT_FAILURE);
        }
        STATS_LOCK();
        stats.hash_power_level = hashpower;
        stats.hash_bytes = hashsize(hashpower) * sizeof(void *);
        STATS_UNLOCK();
    }

看以上代码，先分配hash(hashpower)个指针的存储空间，用来存每个item的指针，然后就是更新stats信息

###根据key查找value

    item *assoc_find(const char *key, const size_t nkey, const uint32_t hv) {
        item *it;
        unsigned int oldbucket;
    
        if (expanding &&
            (oldbucket = (hv & hashmask(hashpower - 1))) >= expand_bucket)
        {
            it = old_hashtable[oldbucket];
        } else {
            it = primary_hashtable[hv & hashmask(hashpower)];
        }
    
        item *ret = NULL;
        int depth = 0;
        while (it) {
            if ((nkey == it->nkey) && (memcmp(key, ITEM_key(it), nkey) == 0)) {
                ret = it;
                break;
            }
            it = it->h_next;
            ++depth;
        }
        MEMCACHED_ASSOC_FIND(key, nkey, depth);
        return ret;
    }

首先，判断hash是否正在扩展，如果是在扩展，则判断数据是在主hash表，还是原hash表。然后根据hash表定位到具体的位置，由于memcache采用的是开链法解决冲突，所以就需要把所有开链上的位置遍历查找，直到碰到和要查找元素的key相同的元素

###插入一个元素

    /* Note: this isn't an assoc_update.  The key must not already exist to call this */
    int assoc_insert(item *it, const uint32_t hv) {
        unsigned int oldbucket;
    
    //    assert(assoc_find(ITEM_key(it), it->nkey) == 0);  /* shouldn't have duplicately named things defined */
    
        if (expanding &&
            (oldbucket = (hv & hashmask(hashpower - 1))) >= expand_bucket)
        {
            it->h_next = old_hashtable[oldbucket];
            old_hashtable[oldbucket] = it;
        } else {
            it->h_next = primary_hashtable[hv & hashmask(hashpower)];
            primary_hashtable[hv & hashmask(hashpower)] = it;
        }
    
        hash_items++;
        if (! expanding && hash_items > (hashsize(hashpower) * 3) / 2) {
            assoc_start_expand();
        }
    
        MEMCACHED_ASSOC_INSERT(ITEM_key(it), it->nkey, hash_items);
        return 1;
    }

首先，根据hash的位置决定是在原hash还是主hash表，如果冲突因子大于1.5，则扩展hash表

###删除一个元素

    void assoc_delete(const char *key, const size_t nkey, const uint32_t hv) {
        item **before = _hashitem_before(key, nkey, hv);
    
        if (*before) {
            item *nxt;
            hash_items--;
            /* The DTrace probe cannot be triggered as the last instruction
             * due to possible tail-optimization by the compiler
             */
            MEMCACHED_ASSOC_DELETE(key, nkey, hash_items);
            nxt = (*before)->h_next;
            (*before)->h_next = 0;   /* probably pointless, but whatever. */
            *before = nxt;
            return;
        }
        /* Note:  we never actually get here.  the callers don't delete things
           they can't find. */
        assert(*before != 0);
    }

首先，找到元素的前一个指针，然后修改前一个指针的指向即可

###hash表扩展

    /* grows the hashtable to the next power of 2. */
    static void assoc_expand(void) {
        old_hashtable = primary_hashtable;
    
        primary_hashtable = calloc(hashsize(hashpower + 1), sizeof(void *));
        if (primary_hashtable) {
            if (settings.verbose > 1)
                fprintf(stderr, "Hash table expansion starting\n");
            hashpower++;
            expanding = true;
            expand_bucket = 0;
            STATS_LOCK();
            stats.hash_power_level = hashpower;
            stats.hash_bytes += hashsize(hashpower) * sizeof(void *);
            stats.hash_is_expanding = 1;
            STATS_UNLOCK();
        } else {
            primary_hashtable = old_hashtable;
            /* Bad news, but we can keep running. */
        }
    }

这个函数的任务就是把hash表的容易扩展一倍，并不涉及到具体的指针移动过程，而后面的`assoc_maintenance_thread`是具体的移动过程

###hash表具体的移动过程

    static void *assoc_maintenance_thread(void *arg) {
    
        while (do_run_maintenance_thread) {
            int ii = 0;
    
            /* Lock the cache, and bulk move multiple buckets to the new
             * hash table. */
            item_lock_global();
            mutex_lock(&cache_lock);
    
			/*默认一次移动一个开链的数据*/
            for (ii = 0; ii < hash_bulk_move && expanding; ++ii) {
                item *it, *next;
                int bucket;
    
                for (it = old_hashtable[expand_bucket]; NULL != it; it = next) {
                    next = it->h_next;
    
                    bucket = hash(ITEM_key(it), it->nkey, 0) & hashmask(hashpower);
                    it->h_next = primary_hashtable[bucket];
                    primary_hashtable[bucket] = it;
                }
    
                old_hashtable[expand_bucket] = NULL;
    
                expand_bucket++;
                if (expand_bucket == hashsize(hashpower - 1)) {
                    expanding = false;
                    free(old_hashtable);
                    STATS_LOCK();
                    stats.hash_bytes -= hashsize(hashpower - 1) * sizeof(void *);
                    stats.hash_is_expanding = 0;
                    STATS_UNLOCK();
                    if (settings.verbose > 1)
                        fprintf(stderr, "Hash table expansion done\n");
                }
            }
    
            mutex_unlock(&cache_lock);
            item_unlock_global();
    
            if (!expanding) {
                /* finished expanding. tell all threads to use fine-grained locks */
                switch_item_lock_type(ITEM_LOCK_GRANULAR);
                slabs_rebalancer_resume();
                /* We are done expanding.. just wait for next invocation */
                mutex_lock(&cache_lock);
                started_expanding = false;
                pthread_cond_wait(&maintenance_cond, &cache_lock);
                /* Before doing anything, tell threads to use a global lock */
                mutex_unlock(&cache_lock);
                slabs_rebalancer_pause();
                switch_item_lock_type(ITEM_LOCK_GLOBAL);
                mutex_lock(&cache_lock);
                assoc_expand();
                mutex_unlock(&cache_lock);
            }
        }
        return NULL;
    }

默认把hash表的一个开链的数据给挪到主hash表中，具体的过程就是一个个的指针就是修改