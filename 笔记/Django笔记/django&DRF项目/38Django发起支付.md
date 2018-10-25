# Django发起支付

[TOC]

## 1. 后端接口设计

**请求方式**： GET `/orders/(?P<order_id>\d+)/payment/`

**请求参数**： 路径参数

| 参数     | 类型 | 是否必须 | 说明     |
| -------- | ---- | -------- | -------- |
| order_id | str  | 是       | 订单编号 |

**返回数据**： JSON

| 返回值     | 类型 | 是否必须 | 说明           |
| ---------- | ---- | -------- | -------------- |
| alipay_url | str  | 是       | 支付宝支付链接 |

## 2. 后端实现

在配置文件中编辑支付宝的配置信息

```python
# 支付宝
ALIPAY_APPID = "2016081600258081"  # 自己的
ALIPAY_URL = "https://openapi.alipaydev.com/gateway.do"
ALIPAY_DEBUG = True
```

在payment/views.py中创建视图

```python
from django.shortcuts import render
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from alipay import AliPay
from django.conf import settings
import os
from rest_framework.permissions import IsAuthenticated

from orders.models import OrderInfo
from .models import Payment
# Create your views here.


class PaymentView(APIView):
    """支付"""

    permission_classes = [IsAuthenticated]

    def get(self, request, order_id):
        """
        :param order_id: 用户要支付的订单的id
        :return: alipay_url
        """
        # 使用order_id查询出要支付的订单信息
        try:
            order = OrderInfo.objects.get(order_id=order_id, user=request.user, status=OrderInfo.ORDER_STATUS_ENUM['UNPAID'])
        except OrderInfo.DoesNotExist:
            return Response({'message':'订单不存在'}, status=status.HTTP_400_BAD_REQUEST)

        # 创建SDK Alipay对象
        alipay = AliPay(
            appid=settings.ALIPAY_APPID,
            app_notify_url=None,  # 默认回调url

            app_private_key_path=os.path.join(os.path.dirname(os.path.abspath(__file__)), "keys/app_private_key.pem"),
            alipay_public_key_path=os.path.join(os.path.dirname(os.path.abspath(__file__)), "keys/alipay_public_key.pem"),  # 支付宝的公钥，验证支付宝回传消息使用，不是你自己的公钥,

            sign_type="RSA2",  # RSA 或者 RSA2
            debug=settings.ALIPAY_DEBUG  # 默认False
        )

        # 使用Alipay对象对接支付宝的支付接口，得到进入到支付宝扫码登录界面
        # 电脑网站支付，需要跳转到https://openapi.alipaydev.com/gateway.do? + order_string
        order_string = alipay.api_alipay_trade_page_pay(
            out_trade_no=order_id,
            total_amount=str(order.total_amount),
            subject="美多商城%s" % order_id,
            return_url="http://www.meiduo.site:8080/pay_success.html", # 支付结束后重定向支付结果的页面
        )

        # 响应用户alipay_url
        alipay_url = settings.ALIPAY_URL + '?' + order_string
        return Response({'alipay_url':alipay_url})
```

## 3. 保存支付结果

用户支付成功后，支付宝会将用户重定向到<http://www.meiduo.site:8080/pay_success.html，并携带支付结果数据>

![支付成功](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/%E6%94%AF%E4%BB%98%E5%AE%8C%E6%88%90.png)

![支付成功跳转数据](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/%E6%94%AF%E4%BB%98%E6%88%90%E5%8A%9F%E8%B7%B3%E8%BD%AC%E6%95%B0%E6%8D%AE.png)

前端页面将此数据发送给后端，后端检验并保存支付结果

### 1. 后端接口设计

**请求方式**： PUT /payment/status/?支付宝参数

**请求参数**： 查询字符串参数， 见上面表格

**返回数据**： JSON

| 返回值   | 类型 | 是否必须 | 说明         |
| -------- | ---- | -------- | ------------ |
| trade_id | str  | 否       | 支付宝流水号 |

