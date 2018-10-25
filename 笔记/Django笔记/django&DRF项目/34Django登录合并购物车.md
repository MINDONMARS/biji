# Django登录合并购物车

[TOC]

在用户登录时，将cookie中的购物车数据合并到redis中，并清除cookie中的购物车数据。

普通登录和QQ登录都要合并，所以将合并逻辑放到公共函数里实现

## 1.⼯具⽅法

**在carts/utils.py中创建merge_cart_cookie_to_redis方法**

```python
import base64, pickle
from django_redis import get_redis_connection


def merge_cart_cookie_to_redis(request, user, response):
    """购物车合并工具方法"""

    # 读取cookie中的购物车数据
    cart_str = request.COOKIES.get('cart')

    if not cart_str:
        # 如果没有cookie购物车数据不存在，就不需要合并，直接响应
        return response

    cart_str_bytes = cart_str.encode()
    cart_dict_bytes = base64.b64decode(cart_str_bytes)
    cookie_cart_dict = pickle.loads(cart_dict_bytes)

    # 读取redis中的购物车数据
    redis_conn = get_redis_connection('cart')

    redis_cart_dict = redis_conn.hgetall('cart_%s' % user.id)
    redis_cart_selected = redis_conn.smembers('selected_%s' % user.id)

    # 合并cookie到redis(核心代码)
    for sku_id, cookie_dict in cookie_cart_dict.items():
        # 将cookie_dict合并到redis_cart_dict
        redis_cart_dict[sku_id] = cookie_dict['count']

        # 将sku_id合并到redis_cart_selected
        if cookie_dict['selected']:
            redis_cart_selected.add(sku_id)

    # 需要将合并的数据同步到redis（核心代码）,也可以使用pipeline()实现
    redis_conn.hmset('cart_%s' % user.id, redis_cart_dict)
    redis_conn.sadd('selected_%s' % user.id, *redis_cart_selected)

    # 合并结束后，需要清空cookie中的购物车
    response.delete_cookie('cart')

    return response
```

## 2.修改登录视图实现登录合并购物⻋

### 1.重写JWT的登录视图

users.views

```python
class UserAuthorizeView(ObtainJSONWebToken):
    """
 	⽤户认证
 	"""
 	def post(self, request, *args, **kwargs):
        # 调⽤⽗类的⽅法，获取drf jwt扩展默认的认证⽤户处理结果
        response = super().post(request, *args, **kwargs)
        # 仿照drf jwt扩展对于⽤户登录的认证⽅式，判断⽤户是否认证登录成功
        # 如果⽤户登录认证成功，则合并购物⻋
        serializer = self.get_serializer(data=request.data)
        if serializer.is_valid():
            user = serializer.validated_data.get('user')
            response = merge_cart_cookie_to_redis(request, response, user)
        return response
```

### 2.更换登录路由

```python
# JWT登录
# url(r'^authorizations/$', obtain_jwt_token),
url(r'^authorizations/$', views.UserAuthorizeView.as_view()),
```

### 3.QQ登录逻辑增加合并购物⻋

```python
response = Response({
    'token': token,
    'user_id': user.id,
    'username': user.username
})

# 合并购物⻋
response = merge_cart_cookie_to_redis(request, response, user)

return response
```

