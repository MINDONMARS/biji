# DRF视图高级用法

[TOC]

## 认证Authentication

识别⽤户是谁 (可以是匿名⽤户, 可以是登录⽤户)

### 配置

#### 全局配置(用)

```python
# DRF配置
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework.authentication.BasicAuthentication', # 基本认证
      'rest_framework.authentication.SessionAuthentication', # session认证
    )
}
```

#### 视图中配置

```python
class ExampleView(APIView):
	authentication_classes = (SessionAuthentication, BasicAuthentication)
```

#### 认证失败

	认证失败会有两种可能的返回值：
	• 401 Unauthorized 未认证
	• 403 Permission Denied 权限被禁⽌

## 权限Permissions

权限控制可以限制⽤户对于视图的访问和对于具体数据对象的访问。
• 在执⾏视图的dispatch()⽅法前，会先进⾏视图访问权限的判断
• 在通过get_object()获取具体对象时，会进⾏对象访问权限的判断

### 限制⽤户的权限  (提供的权限)

	• AllowAny 允许所有⽤户
	• IsAuthenticated 仅通过认证的⽤户
	• IsAdminUser 仅管理员⽤户
	• IsAuthenticatedOrReadOnly 认证的⽤户可以完全操作，否则只能get读取

### 配置视图访问权限

#### 全局权限

```python
REST_FRAMEWORK = {
     'DEFAULT_PERMISSION_CLASSES': (
     'rest_framework.permissions.IsAuthenticated',
     )
}
```

#### 视图中权限

```python
class BookInfoViewSet(ReadOnlyModelViewSet):
    """使⽤ReadOnlyModelViewSet实现返回列表数据和单⼀数据
    """
    queryset = BookInfo.objects.all()
    serializer_class = serializers.BookInfoSerializer
    # 只有登录⽤户才能访问
    permission_classes = [IsAuthenticated]
```

### ⾃定义权限

如需⾃定义权限，需继承rest_framework.permissions.BasePermission⽗类，并实现以下两个任何⼀个⽅法或全部
• .has_permission(self, request, view)
是否可以访问视图， view表示当前视图对象
• .has_object_permission(self, request, view, obj)
是否可以访问数据对象， view表示当前视图， obj为数据对象(单一对象)

## 限流Throttling

可以对接⼝访问的频次进⾏限制，以减轻服务器压⼒。

### 配置

#### 全局配置

```python
# DEFAULT_THROTTLE_RATES 可以使⽤ second, minute, hour 或day来指明周期。
# DRF配置
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': (
        'rest_framework.throttling.AnonRateThrottle', # 匿名⽤户限流
        'rest_framework.throttling.UserRateThrottle' # 登录⽤户限流
    ),
    'DEFAULT_THROTTLE_RATES': {
        'anon': '1/minute', # 匿名⽤户限流
        'user': '3/minute' # 登录⽤户限流
    }
}
```

#### 视图中配置

```python
class ExampleView(APIView):
 throttle_classes = (UserRateThrottle,)
```

## 过滤Filtering

1. ### 安装过滤模块 

   `pip install django-filte`

2. ### 注册应⽤

   ```python
   INSTALLED_APPS = [
       'django.contrib.admin',
       'django.contrib.auth',
       'django.contrib.contenttypes',
       'django.contrib.sessions',
       'django.contrib.messages',
       'django.contrib.staticfiles',
   
       'rest_framework', # DRF
       'django_filters', # DRF过滤
   
       'users.apps.UsersConfig', # 安装users应⽤,演示基本使⽤
       'request_response.apps.RequestResponseConfig', # 演示请求和响应
       'booktest.apps.BooktestConfig', # 图书英雄管理应⽤
   ]
   ```

3. ### 配置过滤后端

   ```python
   # DRF配置
   REST_FRAMEWORK = {
       # 过滤后端
       'DEFAULT_FILTER_BACKENDS': ('django_filters.rest_framework.DjangoFilterBackend',)
   }
   ```

4. ### 视图添加过滤字段

   ```python
   class BookInfoViewSet(ReadOnlyModelViewSet):
       """使⽤ReadOnlyModelViewSet实现返回列表数据和单⼀数据
       """
       queryset = BookInfo.objects.all()
       serializer_class = serializers.BookInfoSerializer
       # 只有登录⽤户才能访问
       permission_classes = [IsAuthenticated, MyPermission]
       # 过滤字段
       filter_fields = ('btitle', 'bread')
   ```

5. ### 测试

   http://127.0.0.1:8000/books/?btitle=射雕英雄传
   http://127.0.0.1:8000/books/?bread=0

## 排序OrderingFilter

对于列表数据，REST framework提供了OrderingFilter过滤器来帮助我们快速指明数据按照指定字段进⾏排序。

### 使⽤⽅法

1. 在类视图中设置filter_backends，使⽤rest_framework.filters.OrderingFilter过滤器
2. REST framework会在请求的查询字符串参数中检查是否包含了ordering参数，如果包含了ordering参数，则按照ordering参数指明的排序字段对数据集进⾏排序
3. 前端可以传递的ordering参数的可选字段值需要在ordering_fields中指明
4. 测试:http://127.0.0.1:8000/books/?ordering=-bread(-表示由大到小排序)

