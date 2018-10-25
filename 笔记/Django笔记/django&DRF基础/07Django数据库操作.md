# Django数据库操作

[TOC]

用shell操作

`	python manage.py shell`

导入模型类

`from booktest.models import BookInfo, HeroInfo`

## 增(两种方法):

1. ### save()

   ```python
   from datetime import date
   book = BookInfo(
   btitle='西游记',
   bpub_date=date(1988,1,1),
   bread=10,
   bcomment=10
   )
   book.save()
   # 外键传模型类对象保存
   hero = HeroInfo(
   hname='孙悟空',
   hgender=0,
   hbook=book
   )
   hero.save()
   # 外键传id保存(表中默认字段加了_id)
   hero2 = HeroInfo(
   hname='猪八戒',
   hgender=0,
   hbook_id=book.id
   hero2.save()
   ```

2. ### create()

   通过模型类.objects.create()保存。 

   ```python
   HeroInfo.objects.create(
       hname='沙悟净',
       hgender=0,
       hbook=book
   )
   # 输出<HeroInfo: 沙悟净>
   ```

## 查询

1. ### 基本查询

   get: 查询单一结果, 如果不存在报错**DoesNotExist**异常 

   ```python
   BookInfo.objects.get(id=3)
   ```

   all: 查询多个结果

   ```python
   BookInfo.objects.all()
   ```

   count: 查询结果数量

   ```python
   BookInfo.objects.count()
   ```

2. ### 过滤查询

   |   方法    |     作用     |
   | :-------: | :----------: |
   |   get()   | 过滤单一结果 |
   |  filte()  | 过滤多个结果 |
   | exclude() |     取反     |

   1. 相等

      exact：判等

      ```python
      BookInfo.objects.filter(id__exact=1)
      简写 
      BookInfo,objects.filter(id=1)
      ```

   2. 模糊查询

      | contains            | BookInfo.objects.filter(btitle__contains='传') |
      | ------------------- | ---------------------------------------------- |
      | endswith/startswith | BookInfo.objects.filter(btitle__endswith='部') |

      以上运算符都区分大小写，在这些运算符前加上i表示不区分大小写，如iexact、icontains、istartswith、iendswith. 

      ------

   3. 空查询

      `isnull`：是否为null

      ```
      BookInfo.objects.filter(btitle__isnull=False)
      ```

   4. 范围查询

      `in`： 是否包含在范围内

      ```
      BookInfo.objects.filter(id__in=[1, 2, 3])
      ```

   5. 比较查询

      | 比较运算符 |                   例                   |
      | :--------: | :------------------------------------: |
      |    gt >    |   BookInfo.objects.filter(id__gt=3)    |
      |   gte >=   | BookInfo.objects.filter(bread__gte=58) |
      |    lt <    |   BookInfo.objects.filter(id__lt=3)    |
      |   lte <=   | BookInfo.objects.filter(bread__lte=58) |

      不等于用exclude

      ```
      BookInfo.objects.exclude(bread=58)
      ```

   6. 日期查询

      **year、month、day、week_day、hour、minute、second：对日期时间类型的属性进行运算。** 

      ```
      BookInfo.objects.filter(bpub_date__year=1988)
      <QuerySet [<BookInfo: 西游记>]>
      BookInfo.objects.filter(bpub_date__gt=date(1980, 1, 1))
      ```

   7. F对象

      不同属性（字段）进行比较

      ```python
      # 导入
      from django.db.models import F
      
      BookInfo.objects.filter(bread__gte=F('bcomment'))
      
      # 可以在F对象上使用数学运算
      BookInfo.objects.filter(bread__gte=F('bcomment') * 2)
      ```

   8. Q对象

      多个过滤器逐个调用表示与或非逻辑运算（&， |，~）

      ```python
      # 导入
      from django.db.models import Q
      # 不用Q
      BookInfo.objects.filter(bread__gt=20, id__lt=3)
      BookInfo.objects.filter(bread__gt=20).filter(id__lt=3)
      # 用Q
      BookInfo.objects.filter(Q(id__lt=3) & Q(bread__gt=20))
      # ~
      BookInfo.objects.filter(~Q(id__lt=5))
      ```

   9. 聚合函数

      使用aggregate()过滤器调用聚合函数。聚合函数包括：**Avg** 平均，**Count** 数量，**Max** 最大，**Min** 最小，**Sum** 求和，被定义在django.db.models中 

      ```python
      from django.db.models import Sum
      # 求和
      BookInfo.objects.aggregate(Sum('bread'))
      # 注意aggregate的返回值是一个字典类型
      # Out[27]: {'bread__sum': 136}
      ```

      使用count时一般不使用aggregate()过滤器

      ```python
      BookInfo.objects.count()
      # 注意count函数的返回值是一个数字
      ```

   	 排序	

