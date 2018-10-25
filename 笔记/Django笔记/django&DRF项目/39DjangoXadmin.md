# DjangoXadmin

[TOC]

![xadmin](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/xadmin.png)

xadmin是Django的第三方扩展，可是使Django的admin站点使用更方便。

## 1. 安装

通过如下命令安装xadmin的最新版

```shell
pip install https://github.com/sshwsfc/xadmin/tarball/master
```

在配置文件中注册如下应用

```python
INSTALLED_APPS = [
    ...
    'xadmin',
    'crispy_forms',
    'reversion',
    ...
]
```

xadmin有建立自己的数据库模型类，需要进行数据库迁移

```shell
python manage.py makemigrations
python manage.py migrate
```

在总路由中添加xadmin的路由信息

```python
import xadmin

urlpatterns = [
    # url(r'^admin/', admin.site.urls),
    url(r'xadmin/', include(xadmin.site.urls)),
    ...
]
```

## 2. 使用

- xadmin不再使用Django的admin.py，而是需要编写代码在adminx.py文件中。
- xadmin的站点管理类不用继承`admin.ModelAdmin`，而是直接继承`object`即可。

在goods应用中创建adminx.py文件。

#### 站点的全局配置

```python
import xadmin
from xadmin import views

from . import models

class BaseSetting(object):
    """xadmin的基本配置"""
    enable_themes = True  # 开启主题切换功能
    use_bootswatch = True

xadmin.site.register(views.BaseAdminView, BaseSetting)

class GlobalSettings(object):
    """xadmin的全局配置"""
    site_title = "美多商城运营管理系统"  # 设置站点标题
    site_footer = "美多商城集团有限公司"  # 设置站点的页脚
    menu_style = "accordion"  # 设置菜单折叠

xadmin.site.register(views.CommAdminView, GlobalSettings)
```

![xadmin全局配置](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/xadmin%E5%85%A8%E5%B1%80%E9%85%8D%E7%BD%AE.png)

#### 站点Model管理

xadmin可以使用的页面样式控制基本与Django原生的admin一直。

- **list_display** 控制列表展示的字段
- **search_fields** 控制可以通过搜索框搜索的字段名称，xadmin使用的是模糊查询
- **list_filter** 可以进行过滤操作的列
- **ordering** 默认排序的字段
- **readonly_fields** 在编辑页面的只读字段
- **exclude** 在编辑页面隐藏的字段
- **list_editable** 在列表页可以快速直接编辑的字段
- **show_detail_fileds** 在列表页提供快速显示详情信息
- **refresh_times** 指定列表页的定时刷新
- **list_export** 控制列表页导出数据的可选格式
- **show_bookmarks** 控制是否显示书签功能
- **data_charts** 控制显示图标的样式
- **model_icon** 控制菜单的图标

1）model_icon

```python
class SKUAdmin(object):
    model_icon = 'fa fa-gift'

xadmin.site.register(models.SKU, SKUAdmin)
```

![model_icon](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/model_icon.png)

可选的图标样式参考<http://fontawesome.dashgame.com>/

2） list_display

```python
    list_display = ['id', 'name', 'price', 'stock', 'sales', 'comments']
```

![list_display](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/list_display.png)

3）search_fields

```python
    search_fields = ['id','name']
```

![search_fields](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/search_fields.png)

4）list_filter

```python
    list_filter = ['category']
```

![list_filter](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/list_filter.png)

5）list_editable

```python
    list_editable = ['price', 'stock']
```

![list_editable](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/list_editable.png)![list_editable](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/list_editable2.png)

6）show_detail_fields

```python
    show_detail_fields = ['name']
```

![show_detail_fields](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/show_detail_fields.png)

![show_detail_fields2](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/show_detail_fields2.png)

7）show_bookmarks

```python
    show_bookmarks = True
```

![bookmarks](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/bookmarks.png)

8）list_export

```python
    list_export = ['xls', 'csv', 'xml']
```

