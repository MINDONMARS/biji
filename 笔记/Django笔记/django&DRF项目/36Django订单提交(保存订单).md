# Django订单提交(保存订单)

[TOC]

## 1. 后端接口设计

**请求方式** ： POST /orders/

**请求参数**： JSON 或 表单

| 参数       | 类型 | 是否必须 | 说明       |
| ---------- | ---- | -------- | ---------- |
| address    | int  | 是       | 收货地址id |
| pay_method | int  | 是       | 支付方式   |

**返回数据**： JSON

| 参数     | 类型 | 是否必须 | 说明     |
| -------- | ---- | -------- | -------- |
| order_id | char | 是       | 订单编号 |

### 1.视图

```python
class CommitOrderView(CreateAPIView):
    """提交订单"""

    # 必须用户登录才能进行访问
    permission_classes = [IsAuthenticated]
    # 指定序列化器
    serializer_class = serializers.CommitOrderSerializer
```

### 2.Django中使⽤事务说明

#### 1.场景说明

- 在提交订单逻辑中，设计到三张表的操作

  OrderInfo
  OrderGoods
  SKU

- 这三张表必须要么⼀起成功，要么⼀起失败

- 所以需要使⽤到事务来处理

#### 2.并发下单库存问题说明

![并发下单](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/%E5%B9%B6%E5%8F%91%E4%B8%8B%E5%8D%95.png)

#### 解决办法：

- **悲观锁**

  当查询某条记录时，即让数据库为该记录加锁，锁住记录后别人无法操作，使用类似如下语法

  ```python
  select stock from tb_sku where id=1 for update;
  
  SKU.objects.select_for_update().get(id=1)
  ```

  悲观锁类似于我们在多线程资源竞争时添加的互斥锁，容易出现**死锁现象**，采用不多。

- **乐观锁**

  乐观锁并不是真实存在的锁，而是在更新的时候判断此时的库存是否是之前查询出的库存，如果相同，表示没人修改，可以更新库存，否则表示别人抢过资源，不再执行库存更新。类似如下操作

  ```python
  update tb_sku set stock=2 where id=1 and stock=7;
  
  SKU.objects.filter(id=1, stock=7).update(stock=2)
  ```

- **任务队列**

  将下单的逻辑放到任务队列中（如celery），将并行转为串行，所有人排队下单。比如开启只有一个进程的Celery，一个订单一个订单的处理。

#### 3. 需要修改MySQL的事务隔离级别

事务隔离级别指的是在处理同一个数据的多个事务中，一个事务修改数据后，其他事务何时能看到修改后的结果。

MySQL数据库事务隔离级别主要有四种：

- **Serializable** 串行化，一个事务一个事务的执行
- **Repeatable read** 可重复读，无论其他事务是否修改并提交了数据，在这个事务中看到的数据值始终不受其他事务影响
- **Read committed** 读取已提交，其他事务提交了对数据的修改后，本事务就能读取到修改后的数据值
- **Read uncommitted** 读取未提交，其他事务只要修改了数据，即使未提交，本事务也能看到修改后的数据值。

MySQL数据库默认使用可重复读（ Repeatable read），而使用乐观锁的时候，如果一个事务修改了库存并提交了事务，那其他的事务应该可以读取到修改后的数据值，所以不能使用可重复读的隔离级别，应该修改为读取已提交Read committed。

**修改方法：**

![打开配置文件](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/%E7%BC%96%E8%BE%91%E6%95%B0%E6%8D%AE%E5%BA%93%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6.jpeg)

![修改隔离级别](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/%E4%BF%AE%E6%94%B9mysql%E7%9A%84%E4%BA%8B%E5%8A%A1%E9%9A%94%E7%A6%BB%E7%BA%A7%E5%88%AB.jpeg)

#### 4.Django中事务介绍

- 在Django中可以通过django.db.transaction模块提供的atomic来定义⼀个事务

- atomic定义事务

  - 装饰器⽤法

    ```python
    from django.db import transaction
    
    @transaction.atomic
    def viewfunc(request):
        # 这些代码会在⼀个事务中执⾏
        ...
    ```

  - with 语句⽤法

    ```python
    def viewfunc(request):
        # 这部分代码不在事务中，会被Django⾃动提交
        ...
        with transaction.atomic():
            # 这部分
    ```

  - 提交回滚事务

    ```python
    # 创建保存点
    save_id = transaction.savepoint()
    # 回滚到保存点
    transaction.savepoint_rollback(save_id)
    # 提交从保存点到当前状态的所有数据库事务操作
    transaction.savepoint_commit(save_id)
    ```

### 3.乐观锁解决并发下单库存问题

#### 乐观锁说明

乐观锁并不是真实存在的锁，⽽是在更新的时候判断此时的库存是否是之前查询出的库存。
如果相同，表示没⼈修改，可以更新库存，否则表示别⼈抢过资源，不再执⾏库存更新。
类似如下操作

```python
update tb_sku set stock=2 where id=1 and stock=7;
SKU.objects.filter(id=1, stock=7).update(stock=2)
```

项目代码:

```python
# 更新库存
new_stock = origin_stock - sku_count
new_sales = origin_sales + sku_count
# 乐观锁更新库存
result = SKU.objects.filter(id=sku_id, stock=origin_stock).update(stock=new_stock, sales=new_sales)
# 如果下单失败，但是库存⾜够时，继续下单，知道下单成功或者库存确实不⾜就停⽌
if result == 0:
    continue
```

### 4.序列化器

