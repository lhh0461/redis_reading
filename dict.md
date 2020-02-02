# dict数据结构的实现
    //这个结构代表一个节点
    typedef struct dictEntry {
        void *key;
        union {
            void *val;
            uint64_t u64;
            int64_t s64;
            double d;
        } v;
        struct dictEntry *next; //链表的下一个节点
    } dictEntry;

    //这个是哈希表的结构，每一个字典有两个这个结构，目的用来实现增量重哈希，把数据从旧表迁移到新表
    //redis的哈希表使用链表来解决冲突
    typedef struct dictht {
        dictEntry **table;          //哈希节点的指针数组
        unsigned long size;         //数组大小
        unsigned long sizemask;     //数组大小的模((1<<size)-1)
        unsigned long used;         //节点数量
    } dictht;
    
    typedef struct dict {
        dictType *type;             //哈希表类型，不同的类型的实现不同的hash接口，面向对象的思想
        void *privdata;             //表类型的用户数据
        dictht ht[2];               //两个哈希表，用于重哈希
        long rehashidx;             //重哈希的下表如果在重哈希中为正在处理的重哈希下标，否则为-1
        unsigned long iterators;    //当前有多少迭代器正在运行
    } dict;

# dict的常见操作
## dict的创建
    /* Create a new hash table */
    dict *dictCreate(dictType *type,
            void *privDataPtr)
    {
        dict *d = zmalloc(sizeof(*d));
    
        _dictInit(d,type,privDataPtr);
        return d;
    }

## dict的删除
    /* Clear & Release the hash table */
    void dictRelease(dict *d)
    {
        _dictClear(d,&d->ht[0],NULL);
        _dictClear(d,&d->ht[1],NULL);
        zfree(d);
    }

    /* Destroy an entire dictionary */
    int _dictClear(dict *d, dictht *ht, void(callback)(void *))
    {
        unsigned long i;
    
        /* Free all the elements */
        for (i = 0; i < ht->size && ht->used > 0; i++) {
            dictEntry *he, *nextHe;
    
            if (callback && (i & 65535) == 0) callback(d->privdata);
    
            if ((he = ht->table[i]) == NULL) continue;
            while(he) {
                nextHe = he->next;
                dictFreeKey(d, he);
                dictFreeVal(d, he);
                zfree(he);
                ht->used--;
                he = nextHe;
            }
        }
        /* Free the table and the allocated cache structure */
        zfree(ht->table);
        /* Re-initialize the table */
        _dictReset(ht);
        return DICT_OK; /* never fails */
    }

    static void _dictReset(dictht *ht)
    {
        ht->table = NULL;
        ht->size = 0;
        ht->sizemask = 0;
        ht->used = 0;
    }



## dict的添加元素操作
    /* Add an element to the target hash table */
    int dictAdd(dict *d, void *key, void *val)
    {
        dictEntry *entry = dictAddRaw(d,key,NULL);
    
        if (!entry) return DICT_ERR;
        dictSetVal(d, entry, val);
        return DICT_OK;
    }

## dict的查找元素操作
    dictEntry *dictFind(dict *d, const void *key)
    {
        dictEntry *he;
        uint64_t h, idx, table;
    
        if (d->ht[0].used + d->ht[1].used == 0) return NULL; /* dict is empty */
        if (dictIsRehashing(d)) _dictRehashStep(d);
        h = dictHashKey(d, key);
        for (table = 0; table <= 1; table++) {
            idx = h & d->ht[table].sizemask;
            he = d->ht[table].table[idx];
            while(he) {
                if (key==he->key || dictCompareKeys(d, key, he->key))
                    return he;
                he = he->next;
            }
            if (!dictIsRehashing(d)) return NULL;
        }
        return NULL;
    }

# dict删除元素
    /* Remove an element, returning DICT_OK on success or DICT_ERR if the
     * element was not found. */
    int dictDelete(dict *ht, const void *key) {
        return dictGenericDelete(ht,key,0) ? DICT_OK : DICT_ERR;
    }


    /* Search and remove an element. This is an helper function for
     * dictDelete() and dictUnlink(), please check the top comment
      * of those functions. */
    static dictEntry *dictGenericDelete(dict *d, const void *key, int nofree) {
        uint64_t h, idx;
        dictEntry *he, *prevHe;
        int table;
    
        if (d->ht[0].used == 0 && d->ht[1].used == 0) return NULL;
    
        if (dictIsRehashing(d)) _dictRehashStep(d);
        h = dictHashKey(d, key);
    
        for (table = 0; table <= 1; table++) {
            idx = h & d->ht[table].sizemask;
            he = d->ht[table].table[idx];
            prevHe = NULL;
            while(he) {
                if (key==he->key || dictCompareKeys(d, key, he->key)) {
                    /* Unlink the element from the list */
                    if (prevHe)
                        prevHe->next = he->next;
                    else
                        d->ht[table].table[idx] = he->next;
                    if (!nofree) {
                        dictFreeKey(d, he);
                        dictFreeVal(d, he);
                        zfree(he);
                    }
                    d->ht[table].used--;
                    return he;
                }
                prevHe = he;
                he = he->next;
            }
            if (!dictIsRehashing(d)) break;
        }
        return NULL; /* not found */
    }



