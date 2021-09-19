# 为网站开启Nginx缓存加速，支持html伪静态页面

## 一、代理模式

代理模式，即在使用 Nginx 反向代理时缓存指定内容，所用模块为 **proxy_cache**。这里网络上的很多教程会说，这个模式必须在反向代理中才能使用，说的好像不能用在只有一台服务器的情况似的。其实不然，我们用点小技巧，将 Nginx 本机的80端口代理转发到 本机的 8080 端口即可变相的开启反向代理模式，在这期间，就完全可以指定缓存内容了，且继续往下看！

### ①、下载模块

所用模块为 ngx_cache_purge，官方地址：http://labs.frickle.com/files/，我们可以挑选一个新版本下载到服务器上，比如 [http://labs.frickle.com/files/ngx_cache_purge-2.3.tar.gz](https://zhangge.net/goto/aHR0cDovL2xhYnMuZnJpY2tsZS5jb20vZmlsZXMvbmd4X2NhY2hlX3B1cmdlLTIuMy50YXIuZ3o=)

```
cd /usr/local/src
#下载
wget http://labs.frickle.com/files/ngx_cache_purge-2.3.tar.gz
#解压
tar zxvf ngx_cache_purge-2.3.tar.gz
```

### ②、重新编译

所以先执行 -V 命令查看 Nginx 是否已经编译了该模块，

```
[marsge@Mars_Server ~]$ nginx -V
Tengine version: Tengine/2.1.0 (nginx/1.6.2)
built by gcc 4.4.7 20120313 (Red Hat 4.4.7-11) (GCC) 
TLS SNI support enabled
configure arguments: --user=www --group=www --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_gzip_static_module --with-http_concat_module --add-module=../ngx_cache_purge-2.3 --with-http_image_filter_module --add-module=../ngx_slowfs_cache-1.9
loaded modules:
    ngx_core_module (static)
    ngx_errlog_module (static)
...
```

如果编译参数中找不到 ngx_cache_purge，就需要重新编译 Nginx ，新增编译参数：

```
#注意第一步解压的文件夹和nginx源码在同一个目录
--add-module=../ngx_cache_purge-2.3
```

我现在用的是淘宝开放的 Tengine ，可以使用动态加载模块功能，如果是原版 Nginx ，可以参考张戈博客之前分享的文章，在原来的基础上加上上述参数重新编译 Nginx 即可：

