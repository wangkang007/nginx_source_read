# ngx_rbtree源码解析

## 1. 数据结构
```c
typedef struct ngx_rbtree_s  ngx_rbtree_t;

struct ngx_rbtree_s {
   ngx_int_t       key;
   ngx_rbtree_t   *left;//左节点
   ngx_rbtree_t   *right;//右节点
   ngx_rbtree_t   *parent;//父节点
   char            color;//颜色
};

```
## 2. 红黑树简介


## 参考
http://blog.csdn.net/pengyelovehk/article/details/52131728