```python
from django.utils import timezone
from rest_framework import serializers
from decimal import Decimal
from django_redis import get_redis_connection
from django.db import transaction

from goods.models import SKU
from .models import OrderInfo, OrderGoods


class CommitOrderSerializer(serializers.ModelSerializer):
    """提交订单"""

    class Meta:
        model = OrderInfo
        # order_id ：输出；address 和 pay_method : 输入
        fields = ('order_id', 'address', 'pay_method')
        read_only_fields = ('order_id',)
        # 指定address 和 pay_method 为输出
        extra_kwargs = {
            'address': {
                'write_only': True,
            },
            'pay_method': {
                'write_only': True,
            }
        }

    def create(self, validated_data):
        """重写create方法实现订单表、订单商品表、sku、spu信息的存储
        """
        # 获取当前保存订单时需要的信息
        user = self.context['request'].user
        # order_id == '时间+userid' == '20180914122126000000001'
        order_id = timezone.now().strftime('%Y%m%d%H%M%S') + ('%09d' % user.id)
        address = validated_data.get('address')
        pay_method = validated_data.get('pay_method')

        # 从这里"明显"的开启一次事务
        with transaction.atomic():

            # 在数据操作的之前，就创建一个保存点，将来用于回滚和提交到此
            save_id = transaction.savepoint()

            try:
                # 保存订单基本信息 OrderInfo（一）
                order = OrderInfo.objects.create(
                    order_id=order_id,
                    user = user,
                    address = address,
                    total_count = 0,
                    total_amount = Decimal('0.00'),
                    freight = Decimal('10.00'),
                    pay_method = pay_method,
                    # status = '待支付' if '支付方式是支付宝支付' else '货到付款'
                    status=OrderInfo.ORDER_STATUS_ENUM['UNPAID'] if pay_method == OrderInfo.PAY_METHODS_ENUM['ALIPAY'] else OrderInfo.ORDER_STATUS_ENUM['UNSEND']
                )

                # 从redis读取购物车中被勾选的商品信息
                redis_conn = get_redis_connection('cart')

                # 读取购物车数据
                # redis_cart = {b'sku_id1':b'count_1', b'sku_id2':b'count_2',b'sku_id3':b'count_3'}
                redis_cart = redis_conn.hgetall('cart_%s' % user.id)
                # 读取被勾选的商品的sku_ids
                # redis_selected = [b'sku_id2', b'sku_id3']
                redis_selected = redis_conn.smembers('selected_%s' % user.id)

                # 构造被勾选的商品字典
                # carts = {sku_id2:count_2, sku_id3:coun_3}
                carts = {}
                for sku_id in redis_selected:
                    carts[int(sku_id)] = int(redis_cart[sku_id])

                # 获取要购买的商品的sku_id
                # sku_ids = [sku_id2, sku_id3]
                sku_ids = carts.keys()

                # 遍历购物车中被勾选的商品信息
                for sku_id in sku_ids:
                    while True:
                        # 获取sku对象
                        sku = SKU.objects.get(id=sku_id)

                        # 读取原始库存和销量
                        origin_stock = sku.stock
                        origin_sales = sku.sales

                        # 判断库存 
                        sku_count = carts.get(sku.id)
                        # 如果要购买的商品数量大于库存就抛出异常
                        if sku_count > origin_stock:
                            # 出错就回滚到保存点
                            transaction.savepoint_rollback(save_id)
                            raise serializers.ValidationError('库存不足')

                        # 模拟网络延迟,为了放大多人同时下单的错误效果，没有实际意义
                        # import time
                        # time.sleep(5)

                        # 减少库存，增加销量 SKU 
                        # sku.stock -= sku_count # sku.stock = sku.stock - sku_count
                        # sku.sales += sku_count # sku.sales = sku.sales + sku_count
                        # sku.save()

                        # 使用乐观锁并发下单，修改库存和销量
                        new_stock = origin_stock - sku_count
                        new_sales = origin_sales + sku_count

                        # 在跟新数据之前，使用原始库存查询该商品记录是否存在，如果存在就跟新新的库存；反之，会返回0
                        result = SKU.objects.filter(id=sku_id, stock=origin_stock).update(stock=new_stock, sales=new_sales)
                        if result == 0:
                            # 如果直接抛出异常，表示用户购买商品只有一次机会，即使库存满足也会下单失败
                            # raise serializers.ValidationError('下单失败')
                            continue

                        # 修改SPU销量
                        sku.goods.sales += sku_count
                        sku.goods.save()

                        # 保存订单商品信息 OrderGoods（多）
                        OrderGoods.objects.create(
                            order=order, # order_id=order.id
                            sku = sku, # sku_id=sku.id
                            count = sku_count,
                            price = sku.price
                        )

                        # 累加计算总数量和总价
                        order.total_count += sku_count
                        order.total_amount += (sku.price * sku_count)

                        # 如果下单成功就break，打破while死循环，进入下一个商品的购买
                        break

                # 最后加入邮费和保存订单信息
                order.total_amount += order.freight
                order.save()
            except serializers.ValidationError:
                # 抛出异常以前，不需要在这里回滚，因为之前回滚了才抛出次异常
                raise
            except Exception:
                # 回滚到保存单
                transaction.savepoint_rollback(save_id)
                raise # 只要有异常就抛出，不关心是什么异常

            # 执行成功就提交事务到保存点
            transaction.savepoint_commit(save_id)

        # 清除购物车中已结算的商品
        pl = redis_conn.pipeline()
        pl.hdel('cart_%s' % user.id, *redis_selected)
        pl.srem('selected_%s' % user.id, *redis_selected)
        pl.execute()

        # 响应数据
        return order


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

