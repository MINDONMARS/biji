# Django类视图

[TOC]

提高代码可读性/一个视图用提供多种请求方式(减少逻辑)

- **代码可读性好**
- **类视图相对于函数视图有更高的复用性**， 如果其他地方需要用到某个类视图的某个特定逻辑，**直接继承**该类视图即可

```python
# 导入View
from django.views.generic import View
# 继承View
class RegisterView(View):
    """类视图：处理注册"""

    def get(self, request):
        """处理GET请求，返回注册页面"""
        return render(request, 'register.html')

    def post(self, request):
        """处理POST请求，实现注册逻辑"""
        return HttpResponse('这里实现注册逻辑')
```

## 类视图使用

定义类视图需要继承自Django提供的父类**View**，可使用

`from django.views.generic import View`或者

`from django.views.generic.base import View` 导入。 

```python
# 配置路由时，使用类视图的 as_view() 方法来添加。

urlpatterns = [
    # 视图函数：注册
    # url(r'^register/$', views.register, name='register'),
    # 类视图：注册
    url(r'^register/$', views.RegisterView.as_view(), name='register'),
]
```

## 类视图装饰器

```python
def my_decorator(func):
    def wrapper(request, *args, **kwargs):
        print('自定义装饰器被调用了')
        print('请求路径%s' % request.path)
        return func(request, *args, **kwargs)
    return wrapper

@method_decorator(my_decorator, name='dispatch')  # 给类视图中所有请求方式添加装饰器(可指定'get', 'post'...)
class DemoView(View):
    def get(self, request):
        print('get方法')
        return HttpResponse('ok')

    def post(self, request):
        print('post方法')
        return HttpResponse('ok')
```

在URL配置中装饰(不利于代码完整性, 不建议使用)

```python
urlpatterns = [
    url(r'^demo/$', my_decorate(DemoView.as_view()))
]
```

如果需要为类视图的多个方法添加装饰器，但又不是所有的方法（为所有方法添加装饰器参考上面例子），可以直接在需要添加装饰器的方法上使用method_decorator，如下所示(调用修改参数会出错!!!)

```python
from django.utils.decorators import method_decorator

# 为特定请求方法添加装饰器
class DemoView(View):

    @method_decorator(my_decorator)  # 为get方法添加了装饰器
    def get(self, request):
        print('get方法')
        return HttpResponse('ok')

    @method_decorator(my_decorator)  # 为post方法添加了装饰器
    def post(self, request):
        print('post方法')
        return HttpResponse('ok')

    def put(self, request):  # 没有为put方法添加装饰器
        print('put方法')
        return HttpResponse('ok')
```

## 类视图Mixin扩展类

使用面向对象多继承的特性，可以通过定义父类（作为扩展类），在父类中定义想要向类视图补充的方法，类视图继承这些扩展父类，便可实现代码复用。

定义的扩展父类名称通常以Mixin结尾。

```python
from django.http import HttpResponse
from django.shortcuts import render
from django.template import loader
from django.utils.decorators import method_decorator
from django.views.generic import View


# Create your views here.


def my_decorator(func):
    def wrapper(request, *args, **kwargs):
        print(request.path)
        print(111111)
        response = func(request, *args, **kwargs)
        print(222222)
        return response
    return wrapper


class GetMixin(object):
    def get(self, request):
        print(request.method)
        return HttpResponse('get')


class PostMixin(object):
    def post(self, request):
        print(request.method)
        return HttpResponse('post')


class GetPostMixin(GetMixin, PostMixin):
    pass


@method_decorator(my_decorator, name='dispatch')
class Index(View, GetPostMixin):

    # def get(self, request):
    #     return HttpResponse('get')
    #
    # def post(self, request):
    #     return HttpResponse('post')
    def put(self, request):
        # # 获取模板
        # template = loader.get_template('index.html')

        context = {
            'city': '北京',
            'adict': {
                'name': '西游记',
                'author': '吴承恩'
            },
            'alist': [1, 2, 3, 4, 5]
        }
        # # 渲染模板
        # return HttpResponse(template.render(context))
		
        return render(request, 'index.html', context)
```

