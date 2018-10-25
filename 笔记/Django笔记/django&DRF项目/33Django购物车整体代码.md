# Django购物车整体代码

[TOC]

## views

```python
from django.shortcuts import render
from rest_framework.views import APIView
from rest_framework.permissions import IsAuthenticated
from django_redis import get_redis_connection
from rest_framework.response import Response
from rest_framework import status
import base64, pickle

from . import serializers
from goods.models import SKU
# Create your views here.



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


# url(r'^carts/$', views.CartView.as_view()),
class CartView(APIView):
    """购物车增删改查
    用户未登录可以访问
    用户登录也可以访问
    """
    # permission_classes = [IsAuthenticated] # 如果加了权限，未登录用户无法访问，没加权限登录用户无意义
    def perform_authentication(self, request):
        """执行认证的方法
        重写之后，直接pass,不对用户做认证，认证的行为放在具体的业务逻辑需要的地方，程序员自己写代码认证
        如果写pass就是取消该视图的认证行为，需要自己写认证代码
        """
        pass

    def post(self, request):
        """添加购物车"""
        # 使用序列化器对购物车数据进行校验
        serializer = serializers.CartSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        # 读取校验之后的数据
        sku_id = serializer.validated_data.get('sku_id')
        count = serializer.validated_data.get('count')
        selected = serializer.validated_data.get('selected')

        # 判断用户是否登录
        try:
            user = request.user
        except Exception:
            user = None

        if user is not None and user.is_authenticated:
            # 用户已登录，操作redis购物车
            # 获取连接到redis的对象
            redis_conn = get_redis_connection('cart')
            pl = redis_conn.pipeline()

            # 给redis购物车做增量存储、计算
            # redis_conn.hincrby(name, key, amount=1)
            pl.hincrby('cart_%s' % user.id, sku_id, count)

            # 存储是否被勾选
            if selected:
                pl.sadd('selected_%s' % user.id, sku_id)

            # 接的执行
            pl.execute()

            # 响应
            return Response(serializer.data, status=status.HTTP_201_CREATED)

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
                cart_dict = {} # 保证用户即使是第一次使用cookie保存购物车数据，也有字典对象可以操作

            # 添加购物车数据到字典
            if sku_id in cart_dict:
                # 表示要添加的商品，在购物车已经存在
                origin_count = cart_dict[sku_id]['count']
                count += origin_count # count = count + origin_count

            cart_dict[sku_id] = {
                'count':count,
                'selected':selected
            }

            # 需要将cart_dict数据转成字符串cookie_cart_str
            cookie_cart_dict_bytes = pickle.dumps(cart_dict)
            cookie_cart_str_bytes = base64.b64encode(cookie_cart_dict_bytes)
            cookie_cart_str = cookie_cart_str_bytes.decode()

            # 需要将cookie_cart_str数据写入到浏览器的cookie
            response = Response(serializer.data, status=status.HTTP_201_CREATED)
            response.set_cookie('cart', cookie_cart_str)
            return response


    def get(self, request):
        """查询购物车"""
        # 判断用户是否登录
        try:
            user = request.user
        except Exception:
            user = None

        if user is not None and user.is_authenticated:
            # 用户已登录，操作redis购物车
            # 获取连接到redis的对象
            redis_conn = get_redis_connection('cart')

            # 查询redis中的购物车数据:
            # 注意点：python3中的从redis读取的数据都是bytes类型的数据
            # redis_cart_dict = {b'sku_id_1': b'count_1', b'sku_id_2': b'count_2'}
            redis_cart_dict = redis_conn.hgetall('cart_%s' % user.id)
            # 查询是否被勾选
            # redis_selected = [b'sku_id_1']
            redis_selected = redis_conn.smembers('selected_%s' % user.id)
            """
            {
                sku_id10: {
                    "count": 10,  // 数量
                    "selected": True  // 是否勾选
                },
                sku_id20: {
                    "count": 20,
                    "selected": False
                },
                ...
            }
            """
            # 定义空的字典
            cart_dict = {}
            for sku_id, count in redis_cart_dict.items():
                cart_dict[int(sku_id)] = {
                    'count': int(count),
                    'selected': sku_id in redis_selected # 如果sku_id在redis_selected中，返回True;反之，返回False
                }

        else:
            # 用户已登录，操作cookie购物车
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

        # 使用序列化器对redis购物车或者cookie购物车进行统一的序列化
        sku_ids = cart_dict.keys()
        skus = SKU.objects.filter(id__in=sku_ids)

        # 需要给每一个sku增加count和selected属性
        for sku in skus:
            sku.count = cart_dict[sku.id]['count']
            sku.selected = cart_dict[sku.id]['selected']

        # 创建序列化器对象,完成序列化并响应
        serializer = serializers.CartSKUSerializer(skus, many=True)
        return Response(serializer.data)


    def put(self, request):
        """修改购物车"""
        # 使用序列化器对购物车数据进行校验
        serializer = serializers.CartSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        # 读取校验之后的数据
        sku_id = serializer.validated_data.get('sku_id')
        count = serializer.validated_data.get('count')
        selected = serializer.validated_data.get('selected')

        # 判断用户是否登录
        try:
            user = request.user
        except Exception:
            user = None

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


    def delete(self, request):
        """删除购物车"""
        # 使用序列化器对购物车数据进行校验
        serializer = serializers.CartDeleteSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)

        # 读取校验之后的数据
        sku_id = serializer.validated_data.get('sku_id')

        # 判断用户是否登录
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

## serializers

```python
from rest_framework import serializers

from goods.models import SKU


class CartSelectAllSerializer(serializers.Serializer):
    """
    购物车全选
    """
    selected = serializers.BooleanField(label='全选')


class CartDeleteSerializer(serializers.Serializer):
    """删除购物车的序列化器"""

    sku_id = serializers.IntegerField(label='商品ID', min_value=1)

    def validate_sku_id(self, value):
        """对sku_id进行校验，判断其是否存在"""
        try:
            SKU.objects.get(id=value)
        except SKU.DoesNotExist:
            raise serializers.ValidationError('sku_id 不存在')
        return value


class CartSKUSerializer(serializers.ModelSerializer):
    """
    购物车商品数据序列化器
    """
    count = serializers.IntegerField(label='数量')
    selected = serializers.BooleanField(label='是否勾选')

    class Meta:
        model = SKU
        fields = ('id', 'count', 'name', 'default_image_url', 'price', 'selected')


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

## urls

```python
from django.conf.urls import url

from . import views


urlpatterns = [
    # 购物车增删改查
    url(r'^carts/$', views.CartView.as_view()),
    # 购物车全选
    url(r'^carts/selection/$', views.CartSelectAllView.as_view()),
]
```



