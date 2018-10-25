# Django部署

[TOC]

![服务器架构](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/%E6%9C%8D%E5%8A%A1%E5%99%A8%E6%9E%B6%E6%9E%84%E5%9B%BE.png)

### 1. 静态文件

**提示**

当Django运行在生产模式时，将不再提供静态文件的支持，需要将静态文件交给静态文件服务器。

我们先收集所有静态文件。项目中的静态文件除了我们使用的front_end_pc中之外，django本身还有自己的静态文件，如果rest_framework、xadmin、admin、ckeditor等。我们需要收集这些静态文件，集中一起放到静态文件服务器中。

**我们要将收集的静态文件放到front_end_pc目录下的static目录中，所以先创建目录static。**

Django提供了收集静态文件的方法。先在配置文件中配置收集之后存放的目录

```python
STATIC_ROOT = os.path.join(os.path.dirname(os.path.dirname(BASE_DIR)), 'front_end_pc/static')
```

然后执行收集命令

```shell
python manage.py collectstatic
```

我们使用Nginx服务器作为静态文件服务器

打开Nginx的配置文件

```shell
sudo vim /usr/local/nginx/conf/nginx.conf
```

在server部分中配置

![1537451731607](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1537451731607.png)

```python
server {
         listen       80;
         server_name  www.meiduo.site;

        location / {
             root   /home/python/Desktop/front_end_pc;
             index  index.html index.htm;
         }

        # 余下省略
}
```

server部分说明

- listen 处理静态业务逻辑的端⼝

- server_name

  - 处理静态业务逻辑的域名

  - ⽤户直接访问的

- location 当访问静态⽂件时指明静态⽂件在nginx服务器的位置

重启Nginx服务器

```shell
sudo /usr/local/nginx/sbin/nginx -s reload
```

> 首次启动nginx服务器
>
> ```shell
> sudo /usr/local/nginx/sbin/nginx
> ```
>
> 重启 
>
> ```shell
> sudo /usr/local/nginx/sbin/nginx -s reload
> ```
>
> 停止nginx服务器
>
> ```shell
> sudo /usr/local/nginx/sbin/nginx -s stop
> ```

配置nginx服务器静态域名

![1537451875560](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1537451875560.png)

测试
访问主⻚
http://www.meiduo.site

### 2. 动态接口

在项目中复制开发配置文件dev.py 到生产配置prod.py

修改配置文件prod.py中

```python
DEBUG = False

ALLOWED_HOSTS = [...,  'www.meiduo.site']  # 添加www.meiduo.site

CORS_ORIGIN_WHITELIST = (
    '127.0.0.1:8080',
    'localhost:8080',
    'www.meiduo.site:8080',
    'api.meiduo.site:8000',
    'www.meiduo.site',  # 添加
)
```

指定wsgi.py启动配置⽂件为prod.py

```python
import os
from django.core.wsgi import get_wsgi_application


os.environ.setdefault("DJANGO_SETTINGS_MODULE", "meiduo_mall.settings.prod")
```

django的程序通常使用uwsgi服务器来运行

安装uwsgi

```python
pip install uwsgi
```

在项目目录/meiduo_mall 下创建uwsgi配置文件 uwsgi.ini

```ini
[uwsgi]
#使用nginx连接时使用，Django程序所在服务器地址
socket=10.211.55.2:8001
#直接做web服务器使用，Django程序所在服务器地址
#http=10.211.55.2:8001
#项目目录
chdir=/Users/delron/Desktop/meiduo/meiduo_mall
#项目中wsgi.py文件的目录，相对于项目目录
wsgi-file=meiduo_mall/wsgi.py
# 进程数
processes=4
# 线程数
threads=2
# uwsgi服务器的角色
master=True
# 存放进程编号的文件
pidfile=uwsgi.pid
# 日志文件，因为uwsgi可以脱离终端在后台运行，日志看不见。我们以前的runserver是依赖终端的
daemonize=uwsgi.log
# 指定依赖的虚拟环境
virtualenv=/Users/delron/.virtualenv/meiduo
```

启动uwsgi服务器

```shell
uwsgi --ini uwsgi.ini
```

> 注意如果想要停止服务器，除了可以使用**kill**命令之外，还可以通过
>
> ```shell
> uwsgi --stop uwsgi.pid
> ```

修改Nginx配置文件nginx.conf，让Nginx接收到请求后转发给uwsgi服务器

```shell
sudo vim /usr/local/nginx/conf/nginx.conf
```

upstream meiduo { }

- 指定uwsgi服务器的IP和端⼝

- ⽤于nginx服务器分发动态业务逻辑

  ![1537452157942](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1537452157942.png)

  - server部分说明
  - listen 处理动态业务逻辑的端⼝
  - server_name
  - 处理动态业务逻辑的域名
  - ⻚⾯内部JS访问的
  - location 当访问动态数据时指明uwsgi服务器位置

```python
     upstream meiduo {
         server 10.211.55.2:8001;  # 此处为uwsgi运行的ip地址和端口号
         # 如果有多台服务器，可以在此处继续添加服务器地址
     }

     #gzip  on;
     server {
         listen  8000;
         server_name api.meiduo.site;

         location / {
             include uwsgi_params;
             uwsgi_pass meiduo;
         }

     }


     server {
         listen       80;
         server_name  www.meiduo.site;

         #charset koi8-r;

         #access_log  logs/host.access.log  main;
         location /xadmin {
             include uwsgi_params;
             uwsgi_pass meiduo;
         }

         location /ckeditor {
             include uwsgi_params;
             uwsgi_pass meiduo;
         }

         location / {
             root   /home/python/Desktop/front_end_pc;
             index  index.html index.htm;
         }


         error_page   500 502 503 504  /50x.html;
         location = /50x.html {
             root   html;
         }

     }
```

重启nginx

```shell
sudo /usr/local/nginx/sbin/nginx -s reload
```

配置nginx服务器静态域名

![1537452327981](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1537452327981.png)

测试
http://www.meiduo.site/list.html?cat=115
既有静态的，也有动态的