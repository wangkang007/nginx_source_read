# nginx_buf源码解析
## 前言

## 1.数据结构
```c
typedef void *            ngx_buf_tag_t;

typedef struct ngx_buf_s  ngx_buf_t;

struct ngx_buf_s {
    u_char          *pos;//待处理数据的开始标记
    u_char          *last;//待处理数据的结尾标记
    off_t            file_pos;//处理文件时，待处理的文件开始标记
    off_t            file_last;//处理文件时，待处理的文件结尾标记

    int              type;
    u_char          *start;  //buffer的起始地址       /* start of buffer */
    u_char          *end;    //buffer的end地址       /* end of buffer */
    ngx_buf_tag_t    tag;//表示当前缓冲区的类型，例如由哪个模块使用就指向这个模块ngx_module_t变量的地址
    ngx_file_t      *file; // 引用的文件
     /*
     * 当前缓冲区的影子缓冲区，该成员很少用到。当缓冲区转发上游服务器的响应时才使用了shadow成员，
     * 这是因为nginx太节约内存了，分配一块内存并使用ngx_buf_t表示接收到的上游服务器响应后，
     * 在向下游客户端转发时可能会把这块内存存储到文件中，也可能直接向下游发送，此时nginx绝对不会
     * 重新复制一份内存用于新的目的，而是再次建立一个ngx_buf_t结构体指向原内存，这样多个ngx_buf_t
     * 结构体指向了同一份内存，它们之间的关系就通过shadow成员来引用，一般不建议使用。
     */
    ngx_buf_t       *shadow;


    /* the buf's content could be changed */
    /* 临时内存标志位，为1时表示数据在内存中且这段内存可以修改 */
    unsigned         temporary:1;

    /*
     * the buf's content is in a memory cache or in a read only memory
     * and must not be changed
     */
    /* 标志位，为1时表示数据在内存中且这段内存不可以修改 */
    unsigned         memory:1;

    /* the buf's content is mmap()ed and must not be changed */
    /* 标志位，为1时表示这段内存是用nmap系统调用映射过来的，不可以修改 */
    unsigned         mmap:1;
 /* 标志位，为1时表示可回收 */
    unsigned         recycled:1;
    /* 标志位，为1时表示这段缓冲区处理的是文件而不是内存 */
    unsigned         in_file:1;
    /* 标志位，为1时表示需要执行flush操作 */
    unsigned         flush:1;
    /*
     * 标志位，表示是否是最后一块缓冲区，因为ngx_buf_t可以由ngx_chain_t链表串联起来，因此为1时，
     * 表示当前是最后一块待处理的缓冲区。   
     */
    unsigned         last_buf:1;
 /* 标志位，表示是否是最后一个影子缓冲区，与shadow域配合使用。通常不建议使用它 */
    unsigned         last_shadow:1;
     /* 标志位，表示当前缓冲区是否属于临时文件 */
    unsigned         temp_file:1;

    unsigned         zerocopy_busy:1;

    /* STUB */ int   num;
};
```

```c
typedef struct ngx_chain_s       ngx_chain_t;

struct ngx_chain_s {
    ngx_buf_t    *buf;
    ngx_chain_t  *next;
};

typedef struct {
    ngx_buf_t                   *buf;
    ngx_chain_t                 *in;
    ngx_chain_t                 *free;
    ngx_chain_t                 *busy;

    unsigned                     sendfile;
    unsigned                     need_in_memory;
    unsigned                     need_in_temp;

    ngx_pool_t                  *pool;
    ngx_int_t                    allocated;
    ngx_bufs_t                   bufs;
    ngx_buf_tag_t                tag;

    ngx_output_chain_filter_pt   output_filter;
    void                        *filter_ctx;
} ngx_output_chain_ctx_t;

```
## 参考文章
http://blog.csdn.net/initphp/article/details/50611021