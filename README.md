====================================================  
# nginx静态文件缓存

nginx的一大功能就是完成静态资源的分离部署，减轻后端服务器的压力，如果给这些静态资源再加一级nginx的缓存，可以进一步提升访问效率。

### 第一步：添加nginx.conf的http级别的缓存配置

```
    proxy_connect_timeout 500;
    #跟后端服务器连接的超时时间_发起握手等候响应超时时间
    proxy_read_timeout 600;
    #连接成功后_等候后端服务器响应的时间_其实已经进入后端的排队之中等候处理
    proxy_send_timeout 500;
    #后端服务器数据回传时间_就是在规定时间内后端服务器必须传完所有数据
    proxy_buffer_size 128k;
    #代理请求缓存区_这个缓存区间会保存用户的头信息以供Nginx进行规则处理_一般只要能保存下头信息即可  
    proxy_buffers 4 128k;
    #同上 告诉Nginx保存单个用的几个Buffer最大用多大空间
    proxy_busy_buffers_size 256k;
    #如果系统很忙的时候可以申请更大的proxy_buffers 官方推荐*2
    proxy_temp_file_write_size 128k;
    #proxy缓存临时文件的大小
    proxy_temp_path /usr/local/nginx/temp;
    #用于指定本地目录来缓冲较大的代理请求
    proxy_cache_path /usr/local/nginx/cache levels=1:2 keys_zone=cache_one:200m inactive=1d max_size=30g;
    #设置web缓存区名为cache_one,内存缓存空间大小为12000M，自动清除超过15天没有被访问过的缓存数据，硬盘缓存空间大小200g
```

此处的重点在最后一句，缓存存储路径为：/usr/local/nginx/cache，levels=1:2代表缓存的目录结构为2级目录

### 第二步：在访问静态文件的location上添加缓存

```
#静态数据保存时效
location ~ \.html$ {
      proxy_pass http://source.qingk.cn;
      proxy_redirect off;
      proxy_cache cache_one;
      #此处的cache_one必须于上一步配置的缓存区域名称相同
      proxy_cache_valid 200 304 12h;
      proxy_cache_valid 301 302 1d;
      proxy_cache_valid any 1m;
      #不同的请求设置不同的缓存时效
      proxy_cache_key $uri$is_args$args;
      #生产缓存文件的key，通过4个string变量结合生成
      expires 30d;
      #其余类型的缓存时效为30天
      proxy_set_header X-Forwarded-Proto $scheme;
}
```

此处需要注意3点：

1、只有在proxy_pass的时候，才会生成缓存，下一次请求执行到proxy_pass的时候会判断是否有缓存，如果有则直接读缓存，返回给客户端，不会执行proxy_pass；如果没有，则执行proxy_pass，并按照规则生成缓存文件；可以到nginx的cache文件夹下看是否生成了缓存文件。

2、proxy_set_header Host $host 这一句可能导致缓存失败，所以不能配置这一句。我在测试的时候遇到了这个问题，不明原理。

3、proxy_pass使用upstream出差，换成域名或ip则可行。

### 第三步：在proxy_pass跳转的location中配置静态文件的路径

```
location ~ .*\.(html)$ {
    default_type 'text/html';
    root "/usr/local/openresty/nginx/html";
}
将nginx本地存放静态文件的路径配到root指令处

如果没有这一句：default_type 'text/html'，所有的请求都默认是下载文件，而不是访问html页面

到此，静态文件缓存已经配置完成。但是还差很重要的最后一步，缓存生成之后会阻止访问进入后台和nginx本地，如果有更新，则更新内容无法生效，还需要一种手动清除缓存的机制。
```

### 第四步：清除缓存

缓存文件是根据proxy_cache_key这个指令生成的，所以找到对应的缓存文件，删除即可


location ~ /purge(/.*) {
    #删除指定缓存区域cache_one的特定缓存文件$1$is_args$args
    proxy_cache_purge cache_one $1$is_args$args;
    #运行本机和10.0.217.0网段的机器访问，拒绝其它所有  
    allow           127.0.0.1;
    allow           10.0.217.0/24;
    deny          all;
}


删除缓存用到proxy_cache_purge指令。

至此缓存生成和特定清除机制都已经实现。

nginx静态资源缓存策略配置


1. 问题-背景
   以前也经常用nginx，但用的不深，通常是简单的设置个location用来做反向代理。直到今天给客户做项目碰到缓存问题：客户有个app，只是用原生做了个壳，里面的内容都是用h5写的，我们半途接手将新版本静态资源部署到服务器上后，发现手机端一直显示老的页面，一抓包，发现手机端根本就没有去请求新的html页面，定位是缓存问题。

