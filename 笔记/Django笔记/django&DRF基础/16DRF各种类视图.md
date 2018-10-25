# DRF类视图

[TOC]

## DRF中类视图概览

![通用视图的继承关系](F:\python就业班课件\Django基础\Django框架基础_day05\02-其他资料\通用视图的继承关系.png)

## 使⽤APIView基类视图

### APIView基类视图介绍

##### 1.位置 rest_framework.views.APIView

##### 2.继承 APIView是REST framework提供的所有视图的基类，继承⾃Django的View⽗类

##### 3.APIView与View对⽐

• 传⼊到视图⽅法中的是REST framework的Request对象，⽽不是Django的HttpRequeset对象；
• 视图⽅法可以返回REST framework的Response对象，视图会为响应数据设置（render）符合前端要求的格式；
• 任何APIException异常都会被捕获到，并且处理成合适的响应信息；
• 在进⾏dispatch()分发前，会对请求进⾏身份认证、权限检查、流量控制。

##### 4.相同点

在APIView中仍以常规的类视图定义⽅法来实现get() 、post() 或者其他请求⽅式的⽅法。
依然以 as_view() 做请求分发

##### 5.⽀持定义的属性：

• authentication_classes 列表或元祖，身份认证类
• permissoin_classes 列表或元祖，权限检查类
• throttle_classes 列表或元祖，流量控制类

### 例: 需求 使⽤APIView实现“获取所有图书信息”接⼝

后端接⼝定义

```python
# 视图
class BookListView(APIView):
	def get(self, request):
        """
        GET /books/
        :param request: Request类型的对象
        :return: JSON
        """	
 		pass
# 路由 # 演示APIView,实现获取所有图书信息接⼝
url(r'^books/$', views.BookListView.as_view()),
```

```python
# 序列化器
# 可使⽤之前定义过的序列化器
class BookInfoSerializer(serializers.Serializer):
	pass
class BookInfoModelSerializer(serializers.ModelSerializer):
	pass
```

## 使⽤GenericAPIView基类视图

**GenericAPIView**基类视图介绍:

1.位置:  rest_framework.generics.GenericAPIView

2.继承:  继承⾃APIVIew

3.说明:  主要增加了操作序列化器和数据库查询的⽅法，作⽤是为下⾯Mixin扩展类的执⾏提供⽅法⽀持。**通常在使⽤时，可搭配⼀个或多个Mixin扩展类**。

**提供的关于序列化器使⽤的属性与⽅法**

• 属性：
◦ **serializer_class** 指明视图使⽤的序列化器
• ⽅法：
◦ **get_serializer_class(self)**
返回序列化器类，默认返回serializer_class，可以重写
◦ **get_serializer(self, args, *kwargs)**
返回序列化器对象，主要⽤来提供给Mixin扩展类使⽤，如果我们在视图中想要获取序列化器对象，也可以直接调⽤此⽅法。

**注意** get_serializer(self, args, *kwargs)

<u>该⽅法在提供序列化器对象的时候，会向序列化器对象的context属性补充三个数据：request、format、view，这三个数据对象可以在定义序列化器时使⽤。</u>
• request 当前视图的请求对象
• view 当前请求的类视图对象
• format 当前请求期望返回的数据格式

**提供的关于数据库查询的属性与⽅法**

• 属性：
◦ queryset 指明使⽤的数据查询集
• ⽅法：
◦ get_queryset(self)
返回视图使⽤的查询集，主要⽤来提供给Mixin扩展类使⽤，是列表视图与详情视图获取数据的基础，默认返回queryset属性，可以重写
◦ get_object(self)
返回详情视图所需的模型类数据对象，主要⽤来提供给Mixin扩展类使⽤。
在试图中可以调⽤该⽅法获取详情信息的模型类对象。
若详情访问的模型类对象不存在，会返回404。
该⽅法会默认使⽤APIView提供的check_object_permissions⽅法检查当前对象是否有权限被访问。

**其他可以设置的属性**

• pagination_class 指明分⻚控制类
• filter_backends 指明过滤控制后端

### 例: 需求 

### 1.使⽤GenericAPIView实现“获取所有图书信息”接⼝
2.使⽤GenericAPIView实现“获取单⼀图书信息”接⼝

```python
# 所有
# 视图
class BookListView(GenericAPIView):
    # 指定查询集
    queryset = BookInfo.objects.all()
    # 指定序列化器
    serializer_class = BookInfoSerializer
    def get(self, request):
        """
        GET /books/
        :param request: Request类型的对象
        :return: JSON
        """
        # 查询数据库
        qs = self.get_queryset()
        # 实现序列化
        serializer = self.get_serializer(qs, many=True)
        # 响应序列化结果
        return Response(serializer.data)
    
# 序列化器
可使⽤之前定义过的序列化器
class BookInfoSerializer(serializers.Serializer):
 pass
class BookInfoModelSerializer(serializers.ModelSerializer):
 pass
# 路由 # 演示APIView,GenericAPIView实现获取所有图书信息接⼝
url(r'^books/$', views.BookListView.as_view()),

# 单一
class BookDetailView(GenericAPIView):
    # 指定查询集
    queryset = BookInfo.objects.all()
    # 指定序列化器
    serializer_class = BookInfoSerializer
    def get(self, request, pk):
        """
        GET /books/<pk>/
        :param request: Request类型的对象
        :param pk: 要访问的数据库记录
        :return: JSON
        """
        # 查询数据库:默认根据pk查询数据库单⼀结果
        book = self.get_object()
        # 实现序列化
        serializer = self.get_serializer(book)
        # 响应序列化结果
        return Response(serializer.data)
# 序列化器
# .路由 # 演示GenericAPIView,实现获取单⼀图书信息接⼝
url(r'^books/(?P<pk>\d+)/$', views.BookDetailView.as_view()),
```

## 使⽤Mixin扩展类(继承GenericAPIView⽗类)

##### Mixin扩展类介绍

提供了⼏种后端视图（对数据资源进⾏增删改查）处理流程的实现，如果需要编写的视图属于这五种，则视图可以通过继承相应的扩展类来复⽤代码，减少⾃⼰编写的代码量。
这五个扩展类需要搭配GenericAPIView⽗类，因为五个扩展类的实现需要调⽤GenericAPIView提供的序列化器与数据库查询的⽅法。

```
ListModelMixin
CreateModelMixin
RetrieveModelMixin
UpdateModelMixin
DestroyModelMixin
```

### 例: 需求 

### 使⽤mixins扩展类搭配GenericAPIView基类视图，实现“获取所有图书信息”接⼝

```python
class BookListView(mixins.ListModelMixin, GenericAPIView):
    # 指定查询集
    queryset = BookInfo.objects.all()
    # 指定序列化器
    serializer_class = BookInfoSerializer
    def get(self, request):
        """
        GET /books/
        :param request: Request类型的对象
        :return: JSON
        """
    	return self.list(request)
```

## 使⽤GenericAPIView的⼦类视图

##### ⼦类视图介绍

```
CreateAPIView
ListAPIView
RetrieveAPIView
DestoryAPIView
UpdateAPIView
RetrieveUpdateAPIView
。。。。。。
```

### 例: 需求 

### 使⽤使⽤GenericAPIView的⼦类视图，实现“获取所有图书信息”接⼝

```python
# GET /books/
class BookListView(ListAPIView):
    # 指定查询集
    queryset = BookInfo.objects.all()
    # 指定序列化器
    serializer_class = BookInfoSerializer
```

