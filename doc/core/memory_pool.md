# 解码Nginx：内存池  

梁涛（@无锋之刃）  
2012-11-17 ~ 2012-11-18  

nginx-1.2.5  

## 背景  

对于HTTP服务器而言，“每秒处理请求数”是非常关键的性能指标。处理路径上每一步动作的时间开销，都将对该指标造成影响。特别地，内存分配/释放作为最基础的动作，对总处理时间施加的影响更为巨大——想想那些伴随复杂数据结构或细碎逻辑产生的内存分配/释放动作，很可能在单次处理中执行数十次，假设每次动作都调用malloc()/free()函数，时间开销将颇为可观、无法接受。由此，Nginx针对HTTP应用的业务特点，重新设计出一套内存管理机制，以内存池的形式提供给上层结构使用，从而有效地优化性能。  

根据分配/释放频繁程度，HTTP应用的内存使用特征可以划分为两类：  

1. 进程初始化时分配、终止时释放的大块内存，供应给配置文件等全程不会变化的数据结构使用；  
2. 处理单次请求时反复分配/释放的小块内存，供应给字符串处理等小型结构体和临时数据使用。  

对于前者，使用malloc()/free()带来的时间开销并无大碍；对于后者，一旦使用完后即刻调用free()释放内存，则会产生许多不必要的时间开销。更理想的作法是将零碎分配得来的内存收集起来，在某个时间点集中释放掉，亦即“多次分配，一次释放”。通过设计得当的数据结构和接口，可以在确保“谁分配，谁释放”的内存使用原则下，提供更好的性能。  

## 内存管理模型  

Nginx将内存管理模型组织为两层结构的内存池：  

1. 第一层结构采用“向系统请求储备内存块（多次）——按需分配零碎内存块（多次）——集中释放零碎内存块（单次）”的设计思路，每次分配不超过某个固定长度上限的零碎内存块；  
2. 第二层结构采用简单的“向系统请求独立内存块（多次+转发）”的设计思路，亦即转发分配请求给malloc()，以分配第一层不负责管理的更大块的独立内存块。  

除此以外，还包装了malloc()/free()以及更为底层的memalign()/posix_memalign()，用以分配不归内存池管理的内存块。  

### 第一层结构  

    ngx_create_pool()              ngx_palloc()/ngx_pnalloc()        ngx_palloc()/ngx_pnalloc()        ...  
     |                              |                                 |  
     |                              \- ngx_palloc_block()             \- ngx_palloc_block()  
     |                                  |                                 |  
     \- ngx_memalign() ------\          \- ngx_memalign() ------\         \- ngx_memalign() ----\  
                             |                                  |                               |  
                             |                                  |                               |  
          ngx_pool_t         V                 ngx_pool_data_t  V              ngx_pool_data_t  V  
    /--> +-------------------+         /----> +-----------------+         /-> +-----------------+  /-> ...  
    |    | ngx_pool_data_t d |         |      | last            | --\     |   | last            |  |  
    |    |  +----------------+         |      +-----------------+   |     |   +-----------------+  |  
    |    |  | last           | --\     |      | end             | --+--\  |   | end             |  |  
    |    |  +----------------+   |     |      +-----------------+   |  |  |   +-----------------+  |  
    |    |  | end            | --+--\  |      | next            | --+--+--/   | next            |--/  
    |    |  +----------------+   |  |  |      +-----------------+   |  |      +-----------------+  
    |    |  | next           | --+--+--/      | failed          |   |  |      | failed          |  
    |    |  +----------------+   |  |         +-----------------+   |  |      +-----------------+  
    |    |  | failed         |   |  |         | allocated       |   |  |      | allocated       |  
    |    +--+----------------+   |  |         | area            |   |  |      | area            |  
    |    | max               |   |  |         /-----------------/ <-/  |      /-----------------/  
    |    +-------------------+   |  |         / unallocated     /      |      / unallocated     /  
    \--- | current           |   |  |         | area            |      |      | area            |  
         +-------------------+   |  |         |                 |      |      |                 |  
         | chain             | --+--+-----\   |                 |      |      |                 |  
         +-------------------+   |  |     |   |                 |      |      |                 |  
         | large             | --+--+--\  |   +-----------------+ <----/      +-----------------+  
         +-------------------+   |  |  |  |  
    /--- | cleanup           |   |  |  |  |  
    |    +-------------------+   |  |  |  |  
    |    | log               |   |  |  |  |  
    |    +-------------------+   |  |  |  \-> ???  
    |    | allocated         |   |  |  |  
    |    | area              |   |  |  |  
    |    /-------------------/ <-/  |  \----> 第二层结构  
    |    / unallocated       /      |         (ngx_pool_large_t)  
    |    | area              |      |  
    |    |                   |      |  
    |    |                   |      |  
    |    |                   |      |  
    |    +-------------------+ <----/  
    |  
    |  
    |     ngx_pool_cleanup_t  
    \--> +-------------------+  
         |  handler          |  
         +-------------------+  
         |  data             |  
         +-------------------+  
    /--- |  next             |  
    |    +-------------------+  
    |  
    |     ngx_pool_cleanup_t  
    \--> +-------------------+  
         |  handler          |  
         +-------------------+  
         |  data             |  
         +-------------------+  
    /--- |  next             |  
    |    +-------------------+  
    |  
    |   
    \--> ...

    注意点：  
      1. ngx_pool_t和ngx_pool_data_t结构体存放在分配出的储备内存块首址处，占用固定空间；  
      2. 内存长度大于max字段值的分配请求将转发给第二层结构处理；  
      3. 使用ngx_pool_data_t结构体构造出单向链表，管理各储备内存块；  
      4. 使用ngx_pool_cleanup_t结构体构造出单向链表，管理各清理回调函数。  

