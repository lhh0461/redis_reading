#dict数据结构的实现
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

#dict的扩充操作
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
    
        /* If the hash table is empty expand it to the initial size. */
        if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);
    
        /* If we reached the 1:1 ratio, and we are allowed to resize the hash
         * table (global setting) or we should avoid it but the ratio between
         * elements/buckets is over the "safe" threshold, we resize doubling
         * the number of buckets. */
        if (d->ht[0].used >= d->ht[0].size &&
                (dict_can_resize ||
                 d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
        {
            return dictExpand(d, d->ht[0].used*2);
        }
        return DICT_OK;
    }
