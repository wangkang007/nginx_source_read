# nginx_pool源码解析
## 前言
一直都听说nginx的内存管理比较好，所以这里从0.1.0版本开始看，低版本也许有很多可以优化的地方，但是更能看清楚nginx模块最初的逻辑，这里讲解的可能会跟网上有不一样的地方,这个也是最原始代码，但是会该类文章会讲解每个版本的变动是什么，这样就能更好的更简单的去理解nginx
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

nginx的内存池是一个很好的设计，在程序关闭的时候能够一次free掉所有的内存，nginx把内存分成两种类型 `ngx_pool_s`和` ngx_pool_large_s` ，分别是小内存块和大内存块，那什么样的内存叫做小内存块？在nginx的定义中只要小于页内存并且小于当前内存池内存块内存的都是小内存块，反之是大内存块
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
    //size小于页内存并且小于当前当前内存池的内存块的内存
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

## 内存销毁

```c
//遍历所有内存块节点销毁指定的p区域的内存
ngx_int_t ngx_pfree(ngx_pool_t *pool, void *p)
{
    ngx_pool_large_t  *l;

    for (l = pool->large; l; l = l->next) {
        if (p == l->alloc) {
            ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, pool->log, 0,
                           "free: " PTR_FMT, l->alloc);
            free(l->alloc);
            l->alloc = NULL;

            return NGX_OK;
        }
    }

    return NGX_DECLINED;
}
//销毁所有内存
void ngx_destroy_pool(ngx_pool_t *pool)
{
    ngx_pool_t        *p, *n;
    ngx_pool_large_t  *l;
    //优先销毁large内存
    for (l = pool->large; l; l = l->next) {

        ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, pool->log, 0,
                       "free: " PTR_FMT, l->alloc);
        //判断是否分配了内存 如果分配了直接free
        if (l->alloc) {
            free(l->alloc);
        }
    }
//如果开启了nginx debug 就打印 pool中的内存情况
#if (NGX_DEBUG)

    /*
     * we could allocate the pool->log from this pool
     * so we can not use this log while the free()ing the pool
     */

    for (p = pool, n = pool->next; /* void */; p = n, n = n->next) {
        ngx_log_debug2(NGX_LOG_DEBUG_ALLOC, pool->log, 0,
                       "free: " PTR_FMT ", unused: " SIZE_T_FMT,
                       p, p->end - p->last);

        if (n == NULL) {
            break;
        }
    }

#endif
    //销毁正常的内存 poll
    for (p = pool, n = pool->next; /* void */; p = n, n = n->next) {
        free(p);

        if (n == NULL) {
            break;
        }
    }
}
```
待解决问题
1.ngx_align为什么需要内存对齐
2.画图使用内存池的时候内存分布情况


参考
内存分部情况
https://www.cnblogs.com/xiekeli/archive/2012/10/17/2727432.html
http://blog.csdn.net/daniel_ustc/article/details/11645293
http://lib.csdn.net/article/liveplay/47854