|      |                                           |
| ---- | ----------------------------------------- |
| 降序 | BookInfo.objects.all().order_by('-bread') |
| 升序 | BookInfo.objects.all().order_by('bread')  |

## 关联查询

1. ### 普通关联

   ```python
   # 一对多
   b = BookInfo.objects.get(id=1)
   b.heroinfo_set.all()
   # 多对一
   h = HeroInfo.objects.get(id=1)
   h.hbook
   h.hbook_id
   ```

2. ### 关联过滤查询

   ```python
   # 多对一
   BookInfo.objects.filter(heroinfo__hname__contains='孙')
   # 一对多
   HeroInfo.objects.filter(hbook__btitle='天龙八部')
   # 查询图书阅读量大于30的所有英雄
   HeroInfo.objects.filter(hbook__bread__gt=30)
   ```

## 修改

1. ### save()

   ```
   hero = HeroInfo.objects.get(hname='猪悟能')
   hero.hname = '猪八戒'
   hero.save
   ```

2. ### update()

   返回受影响的行数 

   ```
   HeroInfo.objects.filter(hname='沙僧').update(hname='沙悟净')
   ```

## 删除

1. ### **模型类对象delete**()

   ```
   hero = HeroInfo.objects.get(hname='猪八戒')
   hero.delete()
   ```

2. ### **模型类.objects.filter().delete()** 

   ```
   HeroInfo.objects.filter(hname='沙悟净').delete()
   ```

## 查询集 QuerySet

- exists()：判断查询集中是否有数据，如果有则返回True，没有则返回False。
- 注意：不支持负数索引 

#### 1）惰性执行

创建查询集不会访问数据库，直到调用数据时，才会访问数据库，调用数据的情况包括迭代、序列化、与if合用

例如，当执行如下语句时，并未进行数据库查询，只是创建了一个查询集qs

```
qs = BookInfo.objects.all()
```

继续执行遍历迭代操作后，才真正的进行了数据库的查询

```
for book in qs:
    print(book.btitle)
```

#### 2）缓存

经过存储后，可以重用查询集，第二次使用缓存中的数据。

```
qs=BookInfo.objects.all()
[book.id for book in qs]
[book.id for book in qs]
```

#### 限制查询集

**对查询集进行切片后返回一个新的查询集，不会立即执行查询** 

```
qs = BookInfo.objects.all()[0:2]
```

## 管理器Manager

	管理器是Django的模型进行数据库操作的接口，Django应用的每个模型类都拥有至少一个管理器。
	
	我们在通过模型类的**objects**属性提供的方法操作数据库时，即是在使用一个管理器对象objects。当没有为模型类定义管理器时，Django会为每一个模型类生成一个名为objects的管理器，它是**models.Manager**类的对象。

### 自定义管理器

在models.py中

```python
# 继承  models.Manager
class BookInfoManage(models.Manager):

    def all(self):
        return super().filter(is_delete=1)
	#  在管理器类中补充定义新的方法
    def create_book(self, btitle, bpub_date):
        book = self.model()
        book.bpub_date = bpub_date
        book.btitle = btitle
        book.save()
        return book
# 在对应的类中用属性接收该对象
class BookInfo(models.Model):
    book_obj = BookInfoManage()
# 用法
BookInfo.book_obj.all()
```