2. 配置
   乍一看，客户原来的配置好像没什么问题，该有的也全有了

# 这是客户原来的配置

    server {
        listen       80 default_server;
        server_name  xxx.xxx.com;
        root         /app/xxx/html/mobile/;
    location ~ .*\.(?:jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|mp4|ogg|ogv|webm)$
    {
        expires      7d;
    }
     
    location ~ .*\.(?:js|css)$
    {
        expires      7d;
    }
    location ~ .*\.(?:htm|html)$
    {
        add_header Cache-Control "private, no-store, no-cache, must-revalidate, proxy-revalidate";
    }
     
    location ^~/mobile/
    {
        alias /app/xxx/html/mobile/;
    }

}

乍看没问题，但就是没有生效，由于查找nginx文档，发现nginx的location有优先级之分（是否生效与放置的位置没有关系）。

2.1 nginx location的四种类别
【=】模式: location = path,此种模式优先级最高（但要全路径匹配） 
【^~】模式:location ^~ path,此种模式优先级第二高于正则； 
【~ or ~*】模式:location ~ path,正则模式，优先级第三，【~】正则匹配区分大小写，【~*】正则匹配不区分大小写； 
【path】模式: location path,中间什么都不加，直接跟路径表达式； 
注意：一次请求只能匹配一个location，一旦匹配成功后，便不再继续匹配其余location;

一对照，发现location ^~优先级高于那些正则的缓存策略，所以缓存策略肯定不会对其生效，一翻查找下，终于解决了，配置如下：

    server {
        listen       80 default_server;
        server_name  xxx.xxx.com;
        root         /app/xxx/html/mobile/;
    location ~ .*\.(?:jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|mp4|ogg|ogv|webm)$
    {
        expires      7d;
    }
     
    location ~ .*\.(?:js|css)$
    {
        expires      7d;
    }
    location ~ .*\.(?:htm|html)$
    {
        add_header Cache-Control "private, no-store, no-cache, must-revalidate, proxy-revalidate";
    }
     
    location ^~/mobile/
    {
        alias /app/xxx/html/mobile/;
    # 将缓存策略用if语句写在location里面，生效了
        if ($request_filename ~* .*\.(?:htm|html)$)
        {
            add_header Cache-Control "private, no-store, no-cache, must-revalidate, proxy-revalidate";
        }
        if ($request_filename ~* .*\.(?:js|css)$)
        {
            expires      7d;
        }
     
        if ($request_filename ~* .*\.(?:jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|mp4|ogg|ogv|webm)$)
        {
            expires      7d;
        }
    }

}

3. 配置优化
   上面的配置虽然解决了缓存问题，但一看就发现冗余代码较多，应该不是最佳实践，于是请教了前公司专业运维同事，优化后的配置如下：

    server {
        listen       80 default_server;
        server_name  xxx.xxx.com;
        # 通过此语句来映射静态资源
        # 查找客户最初的配置，发现也有此项，但配置错误，后面多加了一个mobile, 解析时目录变成了/app/xxx/html/mobile/mobile，报404
        # 估计也是因为此处配置错误不生效，后面才又加了个location来映射，但location又不能继承外层的缓存策略
        # 估计原来配置此nginx的人也是个半吊子
        root         /app/xxx/html/;
    location ~ .*\.(?:jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|mp4|ogg|ogv|webm)$
    {
        expires      7d;
    }
     
    location ~ .*\.(?:js|css)$
    {
        expires      7d;
    }
    location ~ .*\.(?:htm|html)$
    {
        add_header Cache-Control "private, no-store, no-cache, must-revalidate, proxy-revalidate";
    }

}    

4. 深化
   项目通常就是静态资源与接口，接口一般都很少碰到缓存问题（因为很少有人去给接口配置缓存策略，不配置的话就不缓存），碰到缓存问题的通常都是静态资源。 
   静态资源——html: 
   html文件最容易碰到缓存问题，重新发版后，一旦客户端继续使用原来的缓存，那么在原来的缓存过期之前，没有任何办法去触使客户端更新，除非一个个通知android客户手动清除app缓存数据，通知IOS用户卸载重装。所以配置html缓存策略时要格外小心，我们项目是不缓存html文件； 
   静态资源——js/css/各种类型的图片: 
   此类资源改动较少，为了提升用户体验，一般都需要配置缓存，但反而不容易碰到缓存问题。因为现在的前程工程也都需要build，在build时工具会自动在文件名上加时间戳，这样一发新版时，只要客户端请求了新版的html，里面引用的js/css/jpg等都已经换了路径，肯定也就不会使用本地的缓存了。

Nginx 下缓存静态文件（如css js)