> [Nginx在线服务状态下平滑升级或新增模块的详细操作记录](https://zhangge.net/4856.html)

### ③、新增配置

A. 在 http 上下文中新增缓存配置：

```
http {
                #以上略                
                ##cache##
                proxy_connect_timeout 5;
                proxy_read_timeout 60;
                proxy_send_timeout 5;
                proxy_buffer_size 16k;
                proxy_buffers 4 64k;
                proxy_busy_buffers_size 128k;
                proxy_temp_file_write_size 128k;
                proxy_temp_path /tmp/temp_cache1; #临时缓存目录
                proxy_cache_path /tmp/cache1 levels=1:2 keys_zone=cache_one:200m inactive=30d max_size=5g; #设置缓存存放，不懂的参数自己百度搜索下
                ##end##
                #以下略
....
}
```

Ps：上述配置中出现的目录，请在保存配置后，使用 mkdir 手动创建。

B. 如下修改网站原来的server 模块：

```
server
        {
         #将之前监听修改成监听本地8080，其他配置保持不变    
         listen 127.0.0.1:8080;
         server_name zhang.ge;
#以下内容略...
}
```

B. 如下新增一个反向代理Server模块，用于转发请求到本地8080，变相实现反向代理模式：

```
server {
        listen 80;
        server_name zhang.ge;
        #缓存清理模块
        location ~ /purge(/.*) {
              allow 127.0.0.1;
              allow 192.168.1.101; #此处表示允许访问缓存清理页面的IP
              deny all;
              proxy_cache_purge cache_one $host$1$is_args$args;
        }
        #缓存html页面，可以缓存伪静态【这是亮点！】
        location ~ .*\.html$ {
              proxy_pass http://127.0.0.1:8080;
              proxy_cache_key $host$uri$is_args$args;
              proxy_redirect off;
              proxy_set_header Host $host;
              proxy_cache cache_one;
              #状态为200、302的缓存1天
              proxy_cache_valid 200 302 1d;
              #状态为301的缓存2天
              proxy_cache_valid 301 2d;
              proxy_cache_valid any 1m;
              #浏览器过期时间设置4小时
              expires 4h;
              #忽略头部禁止缓存申明，类似与CDN的强制缓存功能
              proxy_ignore_headers "Cache-Control" "Expires" "Set-Cookie";
              #在header中插入缓存状态，命中缓存为HIT，没命中则为MISS
              add_header Nginx-Cache "$upstream_cache_status";
        }
        #图片缓存设置，如果不是使用了Nginx缩略图功能，这个可以不用，效果不明显
        location ~ .*\.(gif|jpg|png|css|jsico)(.*) {
              proxy_pass http://127.0.0.1:8080;
              proxy_cache_key $host$uri$is_args$args;
              proxy_redirect off;
              proxy_set_header Host $host;
              proxy_cache cache_one;
              proxy_cache_valid 200 302 30d;
              proxy_cache_valid 301 1d;
              proxy_cache_valid any 1m;
              expires 30d;
              proxy_ignore_headers "Cache-Control" "Expires" "Set-Cookie";
              add_header Nginx-Cache "$upstream_cache_status";
        }
        #动态页面直接放过不缓存
        location ~ .*\.(php)(.*){
             proxy_pass http://127.0.0.1:8080;
             proxy_set_header        Host $host;
             proxy_set_header        X-Real-IP $remote_addr;
             proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
        }
        #设置缓存黑名单，不缓存指定页面，比如wp后台或其他需要登录态的页面，用分隔符隔开
        location ~ ^/(wp-admin|system)(.*)$ {
             proxy_pass http://127.0.0.1:8080;
             proxy_set_header        Host $host;
             proxy_set_header        X-Real-IP $remote_addr;
             proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
        }
        #缓存以斜杠结尾的页面，类似于CDN的目录缓存，如果存在问题请取消缓存机制
        location ~ ^(.*)/$ {
              proxy_pass http://127.0.0.1:8080;
              proxy_redirect off;
              proxy_cache_key $host$uri$is_args$args;
              proxy_set_header Host $host;
              proxy_cache cache_one;
              proxy_cache_valid 200 302 1d;
              proxy_cache_valid 301 1d;
              proxy_cache_valid any 1m;
              expires 1h;
              proxy_ignore_headers "Cache-Control" "Expires" "Set-Cookie";
              add_header Nginx-Cache "$upstream_cache_status";
        }
       location / {
             proxy_pass http://127.0.0.1:8080;
             proxy_set_header        Host $host;
             proxy_set_header        X-Real-IP $remote_addr;
             proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
        }
}
```

全部保存后，执行 nginx -s reload 让配置生效即可。现在你再去访问网站的 html 页面，刷新一次就可以看到效果了！加载速度绝逼会有质的飞跃！而且你可以在F12开发模式的 Network 状态中看到 Nginx-Cache HIT 的标识！

### ④、清理缓存

清理缓存就有点麻烦了，我弄了多次也还是感觉不怎么好用！网上也有不少先驱分享了自动清理脚本或批量清理代码等。不过用了下也是不咋的好用。

还是说一下清理方法吧！在A部分的配置中，我们已经加入了purge缓存清理页面，假设一个URL为http://192.168.1.1/test.html，通过访问http://192.168.1.1/purge/test.html 就可以清除该URL的缓存（我实际测试经常是404...）。

## 二、本地模式

第一种代理模式，我们是利用本地转发变相实现反向代理下的 Nginx 缓存功能，并且可以缓存 html 伪静态页面。从整体的配置可以看出，已经非常接近百度云加速等CDN的缓存功能了！对于理解CDN缓存还是有不小的帮助的！

现在分享一下，如果不用反向代理模式，该如何实现 Nginx 缓存呢？很简单，进一步借助 **ngx_slowfs_cache** 模块即可，这也是张戈博客在用模式，如何实现，且继续往下看。

### ①、下载模块

这个模式需要下载2个缓存模块：**ngx_cache_purge** 和 **ngx_slowfs_cache** 。这2个模块都出自一个网站，下载地址依然是 http://labs.frickle.com/files/ ，挑选一个最新版下载即可，比如：

[http://labs.frickle.com/files/ngx_slowfs_cache-1.9.tar.gz](https://zhangge.net/goto/aHR0cDovL2xhYnMuZnJpY2tsZS5jb20vZmlsZXMvbmd4X3Nsb3dmc19jYWNoZS0xLjkudGFyLmd6)

### ②、重新编译

和第一种模式一样，新增2个 --add-module 重新编译 Nginx即可：

```
--add-module=../ngx_cache_purge-2.3 --add-module=../ngx_slowfs_cache-1.9
```

具体就不赘述了，参考上文和博客之前的分享就可以搞定了。

### ③、新增配置

I. 在 http 上下文新增如下配置：

```
http {
     #以上略
     slowfs_cache_path /home/wwwroot/cache  levels=1:2   keys_zone=fastcache:256m inactive=1d max_size=5g;
slowfs_temp_path  /tmp/nginx_temp_cache 1 2;
     #以下略
}
```

Ps：以上配置中所涉及的目录请手动创建。

II. 在 server 模块中新增如下配置：

```
#新增缓存清理配置
location ~ /purge(/.*) {
            allow               127.0.0.1;
            allow               192.168.1.101;
            deny                all;
            slowfs_cache_purge  fastcache $1;
        }
#在上一篇文章的缩略图模块后新增缓存，让生成的缩略图缓存到磁盘（具体请看上一篇文章）
location ~ .*\.(gif|jpg|jpeg|png|bmp)$ {
            set $width '';
            set $height '';
            set $width $arg_width;
            set $height $arg_height;
            if ( $width != '' ) {
               add_header Thumbnail "By Nginx";
            }
            if ( $height = '' ) {
                set $height '-';
            }
            if ( $width = '' ) {
                set $width '-';
                set $height '-';
            }
            image_filter resize $width $height;
            image_filter_buffer 5M;
            image_filter_jpeg_quality 80;
            image_filter_transparency on;
 
            #新增缓存配置
            slowfs_cache        fastcache;
            slowfs_cache_key    $uri;
            slowfs_cache_valid  1d;
            #在header中插入缓存标识，比如：HIT/MISS from zhang.ge
            add_header X-Cache '$slowfs_cache_status from $host';
            expires  max;
}
```

保存后，执行 nginx -s reload 重载 Nginx即可。测试中发现，这种模式貌似无法缓存html伪静态页面，稍有遗憾，有兴趣的童鞋可以深入研究看看，可能是我没测试到位。
