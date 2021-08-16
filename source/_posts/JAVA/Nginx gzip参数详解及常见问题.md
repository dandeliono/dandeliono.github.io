# Nginx gzip参数详解及常见问题
## 1、Nginx gzip 功能

Nginx 实现资源压缩的原理是通过 ngx_http_gzip_module 模块拦截请求，并对需要做 gzip 的类型做 gzip，ngx_http_gzip_module 是 Nginx 默认集成的，**不需要重新编译，直接开启即可**。

## 2、参数详解

### gzip on

这个没的说，打开或关闭 gzip

### **gzip_buffers**

设置用于处理请求压缩的缓冲区数量和大小。比如 32 4K 表示按照内存页（one memory page）大小以 4K 为单位（即一个系统中内存页为 4K），申请 32 倍的内存空间。建议此项不设置，使用默认值。

### **gzip_comp_level**

设置 gzip 压缩级别，级别越底压缩速度越快文件压缩比越小，反之速度越慢文件压缩比越大

我们以一个大小为 92.6K 的脚本文件为例，如下所示。其中最后三个数值分别表示压缩比、包大小、平均处理时间（使用 ab 压测，100 用户并发下， `./ab -n 10000 -c 100 -H 'Accept-Encoding: gzip' http://10.27.180.75/jquery.js` ）以及 CPU 消耗。

从这我们可以得出结论：

-   随着压缩级别的升高，压缩比有所提高，但到了级别 6 后，很难再提高；
-   随着压缩级别的升高，处理时间明显变慢;
-   gzip 很消耗 cpu 的性能，高并发情况下 cpu 达到 100%;

因此，建议：

-   不是压缩级别越高越好，其实 gzip_comp_level 1 的压缩能力已经够用了，后面级别越高，压缩的比例其实增长不大，反而很吃处理性能。
-   压缩一定要和静态资源缓存相结合，缓存压缩后的版本，否则每次都压缩高负载下服务器肯定吃不住。

### **gzip_disable**

通过表达式，表明哪些 UA 头不使用 gzip 压缩

### **gzip_min_length**

当返回内容大于此值时才会使用 gzip 进行压缩, 以 K 为单位, 当值为 0 时，所有页面都进行压缩。

### **gzip_http_version**

用于识别 http 协议的版本，早期的浏览器不支持 gzip 压缩，用户会看到乱码，所以为了支持前期版本加了此选项。默认在 http/1.0 的协议下不开启 gzip 压缩。

因为浏览器基本上都支持 HTTP/1.1。然而这里面却存在着一个很容易掉入的坑，也是笔者从生产环境中一个诡异问题中发现的：

**明明开启 gzip 功能，但是未生效。** 

**原因定位：** 为什么这样呢？  
在应用服务器前，公司还有一层 Nginx 的集群作为七层负责均衡，在这一层上，是没有开启 gzip 的。

如果我们使用了 proxy_pass 进行反向代理，那么 nginx 和后端的 upstream server 之间默认是用 HTTP/1.0 协议通信的。  
如果我们的 Cache Server 也是 nginx，而前端的 nginx 没有开启 gzip。  
同时，我们后端的 nginx 上没有设置 gzip_http_version 为 1.0，那么 Cache 的 url 将不会进行 gzip 压缩。

**[![](https://images2018.cnblogs.com/blog/1076553/201806/1076553-20180625145720171-2067830880.png)
](https://images2018.cnblogs.com/blog/1076553/201806/1076553-20180625145720171-2067830880.png)**

我相信，以后还有人会入坑，比如你用 Apache ab 做压测，如果不是设置 gzip_http_version 为 1.0，你也压不出 gzip 的效果（同样的道理）。希望写在这里对大家有帮助

### **gzip_proxied**

Nginx 做为反向代理的时候启用：

-   off – 关闭所有的代理结果数据压缩
-   expired – 如果 header 中包含”Expires” 头信息，启用压缩
-   no-cache – 如果 header 中包含”Cache-Control:no-cache” 头信息，启用压缩
-   no-store – 如果 header 中包含”Cache-Control:no-store” 头信息，启用压缩
-   private – 如果 header 中包含”Cache-Control:private” 头信息，启用压缩
-   no_last_modified – 启用压缩，如果 header 中包含”Last_Modified” 头信息，启用压缩
-   no_etag – 启用压缩，如果 header 中包含 “ETag” 头信息，启用压缩
-   auth – 启用压缩，如果 header 中包含 “Authorization” 头信息，启用压缩
-   any – 无条件压缩所有结果数据

### **gzip_vary**

增加响应头”Vary: Accept-Encoding”

### **gzip_types**

设置需要压缩的 MIME 类型, 如果不在设置类型范围内的请求不进行压缩

[参考文章](https://www.darrenfang.com/2015/01/setting-up-http-cache-and-gzip-with-nginx/)

所以 MIME-TYPE 中应该新增字体类型：

| 字体类型扩展名 | Content-type |
| .eot | application/vnd.ms-fontobject |
| .ttf | font/ttf |
| .otf | font/opentype |
| .woff | font/x-woff |
| .svg | image/svg+xml |

\_\_EOF\_\_ 
 [https://www.cnblogs.com/xzkzzz/p/9224358.html](https://www.cnblogs.com/xzkzzz/p/9224358.html)
