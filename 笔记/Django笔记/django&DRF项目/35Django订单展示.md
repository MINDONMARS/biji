# Django订单展示

[TOC]

## 1.订单表设计

![1536983145170](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1536983145170.png)

## 2.订单确认(去结算)代码说明

### 0.逻辑说明 

点击去结算时，会读取购物⻋⻚⾯被勾选的商品

**订单号不再采用数据库自增主键，而是由后端生成创建。**

### 1.定义模型类

创建订单应用orders，编辑模型类models.py

```python
from django.db import models
from meiduo_mall.utils.models import BaseModel
from users.models import User, Address
from goods.models import SKU

# Create your models here.


class OrderInfo(BaseModel):
    """
    订单信息
    """
    PAY_METHODS_ENUM = {
        "CASH": 1,
        "ALIPAY": 2
    }

    PAY_METHOD_CHOICES = (
        (1, "货到付款"),
        (2, "支付宝"),
    )

    ORDER_STATUS_ENUM = {
        "UNPAID": 1,
        "UNSEND": 2,
        "UNRECEIVED": 3,
        "UNCOMMENT": 4,
        "FINISHED": 5
    }

    ORDER_STATUS_CHOICES = (
        (1, "待支付"),
        (2, "待发货"),
        (3, "待收货"),
        (4, "待评价"),
        (5, "已完成"),
        (6, "已取消"),
    )

    order_id = models.CharField(max_length=64, primary_key=True, verbose_name="订单号")
    user = models.ForeignKey(User, on_delete=models.PROTECT, verbose_name="下单用户")
    address = models.ForeignKey(Address, on_delete=models.PROTECT, verbose_name="收获地址")
    total_count = models.IntegerField(default=1, verbose_name="商品总数")
    total_amount = models.DecimalField(max_digits=10, decimal_places=2, verbose_name="商品总金额")
    freight = models.DecimalField(max_digits=10, decimal_places=2, verbose_name="运费")
    pay_method = models.SmallIntegerField(choices=PAY_METHOD_CHOICES, default=1, verbose_name="支付方式")
    status = models.SmallIntegerField(choices=ORDER_STATUS_CHOICES, default=1, verbose_name="订单状态")

    class Meta:
        db_table = "tb_order_info"
        verbose_name = '订单基本信息'
        verbose_name_plural = verbose_name


class OrderGoods(BaseModel):
    """
    订单商品
    """
    SCORE_CHOICES = (
        (0, '0分'),
        (1, '20分'),
        (2, '40分'),
        (3, '60分'),
        (4, '80分'),
        (5, '100分'),
    )
    order = models.ForeignKey(OrderInfo, related_name='skus', on_delete=models.CASCADE, verbose_name="订单")
    sku = models.ForeignKey(SKU, on_delete=models.PROTECT, verbose_name="订单商品")
    count = models.IntegerField(default=1, verbose_name="数量")
    price = models.DecimalField(max_digits=10, decimal_places=2, verbose_name="单价")
    comment = models.TextField(default="", verbose_name="评价信息")
    score = models.SmallIntegerField(choices=SCORE_CHOICES, default=5, verbose_name='满意度评分')
    is_anonymous = models.BooleanField(default=False, verbose_name='是否匿名评价')
    is_commented = models.BooleanField(default=False, verbose_name='是否评价了')

    class Meta:
        db_table = "tb_order_goods"
        verbose_name = '订单商品'
        verbose_name_plural = verbose_name
```

### 2.后端接口设计

**请求方式** ： GET /orders/settlement/

**请求参数**： 无

**返回数据**： JSON

| 参数              | 类型    | 是否必须 | 说明           |
| ----------------- | ------- | -------- | -------------- |
| freight           | decimal | 是       | 运费           |
| skus              | sku[]   | 是       | 结算的商品列表 |
| id                | int     | 是       | 商品id         |
| name              | str     | 是       | 商品名称       |
| default_image_url | str     | 是       | 商品默认图片   |
| price             | decimal | 是       | 商品单价       |
| count             | int     | 是       | 商品数量       |

```json
{
    "freight":"10.00",
    "skus":[
        {
            "id":10,
            "name":"华为 HUAWEI P10 Plus 6GB+128GB 钻雕金 移动联通电信4G手机 双卡双待",
             "default_image_url":"http://image.meiduo.site:8888/group1/M00/00/02/CtM3BVrRchWAMc8rAARfIK95am88158618",
            "price":"3788.00",
            "count":1
        },
        {
            "id":16,
            "name":"华为 HUAWEI P10 Plus 6GB+128GB 曜石黑 移动联通电信4G手机 双卡双待",
            "default_image_url":"http://image.meiduo.site:8888/group1/M00/00/02/CtM3BVrRdPeAXNDMAAYJrpessGQ9777651",
            "price":"3788.00",
            "count":1
        }
    ]
}
```

### 3.视图

```python
from django.shortcuts import render
from rest_framework.views import APIView
from rest_framework.permissions import IsAuthenticated
from django_redis import get_redis_connection
from decimal import Decimal
from rest_framework.response import Response
from rest_framework.generics import CreateAPIView

from goods.models import SKU
from . import serializers
# Create your views here.


class CommitOrderView(CreateAPIView):
    """提交订单"""

    # 必须用户登录才能进行访问
    permission_classes = [IsAuthenticated]
    # 指定序列化器
    serializer_class = serializers.CommitOrderSerializer


class OrderSettlementView(APIView):
    """订单结算"""

    # 必须用户登录才能进行访问
    permission_classes = [IsAuthenticated]

    def get(self, request):
        """获取"""
        user = request.user

        # 从购物车中获取用户勾选要结算的商品信息
        redis_conn = get_redis_connection('cart')
        redis_cart = redis_conn.hgetall('cart_%s' % user.id)
        cart_selected = redis_conn.smembers('selected_%s' % user.id)

        # 读取redis被勾选的商品（核心代码）
        cart = {}
        for sku_id in cart_selected:
            cart[int(sku_id)] = int(redis_cart[sku_id])

        # 查询商品信息
        skus = SKU.objects.filter(id__in=cart.keys())
        for sku in skus:
            sku.count = cart[sku.id]

        # 运费
        freight = Decimal('10.00')

        serializer = serializers.OrderSettlementSerializer({'freight': freight, 'skus': skus})
        return Response(serializer.data)
```

### 4.序列化器

```python
from rest_framework import serializers

from goods.models import SKU

class CartSKUSerializer(serializers.ModelSerializer):
    """
    购物车商品数据序列化器
    """
    count = serializers.IntegerField(label='数量')

    class Meta:
        model = SKU
        fields = ('id', 'name', 'default_image_url', 'price', 'count')


class OrderSettlementSerializer(serializers.Serializer):
    """
    订单结算数据序列化器
    """
    # float 1.23 ==> 123 * 10 ^ -2  --> 1.299999999
    # Decimal  1.23    1    23
    # max_digits 一共多少位；decimal_places：小数点保留几位
    freight = serializers.DecimalField(label='运费', max_digits=10, decimal_places=2)
    skus = CartSKUSerializer(many=True)
```

### 5.路由

```python
# 订单
url(r'^', include('orders.urls')),
```

```python
from django.conf.urls import url

from . import views


urlpatterns = [
    # 确认订单
    url(r'^orders/settlement/$', views.OrderSettlementView.as_view()),
]
```

