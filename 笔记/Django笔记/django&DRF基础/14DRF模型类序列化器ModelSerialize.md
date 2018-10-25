# DRF模型类序列化器ModelSerialize

[TOC]

ModelSerializer类是Serializer类的⼦类

1.模型类字段全部映射到序列化器

	准备序列化器

```python
class BookInfoSerializer(serializers.ModelSerializer):
    """BookInfo模型类的序列化器"""
    class Meta:
        model = BookInfo
        fields = '__all__'
```

	创建序列化器对象

```python
>>> from booktest.serializers import BookInfoModelSerializer
>>> s = BookInfoModelSerializer()
>>> s
BookInfoModelSerializer():
    id = IntegerField(label='ID', read_only=True)
    btitle = CharField(label='名称', max_length=20)
    bpub_date = DateField(label='发布⽇期')
    bread = IntegerField(label='阅读量', max_value=2147483647, min_value=-2147483648, required=False)
    bcomment = IntegerField(label='评论量', max_value=2147483647, min_value=-2147483648, required=False)
    is_delete = BooleanField(label='逻辑删除', required=False)
    image = ImageField(allow_null=True, label='图书图⽚', max_length=100, required=False)
```

2.模型类字段部分映射到序列化器

	准备序列化器

```python
class BookInfoSerliazer(serializers.ModelSerializer):
    """BookInfo模型类的序列化器"""
    class Meta:
        model = BookInfo
        fields= ['id', 'btitle', 'bpub_date']
        
```

	创建序列化器对象

```python
>>> from booktest.serializers import BookInfoModelSerializer
>>> s = BookInfoModelSerializer()
>>> s
BookInfoModelSerializer():
    id = IntegerField(label='ID', read_only=True)
    btitle = CharField(label='名称', max_length=20)
    bpub_date = DateField(label='发布⽇期')
```

3.模型类字段排除指定字段

	准备序列化器

```python
class BookInfoModelSerlizer(serializers.ModelSerliazer):
    """BookInfo模型类的序列化器"""
    class Meta:
        model = BookInfo
        exclude = ('image',)
```

	创建序列化器对象

```python
>>> from booktest.serializers import BookInfoModelSerializer
>>> s = BookInfoModelSerializer()
>>> s
BookInfoModelSerializer():
    id = IntegerField(label='ID', read_only=True)
    btitle = CharField(label='名称', max_length=20)
    bpub_date = DateField(label='发布⽇期')
    bread = IntegerField(label='阅读量', max_value=2147483647, min_value=-2147483648, required=False)
    bcomment = IntegerField(label='评论量', max_value=2147483647, min_value=-2147483648, required=False)
    is_delete = BooleanField(label='逻辑删除', required=False)
```

4.添加额外参数

	准备序列化器

```python
class BookInfoModelSerializer(serializers.ModelSerializer):
    
    class Meta:
        model = BookInfo
        exclude = ('image',)
        extra_kwargs = {
            'bread': {'min_value': 0, 'required': True},
            'bcomment': {'min_value': 0, 'required':True}
        }
```

```python
>>> from booktest.serializers import BookInfoModelSerializer
>>> s = BookInfoModelSerializer()
>>> s
BookInfoModelSerializer():
    id = IntegerField(label='ID', read_only=True)
    btitle = CharField(label='名称', max_length=20)
    bpub_date = DateField(label='发布⽇期')
    bread = IntegerField(label='阅读量', max_value=2147483647, min_value=0, required=True)
    bcomment = IntegerField(label='评论量', max_value=2147483647, min_value=0, required=True)
```

5.指明只读字段

```python
class BookInfoModelSerializer(serializers.ModelSerializer):
    
    class Meta:
        model = BookInfo
        exclude = ('image',)
        extra_kwargs = {
            'bread':{'min_value': 0, 'required': True},
            'bcomment': {'min_value': 0, 'required': True}
        }
        read_only_fields = ['bread', 'bcomment']
```

```python
>>> from DRF.serializers import BookInfoModelSerializer
>>> s = BookInfoModelSerializer()
>>> s
BookInfoModelSerializer():
    id = IntegerField(label='ID', read_only=True)
    btitle = CharField(label='书名', max_length=20)
    bpub_date = DateField(label='⽇期')
    bread = IntegerField(label='阅读量', min_value=0, read_only=True)
    bcomment = IntegerField(label='评论量', min_value=0, read_only=True)
    is_delete = BooleanField(label='逻辑删除', required=False)
```