目的：缓存nginx服务器的静态文件。如css,js,htm,html,jpg,gif,png,flv,swf，这些文件都不是经常更新。便于缓存以减轻服务器的压力。
实现： nginxproxy_cache可以将用户的请缓存到本地一个目录，当下一个请求时可以直接调取缓存文件，就不用去后端服务器去取文件了。
配置： 打开配置文件/etc/nginx/nginx.conf

```
user  www www;
worker_processes 2;
error_log /var/log/nginx/nginx_error.log  crit;
worker_rlimit_nofile 65535;
events
{
  use epoll;
  worker_connections 65535;
}

http
{
 include      mime.types;
  default_type application/octet-stream;

  server_names_hash_bucket_size 128;
  client_header_buffer_size 32k;
  large_client_header_buffers 4 32k;
  client_max_body_size 8m;

  sendfile on;
 tcp_nopush    on;
  keepalive_timeout 0;
  tcp_nodelay on;

  fastcgi_connect_timeout 300;
  fastcgi_send_timeout 300;
  fastcgi_read_timeout 300;
  fastcgi_buffer_size 64k;
  fastcgi_buffers 4 64k;
  fastcgi_busy_buffers_size 128k;
  fastcgi_temp_file_write_size 128k;
  ##cache##
  proxy_connect_timeout 5;
  proxy_read_timeout 60;
  proxy_send_timeout 5;
  proxy_buffer_size 16k;
  proxy_buffers 4 64k;
  proxy_busy_buffers_size 128k;
  proxy_temp_file_write_size 128k;
  proxy_temp_path /home/temp_dir;
  proxy_cache_path /home/cache levels=1:2keys_zone=cache_one:200m inactive=1d max_size=30g;
  ##end##

 gzip   on;
 gzip_min_length   1k;
  gzip_buffers  4 8k;
  gzip_http_version 1.1;
  gzip_types  text/plain application/x-javascript text/css application/xml;
  gzip_disable "MSIE [1-6]\.";

  log_format access  '$remote_addr - $remote_user [$time_local]"$request" '
            '$status $body_bytes_sent "$http_referer" '
            '"$http_user_agent" $http_x_forwarded_for';
  upstream appserver { 
       server 192.168.1.251;
  }
  server {
       listen      80 default;
       server_name blog.slogra.com;
        location~ .*\.(gif|jpg|png|htm|html|css|js|flv|ico|swf)(.*) {
             proxy_pass http://appserver ;
             proxy_redirect off;
             proxy_set_header Host $host;
             proxy_cache cache_one;
             proxy_cache_valid 200 302 1h;
             proxy_cache_valid 301 1d;
             proxy_cache_valid any 1m;
             expires 30d;
       }
       location ~ .*\.(php)(.*){
            proxy_pass http://appserver ;
            proxy_set_header       Host $host;
            proxy_set_header       X-Real-IP $remote_addr;
            proxy_set_header  
            }
      }
```

===============================================  
···
http {
  proxy_cache_path /tmp/nginx/cache
  levels=1:2
  keys_zone=main:10m
  max_size=1g inactive=1d;
  proxy_temp_path /tmp/nginx/tmp;
  
  server {
    listen 80;
    server_name app.example.com;
    rewrite ^ https://$server_name$request_uri? permanent;
  }
  
  server {
    set $cache_key $scheme$host$uri$is_args$args;
    
    listen 443;
    server_name app.example.com;
  
    ssl on;
    ssl_certificate /etc/nginx/ssl/ssl.crt;
    ssl_certificate_key /etc/nginx/ssl/ssl.key;
  
    location = /favicon.ico {
      root /home/ubuntu/app/bundle/programs/client/app;
      access_log off;
      expires 1w;
    }
  
    location ~* "^/[a-z0-9]{40}\.(css|js)$" {
      root /home/ubuntu/app/bundle/programs/client;
      access_log off;
      expires max;
    }
  
    location ~ "^/packages" {
      root /home/ubuntu/app/bundle/programs/client;
      access_log off;
    }
  
    location / {
      proxy_pass http://localhost:3000;
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
      proxy_set_header Host $host;
    }
    
    location /static {
      proxy_pass http://localhost:3000/static;
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
      proxy_set_header Host $host;
      
      proxy_cache main;
			proxy_cache_key $cache_key;
			proxy_cache_valid 1d; #time till cache goes stale
			proxy_cache_use_stale error
					  timeout
					  invalid_header
					  http_500
					  http_502
					  http_504
					  http_404;
      
      expires 365d;
      gzip on;
      gzip_min_length  1100;
      gzip_buffers  4 32k;
      gzip_types    text/plain application/x-javascript text/xml text/css;
      gzip_vary on;
    }
  }

}
···
