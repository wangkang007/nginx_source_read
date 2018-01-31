# ngx_list源码解析
## 1. 数据结构
```c
struct ngx_list_part_s {
    void             *elts;////首元素地址
    ngx_uint_t        nelts; //数组中已经使用的元素个数
    ngx_list_part_t  *next;//下一个节点
};


typedef struct {
    ngx_list_part_t  *last;
    ngx_list_part_t   part;//首节点的list
    size_t            size;//每个元素占用的内存大小
    ngx_uint_t        nalloc;//当前数组中能够容纳元素的个数
    ngx_pool_t       *pool;
} ngx_list_t;
```

## 2. 初始化

```c

ngx_inline static ngx_int_t ngx_list_init(ngx_list_t *list, ngx_pool_t *pool,
                                          ngx_uint_t n, size_t size)
{
    //分配首节点中的 list内存n*size
    if (!(list->part.elts = ngx_palloc(pool, n * size))) {
        return NGX_ERROR;
    }

    list->part.nelts = 0;
    list->part.next = NULL;
    list->last = &list->part;
    list->size = size;
    list->nalloc = n;
    list->pool = pool;
    return NGX_OK;
}
```
## 3.push元素

```c
void *ngx_list_push(ngx_list_t *l)
{
    void             *elt;
    ngx_list_part_t  *last;

    last = l->last;
    //  如果超过了容量
    if (last->nelts == l->nalloc) {

        /* the last part is full, allocate a new list part */
        //分配一个新的节点
        if (!(last = ngx_palloc(l->pool, sizeof(ngx_list_part_t)))) {
            return NULL;
        }
        //分配新节点的存储空间
        if (!(last->elts = ngx_palloc(l->pool, l->nalloc * l->size))) {
            return NULL;
        }

        last->nelts = 0;
        last->next = NULL;
        //将当前list的last的next指向 新建的节点
        l->last->next = last;
        l->last = last;
    }
    //指针移动的到已经使用元素地址的末尾
    elt = (char *) last->elts + l->size * last->nelts;
    last->nelts++;

    return elt;
}

```