### 2. 后端实现

在payment/views.py中创建视图

```python
class PaymentStatusView(APIView):
    """获取支付结果
    1.使用支付宝交易流水号绑定美多商城订单id
    2.还要修改订单的状态
    """
    def put(self, request):
        """
        读取查询字符串中支付宝重定向的参数
        out_trade_no=20180917012326000000001 (美多商城维护订单编号)
        sign：sign=q722wKrDJvRWrNNs5gwmLuKFW（验证本次通知、重定向是否来源自支付宝）
        trade_no：trade_no=2018091721001004510500275863（支付宝生成的交易流水号）
        """
        # data = ?charset=utf-8&out_trade_no=20180917012326000000001&method=alipay.trade.page.pay.return&total_amount=3798.00&sign=q722wKrDJvRWrNNs5gwmLuKFWNLhyfzqisWFAhQ4aqK6RuUpo73%2BZzSO5hwdglPlapGHYhR0ZpsB%2FlAH8SyAG6sU49VkvM3Juyhlr1d8eL62N5NCy6q1rCd2PN%2FRbK4xouzrbIxESBruFVBFabWaAcAH5iOB2yknvST9x5wWd09jIHtoIZ515nZC98ud0v298kYWe%2FY63iMVOkrVC55Lx0ebgU%2FsmnWv3DFIgeW6UiEno%2BwjeezYn6u8XCqJGrCkBXLBr9X3tk%2BN%2FwbgankTYJvLtwMkc4nZsWrPIt7eXI3wtq4u341gEkgaEoNDMgtC0CaTD9PDNVT89ASiayvFqw%3D%3D&trade_no=2018091721001004510500275863&auth_app_id=2016082100308405&version=1.0&app_id=2016082100308405&sign_type=RSA2&seller_id=2088102172481561&timestamp=2018-09-17+09%3A52%3A40
        # request.query_params ： 返回的是QueryDict类型的字典，只有get(),getlist()方法，没有pop()方法
        query_dict = request.query_params

        # 为了能够在后面调用pop()方法，需要将QueryDict类型的字典，强转成标准字典类型
        data = query_dict.dict()

        # 从查询字符串中剔除掉'sign'，并接受sign的值，以备后用
        signature = data.pop("sign")

        # 创建SDK Alipay对象
        alipay = AliPay(
            appid=settings.ALIPAY_APPID,
            app_notify_url=None,  # 默认回调url

            app_private_key_path=os.path.join(os.path.dirname(os.path.abspath(__file__)), "keys/app_private_key.pem"),
            alipay_public_key_path=os.path.join(os.path.dirname(os.path.abspath(__file__)),
                                                "keys/alipay_public_key.pem"),  # 支付宝的公钥，验证支付宝回传消息使用，不是你自己的公钥,

            sign_type="RSA2",  # RSA 或者 RSA2
            debug=settings.ALIPAY_DEBUG  # 默认False
        )

        # verify()方法使用data结合RSA2签名算法，生成一个新的signature，然后跟前面signature进行对比
        # 如果对比成功，就表示当前的重定向来自于支付宝；反之，就不是响应错误
        success = alipay.verify(data, signature)

        if success:
            # 认证成功
            # 0.读取数据
            order_id = data.get('out_trade_no')
            trade_id = data.get('trade_no')

            # 1. 使用支付宝交易流水号绑定美多商城订单id
            Payment.objects.create(
                order_id=order_id,
                trade_id = trade_id
            )

            # 2. 还要修改订单的状态
            # OrderInfo.objects.filter('订单状态是否是待支付').update('待发货')
            OrderInfo.objects.filter(order_id=order_id, status=OrderInfo.ORDER_STATUS_ENUM['UNPAID']).update(status=OrderInfo.ORDER_STATUS_ENUM["UNSEND"])

            return Response({'trade_id': trade_id})
        else:
            # 认证失败
            return Response({'message': '非法请求'}, status=status.HTTP_403_FORBIDDEN)
```

