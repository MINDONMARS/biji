# Django购物⻋商品全选

[TOC]

## 后端接口设计

**请求方式** ： PUT /cart/selection/

**请求参数**： JSON 或 表单

| 参数     | 类型 | 是否必须 | 说明                                      |
| -------- | ---- | -------- | ----------------------------------------- |
| selected | bool | 是       | 是否全选，true表示全选，false表示取消全选 |

**返回数据**：JSON

| 返回值  | 类型 | 是否必须 | 说明 |
| ------- | ---- | -------- | ---- |
| message | str  | 是       | ok   |

### 1.视图

```python
class CartSelectAllView(APIView):
    """购物车全选 """

    def perform_authentication(self, request):
        """
        重写父类的用户验证方法，不在进入视图前就检查JWT
        """
        pass

    def put(self, request):
        """购物车商品全选"""

        # 序列化器使用
        serializer = serializers.CartSelectAllSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        selected = serializer.validated_data['selected']

        try:
            user = request.user
        except Exception:
            # 验证失败，用户未登录
            user = None

        if user is not None and user.is_authenticated:
            # 用户已登录，操作redis购物车
            redis_conn = get_redis_connection('cart')

            # 查询出所有的购物车数据
            redis_cart_dict = redis_conn.hgetall('cart_%s' % user.id)
            # 取出所有的sku_id
            sku_ids = redis_cart_dict.keys()

            # 将所有的sku_id添加到选中状态集合中
            if selected:
                redis_conn.sadd('selected_%s' % user.id, *sku_ids)
            else:
                redis_conn.srem('selected_%s' % user.id, *sku_ids)

            return Response({'message':'OK'})
        else:
            # 用户未登录，操作cookie购物车
            cart_str = request.COOKIES.get('cart')

            # 读取cookie中的购物车数据
            if cart_str:
                cart_str_bytes = cart_str.encode()
                cart_dict_bytes = base64.b64decode(cart_str_bytes)
                cart_dict = pickle.loads(cart_dict_bytes)
            else:
                cart_dict = {}

            # 遍历购物车字典将所有的sku_id对应的selected设置为用户传入的selected
            for sku_id in cart_dict:
                cart_dict[sku_id]['selected'] = selected

            # 构造购物车字符串
            cookie_cart_dict_bytes = pickle.dumps(cart_dict)
            cookie_cart_str_bytes = base64.b64encode(cookie_cart_dict_bytes)
            cookie_cart_str = cookie_cart_str_bytes.decode()

            # cookie中写入购物车字符串
            response = Response({'message': 'OK'})
            response.set_cookie('cart', cookie_cart_str)
            return response
```

### 2.序列化器

```python
class CartSelectAllSerializer(serializers.Serializer):
    """
    购物车全选
    """
    selected = serializers.BooleanField(label='全选')
```

### 3.路由

```python
urlpatterns = [
    ......
    # 购物车全选
    url(r'^carts/selection/$', views.CartSelectAllView.as_view()),
]
```