### 第二层结构  

    ngx_palloc()/ngx_pnalloc()          ngx_palloc()/ngx_pnalloc()  
     |                                   |  
     \- ngx_palloc_large() --\           \- ngx_palloc_large() --\  
                             |                                   |  
          ngx_pool_large_t   V                ngx_pool_large_t   V  
         +-------------------+       /-----> +-------------------+       /-----> ...  
         |  alloc            |       |       |  alloc            |       |  
         +-------------------+       |       +-------------------+       |  
         |  next             | ------/       |  next             | ------/  
         +-------------------+               +-------------------+  

    注意点：  
      1. ngx_pool_large_t结构体是从ngx_pool_data_t结构管理的储备内存块中分配出来的，并且会适当复用。  

## 内存池接口函数  

### ngx_create_pool(size, log)  

ngx_create_pool()负责创建内存池，动作序列如下：  

1. 向系统请求size参数指定大小的储备内存块（按NGX_POOL_ALIGNMENT对齐，默认16字节），逻辑上划分为（ngx_pool_t结构体＋剩余内存区）两部分；  
2. 初始化ngx_pool_data_t结构体各个字段（参考前一节，last字段指向初始分配地址）；  
3. 计算单次分配的内存块长度上限（max字段，最大不超过size参数值或NGX_MAX_ALLOC_FROM_POOL）；  
4. 初始化其它字段（current字段指向自己）。  

### ngx_destroy_pool(pool)  

ngx_destroy_pool()负责销毁内存池，动作序列如下：  

1. 调用各个清理回调函数；  
2. 释放各个ngx_pool_large_t结构体管理的储备内存块；  
3. 释放各个ngx_pool_data_t结构体管理的独立内存块。  

### ngx_reset_pool(pool)  

ngx_reset_pool()负责释放零碎内存块，动作序列如下：  

1. 释放各个ngx_pool_large_t结构体管理的内存块；  
2. 简单地将各个ngx_pool_data_t结构体的last字段复原成初始分配地址，而不是释放。  

### ngx_palloc(pool, size)/ngx_pnalloc(pool, size)/ngx_pcalloc(pool, size)  

ngx_palloc()负责分配零碎内存块，动作序列如下：  

1. 若size参数大于max字段值，转发给ngx_palloc_large()处理（调用更底层的分配函数）；  
2. 以current字段所指ngx_pool_data_t结构体为起点，在单向链表中搜索第一个能执行分配动作的结构体，完成分配（分配首址按NGX_ALIGNMENT对齐）；  
3. 若以上动作均告失败，转发给ngx_palloc_block()处理（先分配新的ngx_pool_data_t结构体，再分配零碎内存块）。  

ngx_pnalloc()动作与ngx_palloc()类似，但不对齐分配首址。  
ngx_pcalloc()将分配请求转发给ngx_palloc()处理，并对返回的零碎内存块进行清零初始化。  

### ngx_palloc_block(pool, size)  

ngx_palloc_block()负责向系统请求新的储备内存块，最终完成分配，动作序列如下：  

1. 向系统请求指定大小的储备内存块（按NGX_POOL_ALIGNMENT对齐，默认16字节），逻辑上划分为（ngx_pool_data_t结构体＋剩余内存区）两部分；  
2. 初始化各个字段（包括分配首址，按NGX_ALIGNMENT对齐）；  
3. 调整current字段的指向（实际上维持着一个“... 5 5 5 4 3 2 1 0 0”的failed字段值序列），将新的ngx_pool_data_t结构体挂入单向链表中。  

### ngx_palloc_large(pool, size)/ngx_pmemalign(pool, size, alignment)  

ngx_palloc_large负责分配不归第一层结构管理的独立内存块，动作序列如下：  

1. 调用ngx_alloc()分配新的独立内存块；  
2. 搜索第一个空闲的ngx_pool_large_t结构体，保存并返回独立内存块首址；  
3. 若前一步失败，从本内存池中分配新的ngx_pool_large_t结构体并加入单向链表中，保存并返回独立内存块首址。  

ngx_pmemalign()动作与ngx_palloc_large()类似，但会按alignment参数对齐独立内存块首址。  

### ngx_pfree(pool, p)  

ngx_pfree()负责“释放”内存块，动作序列如下：  

1. 如果待“释放”对象是独立内存块，则转调用ngx_free()释放。  

实际上该函数并不真正释放零碎内存块，而是尽可能将释放动作推迟到ngx_reset_pool()/ngx_destroy_pool()中。  

### ngx_pool_cleanup_add(p, size)  

ngx_pool_cleanup_add()负责分配新的ngx_pool_cleanup_t结构体，以便调用端注册回调函数。动作序列如下：  

1. 从本内存池中分配一个ngx_pool_cleanup_t结构体；  
2. 若size参数不为0，尝试从本内存池分配相应大小的一块内存并作为回调参数；  
3. 将新的结构体挂到单向链表中；  
4. 调整cleanup字段，保持回调函数按“先进后出”的性质（类似于压栈操作）。  

### ngx_pool_run_cleanup_file(p, fd)/ngx_pool_cleanup_file(data)/ngx_pool_cleanup_file(data)  

ngx_pool_run_cleanup_file()负责搜索注册在回调函数链表中的、与fd参数指定文件句柄对应的回调函数并调用之。  
ngx_pool_cleanup_file()提供一个关闭指定文件句柄的缺省实现函数。  
ngx_pool_delete_file()提供一个删除文件并关闭相关文件句柄的缺省实现函数。  
