# nginx_pool源码解析
## 1.数据结构

```c
typedef struct ngx_pool_large_s  ngx_pool_large_t;

struct ngx_pool_large_s {
    ngx_pool_large_t  *next;//next指针
    void              *alloc;//具体的内存
};


struct ngx_pool_s {
    char              *last;//当前内存池分配到此处，即下一次分配从此处开始
    char              *end;//真实的内存尾地址
    ngx_pool_t        *next;//内存块链表
    ngx_pool_large_t  *large;//超出限制的内存分配
    ngx_log_t         *log;
};
 
```

## 创建pool

```c
ngx_pool_t *ngx_create_pool(size_t size, ngx_log_t *log)
{
    ngx_pool_t  *p;

    if (!(p = ngx_alloc(size, log))) {
       return NULL;
    }

    p->last = (char *) p + sizeof(ngx_pool_t);
    p->end = (char *) p + size;
    p->next = NULL;
    p->large = NULL;
    p->log = log;

    return p;
}

```

## 使用内存池分配内存

```c


void *ngx_palloc(ngx_pool_t *pool, size_t size)
{
    char              *m;
    ngx_pool_t        *p, *n;
    ngx_pool_large_t  *large, *last;

    if (size <= (size_t) NGX_MAX_ALLOC_FROM_POOL
        && size <= (size_t) (pool->end - (char *) pool) - sizeof(ngx_pool_t))
    {
        //遍历内存块
        for (p = pool, n = pool->next; /* void */; p = n, n = n->next) {
            //#define ngx_align(p)    (char *) ((NGX_ALIGN_CAST p + NGX_ALIGN) & ~NGX_ALIGN)
            //对末尾last进行内存对齐
            m = ngx_align(p->last);
            //如果内存块中还有多余的内存 那么就使用当前内存块
            if ((size_t) (p->end - m) >= size) {
                p->last = m + size ;

                return m;
            }

            if (n == NULL) {
                break;
            }
        }

        /* allocate a new pool block */
        //如果走到了这里说明没有合适内存块可以使用，那么就创建一块新的内存块，并且挂到链表末尾，创建一块跟尾内存块一样大的内存
        if (!(n = ngx_create_pool((size_t) (p->end - (char *) p), p->log))) {
            return NULL;
        }
        //让m指向新内存块的last地址，并且标记这块内存已经占用，并且返回
        p->next = n;
        m = n->last;
        n->last += size;

        return m;
    }

    /* allocate a large block */
    //走到了这里说明是创建large block
    large = NULL;
    last = NULL;
    //如果pool中已经存在了大内存块
    if (pool->large) {
        for (last = pool->large; /* void */ ; last = last->next) {
            //如果large中的alloc未使用，那么直接使用pool->large 
            if (last->alloc == NULL) {
                large = last;
                last = NULL;
                break;
            }

            if (last->next == NULL) {
                break;
            }
        }
    }
    //如果没有可用的large节点，那么就使用ngx_palloc创建
    if (large == NULL) {
        if (!(large = ngx_palloc(pool, sizeof(ngx_pool_large_t)))) {
            return NULL;
        }

        large->next = NULL;
    }
//分配p节点 内存
#if 0
    if (!(p = ngx_memalign(ngx_pagesize, size, pool->log))) {
        return NULL;
    }
#else
    if (!(p = ngx_alloc(size, pool->log))) {
        return NULL;
    }
#endif
    //如果pool->large==NULL 那么就直接使用刚才创建的large节点
    if (pool->large == NULL) {
        pool->large = large;

    } else if (last) {
        last->next = large;
    }

    large->alloc = p;

    return p;
}
```
待解决问题
1.ngx_align为什么需要内存对齐


参考
http://blog.csdn.net/daniel_ustc/article/details/11645293
http://lib.csdn.net/article/liveplay/47854