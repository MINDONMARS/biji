# DjangoAdmin站点管理

[TOC]

## 管理界面本地化

```python
LANGUAGE_CODE = 'zh-hans' # 使用中国语言
TIME_ZONE = 'Asia/Shanghai' # 使用中国上海时间
```

## 创建超级管理员

键入如下代码依次输入用户名， 邮箱， 密码， 确认密码

```
python manage.py createsuperuser
```

登录

```
http://127.0.0.1:8000/admin/
```

## App应用配置

apps.py文件 

 在配置类中写入如下，在admin站点中会显示应用直观可读的名字

```
verbose_name = '图书管理'
```

## 注册模型类

admin.py文件 

```python
from django.contrib import admin
from demo_class.models import BookInfo,HeroInfo

admin.site.register(BookInfo)
admin.site.register(HeroInfo)
# 到浏览器中刷新页面，可以看到模型类BookInfo和HeroInfo的管理
```

## 定义与使用Admin管理类

定义管理类继承自**admin.ModelAdmin**类 

admin.py文件

```python
from django.contrib import admin

class BookInfoAdmin(admin.ModelAdmin):
    pass
```

使用管理类有两种方式：

- 注册参数

  ```python
  admin.site.register(BookInfo,BookInfoAdmin)
  ```

- 装饰器

  ```python
  @admin.register(BookInfo)
  class BookInfoAdmin(admin.ModelAdmin):
      pass
  ```

## 调整列表页展示

#### 每页条数

```python
# 默认100
list_per_page=100
```

#### "操作选项"的位置

```python
# 默认
actions_on_top=True 
actions_on_bottom=False

# 修改
class BookInfoAdmin(admin.ModelAdmin):
    ...
    actions_on_top = True
    actions_on_bottom = True
```

#### 列表中的列

```
class BookInfoAdmin(admin.ModelAdmin):
    ...
    list_display = ['id','btitle']
```

#### 将方法作为列

在模型类中定义方法（有返回值）

**通过设置short_description属性，可以设置在admin站点中显示的列名。** 

```python
class BookInfo(models.Model):
    ...
    def pub_date(self):
        return self.bpub_date

    pub_date.short_description = '发布日期'  # 设置方法字段在admin中显示的标题
```

```python
# 修改BookInfoAdmin类如下：
class BookInfoAdmin(admin.ModelAdmin):
    ...
    list_display = ['id','atitle','pub_date']
```

方法列是不能排序的，如果需要排序需要为方法指定排序依据。

```python
# admin_order_field=模型类字段
class BookInfo(models.Model):
    ...
    def pub_date(self):
        return self.bpub_date

    pub_date.short_description = '发布日期'
    pub_date.admin_order_field = 'bpub_date'
```

#### 关联对象

在模型类中封装方法，访问关联对象的成员

修改HeroInfo类如下：

```python
class HeroInfo(models.Model):
    ...
    def read(self):
        return self.hbook.bread

    read.short_description = '图书阅读量'
```

修改HeroInfoAdmin类如下：

```python
class HeroInfoAdmin(admin.ModelAdmin):
    ...
    list_display = ['id', 'hname', 'hbook', 'read']
```

#### 右侧栏过滤器

修改HeroInfoAdmin类如下：

```python
class HeroInfoAdmin(admin.ModelAdmin):
    ...
    list_filter = ['hbook', 'hgender']
```

#### 搜索框

修改HeroInfoAdmin类如下：

```python
class HeroInfoAdmin(admin.ModelAdmin):
    ...
    search_fields = ['hname']
```
## admin详情页

### 显示字段

fields = []

```python
class BookInfoAdmin(admin.ModelAdmin):
    ...
    fields = ['btitle', 'bpub_date']
```

### 分组显示

```python
"""
fieldset = (
    ('组1标题',{'fields':('字段1','字段2')}),
    ('组2标题',{'fields':('字段3','字段4')}),
)
class BookInfoAdmin(admin.ModelAdmin):
"""
    ...
    # fields = ['btitle', 'bpub_date']
    fieldsets = (
        ('基本', {'fields': ['btitle', 'bpub_date']}),
        ('高级', {
            'fields': ['bread', 'bcomment'],
            'classes': ('collapse',)  # 是否折叠显示
        })
    )
```

说明：fields与fieldsets两者选一使用。 

#### 关联对象

在一对多的关系中，可以在一端的编辑页面中编辑多端的对象，嵌入多端对象的方式包括表格、块两种。 

- 类型InlineModelAdmin：表示在模型的编辑页面嵌入关联模型的编辑。

- 子类StackedInline：以块的形式嵌入。

- 子类TabularInline：以表格的形式嵌入。


admin.py文件，创建HeroInfoStackInline类。

```python
class HeroInfoStackInline(admin.StackedInline):
    model = HeroInfo  # 要编辑的对象
    extra = 1  # 附加编辑的数量
```

修改BookInfoAdmin类如下：

```python
class BookInfoAdmin(admin.ModelAdmin):
    ...
    inlines = [HeroInfoStackInline]
```

创建HeroInfoTabularInline类。

```python
class HeroInfoTabularInline(admin.TabularInline):
    model = HeroInfo
    extra = 1
```

修改BookInfoAdmin类如下：

```python
class BookInfoAdmin(admin.ModelAdmin):
    ...
    inlines = [HeroInfoTabularInline]
```

## 调整站点信息

Admin站点的名称信息可以自定义 , 直接写在admin.py中

### 网站标签页信息

```python
admin.site.site_title = 'xxx'
```

![1535423604947](C:\Users\ADMINI~1\AppData\Local\Temp\1535423604947.png)

### 页面标题

```python
admin.site.site_header = 'xxx'
```

![1535423424862](C:\Users\ADMINI~1\AppData\Local\Temp\1535423424862.png)

### 首页标题

```python
admin.site.index_title = 'kakaka'
```

![1535423683119](C:\Users\ADMINI~1\AppData\Local\Temp\1535423683119.png)

## 上传图片

Django有提供文件系统支持，在Admin站点中可以轻松上传图片。

使用Admin站点保存图片，需要安装Python的图片操作包

```
pip install Pillow
```

### 配置

默认情况下，Django会将上传的图片保存在本地服务器上，需要配置保存的路径。

我们可以将上传的文件保存在静态文件目录中，如我们之前设置的static_files目录中在settings.py 文件中添加如下上传保存目录信息

```
MEDIA_ROOT=os.path.join(BASE_DIR,"static_files/media")
```

## 为模型类添加ImageField字段

我们为之前的BookInfo模型类添加一个ImageFiled

```python
class BookInfo(models.Model):
    ...
    image = models.ImageField(upload_to='book_img', verbose_name='图片', null=True)
```

- upload_to 选项指明该字段的图片保存在MEDIA_ROOT目录中的哪个子目录

进行数据库迁移操作

```
python manage.py makemigrations
python manage.py migrate
```

## 使用Admin站点上传图片

进入Admin站点的图书管理页面，选择一个图书，能发现多出来一个上传图片的字段

选择一张图片并保存后，图片会被保存在**static_files/media/book_img/**目录下。

在数据库中，我们能看到image字段被设置为图片的路径