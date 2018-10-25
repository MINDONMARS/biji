# DRF视图集

[TOC]

## 一 视图集介绍

### 视图集概览

![视图集类继承关系](F:\python就业班课件\Django基础\Django框架基础_day06\02-其他资料\视图集类继承关系.png)

### 提供的增删改查/⾏为/

	• list() 提供⼀组数据
	• retrieve() 提供单个数据
	• create() 创建数据
	• update() 保存数据
	• destory() 删除数据

### 对于⾏为和分发⾏为的说明

	视图集类不再实现get()、post()等⽅法，⽽是实现动作 action 如 list() 、create() 等。
	视图集只在使⽤as_view()⽅法的时候，才会将action动作与具体请求⽅式对应上。

## 二 视图集基本使⽤

1. ### ViewSet的使⽤

   - ViewSet继承⾃APIView

   - 对于提供的五种⾏为，需要⾃⼰在⼦类中实现

     ```python
     class BookInfoViewSet(ViewSet):
         """ViewSet获取列表和单⼀数据"""
         def list(self, request):
             qs = BookInfo.objects.all()
             serializer = serializers.BookInfoSerializer(qs, many=True)
         	return Response(serializer.data)
         def retrieve(self, request, pk):
             book = BookInfo.objects.get(id=pk)
             serializer = serializers.BookInfoSerializer(book)
             return Response(serializer.data)
     ```

2. ### GenericViewSet的使⽤

   - 继承⾃GenericAPIView

   - 可以搭配mixins扩展类使⽤ ⽆需⾃⼰定义和实现⾏为

     ```python
     # 使⽤视图集获取列表数据和单⼀数据
     class BookInfoViewSet(mixins.ListModelMixin, mixins.RetrieveModelMixin, GenericViewSet):
         """使⽤GenericViewSet实现返回列表数据和单⼀数据"""
         queryset = BookInfo.objects.all()
     	serializer_class = BookInfoSerializer
        
     # 序列化器
     # 可使⽤之前定义过的序列化器
     class BookInfoSerializer(serializers.Serializer):
     	pass
     class BookInfoModelSerializer(serializers.ModelSerializer):
     	pass
     # 路由
     # 演示viewset
     url(r'^books/$', views.BookInfoViewSet.as_view({'get': 'list'})),
     url(r'^books/(?P<pk>\d+)/$', views.BookInfoViewSet.as_view({'get': 'retrieve'})),
     ```

3. ### ReadOnlyModelViewSet的使⽤(封装了上面的继承类)

   **使⽤⽅式**

   ```python
   class BookInfoViewSet(ReadOnlyModelViewSet):
       """使⽤ReadOnlyModelViewSet实现返回列表数据和单⼀数据"""
       queryset = BookInfo.objects.all()
       serializer_class = serializers.BookInfoSerializer
   ```

4. ### ModelViewSet说明

   ```python
   # 源码
   class ModelViewSet(mixins.CreateModelMixin,
       mixins.RetrieveModelMixin,
       mixins.UpdateModelMixin,
       mixins.DestroyModelMixin,
       mixins.ListModelMixin,
       GenericViewSet):
       """
       A viewset that provides default `create()`, `retrieve()`, `update()`,
       `partial_update()`, `destroy()` and `list()` actions.
       """
       pass
   ```

## 三 视图集中⾃定义action动作

### 例: 需求

获取倒叙后的最新书籍数据
修改书籍阅读量

```python
# 视图
class BookInfoViewSet(ReadOnlyModelViewSet):
    """使⽤ReadOnlyModelViewSet实现返回列表数据和单⼀数据
    """
    queryset = BookInfo.objects.all()
    serializer_class = serializers.BookInfoSerializer
    def latest(self, request):
        """获取倒叙后的最新书籍数据
        """
        book = BookInfo.objects.latest('id')
        serializer = serializers.BookInfoSerializer(book)
        return Response(serializer.data)
    def update_bread(self, request, pk):
        """修改书籍阅读量
        """
        book = self.get_object()
        book.bread = request.data.get('bread')
        book.save()
        serializer = serializers.BookInfoS
        
        
# 路由
url(r'^books/$', views.BookInfoViewSet.as_view({'get': 'list'})),
url(r'^books/(?P<pk>\d+)/$', views.BookInfoViewSet.as_view({'get': 'retrieve'})),
url(r'^books/latest/$', views.BookInfoViewSet.as_view({'get': 'latest'})),
url(r'^books/(?P<pk>\d+)/update_bread/$', views.BookInfoViewSet.as_view({'put':'update_bread'}))
```

### action属性补充(可以根据不同的⾏为，指定不同的序列化器)

```python
def get_serializer_class(self):
    if self.action == 'create':
    return OrderCommitSerializer
    else:
    return OrderDataSerializer
```

### 定义路由Routers(路由生成器)

	REST framework提供了两个router
	• **SimpleRouter**
	• **DefaultRouter**

	**例**:  使⽤DefaultRouter定义路由

```python
# 创建路由对象
router = DefaultRouter()
# 将视图集注册到路由
router.register(r'books', views.BookInfoViewSet, base_name='book')
# 视图集路由添加到urlpatterns
urlpatterns += router.urls
```

	**视图集中附加action的声明**

	如果希望DefaultRouter定义路由时把⾃⼰追加的⾏为也定义到路由中，就需要使⽤装饰器，在视图集中附加action的声明

```python
class BookInfoViewSet(ReadOnlyModelViewSet):
    """使⽤ReadOnlyModelViewSet实现返回列表数据和单⼀数据
    """
    queryset = BookInfo.objects.all()
    serializer_class = serializers.BookInfoSerializer
    
    # detail为False 表示路径名格式应该为 books/latest/
    @action(methods=['get'], detail=False)
    def latest(self, request):
        """获取倒叙后的最新书籍数据
        """
        book = BookInfo.objects.latest('id')
        serializer = serializers.BookInfoSerializer(book)
        return Response(serializer.data)
    
    # detail为True，表示路径名格式应该为 books/{pk}/update_bread/
    @action(methods=['put'], detail=True)
    def update_bread(self, request, pk):
        """修改书籍阅读量
        """
        book = self.get_object()
        book.bread = request.data.get('bread')
        book.save()
        serializer = serializers.BookInfoSerializer(book)
        return Response(serializer.data)
```

