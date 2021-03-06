http://hansionxu.blog.163.com/blog/static/24169810920147711260835/

```
Nginx下让SSL支持SPDY协议  

2014-08-07 11:26:00|  分类： web开发 |  标签：spdy  http2.0  ssl   |举报|字号 订阅
    
下载LOFTER客户端
SPDY介绍

SPDY 是 Google 开发的基于传输控制协议（TCP）的应用层协议，开发组正在推动 SPDY 成为正式标准（现为互联网草案）。SPDY 协议类似于 HTTP，但旨在缩短网页的加载时间和提高安全性，通过压缩、多路复用和优先级来缩短加载时间。



Google Chrome 用户打开 chrome://net-internals/#spdy 就会发现你已经在使用 SPDY 协议了。





SPDY 与 HTTP 的关系

　　SPDY 协议只是在性能上对 HTTP 做了很大的优化，其核心思想是尽量减少连接个数，而对于 HTTP 的语义并没有做太大的修改。具体来说是，SPDY 使用了 HTTP 的方法和页眉，但是删除了一些头并重写了 HTTP 中管理连接和数据转移格式的部分，所以基本上是兼容 HTTP 的。



　　Google 在 SPDY 白皮书里表示要向协议栈下面渗透并替换掉传输层协议（TCP），但是因为这样无论是部署起来还是实现起来暂时相当困难，因此 Google 准备先对应用层协议 HTTP 进行改进，先在 SSL 之上增加一个会话层来实现 SPDY 协议，而 HTTP 的 GET 和 POST 消息格式保持不变，即现有的所有服务端应用均不用做任何修改。



　　因此在目前，SPDY 的目的是为了加强 HTTP，是对 HTTP 一个更好的实现和支持。至于未来 SPDY 得到广泛应用后会不会演一出狸猫换太子，替换掉 HTTP 并彻底颠覆整个 Internet 就是 Google 的事情了。



　　为什么要重新建立一个 SPDY ？

　　距离万维网之父蒂姆·伯纳斯-李发明并推动 HTTP 成为如今互联网最流行的协议已经过去十几年了（现用 HTTP 1.1 规范也停滞了 10多年了），随着现在 WEB 技术的飞速发展尤其是 HTML5 的不断演进，包括 WebSockets 协议的出现以及当前网络环境的改变、传输内容的变化，当初的 HTTP 规范已经逐渐无法满足人们的需要了，HTTP 需要进一步发展，因此 HTTPbis 工作组已经被组建并被授权考虑 HTTP 2.0 ，希望能解决掉目前 HTTP 所带来的诸多限制。而 SPDY 正是 Google 在 HTTP 即将从 1.1 跨越到 2.0 之际推出的试图成为下一代互联网通信的协议，长期以来一直被认为是 HTTP 2.0 唯一可行选择。



HTTP 协议的不足

　　1. 单路连接 请求低效

　　HTTP 协议的最大弊端就是每个 TCP 连接只能对应一个 HTTP 请求，即每个 HTTP 连接只请求一个资源，浏览器只能通过建立多个连接来解决。此外在 HTTP 中对请求是严格的先入先出（FIFO）进行的，如果中间某个请求处理时间较长会阻塞后面的请求。

　　（注：虽然 HTTP pipelining 对连接请求做了改善，但复杂度增加很大，并未普及）



　　2. HTTP 只允许由客户端主动发起请求

　　服务端只能等待客户端发送一个请求，在可以满足预加载的现状是一种桎梏。



　　3. HTTP 头冗余

　　HTTP 头在同一个会话里是反复发送的，中间的冗余信息，比如 User-Agent、Host 等不需要重复发送的信息也在反复发送，浪费带宽和资源。



SPDY 协议的优点

　　1. 多路复用 请求优化

　　SPDY 规定在一个 SPDY 连接内可以有无限个并行请求，即允许多个并发 HTTP 请求共用一个 TCP会话。这样 SPDY 通过复用在单个 TCP 连接上的多次请求，而非为每个请求单独开放连接，这样只需建立一个 TCP 连接就可以传送网页上所有资源，不仅可以减少消息交互往返的时间还可以避免创建新连接造成的延迟，使得 TCP 的效率更高。



　　此外，SPDY 的多路复用可以设置优先级，而不像传统 HTTP 那样严格按照先入先出一个一个处理请求，它会选择性的先传输 CSS 这样更重要的资源，然后再传输网站图标之类不太重要的资源，可以避免让非关键资源占用网络通道的问题，提升 TCP 的性能。



　　2. 支持服务器推送技术

　　服务器可以主动向客户端发起通信向客户端推送数据，这种预加载可以使用户一直保持一个快速的网络。



　　3. SPDY 压缩了 HTTP 头

　　舍弃掉了不必要的头信息，经过压缩之后可以节省多余数据传输所带来的等待时间和带宽。



　　4. 强制使用 SSL 传输协议

　　Google 认为 Web 未来的发展方向必定是安全的网络连接，全部请求 SSL 加密后，信息传输更加安全。



SPDY 协议的意义

　　按照 Google 的说法，SPDY 被创造出来的唯一目的就是让 Web 更快（strive to make the whole web fast），其名字 SPDY（Speedy） 也似乎在暗示着这一点。那么 SPDY 的意义又在哪里呢？



　　1. 普通用户：

　　对于使用者来说，隐藏在浏览器下面的 SPDY 相比 HTTP 没有任何区别，但是我们可以感觉到 Google 服务在 Chrome 下异常的快，这就是 SPDY 的功劳了。此外网站信息传输加密后不用担心信息被截取等，大大增加了安全性和保密性。



　　2. 前端人员：

　　对于前端工程师们来说，提升页面效率是一件很重要的事情，目前大多采用像 CSS Sprites 等方法来优化网站，对于因为页面加载时每张图片、icon 都请求一个连接甚至采用在不同页面引用不同图片来降低一个页面内图片的请求数量。而现在有了 SPDY 的请求优化可以将请求顺序进行重排，这样可以在很大程度上缓解页面加载时图片请求带来的影响。例如像极客公园的报名页面，如果报名用户过多，例如极客公园2012年创新大会或极客公园第 27 期长城会，可以很明显的感觉出头像的请求会拖累整体页面加载变慢甚至变卡，相信对于这点，经常上淘宝或刷微博的会深有体会，一旦网速稍微慢点就会出现页面加载异常，还有像苹果 App Store（除去服务器因为地区的延迟），豌豆荚这类应用分发平台上应用图标刷新缓慢等。



　　3. 运维人员：

　　SPDY 在降低连接数目的同时，还使得服务器上每个客户端占用的资源也减少，从而可以释放出更多内存和 CPU 。此外 SPDY 综合起来可以将浏览速度提升一倍，页面加载延迟方面的改进达 64% 。



Nginx的配置SPDY方法(ssl)：

准备SSL：

wget http://www.openssl.org/source/openssl-1.0.1i.tar.gz
tar zxvpf openssl-1.0.1i.tar.gz

然后下载最新的 Nginx，解压、编译、安装：



wget http://nginx.org/download/nginx-1.7.4.tar.gz
 
tar zxvpf nginx-1.7.4.tar.gz
 
cd nginx-1.7.4
 
./configure --with-http_ssl_module --with-http_spdy_module --with-openssl=/mark/openssl-1.0.li
 
make
 
make install


Nginx配置spdy的反向代理服务器配置文件：



server {
 
    listen      80;
 
    server_name b2b.xiaoman.cn;
 
    rewrite     ^   https://$server_name$request_uri? permanent;
 
}
 
server {
 
        listen       443 ssl spdy;
 
        server_name  test.com;
 
        ssl_certificate      ssl/xiaoman.cn.chained.crt; #.pem
 
        ssl_certificate_key  ssl/xiaoman.cn.pem; #.key
 
        ssl_session_cache    shared:SSL:1m;
 
        ssl_session_timeout  5m;
 
        ssl_ciphers  HIGH:!aNULL:!MD5;
 
        ssl_prefer_server_ciphers  on;
 
        location / {
 
                proxy_pass  http://115.1.1.1:8000;
 
                proxy_redirect     off;
 
                proxy_set_header   Host             $host;
 
                proxy_set_header   X-Real-IP        $remote_addr;
 
                proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
 
                proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
 
                proxy_max_temp_file_size 0;
 
                proxy_connect_timeout      90;
 
                proxy_send_timeout         90;
 
                proxy_read_timeout         90;
 
                proxy_buffer_size          4k;
 
                proxy_buffers              4 32k;
 
                proxy_busy_buffers_size    64k;
 
                proxy_temp_file_write_size 64k;
 
        }
 
}




查看chrome中SPDY的使用情况（下图是xiaoman.cn的使用）：chrome://net-internals/#spdy
```
![11](http://img1.ph.126.net/ncKnPn8UC6Ncj7NqNwIc9g==/6608920302981006994.jpg)
