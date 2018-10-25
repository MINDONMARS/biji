# Django保存购物⻋数据

[TOC]

## 1. 后端接口设计

**请求方式** ： POST /carts/

**请求参数**： JSON 或 表单

| 参数     | 类型 | 是否必须 | 说明               |
| -------- | ---- | -------- | ------------------ |
| sku_id   | int  | 是       | 商品sku id         |
| count    | int  | 是       | 数量               |
| selected | bool | 否       | 是否勾选，默认勾选 |

**返回数据**： JSON

| 参数     | 类型 | 是否必须 | 说明               |
| -------- | ---- | -------- | ------------------ |
| sku_id   | int  | 是       | 商品sku id         |
| count    | int  | 是       | 数量               |
| selected | bool | 是       | 是否勾选，默认勾选 |

**访问此接口，无论用户是否登录，前端请求都需携带请求头Authorization，由后端判断是否登录**

### 1.视图

增删改查

```python
class CartView(APIView):
    """操作购物⻋：增删改查"""
    def post(self, request):
        """添加购物⻋"""
        pass

    def get(self, request):
        """读取购物⻋"""
        pass

    def put(self, request):
        """修改购物⻋"""
        pass

    def delete(self, request):
        """删除购物⻋"""
        pass
```

### 2.路由

```python
# 总路由
url(r'^', include('carts.urls')),
# 子路由
urlpatterns = [
    url(r'^carts/$', views.CartView.as_view()),
]
```

## 2.取消购物⻋视图⽤户认证

### 1.说明

**因为前端请求时携带了Authorization请求头（主要是JWT），而如果用户未登录，此请求头的JWT无意义（没有值），为了防止REST framework框架在验证此无意义的JWT时抛出401异常，在视图中需要做两个处理**

- **重写perform_authentication()方法，此方法是REST framework检查用户身份的方法**
- **在获取request.user属性时捕获异常，REST framework在返回user时，会检查Authorization请求头，无效的Authorization请求头会导致抛出异常**

### 2.⽤于认证源代码

APIView.as_view.dispatch.initial.perform_authentication

![1536979982426](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1536979982426.png)

### 3.取消⽤户认证

延后认证

```python
def perform_authentication(self, request):
    """
    重写⽗类的⽤户验证⽅法，不在进⼊视图前就检查JWT
    保证⽤户未登录也可以进⼊下⾯的请求⽅法
    """
    pass
```

## 3.设置cookie有效期

```python
# 需要设置有效期，否则是临时cookie
            response.set_cookie('cart', cookie_cart, max_age=constants.CART_COOKIE_EXPIRES)
            return response
```

在carts中新建constants.py 常量文件

```python
# 购物车cookie的有效期
CART_COOKIE_EXPIRES = 365 * 24 * 60 * 60
```

## 4.序列化器校验参数

### 1.定义序列化器

```python
class CartSerializer(serializers.Serializer):
    """添加购物车序列化器"""
    sku_id = serializers.IntegerField(label='商品ID', min_value=1)
    count = serializers.IntegerField(label='商品商量', min_value=1)
    selected = serializers.BooleanField(label='是否勾选', default=True)

    def validate_sku_id(self, value):
        """对sku_id进行校验，判断其是否存在"""
        try:
            SKU.objects.get(id=value)
        except SKU.DoesNotExist:
            raise serializers.ValidationError('sku_id 不存在')
        return value
```

### 2.使⽤序列化器

```python
def post(self, request):
    """添加购物⻋"""
    # 创建序列化器对象并验证字段
    serializer = serializers.CartSerializer(data=request.data)
    serializer.is_valid(raise_exception=True)
```

### 5.判断⽤户是否登录

```python
def post(self, request):
    """添加购物⻋"""
    # 创建序列化器对象并验证字段
    serializer = serializers.CartSerializer(data=request.data)
    serializer.is_valid(raise_exception=True)
    # 获取user,⽤于判断⽤户是否登录
    try:
        user = request.user
    except Exception:
        user = None
    # 判断⽤户是否登录
    if user is not None and user.is_authenticated:
        # ⽤已登录，操作redis购物⻋
        pass
    else:
        # ⽤户未登录，操作cookie购物⻋
        pass
```

⽤户已登录:

```python
if user is not None and user.is_authenticated:
    # ⽤户已登录，redis操作购物⻋
    redis_conn = get_redis_connection('cart')
    # 创建管道操作redis,提升访问数据库效率
    pl = redis_conn.pipeline()
    # 新增购物⻋数据
    pl.hincrby('cart_%s' % user.id, sku_id, count)
    # 新增选中的状态
    if selected:
        pl.sadd('selected_%s' % user.id, sku_id)
 	# 执⾏管道
	pl.execute()
 	# 响应结果
 	return Response(serializer.data, status=status.HTTP_201_CREATED)
```

⽤户未登录:

```python
# ⽤户未登录，cookie操作购物⻋
cart_str = request.COOKIES.get('cart')

if cart_str:# ⽤户操作过cookie购物⻋
    # 将cart_str转成bytes,再将bytes转成base64的bytes,最后将bytes转字典
	cart_dict = pickle.loads(base64.b64decode(cart_str.encode()))
    
else:# ⽤户从没有操作过cookie购物⻋
    cart_dict = {}
    
# 判断要加⼊购物⻋的商品是否已经在购物⻋中
# 如有相同商品，累加求和，反之，直接赋值
if sku_id in cart_dict:
    # 累加求和
    origin_count = cart_dict[sku_id]['count']
    count += origin_count
cart_dict[sku_id] = {
    'count': count,
    'selected': selected
}

# 将字典转成bytes,再将bytes转成base64的bytes,最后将bytes转字符串
cookie_cart_str = base64.b64encode(pickle.dumps(cart_dict)).decode()

# 创建响应对象
response = Response(serializer.data, status=status.HTTP_201_CREATED)

# 响应结果并将购物⻋数据写⼊到cookie
response.set_cookie('cart', cookie_cart_str)
return response
```

