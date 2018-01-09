#开发环境使用说明

一键搭建开发环境需要先进行如下几步操作：

1.克隆项目到本地并进入项目目录。

2.复制`docker-compose-dev.yml.example`并命名为`docker-compose-dev.yml`。

3.修改其中的`/d/WWW` 和 `/d/persistent/` 为 **你宿主机的代码目录 和持久化数据目录**。

4.如果你本机已经有mysql并且想数据重用，将`/d/server/MySql/data`修改为你自己的mysql数据目录。否则若要初始化一个全新的，改成持久化数据目录即可(如/d/persistent/mysql)

5.在 `nginx\conf.d` 目录下新增你想要自定义的站点conf。

一个例子：

nginx/conf.d/deploy.conf

```php
server {
    listen 80;
    server_name  deploy.cc;
    index index.html index.htm index.php;
    root /ibos/www/gamer/deploy/web;
    location / {
        if (!-f $request_filename){
            rewrite ^/(.*)$ /index.php?/$1 last;
        }
    }
    location ~ .*\.(php|php5)?$
    {
        fastcgi_pass php:9000;
        fastcgi_index index.php;
        include /etc/nginx/fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
    location ^~ /.git
    {
       return 444;
    }
    access_log /ibos/log/nginx/deploy.log main;
}

```

    所有php请求发送到php服务的9000端口处理。
    `/ibos/www`对应yml文件里的`/d/WWW`目录，亦即宿主机的代码目录。

安装好`docker`，在当前目录运行命令 `docker-compose -f .\docker-compose-dev.yml -p dev up -d` 即可启动所有服务。(-p为项目名，可自定义)

默认php版本为 `7.1.11` , 如某些项目需要 `php7.2` 的版本，可使用另外一个已经编译好的镜像。
用法：更改php服务下的image项为：`ccr.ccs.tencentyun.com/game-base/php:[tag]`,当前`[tag]`为`1.0.3`.
注意：需修改`php/php.ini`下的`extension_dir = "/usr/lib/php/7.1/modules/"` 为 `extension_dir = "/usr/lib/php/7.2/modules/"`

要启用xdebug调试，还需要修改`php/php.ini`下的[XDebug]一节，把 `xdebug.remote_host = "192.168.1.9"`,`xdebug.remote_port = 9001`修改为你宿主机的ip地址和ide设置的xdebug监听端口。

yml文件中的redis,memcached,mysql服务可以直接通过服务名+端口连接。 如`php:9000,mysql:3306`

关闭所有服务： `docker-compose -f .\docker-compose-dev.yml -p dev down`


