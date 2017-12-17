# nginx_array源码解析
## 1. 数据结构
```c
struct ngx_array_s {
    void        *elts; //首元素地址
    ngx_uint_t   nelts; //数组中已经使用的元素个数
    size_t       size;//每个元素占用的内存大小
    ngx_uint_t   nalloc;//当前数组中能够容纳元素的个数 
    ngx_pool_t  *pool;//内存池对象 
};

```
## 2.创建
```c

#define ngx_test_null(p, alloc, rc)  if ((p = alloc) == NULL) { return rc; }

ngx_array_t *ngx_create_array(ngx_pool_t *p, ngx_uint_t n, size_t size)
{
    ngx_array_t *a;
    //分配ngx_array_t的内存
    ngx_test_null(a, ngx_palloc(p, sizeof(ngx_array_t)), NULL);
    //分配数组的内存 内存大小为 数组中的个数*size
    ngx_test_null(a->elts, ngx_palloc(p, n * size), NULL);

    a->pool = p;
    a->nelts = 0;
    a->nalloc = n;
    a->size = size;

    return a;
}

```

## 3.销毁

```c
void ngx_destroy_array(ngx_array_t *a)
{
    ngx_pool_t  *p;

    p = a->pool;
    //如果当前数组的尾地址 和pool尾指针地址一致，那么p->last就减去这段地址，相当于不对这个地址做访问，但是内存并没有真正销毁
    if ((char *) a->elts + a->size * a->nalloc == p->last) {
        p->last -= a->size * a->nalloc;
    }
    // 如果p->last的地址等于 a+sizeof(ngx_array_t)的地址，那么直接让p->last指向a
    if ((char *) a + sizeof(ngx_array_t) == p->last) {
        p->last = (char *) a;
    }
}
```

待补充内容:
1.操作内存时候的图
2.为什么要怎么操作内存，不直接释放内存

## 4.ngx_push_array放入元素
```c
void *ngx_push_array(ngx_array_t *a)
{
    void        *elt, *new;
    ngx_pool_t  *p;

    /* array is full */
    //判断如果 array已经满了
    if (a->nelts == a->nalloc) {
        p = a->pool;

        /* array allocation is the last in the pool */
        if ((char *) a->elts + a->size * a->nelts == p->last
            && (unsigned) (p->end - p->last) >= a->size)
        {
            p->last += a->size;
            a->nalloc++;

        /* allocate new array */
        } else {
            ngx_test_null(new, ngx_palloc(p, 2 * a->nalloc * a->size), NULL);

            ngx_memcpy(new, a->elts, a->nalloc * a->size);
            a->elts = new;
            a->nalloc *= 2;
        }
    }

    elt = (char *) a->elts + a->size * a->nelts;
    a->nelts++;

    return elt;
}
```