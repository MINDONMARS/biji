# 配置Django实现数据库读写分离

django在进行数据库操作的时候，读取数据与写数据（增、删、改）可以分别从不同的数据库进行操作。

#### 1. 在配置文件中增加slave数据库的配置

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'HOST': '10.211.55.5',
        'PORT': 3306,
        'USER': 'meiduo',
        'PASSWORD': 'meiduo',
        'NAME': 'meiduo_mall'
    },
    'slave': {
        'ENGINE': 'django.db.backends.mysql',
        'HOST': '10.211.55.5',
        'PORT': 8306,
        'USER': 'root',
        'PASSWORD': 'mysql',
        'NAME': 'meiduo_mall'
    }
}
```

### 2. 创建数据库操作的路由分发类

在meiduo_mall/utils中创建db_router.py

```python
class MasterSlaveDBRouter(object):
    """数据库主从读写分离路由"""

    def db_for_read(self, model, **hints):
        """读数据库"""
        return "slave"

    def db_for_write(self, model, **hints):
        """写数据库"""
        return "default"

    def allow_relation(self, obj1, obj2, **hints):
        """是否运行关联操作"""
        return True
```

#### 3. 配置读写分离路由

在配置文件中增加

```python
# 配置读写分离
DATABASE_ROUTERS = ['meiduo_mall.utils.db_router.MasterSlaveDBRouter']
```