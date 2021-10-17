# 一、Nginx 优化静态资源加载 解决 Waiting(TTFB)时间过长问题

# 前因后果

网站每次加载都需要等待两三秒，一直以为是带宽问题（因为带宽真的小，钱的问题），后来开了全站 CDN 加速依然没有解决问题，今天正好没事就研究研究。

如图：多个静态文件 Waiting(TTFB) 时间过长

![image](https://github.com/Mountains-and-rivers/nginx-cache/blob/main/image/01.png)

如图：Waiting(TTFB) 时间过长

![image](https://github.com/Mountains-and-rivers/nginx-cache/blob/main/image/02n.png)

都是静态资源，文件也不大，试了全站 CDN 和单个文件 CDN 没有任何效果，后来怀疑是Nginx 配置问题。经过查询文档检查配置最终找到问题。

解决办法第一步，启用缓存
Nginx 静态资源配置了禁用缓存，导致每次都重新加载。开启缓存后刷新就很快了，但是第一次加载依然是两三秒。

```
开启缓存静态资源
```

```
# 开启缓存，关闭静态资源日志记录，节省服务器资源
location ~ .*\.(gif|jpg|jpeg|png|ico|css|js|woff|woff2|ttf)$ {
    root	/usr/xxx;
    #禁用缓存
    #add_header Cache-Control no-cache; 
    # 关闭日志
    access_log off;
    #缓存7天
    expires 7d;		
}
```

# 解决办法第二步，启用 `gzip` 压缩

第一次加载依然是两三秒，解决办法是开启静态资源压缩，速度瞬间提升 20 倍哈哈。

> 在 `http` 模块加入以下配置

```
# 开启gzip
gzip  on;
# 启用gzip压缩的最小文件，小于设置值的文件将不会压缩
gzip_min_length 1k;
# gzip 压缩级别，1-10，数字越大压缩的越好，也越占用CPU时间。一般设置1和2
gzip_comp_level 1;
# 进行压缩的文件类型。javascript有多种形式。其中的值可以在 mime.types 文件中找到。
gzip_types text/plain application/javascript application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;
# 是否在http header中添加Vary: Accept-Encoding，建议开启
gzip_vary on;
# 禁用IE 8 gzip
gzip_disable "MSIE [1-8]\.";
# 设置缓存路径并且使用一块最大100M的共享内存，用于硬盘上的文件索引，包括文件名和请求次数，每个文件在1天内若不活跃（无请求）则从硬盘上淘汰，硬盘缓存最大10G，满了则根据LRU算法自动清除缓存。
proxy_cache_path /usr/local/nginx/cache/ levels=1:2 keys_zone=imgcache:100m inactive=1d max_size=10g;
```

```
再来看看现在，静态资源加载的速度，保持在100ms以内
```

![image](https://github.com/Mountains-and-rivers/nginx-cache/blob/main/image/03.png)

解决办法第三步，启用 http2 协议（可选）
HTTP/2（超文本传输协议第2版，最初命名为HTTP 2.0），简称为h2（基于TLS/1.2或以上版本的加密连接）或h2c（非加密连接），是HTTP协议的的第二个主要版本，使用于万维网。

```
如果没有安装http_v2_module模块，则需要重新编译安装。
```

```
./configure --user=nginx --group=nginx --prefix=/usr/local/nginx/ --with-http_addition_module --with-http_flv_module --with-http_gzip_static_module --with-http_realip_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_dav_module --with-http_v2_module --with-http_geoip_module --with-stream --with-stream=dynamic
```

Nginx 启用`http2`并优化`https`性能， 从而获得更好的 TTFB 和减少的延迟。

```
server {
    listen 443 ssl;
    # 改为
    listen 443 ssl http2;
    
    # https 优化
    # 减少SSL缓冲区大小，默认情况下，缓冲区为16k，这是一种“一刀切”的方法，旨在应对较大的响应。但是，为了最大程度地减少TTFB，通常最好使用较小的值。
    ssl_buffer_size 4k;
    ssl_prefer_server_ciphers on;
    ssl_protocols TLSv1.2 TLSv1.3; # 最低支持1.2 协议配置
    # 使用http2并启用Nginx ssl_session_cache将确保初始连接的HTTPS性能更快，并且页面加载速度快于http。
    ssl_session_cache shared:SSL:1m; # 可容纳约4000个会话
    ssl_session_timeout 24h; # 24小时，在此期间可以重复使用会话
    # 由于Nginx尚未正确实现会话票证加密密钥的轮换，因此将其关闭。
    ssl_session_tickets off;
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate cert/xxxxx.pem; # 证书地址
    resolver 8.8.8.8 202.106.0.20 valid=300s;
    resolver_timeout 5s;
}
```

> 参考资料

https://cloud.tencent.com/developer/article/1430637

https://haydenjames.io/nginx-tuning-tips-tls-ssl-https-ttfb-latency/

# 二、nginx提高加载静态文件速度

1.本来对于静态网页，我们不需要放在应用容器中，原因一时由于应用服务器是用来解析动态网页的，针对静态网页本来就性能不高，而且还会占用应用容器的资源，所以我们专门使用nginx用来解析静态网页。

2.当我们使用nginx解析静态网页的时候，网页在加载静态网页的时候的确时很快了，但是当静态网页的大小(size）非常大（因为会包含很多图片）的时候就会加载也会慢，所以我们需要更快地加载网页。

3.我们该怎么使用nginx更快的加载这些静态网页呢？开启nginx的gzip压缩

现在我们在测试一下，访问一个网页正常使用nginx加载需要传输多大size的流量，可以看到一个网页文本7.7k，两张图片分别时11.9k和7.6k

![image](https://github.com/Mountains-and-rivers/nginx-cache/blob/main/image/04.png)

现在我们来配置一下nginx的配置文件里面开启gzip压缩

```
  gzip  on;
    gzip_comp_level  5;
    gzip_min_length  1024;
    gzip_types   text/plain application/x-javascript text/css application/xml text/javascript  image/jpeg image/gif image/png;
```

![image](https://github.com/Mountains-and-rivers/nginx-cache/blob/main/image/05.png)

现在我们可以看到压缩传输后的结果

 4.我们同样可以开启静态文件在客户端进行缓存，那么就不必要从服务端重新获取了，这样也能提高客户端的加载速度

我们在nginx里面的conf/nginx.conf文件开启缓存

![image](https://github.com/Mountains-and-rivers/nginx-cache/blob/main/image/06.png)

这样我们在刷新请求网页第二次的时候，就是从缓存里面获取图片了，这样加载速度就更快了

![image](https://github.com/Mountains-and-rivers/nginx-cache/blob/main/image/07.png)

# 三、Nginx性能优化功能-Gzip压缩(大幅度提高页面加载速度)

```
Nginx开启Gzip压缩功能， 可以使网站的css、js 、xml、html 文件在传输时进行压缩，提高访问速度, 进而优化Nginx性能! Web网站上的图片，视频等其它多媒体文件以及大文件，因为压缩效果不好，所以对于图片没有必要支压缩，如果想要优化，可以图片的生命周期设置长一点，让客户端来缓存。 开启Gzip功能后，Nginx服务器会根据配置的策略对发送的内容, 如css、js、xml、html等静态资源进行压缩, 使得这些内容大小减少，在用户接收到返回内容之前对其进行处理，以压缩后的数据展现给客户。这样不仅可以节约大量的出口带宽，提高传输效率，还能提升用户快的感知体验, 一举两得; 尽管会消耗一定的cpu资源，但是为了给用户更好的体验还是值得的。

经过Gzip压缩后页面大小可以变为原来的30%甚至更小，这样，用户浏览页面的时候速度会快得多。Gzip 的压缩页面需要浏览器和服务器双方都支持，实际上就是服务器端压缩，传到浏览器后浏览器解压并解析。浏览器那里不需要我们担心，因为目前的巨大多数浏览器 都支持解析Gzip过的页面。
```

Gzip压缩作用：将响应报⽂发送⾄客户端之前可以启⽤压缩功能，这能够有效地节约带宽，并提⾼响应⾄客户端的速度。Gzip压缩可以配置http,[server](https://www.centos.bz/tag/server/)和[location](https://www.centos.bz/tag/location/)模块下。Nginx开启Gzip压缩功能的配置如下:

```
#修改nginx配置文件 /usr/local/nginx/conf/nginx.conf
[root@localhost ~]# vim /usr/local/nginx/conf/nginx.conf        #将以下配置放到nginx.conf的http{ ... }节点中

#修改配置为
gzip on;                    #开启gzip压缩功能
gzip_min_length 10k;         #设置允许压缩的页面最小字节数; 这里表示如果文件小于10个字节，就不用压缩，因为没有意义，本来就很小.
gzip_buffers 4 16k;         #设置压缩缓冲区大小，此处设置为4个16K内存作为压缩结果流缓存
gzip_http_version 1.1;      #压缩版本
gzip_comp_level 2;   #设置压缩比率，最小为1，处理速度快，传输速度慢；9为最大压缩比，处理速度慢，传输速度快; 这里表示压缩级别，可以是0到9中的任一个，级别越高，压缩就越小，节省了带宽资源，但同时也消耗CPU资源，所以一般折中为6
gzip types text/css text/xml application/javascript;      #制定压缩的类型,线上配置时尽可能配置多的压缩类型!
gzip_disable "MSIE [1-6]\.";       #配置禁用gzip条件，支持正则。此处表示ie6及以下不启用gzip（因为ie低版本不支持）
gzip vary on;    #选择支持vary header；改选项可以让前端的缓存服务器缓存经过gzip压缩的页面; 这个可以不写，表示在传送数据时，给客户端说明我使用了gzip压缩
```

## 线上使用的Gzip压缩配置

```
[root@external-lb02 ~]# cat /data/nginx/conf/nginx.conf
........
http {
.......
    gzip  on;
    gzip_min_length  1k;
    gzip_buffers     4 16k;
    gzip_http_version 1.1;
    gzip_comp_level 9;
    gzip_types       text/plain application/x-javascript text/css application/xml text/javascript application/x-httpd-php application/javascript application/json;
    gzip_disable "MSIE [1-6]\.";
    gzip_vary on;

}
```

如果不开启Gzip压缩功能(即注释掉Gzip的相关配置), 查看某个图片大小

```
[root@external-lb02 ~]#  ll  -h /data/web//www/test.bmp
-rw-r--r-- 1 root root 453K 3月  14 18:43 /data/web//www/test.bmp
```

如下可知, 文件没有被压缩,文件传输大小还是400多K

![image](https://github.com/Mountains-and-rivers/nginx-cache/blob/main/image/08.png)

如果开启Nginx的Gzip压缩功能(即打开Gzip的相关配置), 然后再次访问test.bmp图片, 发现压缩后的该图片文件传输大小只有200多K !

![image](https://github.com/Mountains-and-rivers/nginx-cache/blob/main/image/09.png)

通过上面测试对比, 发现Nginx开启Gzip压缩功能后, 定义的gzip type的文件在传输时的大小明显变小, 这样这会大大提高nginx访问性能.

直接用curl测试命令:

```
[root@fvtlb02 ~]# curl -I -H "Accept-Encoding: gzip, deflate" "http://fvtvfc-web.kevin.com/service-worker.js"
HTTP/1.1 200 OK
Server: nginx/1.12.2
Date: Mon, 26 Nov 2018 02:19:16 GMT
Content-Type: application/javascript; charset=utf-8
Connection: keep-alive
Vary: Accept-Encoding
Last-Modified: Sun, 25 Nov 2018 22:28:15 GMT
Vary: Accept-Encoding
ETag: W/"5bfb21ff-40be"
Content-Encoding: gzip

如上,response header头信息中出现"Conten_Encoding: gzip" , 就说明Nginx已开启了压缩 (在浏览器访问, 通过F12看请求的响应头部 也是一样)
```

## **Nginx的Gzip压缩功能虽然好用，但是下面两类文件资源不太建议启用此压缩功能。**

**1) 图片类型资源 (还有视频文件)**

原因：图片如jpg、png文件本身就会有压缩，所以就算开启gzip后，压缩前和压缩后大小没有多大区别，所以开启了反而会白白的浪费资源。（可以试试将一张jpg图片压缩为zip，观察大小并没有多大的变化。虽然zip和gzip算法不一样，但是可以看出压缩图片的价值并不大）

**2) 大文件资源**

原因：会消耗大量的cpu资源，且不一定有明显的效果。