![list_export](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/list_export.png)

注意，导出到xls（excel) 需要安装**xlwt**扩展

9）refresh_times

```python
class OrderAdmin(object):
    list_display = ['order_id', 'create_time', 'total_amount', 'pay_method', 'status']
    refresh_times = [3, 5]  # 可选以支持按多长时间(秒)刷新页面
```

![refresh_times](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/refresh_times.png)

10）data_charts

```python
     data_charts = {
        "order_amount": {'title': '订单金额', "x-field": "create_time", "y-field": ('total_amount',),
                       "order": ('create_time',)},
        "order_count": {'title': '订单量', "x-field": "create_time", "y-field": ('total_count',),
                       "order": ('create_time',)},
    }
```

- title 控制图标名称
- x-field 控制x轴字段
- y-field 控制y轴字段，可以是多个值
- order 控制默认排序

![data_charts](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/data_charts.png)

11）readonly_fields

```python
class SKUAdmin(object):
    ...
    readonly_fields = ['sales', 'comments']
```

![readonly_fields](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/readonly_fields.png)

#### 站点保存对象数据方法重写

在Django的原生admin站点中，如果想要在站点保存或删除数据时，补充自定义行为，可以重写如下方法：

- `save_model(self, request, obj, form, change)`
- `delete_model(self, request, obj)`

而在xadmin中，需要重写如下方法：

- `save_models(self)`
- `delete_model(self)`

在方法中，如果需要用到当前处理的模型类对象，需要通过`self.obj`来获取，如

```python
class SKUSpecificationAdmin(object):
    def save_models(self):
        # 保存数据对象
        obj = self.new_obj
        obj.save()

        # 补充自定义行为
        from celery_tasks.html.tasks import generate_static_sku_detail_html
        generate_static_sku_detail_html.delay(obj.sku.id)

    def delete_model(self):
        # 删除数据对象
        obj = self.obj
        sku_id = obj.sku.id
        obj.delete()

        # 补充自定义行为
        from celery_tasks.html.tasks import generate_static_sku_detail_html
        generate_static_sku_detail_html.delay(sku_id)
```

#### 自定义用户管理

xadmin会自动为admin站点添加用户User的管理配置

xadmin使用xadmin.plugins.auth.UserAdmin来配置

如果需要自定义User配置的话，需要先unregister(User)，在添加自己的User配置并注册

```python
import xadmin
# Register your models here.

from .models import User
from xadmin.plugins import auth


class UserAdmin(auth.UserAdmin):
    list_display = ['id', 'username', 'mobile', 'email', 'date_joined']
    readonly_fields = ['last_login', 'date_joined']
    search_fields = ('username', 'first_name', 'last_name', 'email', 'mobile')
    style_fields = {'user_permissions': 'm2m_transfer', 'groups': 'm2m_transfer'}

    def get_model_form(self, **kwargs):
        if self.org_obj is None:
            self.fields = ['username', 'mobile', 'is_staff']

        return super().get_model_form(**kwargs)


xadmin.site.unregister(User)
xadmin.site.register(User, UserAdmin)
```

## 3.用户权限控制说明

在产品运营平台中，是需要对用户进行权限控制的。Django实现了用户权限的控制

- 消费者用户与公司内部运营用户使用一个用户数据库来存储
- 通过is_staff 来区分是运营用户还是消费者用户
- 对于运营用户通过is_superuser 来区分是运营平台的管理员还是运营平台的普通用户
- 对于运营平台的普通用户，通过权限、组和组外权限来控制这个用户在平台上可以操作的数据。
- 对于权限，Django会为每个数据库表提供增、删、改、查四种权限
- 用户最终的权限为 组权限 + 用户特有权限

![权限控制](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/django%E6%9D%83%E9%99%90%E6%8E%A7%E5%88%B6.png)

![用户权限](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/%E7%94%A8%E6%88%B7%E6%9D%83%E9%99%90.png)