# dict的扩充操作
    //redis在dict添加元素时检查是否需要扩容，如下代码所示
    dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)
    {
        long index;
        dictEntry *entry;
        dictht *ht;
    
        //如果这个字典正在重哈希中，执行一次重哈希，重哈希是分步的过程，每一步就是数组重哈希下标的
        //链表所有元素搬移到新的表中
        if (dictIsRehashing(d)) _dictRehashStep(d);
    
        /* 获得新元素key所在的下标, 
         * 如果是已存在的key返回-1. */
        if ((index = _dictKeyIndex(d, key, dictHashKey(d,key), existing)) == -1)
            return NULL;
    
        //分配内存和存储新的节点
        //插入节点到链表头，这让做的假设就是最近插入的节点会访问的更加频繁
        //如果正在重哈希则插入到ht[1]否则插入到ht[0]，如果是在重哈希中，那么两个表都会存着数据
        ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
        entry = zmalloc(sizeof(*entry));
        entry->next = ht->table[index];
        ht->table[index] = entry;
        ht->used++;
    
        /* Set the hash entry fields. */
        dictSetKey(d, entry, key);
        return entry;
    }

    static long _dictKeyIndex(dict *d, const void *key, uint64_t hash, dictEntry **existing)
    {
        unsigned long idx, table;
        dictEntry *he;
        if (existing) *existing = NULL;
    
        //扩展字典如果需要的话
        if (_dictExpandIfNeeded(d) == DICT_ERR)
            return -1;

        //扩展字典如果需要的话
        for (table = 0; table <= 1; table++) {
            //取出这个key所在的下标，下标就是hash与上模
            idx = hash & d->ht[table].sizemask;
            //遍历整个链表，找出key是否存在
            he = d->ht[table].table[idx];
            while(he) {
                if (key==he->key || dictCompareKeys(d, key, he->key)) {
                    if (existing) *existing = he;
                    return -1;
                }
                he = he->next;
            }
            //如果没有重哈希中，直接跳出循环
            if (!dictIsRehashing(d)) break;
        }
        return idx;
    }


    //扩展字典判断逻辑
    static int _dictExpandIfNeeded(dict *d)
    {
        /* Incremental rehashing already in progress. Return. */
        if (dictIsRehashing(d)) return DICT_OK;
    
        //如果为0则是未初始化，扩展为初始化大小
        if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);
    
        /* If we reached the 1:1 ratio, and we are allowed to resize the hash
         * table (global setting) or we should avoid it but the ratio between
         * elements/buckets is over the "safe" threshold, we resize doubling
         * the number of buckets. */
        //注释写的很清楚了，节点数量大于槽数量并且可以重哈希，则扩展2倍
        if (d->ht[0].used >= d->ht[0].size &&
                (dict_can_resize ||
                 d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
        {
            return dictExpand(d, d->ht[0].used*2);
        }
        return DICT_OK;
    }


    /* Expand or create the hash table */
    int dictExpand(dict *d, unsigned long size)
    {
        dictht n; /* the new hash table */
        //_dictNextPower获得大于等于size的最小2的n次幂
        unsigned long realsize = _dictNextPower(size);
    
        /* the size is invalid if it is smaller than the number of
         * elements already inside the hash table */
        if (dictIsRehashing(d) || d->ht[0].used > size)
            return DICT_ERR;
    
        /* Rehashing to the same table size is not useful. */
        if (realsize == d->ht[0].size) return DICT_ERR;
    
        /* Allocate the new hash table and initialize all pointers to NULL */
        n.size = realsize;
        n.sizemask = realsize-1;
        n.table = zcalloc(realsize*sizeof(dictEntry*));
        n.used = 0;
    
        /* Is this the first initialization? If so it's not really a rehashing
         * we just set the first hash table so that it can accept keys. */
        if (d->ht[0].table == NULL) {
            d->ht[0] = n;
            return DICT_OK;
        }
    
        /* Prepare a second hash table for incremental rehashing */
        d->ht[1] = n;
        d->rehashidx = 0;
        return DICT_OK;
    }


# dict的收缩
    //收缩也是调用dictExpand
    int dictResize(dict *d)
    {
        int minimal;
    
        if (!dict_can_resize || dictIsRehashing(d)) return DICT_ERR;
        minimal = d->ht[0].used;
        if (minimal < DICT_HT_INITIAL_SIZE)
            minimal = DICT_HT_INITIAL_SIZE;
        return dictExpand(d, minimal);
    }

