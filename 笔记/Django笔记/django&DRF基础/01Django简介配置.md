# Django

[TOC]

## 简介

### (1)重量级框架

- 提供项目工程管理的自动化脚本工具

- 数据库ORM支持（对象关系映射，英语：Object Relational Mapping）
- 模板
- 表单
- Admin管理站点
- 文件管理
- 认证权限
- session机制
- 缓存

### (2)MVT模式

![mvt](file:///C:/Users/Administrator/Desktop/django%E8%AE%B2%E4%B9%89/images/mvt.png)

- Model数据库交互,数据处理
- View 视图, 接收请求, 进行业务处理, 响应
- Template 模板 构造返回的html

## 环境安装

### (1)创建虚拟环境

```
mkvirtualenv django_py3_1.11 -p python3
```

### (2)安装Django

```
pip install django==1.11
```

## 创建工程

```
创建: django-admin startproject 名字
启动: python manage.py runserver ip:port (不写可用默认ip和端口 127.0.0.1:8000)
默认DEBUG
```

## 创建子应用

```
python manage.py startapp 名字
setting中注册子应用(否则数据库迁移不成功)
install_apps中添加 ['应用名.apps.应用名config']
```

## 创建视图

视图中传入参数(request)

返回响应HTTPResponse(响应内容, 响应类型, 状态码)

在子路由中注册(**需要注意定义路由的顺序，避免出现屏蔽效应。** )

```python
导入views
from . import views
urlpatterns = [
    # 每个路由信息都需要使用url函数来构造
    # url(路径, 视图)
    url(r'^index/$', views.index),
    url(r'^xxx/$', views.视图引用, name='xxx')
]
```

在总路由中注册

```python
导入include 可添加命名空间区分子路由  namespace='xxx'
from django.conf.urls import url, include
from django.contrib import admin

urlpatterns = [
    url(r'^admin/', admin.site.urls),  # django默认包含的

    # 添加
    url(r'^users/', include('users.urls')), 
]
```

## 配置文件

```python
BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))

DEBUG=False  #默认True

LANGUAGE_CODE = 'en-us'  # 语言
TIME_ZONE = 'UTC'  # 时区
# 改成中国大陆(用上海)
LANGUAGE_CODE = 'zh-hans'
TIME_ZONE = 'Asia/Shanghai'
# 静态文件(DEBUG = Fasle 时不提供静态文件)
STATIC_URL = '/static/'
STATICFILES_DIRS = [
    os.path.join(BASE_DIR, 'static_files'),
]
# session存储位置
# 本地缓存(丢失无法找回)
SESSION_ENGINE='django.contrib.sessions.backends.cache'

# 混合存储
SESSION_ENGINE='django.contrib.sessions.backends.cached_db'

# Redis(pip install django-redis)(配置)
CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://127.0.0.1:6379/1",
        "OPTIONS": {
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
        }
    }
}
SESSION_ENGINE = "django.contrib.sessions.backends.cache"
SESSION_CACHE_ALIAS = "default"
```

