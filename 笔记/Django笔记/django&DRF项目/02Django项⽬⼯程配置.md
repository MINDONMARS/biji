# Django项⽬⼯程配置

[TOC]

## 1.创建配置文件夹

准备两个配置⽂件

dev.py 开发模式下使⽤的
prod.py ⽣产环境使⽤的

## 安装rest_frameworker

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    
    'rest_framework', # DRF
]
```

## 2.配置数据库

### MySQL

```python
DATABASES = {
    'default': {
    'ENGINE': 'django.db.backends.mysql',
    'HOST': '192.168.73.128', # 数据库主机
    'PORT': 3306, # 数据库端⼝
    'USER': 'root', # 数据库⽤户名
    'PASSWORD': 'mysql', # 数据库⽤户密码
    'NAME': 'meiduo_mall' # 数据库名字
    }
}
```

### Redis

```python
# 配置redis数据库作为缓存后端
CACHES = {
    "default": {
    "BACKEND": "django_redis.cache.RedisCache",
    "LOCATION": "redis://192.168.73.128:6379/0",
    "OPTIONS": {
    "CLIENT_CLASS": "django_redis.client.DefaultClient",
    	}
    },
    "session": {
    "BACKEND": "django_redis.cache.RedisCache",
    "LOCATION": "redis://192.168.73.128:6379/1",
    "OPTIONS": {
    "CLIENT_CLASS": "django_redis.client.DefaultClient",
    	}
    }
}
SESSION_ENGINE = "django.contrib.sessions.backends.cache"
SESSION_CACHE_ALIAS = "session"
```

## 3.⽇志

### ⽇志集成

```python
# ⽇志
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False, # 是否禁⽤已经存在的⽇志器
    'formatters': { # ⽇志信息显示的格式
    	'verbose': {
    		'format': '%(levelname)s %(asctime)s %(module)s %(lineno)d %(message)s'
    	},
    	'simple': {
    		'format': '%(levelname)s %(module)s %(lineno)d %(message)s'
   		 },
    },
    'filters': { # 对⽇志进⾏过滤
    	'require_debug_true': { # django在debug模式下才输出⽇志
    		'()': 'django.utils.log.RequireDebugTrue',
    	},
    },
    'handlers': { # ⽇志处理⽅法
    	'console': { # 向终端中输出⽇志
    		'level': 'INFO',
    		'filters': ['require_debug_true'],
    		'class': 'logging.StreamHandler',
    		'formatter': 'simple'
    },
    	'file': { # 向⽂件中输出⽇志
    		'level': 'INFO',
    		'class': 'logging.handlers.RotatingFileHandler',
    		'filename': os.path.join(os.path.dirname(BASE_DIR), "logs/meiduo.log"), # ⽇志⽂件的位置
    		'maxBytes': 300 * 1024 * 1024,
    		'backupCount': 10,
    		'formatter': 'verbose'
    	},
    },
    'loggers': { # ⽇志器
    	'django': { # 定义了⼀个名为django的⽇志器
    		'handlers': ['console', 'file'], # 可以同时向终端与⽂件中输出⽇志
    		'propagate': True, # 是否继续传递⽇志信息
    		'level': 'INFO', # ⽇志器接收的最低⽇志级别
    	},
    }
}
```

### ⽇志推送到代码仓库

日志文件中写入`.gitkeep`

忽略文件中写入忽略本地日志`logs/*.log`

## 4.⾃定义异常处理

创建文件meiduo_mall.utils.exceptions

```python
from rest_framework.views import exception_handler as drf_exception_handler
import logging
from django.db import DatabaseError
from redis.exceptions import RedisError
from rest_framework.response import Response
from rest_framework import status
# 获取在配置⽂件中定义的logger，⽤来记录⽇志


logger = logging.getLogger('django')

def exception_handler(exc, context):
    """
    ⾃定义异常处理
    :param exc: 异常
    :param context: 抛出异常的上下⽂(包含request和view对象)
    :return: Response响应对象
    """
    # 调⽤drf框架原⽣的异常处理⽅法
    response = drf_exception_handler(exc, context)
    if response is None:
   		view = context['view']
    	if isinstance(exc, DatabaseError) or isinstance(exc, RedisError):
    		# 数据库异常
    		logger.error('[%s] %s' % (view, exc))
    		response = Response({'message': '服务器内部错误'}, status=status.HTTP_507_INSUFFICIENT_STORAGE)
    return response
```

 	