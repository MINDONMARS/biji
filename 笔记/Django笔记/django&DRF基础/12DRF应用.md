# DRF应用(序列化)

[TOC]

创建序列化器

在应用中新建serializers.py用于保存该应用的序列化器。 

##### 定义方法：

Django REST framework中的Serializer使用类来定义，须继承自rest_framework.serializers.Serializer 

例如，我们已有了一个数据库模型类BookInfo

```python
class BookInfo(models.Model):
    btitle = models.CharField(max_length=20, verbose_name='名称')
    bpub_date = models.DateField(verbose_name='发布日期', null=True)
    bread = models.IntegerField(default=0, verbose_name='阅读量')
    bcomment = models.IntegerField(default=0, verbose_name='评论量')
    image = models.ImageField(upload_to='booktest', verbose_name='图片', null=True)
```

为这个模型类提供一个序列化器，可以定义如下：

```python
class BookInfoSerializer(serializers.Serializer):
    """图书数据序列化器"""
    id = serializers.IntegerField(label='ID', read_only=True)
    btitle = serializers.CharField(label='名称', max_length=20)
    bpub_date = serializers.DateField(label='发布日期', required=False)
    bread = serializers.IntegerField(label='阅读量', required=False)
    bcomment = serializers.IntegerField(label='评论量', required=False)
    image = serializers.ImageField(label='图片', required=False)
```

**注意**：serializer不是只能为数据库模型类定义，也可以为非数据库模型类的数据定义。serializer是独立于数据库之外的存在。 

### 字段与选项

**常用字段类型**：

| 字段              | 字段构造方式                                                 |
| ----------------- | ------------------------------------------------------------ |
| **BooleanField**  | BooleanField()                                               |
| **ImageField**    | ImageField(max_length=None, allow_empty_file=False, use_url=UPLOADED_FILES_USE_URL) |
| **CharField**     | CharField(max_length=None, min_length=None, allow_blank=False, trim_whitespace=True) |
| **EmailField**    | EmailField(max_length=None, min_length=None, allow_blank=False) |
| **RegexField**    | RegexField(regex, max_length=None, min_length=None, allow_blank=False) |
| **FileField**     | FileField(max_length=None, allow_empty_file=False, use_url=UPLOADED_FILES_USE_URL) |
| **IntegerField**  | IntegerField(max_value=None, min_value=None)                 |
| **UUIDField**     | UUIDField(format='hex_verbose')  format:  1) `'hex_verbose'` 如`"5ce0e9a5-5ffa-654b-cee0-1238041fb31a"`  2） `'hex'` 如 `"5ce0e9a55ffa654bcee01238041fb31a"`  3）`'int'` - 如: `"123456789012312313134124512351145145114"`  4）`'urn'` 如: `"urn:uuid:5ce0e9a5-5ffa-654b-cee0-1238041fb31a"` |
| **FloatField**    | FloatField(max_value=None, min_value=None)                   |
| **TimeField**     | TimeField(format=api_settings.TIME_FORMAT, input_formats=None) |
| **DateField**     | DateField(format=api_settings.DATE_FORMAT, input_formats=None) |
| **DecimalField**  | DecimalField(max_digits, decimal_places, coerce_to_string=None, max_value=None, min_value=None) max_digits: 最多位数 decimal_palces: 小数点位置 |
| **DateTimeField** | DateTimeField(format=api_settings.DATETIME_FORMAT, input_formats=None) |

**选项参数：**

| 参数名称            | 作用             |
| ------------------- | ---------------- |
| **max_length**      | 最大长度         |
| **min_lenght**      | 最小长度         |
| **allow_blank**     | 是否允许为空     |
| **trim_whitespace** | 是否截断空白字符 |
| **max_value**       | 最小值           |
| **min_value**       | 最大值           |

#### 通用参数：

| 参数名称           | 说明                                          |
| ------------------ | --------------------------------------------- |
| **read_only**      | 表明该字段仅用于序列化输出，默认False         |
| **write_only**     | 表明该字段仅用于反序列化输入，默认False       |
| **required**       | 表明该字段在反序列化时必须输入，默认True      |
| **default**        | 反序列化时使用的默认值                        |
| **allow_null**     | 表明该字段是否允许传入None，默认False         |
| **validators**     | 该字段使用的验证器                            |
| **error_messages** | 包含错误编号与错误信息的字典                  |
| **label**          | 用于HTML展示API页面时，显示的字段名称         |
| **help_text**      | 用于HTML展示API页面时，显示的字段帮助提示信息 |

