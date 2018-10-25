# Django录⼊商品数据

## 1.站点管理

### 注册⼴告

```python
from django.contrib import admin

from . import models
# Register your models here.


admin.site.register(models.ContentCategory)
admin.site.register(models.Content)
```

### 注册商品

```python
from django.contrib import admin

from . import models
# Register your models here.


admin.site.register(models.GoodsCategory)
admin.site.register(models.GoodsChannel)
admin.site.register(models.Goods)
admin.site.register(models.Brand)
admin.site.register(models.GoodsSpecification)
admin.site.register(models.SpecificationOption)
admin.site.register(models.SKU)
admin.site.register(models.SKUSpecifica
admin.site.register(models.SKUImage)
```

## 2.录⼊storage数据

1.准备新的图⽚数据压缩包
2.删除旧的data⽂件夹
3.拷⻉新的图⽚数据压缩包并解压
解压命令：sudo tar -zxvf data.tar.gz

## 3.录⼊数据库商品数据

mysql -uroot -pmysql meiduo_mall < goods_data.sql
注意
需要绑定image.meiduo.site域名
192.168.73.129绑定到image.meiduo.site