# Django数据库

[TOC]

自带ORM(对象关系映射)

使用django进行数据库开发的步骤如下：

1. 配置数据库连接信息(见配置页)
2. 在models.py中定义模型类
3. 迁移
4. 通过类和对象完成数据增删改查操作

## 定义模型类

### 数据库表名

模型类如果未指定表名, 默认以app应用名_小写模型类名命名(可通过db_table指定)

### 主键

django会为表创建自动增长的主键列，每个模型只能有一个主键列，如果使用选项设置某属性为主键列后django不会再创建自动增长的主键列。

默认创建的主键列属性为id，可以使用pk代替，pk全拼为primary key。

### 属性命名限制

- 不能是python的保留关键字。

- 不允许使用连续的下划线，这是由django的查询方式决定的。

- 定义属性时需要指定字段类型，通过字段类型的参数指定选项，语法如下：

  ```
  属性=models.字段类型(选项)
  ```

#### 字段类型

| 类型             | 说明                                                         |
| ---------------- | ------------------------------------------------------------ |
| AutoField        | 自动增长的IntegerField，通常不用指定，不指定时Django会自动创建属性名为id的自动增长属性 |
| BooleanField     | 布尔字段，值为True或False                                    |
| NullBooleanField | 支持Null、True、False三种值                                  |
| CharField        | 字符串，参数max_length表示最大字符个数                       |
| TextField        | 大文本字段，一般超过4000个字符时使用                         |
| IntegerField     | 整数                                                         |
| DecimalField     | 十进制浮点数， 参数max_digits表示总位数， 参数decimal_places表示小数位数 |
| FloatField       | 浮点数                                                       |
| DateField        | 日期， 参数auto_now表示每次保存对象时，自动设置该字段为当前时间，用于"最后一次修改"的时间戳，它总是使用当前日期，默认为False； 参数auto_now_add表示当对象第一次被创建时自动设置当前时间，用于创建的时间戳，它总是使用当前日期，默认为False; 参数auto_now_add和auto_now是相互排斥的，组合将会发生错误 |
| TimeField        | 时间，参数同DateField                                        |
| DateTimeField    | 日期时间，参数同DateField                                    |
| FileField        | 上传文件字段                                                 |
| ImageField       | 继承于FileField，对上传的内容进行校验，确保是有效的图片      |

#### 选项

| 选项        | 说明                                                         |
| ----------- | ------------------------------------------------------------ |
| null        | 如果为True，表示允许为空，默认值是False                      |
| blank       | 如果为True，则该字段允许为空白，默认值是False                |
| db_column   | 字段的名称，如果未指定，则使用属性的名称                     |
| db_index    | 若值为True, 则在表中会为此字段创建索引，默认值是False        |
| default     | 默认                                                         |
| primary_key | 若为True，则该字段会成为模型的主键字段，默认值是False，一般作为AutoField的选项使用 |
| unique      | 如果为True, 这个字段在表中必须有唯一值，默认值是False        |

**null是数据库范畴的概念，blank是表单验证范畴的**

#### 外键

在设置外键时，需要通过**on_delete**选项指明主表删除数据时，对于外键引用表数据如何处理，在django.db.models中包含了可选常量：

- **CASCADE** 级联，删除主表数据时连通一起删除外键表中数据

- **PROTECT** 保护，通过抛出**ProtectedError**异常，来阻止删除主表中被外键应用的数据

- **SET_NULL** 设置为NULL，仅在该字段null=True允许为null时可用

- **SET_DEFAULT** 设置为默认值，仅在该字段设置了默认值时可用

- **DO_NOTHING** 不做任何操作，如果数据库前置指明级联性，此选项会抛出**IntegrityError**异常 

- **SET()** 设置为特定值或者调用特定方法，如

  ```python
  from django.conf import settings
  from django.contrib.auth import get_user_model
  from django.db import models
  
  def get_sentinel_user():
      return get_user_model().objects.get_or_create(username='deleted')[0]
  
  class MyModel(models.Model):
      user = models.ForeignKey(
          settings.AUTH_USER_MODEL,
          on_delete=models.SET(get_sentinel_user),
      )
  ```



### 例:

```python
from django.db import models


# Create your models here.


class BookInfo(models.Model):
    btitle = models.CharField(max_length=20,
                              verbose_name='书名')
    bpub_date = models.DateField(verbose_name='日期')
    bread = models.IntegerField(default=0,
                                verbose_name='阅读量')
    bcomment = models.IntegerField(default=0,
                                   verbose_name='评论量')
    is_delete = models.BooleanField(default=False,
                                    verbose_name='逻辑删除')

    # 控制表
    class Meta:
        db_table = 'tb_book'  # 指定表名
        verbose_name = '图书'
        verbose_name_plural = verbose_name

    def __str__(self):
        return self.btitle


class HeroInfo(models.Model):
    GENDER_CHOICES = ((0, 'male'),
                      (1, 'female'))
    hname = models.CharField(max_length=20, verbose_name='名称')
    hgender = models.SmallIntegerField(choices=GENDER_CHOICES, default=0)
    hcomment = models.CharField(max_length=200, null=True, verbose_name='描述信息')
    hbook = models.ForeignKey(BookInfo, on_delete=models.CASCADE, verbose_name='图书')
    is_delete = models.BooleanField(default=False, verbose_name='逻辑删除')

    class Meta:
        db_table = 'tb_heros'
        verbose_name = '英雄'
        verbose_name_plural = verbose_name

    def __str__(self):
        return self.hname

```

### 迁移

将模型类同步到数据库中。

#### 生成迁移文件

```
python manage.py makemigrations
```

#### 同步到数据库中

```
python manage.py migrate
```

