# Django删除购物⻋数据

[TOC]

## 1. 后端接口设计

**请求方式** ： DELETE /cart/

**请求参数**：

| 参数   | 类型 | 是否必须 | 说明       |
| ------ | ---- | -------- | ---------- |
| sku_id | int  | 是       | 商品sku id |

**返回数据**：无，状态码204

### 1.序列化器校验参数

```python
class CartDeleteSerializer(serializers.Serializer):
    """
    删除购物⻋数据序列化器
    """
    sku_id = serializers.IntegerField(label='商品id', min_value=1)
    def validate_sku_id(self, value):
        try:
            sku = SKU.objects.get(id=value)
        except SKU.DoesNotExist:
            raise serializers.ValidationError('商品不存在')
        return value
```

使⽤序列化器

```python
def delete(self, request):
    """删除购物⻋"""
    # 创建序列化器对象并验证字段
    serializer = serializers.CartDeleteSerializer(data=request.data)
    serializer.is_valid(raise_exception=True)
    # 获取校验后的参数
    sku_id = serializer.validated_data.get('sku_id')
```

### 2.判断⽤户是否登录

⽤户已登录

```python
		try:
            user = request.user
        except Exception:
            user = None

        if user is not None and user.is_authenticated:
            # 用户已登录，操作redis购物车
            # 创建连接到redis的对象
            redis_conn = get_redis_connection('cart')

            pl = redis_conn.pipeline()
            # 删除购物车记录
            pl.hdel('cart_%s' % user.id, sku_id)
            # 移除选中状态
            pl.srem('selected_%s' % user.id, sku_id)
            # 记得要执行
            pl.execute()

            return Response(status=status.HTTP_204_NO_CONTENT)
```

⽤户未登录

```python
 		else:
            # 用户未登录，操作cookie购物车
            # 获取用户浏览器中的cookie购物车数据
            cart_str = request.COOKIES.get('cart')

            if cart_str:
                # 将购物车字符串转bytes类型的购物车字符串
                cart_str_bytes = cart_str.encode()
                # 将bytes类型的购物车字符串转bytes类型的字典
                cart_dict_bytes = base64.b64decode(cart_str_bytes)
                # 将bytes类型的字典转标准字典
                cart_dict = pickle.loads(cart_dict_bytes)
            else:
                cart_dict = {}

            # 需要将cookie_cart_str数据写入到浏览器的cookie
            response = Response(status=status.HTTP_204_NO_CONTENT)

            # 如果sku_id在字典中，直接删除对应的记录
            if sku_id in cart_dict:
                del cart_dict[sku_id]

                # 需要将cart_dict数据转成字符串cookie_cart_str
                cookie_cart_dict_bytes = pickle.dumps(cart_dict)
                cookie_cart_str_bytes = base64.b64encode(cookie_cart_dict_bytes)
                cookie_cart_str = cookie_cart_str_bytes.decode()

                response.set_cookie('cart', cookie_cart_str)

            return response

```

