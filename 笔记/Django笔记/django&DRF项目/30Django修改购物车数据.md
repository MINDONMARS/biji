# Django修改购物车数据

[TOC]

## 1.幂等和⾮幂等介绍

### 1.概念

- **⾮幂等**:  如果后端处理结果对于这些请求的最终结果不同，跟请求次数相关，则称接⼝⾮幂等
- **幂等**:  对于同⼀个接⼝，进⾏多次相同的请求，如果后端处理结果对于这些请求都是相同的，则称接⼝是幂等的

### 2.表现形式

![1536981619917](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1536981619917.png)

## 2. 后端接口设计

**请求方式** ： PUT /cart/

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

### 1.序列化器校验参数

<u>使⽤新增购物⻋记录的序列化器</u>

```python
def put(self, request):
    """修改购物⻋"""
    # 创建序列化器对象并验证字段
    serializer = serializers.CartSerializer(data=request.data)
    serializer.is_valid(raise_exception=True)
    # 获取校验后的参数
    sku_id = serializer.validated_data.get('sku_id')
    count = serializer.validated_data.get('count')
    selected = serializer.validated_data.get('selected')
```

### 2.判断⽤户是否登录

```python
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
    # ⽤户
```

### 3.⽤户已登录

```python
 if user is not None and user.is_authenticated:
            # 用户已登录，操作redis购物车
            # 获取连接到redis的对象
            redis_conn = get_redis_connection('cart')
            pl = redis_conn.pipeline()

            # 操作redis修改购物车数据：将已经存在的商品数量覆盖即可
            pl.hset('cart_%s' % user.id, sku_id, count)

            # 是否勾选
            if selected:
                # ['sku_id_1', 'sku_id_2']
                pl.sadd('selected_%s' % user.id, sku_id)
            else:
                # ['sku_id_1', 'sku_id_2'] ==> ['sku_id_1']
                pl.srem('selected_%s' % user.id, sku_id)

            # 记得需要调用执行
            pl.execute()

            # 响应
            return Response(serializer.data)
```

### 4.用户未登录

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
                cart_dict = {}  # 保证用户即使是第一次使用cookie保存购物车数据，也有字典对象可以操作

            # 添加购物车数据到字典
            cart_dict[sku_id] = {
                'count': count,
                'selected': selected
            }

            # 需要将cart_dict数据转成字符串cookie_cart_str
            cookie_cart_dict_bytes = pickle.dumps(cart_dict)
            cookie_cart_str_bytes = base64.b64encode(cookie_cart_dict_bytes)
            cookie_cart_str = cookie_cart_str_bytes.decode()

            # 需要将cookie_cart_str数据写入到浏览器的cookie
            response = Response(serializer.data)
            response.set_cookie('cart', cookie_cart_str)
            return response
```