## 创建Serializer对象

```python
Serializer(instance=None, data=empty, **kwarg)
```

说明：

1）用于序列化时，将模型类对象传入**instance**参数

2）用于反序列化时，将要被反序列化的数据传入**data**参数

3）除了instance和data参数外，在构造Serializer对象时，还可通过**context**参数额外添加数据，如

```python
serializer = AccountSerializer(account, context={'request': request})
```

## 基本使用(添加字段，直接查询)

1） 先查询出一个图书对象

```python
from booktest.models import BookInfo

book = BookInfo.objects.get(id=2)
```

2） 构造序列化器对象

```python
from booktest.serializers import BookInfoSerializer

serializer = BookInfoSerializer(book)
```

3）获取序列化数据

通过data属性可以获取序列化后的数据

```python
serializer.data
# {'id': 2, 'btitle': '天龙八部', 'bpub_date': '1986-07-24', 'bread': 36, 'bcomment': 40, 'image': None}
```

## 关联对象嵌套序列化(添加字段，直接查询)

如果需要序列化的数据中包含有其他关联对象，则对关联对象数据的序列化需要指明。

例如，在定义英雄数据的序列化器时，外键hbook（即所属的图书）字段如何序列化？

我们先定义HeroInfoSerialzier除外键字段外的其他部分

```python
class HeroInfoSerializer(serializers.Serializer):
    """英雄数据序列化器"""
    GENDER_CHOICES = (
        (0, 'male'),
        (1, 'female')
    )
    id = serializers.IntegerField(label='ID', read_only=True)
    hname = serializers.CharField(label='名字', max_length=20)
    hgender = serializers.ChoiceField(choices=GENDER_CHOICES, label='性别', required=False)
    hcomment = serializers.CharField(label='描述信息', max_length=200, required=False, allow_null=True)
```

对于关联字段，可以采用以下几种方式：

#### PrimaryKeyRelatedField

此字段将被序列化为关联对象的主键。

```python
hbook = serializers.PrimaryKeyRelatedField(label='图书', read_only=True)
```

指明字段时需要包含read_only=True：

- 包含read_only=True参数时，该字段将不能用作反序列化使用

使用效果：

```python
from booktest.serializers import HeroInfoSerializer
from booktest.models import HeroInfo
hero = HeroInfo.objects.get(id=6)
serializer = HeroInfoSerializer(hero)
serializer.data
# {'id': 6, 'hname': '乔峰', 'hgender': 1, 'hcomment': '降龙十八掌', 'hbook': 2}
```

#### StringRelatedField

此字段将被序列化为关联对象的字符串表示方式（即__str__方法的返回值）

```python
hbook = serializers.StringRelatedField(label='图书')
```

效果:

```python
{'id': 6, 'hname': '乔峰', 'hgender': 1, 'hcomment': '降龙十八掌', 'hbook': '天龙八部'}
```

#### 使用关联对象的序列化器

```python
hbook = BookInfoSerializer()
```

## many参数

如果关联的对象数据不是只有一个，而是包含多个数据，如想序列化图书BookInfo数据，每个BookInfo对象关联的英雄HeroInfo对象可能有多个，此时关联字段类型的指明仍可使用上述几种方式，只是在声明关联字段时，多补充一个many=True参数即可。

此处仅拿PrimaryKeyRelatedField类型来举例，其他相同。

在BookInfoSerializer中添加关联字段：

```python
class BookInfoSerializer(serializers.Serializer):
    """图书数据序列化器"""
    id = serializers.IntegerField(label='ID', read_only=True)
    btitle = serializers.CharField(label='名称', max_length=20)
    bpub_date = serializers.DateField(label='发布日期', required=False)
    bread = serializers.IntegerField(label='阅读量', required=False)
    bcomment = serializers.IntegerField(label='评论量', required=False)
    image = serializers.ImageField(label='图片', required=False)
    heroinfo_set = serializers.PrimaryKeyRelatedField(read_only=True, many=True)  # 新增
```

使用效果：

```python
from booktest.serializers import BookInfoSerializer
from booktest.models import BookInfo
book = BookInfo.objects.get(id=2)
serializer = BookInfoSerializer(book)
serializer.data
# {'id': 2, 'btitle': '天龙八部', 'bpub_date': '1986-07-24', 'bread': 36, 'bcomment': 40, 'image': None, 'heroinfo_set': [6,8, 9]}
```