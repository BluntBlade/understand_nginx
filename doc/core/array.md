# 解码Nginx：数组  

梁涛（@无锋之刃）  
2012-11-24  

nginx-1.2.5  

## 设计思路  

## 数据结构  

            ngx_array_create(pool, n, size)
             |
             +- ngx_palloc()-----\
             |                   |
             \- ngx_palloc()-----+------------\
                                 |            |
              ngx_array_t        V            |
             +-------------------+            |
    /------- | elts              |            |
    |        +-------------------+            |
    |        | nelts             | -------\   |
    |        +-------------------+        |   |
    |    /-- | size        bytes |        |   |
    |    |   +-------------------+        |   |
    |    |   | nalloc        = n | --\    |   |
    |    |   +-------------------+   |    |   |
    |    |   | pool              |   |    |   |
    |    |   +-------------------+   |    |   |
    |    |                           |    |   |
    |    |                           |    |   |
    |    |                       /---+----+---+--+-- ngx_array_push()/
    |    |                       |   |    |      |   ngx_array_push_n()
    |    |                       |   |    |      |    |
    |    |                       V   |    |      |    \- ngx_palloc()--\
    \----+-> +-------------------+  -+-  -+-     |                     |
         |   | elem 0            |   A    A      \---------------------/
         |   /                   /   |    |
         |   /                   /   |    |
         |   |                   |   |    |
         |   +-------------------+   |    |
         |   | elem 1            |   |    |
         |   /                   /   |    |
         |   /                   /   |    |
         |   |                   |   |    |
        -+-  +-------------------+   |    |
         A   | elem 2            |   |    |
         |   /                   /   |    |
         |   /                   /   |    |
         V   |                   |   |    V
        ---  +-------------------+   |   ---
             | unpushed          |   |
             / elements          /   |
             /                   /   |
             /                   /   |
             /                   /   |
             /                   /   |
             /                   /   |
             /                   /   |
             /                   /   |
             |                   |   V
             +-------------------+  --- 

## 数组接口函数  