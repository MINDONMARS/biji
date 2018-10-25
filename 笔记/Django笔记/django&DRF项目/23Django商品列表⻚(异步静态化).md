# Django商品列表⻚(异步静态化)

[TOC]

## 1.商品分类静态化

### 1.准备celery异步任务包 

/celery_tasks/html/tasks.py

```python
from celery_tasks.main import celery_app
from django.template import loader
import os
from django.conf import settings

from goods.utils import get_categories


@celery_app.task(name='generate_static_list_search_html')
def generate_static_list_search_html():
    """
    生成静态的商品列表页和搜索结果页html文件
    """
    # 商品分类菜单
    categories = get_categories()

    # 渲染模板，生成静态html文件
    context = {
        'categories': categories,
    }

    # 渲染模板
    template = loader.get_template('list.html')
    html_text = template.render(context)

    # 将html字符串写入到html文件
    file_path = os.path.join(settings.GENERATED_STATIC_HTML_FILES_DIR, 'list.html')
    with open(file_path, 'w') as f:
        f.write(html_text)
```

### 2.定义异步任务

goods/utils.py

```python
from collections import OrderedDict

from .models import GoodsChannel


def get_categories():
    """
    获取商城商品分类菜单
    :return 菜单字典
    """
    # 商品频道及分类菜单
    # 使用有序字典保存类别的顺序
    # categories = {
    #     1: { # 组1
    #         'channels': [{'id':, 'name':, 'url':},{}, {}...],
    #         'sub_cats': [{'id':, 'name':, 'sub_cats':[{},{}]}, {}, {}, ..]
    #     },
    #     2: { # 组2
    #
    #     }
    # }
    categories = OrderedDict()
    channels = GoodsChannel.objects.order_by('group_id', 'sequence')
    for channel in channels:
        group_id = channel.group_id  # 当前组

        if group_id not in categories:
            categories[group_id] = {'channels': [], 'sub_cats': []}

        cat1 = channel.category  # 当前频道的类别

        # 追加当前频道
        categories[group_id]['channels'].append({
            'id': cat1.id,
            'name': cat1.name,
            'url': channel.url
        })
        # 构建当前类别的子类别
        for cat2 in cat1.goodscategory_set.all():
            cat2.sub_cats = []
            for cat3 in cat2.goodscategory_set.all():
                cat2.sub_cats.append(cat3)
            categories[group_id]['sub_cats'].append(cat2)
    return categories
```

### 3.准备渲染模板

list.html

### 4.站点管理触发异步任务

goods/admin

```python
from django.contrib import admin

from . import models
# Register your models here.


class GoodsCategoryAdmin(admin.ModelAdmin):
    """定义GoodsCategory模型类的站点管理类
    用于监听保存和删除事件，触发异步任务
    """
    def save_model(self, request, obj, form, change):
        """
        监听站点的保存事件
        :param request: 本次保存时的请求
        :param obj: 本次保存时的模型对象
        :param form: 本次保存时操作的表单
        :param change: 本次保存时的数据跟旧数据的不同点
        :return: None
        """
        # 执行父类的保存行为
        obj.save()

        # 追加自己的触发异步任务的行为
        from celery_tasks.html.tasks import generate_static_list_search_html
        generate_static_list_search_html.delay()


    def delete_model(self, request, obj):
        """
        监听站点的删除事件
        :param request: 本次删除时的请求
        :param obj: 本次删除时的模型对象
        :return: None
        """
        # 执行父类的删除行为
        obj.delete()

        # 追加自己的触发异步任务的行为
        from celery_tasks.html.tasks import generate_static_list_search_html
        generate_static_list_search_html.delay()


admin.site.register(models.GoodsCategory, GoodsCategoryAdmin)

admin.site.register(models.GoodsChannel)
admin.site.register(models.Goods)
admin.site.register(models.Brand)
admin.site.register(models.GoodsSpecification)
admin.site.register(models.SpecificationOption)
admin.site.register(models.SKU)
admin.site.register(models.SKUSpecification)
admin.site.register(models.SKUImage)
```

## 2.排序分⻚逻辑

### 1.排序分⻚逻辑分析

#### 1.排序

##### 默认排序

根据商品发布的时间，倒叙，最新发布的商品在最前⾯
-create_time

##### 价格排序 

price 按照价格由低到⾼排序

##### 销量排序

-sales 按照销量由⾼到低

#### 2.分⻚

##### 当前⻚ 

page

##### 每⻚条数 

page_size
默认每⻚5条
最少2条，最多20条

##### 分⻚后端 

⾃定义分⻚后端

### 2.排序分⻚逻辑实现

#### 视图

```python
from django.shortcuts import render
from rest_framework.filters import OrderingFilter
from rest_framework.generics import ListAPIView
from drf_haystack.viewsets import HaystackViewSet

from .models import SKU
from . import serializers
# Create your views here.


# url(r'^categories/(?P<category_id>\d+)/skus/', views.SKUListView.as_view()),
class SKUListView(ListAPIView):
    """商品列表：排序和分页
    """
    # 指定序列化器
    serializer_class = serializers.SKUSerializer

    # 指定排序后端
    filter_backends = [OrderingFilter]
    # 指定排序的字段，由ordering接受
    ordering_fields = ('create_time', 'price', 'sales')

    # 指定查询集
    # queryset = SKU.objects.all()
    def get_queryset(self):
        category_id = self.kwargs.get('category_id')
        # 商品列表中的数据需要满足指定的分类和必须是上架的商品
        return SKU.objects.filter(category_id=category_id, is_launched=True)

```

#### 序列化器

```python
from drf_haystack.serializers import HaystackSerializer
from rest_framework import serializers

from .models import SKU
from goods.search_indexes import SKUIndex


class SKUSerializer(serializers.ModelSerializer):

    class Meta:
        model = SKU
        fields = ['id', 'name', 'price', 'comments', 'default_image_url']
```

#### 分⻚后端

```python
from rest_framework.pagination import PageNumberPagination


class LargeResultsSetPagination(PageNumberPagination):
    """自定义分页后端"""
    # 指定每页最小数据条数
    page_size = 2

    # 指定接受每页条数的字段
    page_size_query_param = 'page_size'

    # 指定每页最大数据条数
    max_page_size = 20
```

配置自定义分页后端

```python
# DRF配置
REST_FRAMEWORK = {
   ......
    # 分页
    'DEFAULT_PAGINATION_CLASS': 'meiduo_mall.utils.pagination.LargeResultsSetPagination',

}
```

#### 4.路由

```python
urlpatterns = [
    # 商品列表
    url(r'^categories/(?P<category_id>\d+)/skus/', views.SKUListView.as_view()),
]
```

