# Django⽤户浏览记录

[TOC]

## 1.⽤户浏览记录的存储设计⽅案

### 1.存储位置

redis

添加配置

```python
	"history": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://192.168.103.135:6379/3",
        "OPTIONS": {
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
        }
    },
```

### 2.存储的数据 

sku_id

### 3.存储类型 

list

用户在访问每个商品详情页面时，都要记录浏览历史记录

历史记录只需保存多个商品的sku_id即可，而且需要保持添加sku_id的顺序，所以采用redis中的列表来保存，redis的数据存储设

### 4.存储格式

每个⽤户使⽤user_id作为list的key, 单独维护⼀条唯⼀的记录

history_user_id' : [sku_id_1, sku_id_2, ...]

### 5.存储逻辑

先去重

```python
redis_conn.lrem('history_%s' % user_id, 0, sku_id)
```


再存储 

```python
redis_conn.lpush('history_%s' % user_id, sku_id)
```

最后截取五个元素 

```python
redis_conn.ltrim('history_%s' % user_id, 0, 4)
```

## 2.保存⽤户浏览记录接⼝实现

后端接口设计

**请求方式**：POST /browse_histories/

**请求参数**：JSON 或 表单

| 参数   | 类型 | 是否必须 | 说明         |
| ------ | ---- | -------- | ------------ |
| sku_id | int  | 是       | 商品sku 编号 |

**返回数据**：JSON

| 返回值 | 类型 | 是否必须 | 说明         |
| ------ | ---- | -------- | ------------ |
| sku_id | int  | 是       | 商品sku 编号 |

### 1.视图

users.views

```python
# url(r'^browse_histories/$', views.UserBrowsingHistoryView.as_view()),
class UserBrowsingHistoryView(CreateAPIView):
    """保存用户浏览记录"""

    # 指定序列化器
    serializer_class = serializers.UserBrowsingHistorySerializer
    # 只有登录用户才能保存浏览记录
    permission_classes = [IsAuthenticated]
```

### 2.序列化器

```python
class UserBrowsingHistorySerializer(serializers.Serializer):
    """用户浏览记录序列化器
    接受sku_id,校验sku_id,保存sku_id到redis
    """
    sku_id = serializers.IntegerField(label='商品ID', min_value=1) # 1123456781

    def validate_sku_id(self, value):
        """
        校验sku_id
        :param value: sku_id
        :return: sku_id
        """
        try:
            SKU.objects.get(id=value)
        except SKU.DoesNotExist:
            raise serializers.ValidationError('sku_id不存在')

        return value

    def create(self, validated_data):
        """
        重写create方法将sku_id到redis
        :param validated_data: {'sku_id':1}
        :return: validated_data
        """

        sku_id = validated_data.get('sku_id')
        user_id = self.context['request'].user.id

        # 创建连接到redis的对象
        redis_conn = get_redis_connection('history')
        pl = redis_conn.pipeline()

        # 去重
        pl.lrem('history_%s' % user_id, 0, sku_id)
        # 添加
        pl.lpush('history_%s' % user_id, sku_id)
        # 截取
        pl.ltrim('history_%s' % user_id, 0, 4)
        # 执行
        pl.execute()

        return validated_data
```

## 3.个⼈中⼼获取⽤户浏览记录

**请求方式**：GET /browse_histories/

**请求参数**： 无

**返回数据**： JSON

| 返回值            | 类型    | 是否必须 | 说明         |
| ----------------- | ------- | -------- | ------------ |
| id                | int     | 是       | 商品sku 编号 |
| name              | str     | 是       | 商品名称     |
| price             | decimal | 是       | 单价         |
| default_image_url | str     | 是       | 默认图片     |
| comments          | int     | 是       | 评论量       |

### 1.视图

```python
# url(r'^browse_histories/$', views.UserBrowsingHistoryView.as_view()),
class UserBrowsingHistoryView(CreateAPIView):
    """保存用户浏览记录"""
	'''
	# 指定序列化器
    serializer_class = serializers.UserBrowsingHistorySerializer
    # 只有登录用户才能保存浏览记录
    permission_classes = [IsAuthenticated]
	'''
    
    def get(self, request):
        """获取用户浏览记录
        """
        # 获取连接到redis的对象
        redis_conn = get_redis_connection('history')

        # 查询出redis中的浏览记录 sku_ids == ['1', '2', ......]
        sku_ids = redis_conn.lrange('history_%s' % request.user.id, 0, -1)

        # 使用sku_ids查询出sku
        sku_list = []
        for sku_id in sku_ids:
            sku = SKU.objects.get(id=sku_id)
            sku_list.append(sku)

        # 序列化sku_list
        serializer = serializers.SKUSerializer(sku_list, many=True)

        return Response(serializer.data)
```

### 2.序列化器

```python
class SKUSerializer(serializers.ModelSerializer):
    """用户浏览记录序列化器"""

    
    class Meta:
        model = SKU
        fields = ['id', 'name', 'price', 'default_image_url', 'comments']
```

