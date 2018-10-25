# Django商品⾸⻚(静态化处理)

[TOC]

## 1.⻚⾯静态化介绍

### 1.为什么要进⾏⻚⾯静态化？

1.美多商城的主⻚频繁被访问，⽽且查询数据量⼤
2.为了减少数据库查询次数和提升访问速度，所以进⾏⻚⾯静态化

### 2.什么是⻚⾯静态化？

1.将动态渲染⽣成的⻚⾯结果保存成html⽂件，放到静态⽂件服务器中
2.⽤户访问的时候访问的直接是处理好之后的html静态⽂件

### 3.⻚⾯静态化注意点

对于⻚⾯中属于每个不同⽤户的不同数据不能静态化: ⽤户名、购物⻋ 不能静态化

可以在⽤户请求完静态化之后的⻚⾯后，在⻚⾯中向后端发送请求，获取属于⽤户的特殊的数据。

## 2.主⻚⻚⾯静态化步骤

### 1.主⻚静态化⼯具⽅法代码说明

contents.crons.py 主⻚数据都是⼴告数据

```python
from collections import OrderedDict
from django.conf import settings
from django.template import loader
import os
import time

from goods.models import GoodsChannel
from .models import ContentCategory


def generate_static_index_html():
    """
    生成静态的主页html文件
    """
    print('%s: generate_static_index_html' % time.ctime())
    # 商品频道及分类菜单
    # 使用有序字典保存类别的顺序
    # categories = {
    #     1: { # 组1
    #         'channels': [{'id':, 'name':, 'url':},{}, {}...],
    #         'sub_cats': [{'id':, 'name':, 'sub_cats':[{},{}]}, {}, {}, ..]
    #     },
    #     2: { # 组2
    #
    #     }
    # }
    categories = OrderedDict()
    channels = GoodsChannel.objects.order_by('group_id', 'sequence')
    for channel in channels:
        group_id = channel.group_id  # 当前组

        if group_id not in categories:
            categories[group_id] = {'channels': [], 'sub_cats': []}

        cat1 = channel.category  # 当前频道的类别

        # 追加当前频道
        categories[group_id]['channels'].append({
            'id': cat1.id,
            'name': cat1.name,
            'url': channel.url
        })
        # 构建当前类别的子类别
        for cat2 in cat1.goodscategory_set.all():
            cat2.sub_cats = []
            for cat3 in cat2.goodscategory_set.all():
                cat2.sub_cats.append(cat3)
            categories[group_id]['sub_cats'].append(cat2)

    # 广告内容
    contents = {}
    content_categories = ContentCategory.objects.all()
    for cat in content_categories:
        contents[cat.key] = cat.content_set.filter(status=True).order_by('sequence')

    # 渲染模板
    context = {
        'categories': categories,
        'contents': contents
    }

    # 获取要渲染的模板文件
    # loader.get_template('index.html') : 使用模板加载器，从/templates/文件夹中读取名为index.html的文件
    template = loader.get_template('index.html')

    # 使用查询出来的上下文渲染index.html
    # html_text : 表示html字符串，下一步需要将html字符串写入到html文件中
    html_text = template.render(context)
    # print(html_text)

    # 静态的主页会被存储到/front_end_pc/文件夹：../front_end_pc/index.html
    file_path = os.path.join(settings.GENERATED_STATIC_HTML_FILES_DIR, 'index.html')

    # encoding ： 用于解决在定时器执行时中文字符编码的问题
    with open(file_path, 'w', encoding='utf-8') as f:
        f.write(html_text)
```

### 2.准备静态主⻚存放位置

将静态⽣成的主⻚放在 front_end_pc/ ⽂件夹下

配置

```python
# 静态化主⻚存储路径
GENERATED_STATIC_HTML_FILES_DIR = os.path.join(os.path.dirname(os.path.dirname(BASE_DIR)), 'front_end_pc')

```

### 3.准备主⻚模板⽂件

1.准备模板⽂件夹

### 2.指定模板加载路径

```python
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [os.path.join(BASE_DIR, 'templates')],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                    'django.template.context_processors.request',
                    'django.contrib.auth.context_processors.auth',
                    'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
```

### 3.准备index.js

### 4.shell 命令测试主⻚静态化

```python
python manage.py shell

from contents.crons import generate_static_index_html

generate_static_index_html()
```

## 3.修改Vue变量语法

1.⽬的 避免和Django模板语⾔冲突
2.修改Vue变量语法

```javascript
var vm = new Vue({
    el: '#app',
    // 修改Vue变量的读取语法，避免和django模板语法冲突
    delimiters: ['[[', ']]'],
    data: {

    },
    mounted: function(){

    },
    methods: {

    }
});
```

## 4.定时任务crontab静态化主⻚

### 1.安装依赖包 

```
pip install django-crontab
```

### 2.注册应⽤

```python
INSTALLED_APPS = [
    ...
    'django_crontab', # 定时任务
    ...
]
```

### 3.设置定时时间

**注意** 定时任务的⽇志⽂件路径借⽤了项⽬中logs⽂件夹的绝对路径

```shel
基本格式 :

* * * * *

分 时 日 月 周      命令

M: 分钟（0-59）。每分钟用*或者 */1表示

H：小时（0-23）。（0表示0点）

D：天（1-31）。

m: 月（1-12）。

d: 一星期内的天（0~6，0为星期天）。
```

```python
# 定时任务
CRONJOBS = [
    # 每1分钟执行一次生成主页静态文件
    ('*/1 * * * *', 'contents.crons.generate_static_index_html', '>> /Users/zhangjie/Desktop/meiduo_SY09/meiduo_mall/logs/crontab.log')
]
```

#### 解决中文字符问题

在定时任务中，如果出现非英文字符，会出现字符异常错误

```python
# 解决crontab中文问题
CRONTAB_COMMAND_PREFIX = 'LANG_ALL=zh_cn.UTF-8'
```

## 5.开启定时任务

添加定时任务到系统中

```shell
python manage.py crontab add
```

显示已经激活的定时任务

```shell
python manage.py crontab show
```

移除定时任务

```shellv
python manage.py crontab remove
```