```python
class BookInfoViewSet(ReadOnlyModelViewSet):
    """使⽤ReadOnlyModelViewSet实现返回列表数据和单⼀数据
    """
    queryset = BookInfo.objects.all()
    serializer_class = serializers.BookInfoSerializer
    
    # 只有登录⽤户才能访问
    permission_classes = [IsAuthenticated, MyPermission]
    
    # 过滤字段
    filter_fields = ('btitle', 'bread')

    # 排序
    filter_backends = [OrderingFilter]
    ordering_fields = ('id', 'bread', 'bpub_date')
```

## 分⻚Pagination

### 全局配置分⻚后端

```python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 100 # 每⻚数⽬
}
```

### ⾃定义分⻚后端

#### PageNumberPagination

可以在⼦类中定义的属性：
	• page_size 每⻚数⽬
	• page_query_param 前端发送的⻚数关键字名，默认为"page"
	• page_size_query_param 前端发送的每⻚数⽬关键字名，默认为None
	• max_page_size 前端最多能设置的每⻚数量

#### ⾃定义分⻚类

```python
class LargeResultsSetPagination(PageNumberPagination):
    """⾃定义分⻚器"""
    page_size = 2 # 每⻚记录条数
    page_size_query_param = 'page_size' # 接受每⻚条数的关键字
    max_page_size = 10 # 每⻚最多记录条数
```

使用方式

```python
class BookInfoViewSet(ReadOnlyModelViewSet):
    """使⽤ReadOnlyModelViewSet实现返回列表数据和单⼀数据
    """
    queryset = BookInfo.objects.all()
    serializer_class = serializers.BookInfoSerializer
    # 只有登录⽤户才能访问
    permission_classes = [IsAuthenticated, MyPermission]
    # 过滤字段
    filter_fields = ('btitle', 'bread')
    # 排序
    filter_backends = [OrderingFilter]
    ordering_fields = ('id', 'bread', 'bpub_date')
    # 分⻚
    pagination_class = LargeResultsSetPagination
```

#### 视图中关闭分⻚功能

`pagination_class = None`

#### 其他分⻚类(偏移分页)

LimitOffsetPagination

可以在⼦类中定义的属性：
	• default_limit 默认限制，默认值与PAGE_SIZE设置⼀直
	• limit_query_param limit参数名，默认'limit'
	• offset_query_param offset参数名，默认'offset'
	• max_limit 最⼤limit限制，默认None

```python
from rest_framework.pagination import LimitOffsetPagination
class BookListView(ListAPIView):
    queryset = BookInfo.objects.all().order_by('id')
    serializer_class = BookInfoSerializer
    pagination_class = LimitOffsetPagination
# 127.0.0.1:8000/books/?offset=3&limit=2
```

## 异常处理 Exceptions

REST framework提供了异常处理，我们可以⾃定义异常处理函数。

创建exceptions.py文件

(捕获数据库异常)

```python
from rest_framework.views import exception_handler
from rest_framework import status
from django.db import DatabaseError
from rest_framework.response import Response
def custom_exception_handler(exc, context):
    """⾃定义异常
    补充捕获DRF以外的异常信息
    """
    response = exception_handler(exc, context)
    print(exc)
    if response is None:
    	if isinstance(exc, DatabaseError):
    		response = Response({'detail': '服务器内部错误'}, status=status.HTTP_507_INSUFFICIENT_STORAGE)
    return response
```

⾃定义异常的使⽤(写入配置文件)

```python
# DRF配置
REST_FRAMEWORK = {
    # 指定⾃定义的异常处理
    'EXCEPTION_HANDLER': 'exceptions.custom_exception_handler'
}
```

REST framework定义的异常
	• APIException 所有异常的⽗类
	• ParseError 解析错误
	• AuthenticationFailed 认证失败
	• NotAuthenticated 尚未认证
	• PermissionDenied 权限决绝
	• NotFound 未找到
	• MethodNotAllowed 请求⽅式不⽀持
	• NotAcceptable 要获取的数据格式不⽀持
	• Throttled 超过限流次数
	• ValidationError 校验失败

## ⾃动⽣成接⼝⽂档

REST framework可以⾃动帮助我们⽣成接⼝⽂档。
接⼝⽂档以⽹⻚的⽅式呈现。
⾃动接⼝⽂档能⽣成的是继承⾃APIView及其⼦类的视图。

#### 安装依赖包 

REST framewrok⽣成接⼝⽂档需要coreapi库的⽀持。
`pip install coreapi`

设置接⼝⽂档访问路径

```python
from rest_framework.documentation import include_docs_urls
urlpatterns = [
    ...
    url(r'^docs/', include_docs_urls(title='My API title'))
]
```

#### 访问

 http://127.0.0.1:8000/docs/

#### ⽂档描述说明的定义位置

1.单⼀⽅法的视图，可直接使⽤类视图的⽂档字符串

```python
class BookListView(generics.ListAPIView):
    """
    返回所有图书信息.
    """
```

2.包含多个⽅法的视图，在类视图的⽂档字符串中，分开⽅法定义

```python
class BookListCreateView(generics.ListCreateAPIView):
    """
    get:
    返回所有图书信息.
    post:
    新建图书
```

3.对于视图集ViewSet，仍在类视图的⽂档字符串中封开定义，但是应使⽤action名称区分

```python
class BookInfoViewSet(mixins.ListModelMixin, mixins.RetrieveModelMixin, GenericViewSet):
    """
    list:
    返回图书列表数据
    retrieve:
    返回图书详情数据
    latest:
    返回最新的图书数据
    read:
    修改图
```

#### 补充

两点说明：
1） 视图集ViewSet中的retrieve名称，在接⼝⽂档⽹站中叫做read
2）参数的Description需要在模型类或序列化器类的字段中以help_text选项定义