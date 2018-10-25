# Django查询购物⻋数据

[TOC]

## 1. 后端接口设计

**请求方式** ： GET /cart/

**请求参数**： 无

**返回数据**： JSON 或 表单

# 查询购物车数据

### 1. 后端接口设计

**请求方式** ： GET /cart/

**请求参数**： 无

**返回数据**： JSON 或 表单

```json
[
    {
        "id": 9,
        "count": 3,
        "name": "华为 HUAWEI P10 Plus 6GB+64GB 钻雕金 移动联通电信4G手机 双卡双待",
        "default_image_url": "http://image.meiduo.site:8888/group1/M00/00/02/CtM3BVrRcUeAHp9pAARfIK95am88523545",
        "price": "3388.00",
        "selected": true
    },
    {
        "id": 12,
        "count": 1,
        "name": "华为 HUAWEI P10 Plus 6GB+64GB 钻雕蓝 移动联通电信4G手机 双卡双待",
        "default_image_url": "http://image.meiduo.site:8888/group1/M00/00/02/CtM3BVrRdICAO_CRAAcPaeOqMpA2024091",
        "price": "3388.00",
        "selected": true
    }
]
```

| 参数              | 类型    | 是否必须 | 说明               |
| ----------------- | ------- | -------- | ------------------ |
| id                | int     | 是       | 商品sku id         |
| count             | int     | 是       | 数量               |
| selected          | bool    | 是       | 是否勾选，默认勾选 |
| name              | str     | 是       | 商品名称           |
| default_image_url | str     | 是       | 商品默认图片       |
| price             | decimal | 是       | 商品单价           |

## 2. 后端实现

### 1.判断⽤户是否登录

```python
def get(self, request):
    """查询购物⻋"""
    # 获取user,⽤于判断⽤户是否登录
    try:
        user = request.user
    except Exception:
        user = None
    # 判断⽤户是否登录
    if user is not None and user.is_authenticated:
        # ⽤户已登录，操作redis购物⻋
        pass
    else:
        # ⽤户未登录，操作cookie购物⻋
        pass
```

### 2.⽤户已登录

1.基础查询

```python
# 判断⽤户是否登录
if user is not None and user.is_authenticated:
    # 创建连接到redis对象
    redis_conn = get_redis_connection('cart')
    # 获取redis中的购物⻋数据
    redis_cart = redis_conn.hgetall('cart_%s' % user.id)
    # 获取redis中的选中状态
    redis_selected = redis_conn.smembers('selected_%s' % user.id)
```

2.和cookie购物⻋统⼀格式

**为了⽅便查询到购物⻋数据后，进⾏统⼀的序列化操作，可以提前将redis中的购物⻋数据构造成字典**

```json
{
    sku_id10: {
    	"count": 10, // 数量
    	"selected": True // 是否勾选
    },
    sku_id20: {
    	"count": 20,
    	"selected": False
    },
    ...
}
```

3.实现

```python
# 将redis中的两个数据统⼀格式，跟cookie中的格式⼀致，⽅便统⼀查询
 cart_dict = {}
 for sku_id, count in redis_cart.items():
    cart_dict[int(sku_id)] = {
        'count':int(count),
        'selected':sku_id in redis_selected
    }
```

### 3.⽤户未登录

```python
# ⽤户未登录，cookie操作购物⻋
cart_str = request.COOKIES.get('cart')

if cart_str: # ⽤户操作过cookie购物⻋
    # 将cart_str转成bytes,再将bytes转成base64的bytes,最后将bytes转字典
    cart_dict = pickle.loads(base64.b64decode(cart_str.encode()))
    
else: # ⽤户从没有操作过cookie购物⻋
    cart_dict = {}
```

### 4.序列化购物⻋数据

#### 1.构造序列化器

```python
class CartSKUSerializer(serializers.ModelSerializer):
    """
    购物⻋商品数据序列化器
    """
    count = serializers.IntegerField(label='数量')
    selected = serializers.BooleanField(label='是否勾选')
    
    class Meta:
        model = SKU
        fields = ('id', 'count', 'name', 'default_image_url', 'price', 'selected')

```

#### 2.使⽤序列化器

```python
# 查询购物⻋数据
sku_ids = cart_dict.keys()
skus = SKU.objects.filter(id__in=sku_ids)

# 补充count和selected字段
for sku in skus:
    sku.count = cart_dict[sku.id]['count']
    sku.selected = cart_dict[sku.id]['selected']
    
# 创建序列化器序列化商品数据
serializer = serializers.CartSKUSerializer(skus, many=True)

# 响应结果
return Response(serializer.data)
```

