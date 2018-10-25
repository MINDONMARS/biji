# Django用户中心收货地址(省市区三级联动)

[TOC]

## 1.准备省市区模型和数据库表(自关联)

### 1.准备模型类

```python
from django.db import models

# Create your models here.


class Area(models.Model):
    """
    行政区划
    """
    name = models.CharField(max_length=20, verbose_name='名称')
    parent = models.ForeignKey('self', on_delete=models.SET_NULL, related_name='subs', null=True, blank=True, verbose_name='上级行政区划')
    # 如果不自定义关联字段 related_name='subs',
    # 那么默认的关联字段就是 area_set

    class Meta:
        db_table = 'tb_areas'
        verbose_name = '行政区划'
        verbose_name_plural = '行政区划'

    def __str__(self):
        return self.name

```

### 2.迁移建表

`python manage.py makemigrations`
`python manage.py migrate`

### 3.导⼊省市区数据

`mysql -h数据库ip地址 -u数据库⽤户名 -p数据库密码 数据库 < areas.sql`
`mysql -h127.0.0.1 -uroot -pmysql meiduo_mall < areas.sql`

## 2.省市区视图集的编写

### 0.视图集选择 

ReadOnlyModelViewSet

- 对数据进⾏查询操作
- 既有查询单⼀数据，⼜查询列表数据
- 数据也都来源于⼀个数据表

### 1.视图集准备

**注意点**

1. 根据action选择不同的序列化器
2. 根据action选择不同的查询集 返回省级数据的查询集parent=None
3. 区域信息不分⻚

```python
from django.shortcuts import render
from rest_framework.viewsets import ReadOnlyModelViewSet

from .models import Area
from . import serializers
# Create your views here.


class AreasViewset(ReadOnlyModelViewSet):
    """省市区三级联动数据"""

    # 禁用分页
    pagination_class = None

    # 指定查询集
    # queryset = Area.objects.all()
    def get_queryset(self):
        # 根据不同的行为，指定不同的查询集
        if self.action == 'list':
            # 如果是list行为，表示返回订顶级数据
            return Area.objects.filter(parent=None)
        else:
            return Area.objects.all()

    # 指定序列化器
    # serializer_class = serializers.AreaSerializer
    def get_serializer_class(self):
        # 根据不同的行为，指定不同的序列化器
        if self.action == 'list':
            return serializers.AreaSerializer
        else:
            return serializers.SubAreaSerializer
```

### 2.序列化器准备

针对两种行为定义不同的序列化器(子级区划查询要关联上级)

```python
from rest_framework import serializers

from .models import Area


class AreaSerializer(serializers.ModelSerializer):
    """省序列化器"""

    class Meta:
        model = Area
        fields = ['id', 'name']


class SubAreaSerializer(serializers.ModelSerializer):
    """城市和区县序列化器"""

    # 使用subs关联AreaSerializer
    # area_set = AreaSerializer(many=True, read_only=True)
    subs = AreaSerializer(many=True, read_only=True)

    class Meta:
        model = Area
        fields = ['id', 'name', 'subs']
```

### 3.路由

主路由

```python
urlpatterns = [
    url(r'^admin/', admin.site.urls),

    ......
    
    # 省市区模块
    url(r'^', include('areas.urls')),
]

```

子路由

```python
from rest_framework.routers import DefaultRouter
from . import views


router = DefaultRouter()
router.register(r'areas', views.AreasViewset, base_name='areas')

urlpatterns = []

urlpatterns += router.urls
```

### 4.使⽤drf-extensions给类视图添加缓存

(省市区的信息)

安装

`pip install drf-extensions`

配置

```python
3.配置
# DRF扩展
REST_FRAMEWORK_EXTENSIONS = {
    # 缓存时间
    'DEFAULT_CACHE_RESPONSE_TIMEOUT': 60 * 60,
    # 缓存存储
    'DEFAULT_USE_CACHE': 'default',
}
```

使用

```python
from rest_framework_extensions.cache.mixins import CacheResponseMixin

class AreasViewSet(CacheResponseMixin, ReadOnlyModelViewSet):
    """省市区视图集：提供单个对象和列表数据"""
    pass
```

