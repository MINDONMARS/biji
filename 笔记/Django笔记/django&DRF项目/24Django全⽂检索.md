# Django全⽂检索

[TOC]

## 1.全⽂检索和Elasticsearch介绍

### 1.全⽂检索

#### 1.like关键字

MySQL数据库的like关键字可以实现关键字搜索

但是效率低下，涉及到多个字段不好实现

#### 2.全⽂检索

使⽤搜索引擎，在指定的任意字段中进⾏检索查询

#### 3.搜索引擎

通过搜索引擎进⾏数据查询时，搜索引擎并不是直接在数据库中进⾏查询，⽽是搜索引擎会对数据库中的数据进⾏⼀遍预处理，单独建⽴起⼀份索引结构数据。

类似于新华字典⼀样的

### 2.Elasticsearch介绍

- 1.开源的 Elasticsearch 是⽬前全⽂搜索引擎的⾸选
- 2.它可以快速地储存、搜索和分析海量数据。维基百科、Stack Overflow、Github 都采⽤它。
- 3.Elasticsearch 的底层是开源库 Lucene。
- 4.但是，你没法直接⽤ Lucene，必须⾃⼰写代码去调⽤它的接⼝。
- 5.Elastic 是 Lucene 的封装，提供了 REST API 的操作接⼝。
- 6.Elasticsearch 是⽤Java实现的。
- 7.注意点 Elasticsearch 不⽀持对中⽂进⾏分词建⽴索引，需要配合扩展elasticsearch-analysis-ik来实现中⽂分词处理。

## 2.Docker安装Elasticsearch

### 1.修改elasticsearch的配置⽂件

/home/python/elasticsearc-2.4.6/config/elasticsearch.yml第54⾏，更改ip地址为本机ip地址

```python
cd elasticsearch-2.4.6/config/
sudo vim elasticsearch.yml
```

### 2.获取安装镜像

```python
sudo docker image pull delron/elasticsearch-ik:2.4.6-1.0
sudo docker load -i elasticsearch-ik-2.4.6_docker.tar
```

## 3.配置haystack对接Elasticsearch

### 1.安装haystack

```python
pip install drf-haystack
pip install elasticsearch==2.4.1
```

### 2.注册应⽤

```
INSTALLED_APPS = [
    ...
    'haystack',
    ...
]
```

### 3.配置haystack

```python
# Haystack
HAYSTACK_CONNECTIONS = {
    'default': {
        'ENGINE': 'haystack.backends.elasticsearch_backend.ElasticsearchSearchEngine',
        'URL': 'http://192.168.103.134:9200/',  # 此处为elasticsearch运行的服务器ip地址，端口号固定为9200
        'INDEX_NAME': 'meiduo_09',  # 指定elasticsearch建立的索引库的名称
    },
}

# 当添加、修改、删除数据时，自动生成索引
HAYSTACK_SIGNAL_PROCESSOR = 'haystack.signals.RealtimeSignalProcessor'
```

### 4.创建索引类并⽣成索引数据

#### 1.使⽤场景

给商品模块的SKU模型类建⽴索引
goods.models.SKU

#### 2.创建索引类

goods.search_indexes.py

```python
from haystack import indexes

from .models import SKU


class SKUIndex(indexes.SearchIndex, indexes.Indexable):
    """
    SKU索引数据模型类
    """
    text = indexes.CharField(document=True, use_template=True)

    def get_model(self):
        """返回建立索引的模型类"""
        return SKU

    def index_queryset(self, using=None):
        """返回要建立索引的数据查询集"""
        return self.get_model().objects.filter(is_launched=True)
```

#### 3.指定复合字段(搜索为关键字在那些字段)

templates/search/indexes/goods/sku_text.txt

```
{{ object.name }}
{{ object.caption }}
{{ object.id }}
```

#### 4.建⽴索引

```
 python manage.py rebuild_index
```

### 5.创建搜索视图并测试搜索

#### 1.搜索视图

goods/views

```python
class SKUSearchViewSet(HaystackViewSet):
    """
    SKU搜索
    """
    # 指定索引模型类
    index_models = [SKU]
    # 指定序列化器
    serializer_class = serializers.SKUIndexSerializer
```

#### 2.搜索序列化器

goods.serializers

```python
from drf_haystack.serializers import HaystackSerializer
from rest_framework import serializers

from .models import SKU
from goods.search_indexes import SKUIndex


class SKUSerializer(serializers.ModelSerializer):

    class Meta:
        model = SKU
        fields = ['id', 'name', 'price', 'comments', 'default_image_url']


class SKUIndexSerializer(HaystackSerializer):
    """
    SKU索引结果数据序列化器
    """
    object = SKUSerializer(read_only=True)

    class Meta:
        index_classes = [SKUIndex]
        fields = ('text', 'object')
```

#### 3.搜索路由

```python
router = DefaultRouter()
router.register('skus/search', views.SKUSearchViewSet, base_name='skus_search')
urlpatterns += router.urls
```

#### 4.测试地址 

http://api.meiduo.site:8000/skus/search/?text=wifi