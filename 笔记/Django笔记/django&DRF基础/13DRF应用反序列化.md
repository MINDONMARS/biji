# DRF应用反序列化

[TOC]

1. ## 需要对数据验证

   - ### 调⽤序列化器进⾏验证

           使用序列化器进行反序列化时，需要对数据进行验证后，才能获取验证成功的数据或保存成模型类对象。

   	步骤:

   	1: 得到需要反序列化的模型对象

   	2: 调用**is_valid()**方法进行验证(验证成功返回True, 失败False)

   	3:验证失败可用序列化器的**errors**属性获取错误信息(字典包含了字段和字			  			段的错误。如果是非字段错误，可以通过修改REST framework配置中的**NON_FIELD_ERRORS_KEY**来控制错误字典中的键名)

   	4:成功可用序列化器的validated_data属性获取反序列化后的数据

   代码展示:

   ```python
   class BookInfoSerializer(serializers.Serializer):
       """图书数据序列化器"""
       id = serializers.IntegerField(label='ID', read_only=True)
       btitle = serializers.CharField(label='名称', max_length=20)
       bpub_date = serializers.DateField(label='发布⽇期', required=False)
       bread = serializers.IntegerField(label='阅读量', required=False)
       bcomment = serializers.IntegerField(label='评论量', required=False)
       image = serializers.ImageField(label='图⽚', required=False)
       heroinfo_set = serializers.PrimaryKeyRelatedField(read_only=True, many=True)
   ```

   验证失败:

   ```python
   >>> from booktest.serializers import BookInfoSerializer
   >>> data = {}
   >>> s = BookInfoSerializer(data=data)
   >>> s.is_valid()
   False
   >>> s.errors
   {'btitle': [ErrorDetail(string='This field is required.', code='required')]}
   ```

   验证成功:

   ```python
   >>> data = {'btitle':'⽔浒传'}
   >>> s = BookInfoSerializer(data=data)
   >>> s.is_valid()
   True
   >>> s.validated_data
   OrderedDict([('btitle', '⽔浒传')])
   ```

   验证的同时抛出异常:

   ```python
   >>> data = {}
   >>> s = BookInfoSerializer(data=data)
   >>> s.is_valid(raise_exception=True)
   Traceback (most recent call last):
    File "<console>", line 1, in <module>
    File "/Users/zhangjie/.virtualenvs/py3_django/lib/python3.6/site-packages/rest_framework/serializers.py", line 244, in is_valid
    raise ValidationError(self.errors)
   rest_framework.exceptions.ValidationError: {'btitle': [ErrorDetail(string='This field is required.', code='required')]	
   ```

   - ### 定义序列化器进⾏验证

     ##### 单个字段验证

     ```python
     准备序列化器
     class BookInfoSerializer(serializers.Serializer):
         """图书数据序列化器"""
         id = serializers.IntegerField(label='ID', read_only=True)
         btitle = serializers.CharField(label='名称', max_length=20)
         bpub_date = serializers.DateField(label='发布⽇期', required=False)
         bread = serializers.IntegerField(label='阅读量', required=False)
         bcomment = serializers.IntegerField(label='评论量', required=False)
         image = serializers.ImageField(label='图⽚', required=False)
         heroinfo_set = serializers.PrimaryKeyRelatedField(read_only=True, many=True) # 新增
         def validate_btitle(self, value):
             """单个字段验证"""
             if 'django' not in value.lower():
                 raise serializers.ValidationError("图书不是关于Django的")
             return value
     ```

     ```python
     >>> from booktest.serializers import BookInfoSerializer
     >>> data = {'btitle':'python'}
     >>> s = BookInfoSerializer(data=data)
     >>> s.is_valid()
     False
     >>> s.errors
     {'btitle': [ErrorDetail(string='图书不是关于Django的', code='invalid')]}
     ```

     ##### 多字段联合验证

     **提示:**  多字段联合验证是在单个字段验证之后进⾏的验证

     ```python
     class BookInfoSerializer(serializers.Serializer):
         """图书数据序列化器"""
         id = serializers.IntegerField(label='ID', read_only=True)
         btitle = serializers.CharField(label='名称', max_length=20)
         bpub_date = serializers.DateField(label='发布⽇期', required=False)
         bread = serializers.IntegerField(label='阅读量', required=False)
         bcomment = serializers.IntegerField(label='评论量', required=False)
         image = serializers.ImageField(label='图⽚', required=False)
         heroinfo_set = serializers.PrimaryKeyRelatedField(read_only=True, many=True) # 新增
         def validate_btitle(self, value):
             """单个字段验证"""
             if 'django' not in value.lower():
                 raise serializers.ValidationError("图书不是关于Django的")
             return value
         def validate(self, attrs):
             """多字段联合验证"""
             bread = attrs['bread']
             bcomment = attrs['bcomment']
             if bread < bcomment:
                 raise serializers.ValidationError('阅读量⼩于评论量')
             return attrs
     ```

     ```python
     >>> from booktest.serializers import BookInfoSerializer
     >>> data = {'btitle':'django', 'bread':10, 'bcomment':20}
     >>> s = BookInfoSerializer(data=data)
     >>> s.is_valid()
     False
     >>> s.errors
     {'non_field_errors': [ErrorDetail(string='阅读量⼩于评论量', code='invalid')]}
     >>> 
     
     'non_field_errors' 多字段联合校验时的错误信息key
     ```

     validators（了解）

   ```python
   # 1.准备序列化器
   def about_django(value):
       if 'django' not in value.lower():
           raise serializers.ValidationError("图书不是关于Django的")
           
           
   class BookInfoSerializer(serializers.Serializer):
       """图书数据序列化器"""
       id = serializers.IntegerField(label='ID', read_only=True)
       btitle = serializers.CharField(label='名称', max_length=20, validators=[about_django])
       bpub_date = serializers.DateField(label='发布⽇期', required=False)
       bread = serializers.IntegerField(label='阅读量', required=False)
       bcomment = serializers.IntegerField(label='评论量', required=False)
       image = serializers.ImageField(label='图⽚', required=False)
       heroinfo_set = serializers.PrimaryKeyRelatedField(read_only=True, many=True)
       
       
   # 2.验证
   >>> from booktest.serializers import BookInfoSerializer
   >>> data = {'btitle':'python'}
   >>> s = BookInfoSerializer(data=data)
   >>> s.is_valid()
   False
   >>> s.errors
   {'btitle': [
   ```

2. ## 保存序列化器验证后的数据

   准备序列化器

   ```python
   class BookInfoSerializer(serializers.Serializer):
       """图书数据序列化器"""
       id = serializers.IntegerField(label='ID', read_only=True)
       btitle = serializers.CharField(label='名称', max_length=20, validators=[about_django])
       bpub_date = serializers.DateField(label='发布⽇期', required=False)
       bread = serializers.IntegerField(label='阅读量', required=False)
       bcomment = serializers.IntegerField(label='评论量', required=False)
       image = serializers.ImageField(label='图⽚', required=False)
       heroinfo_set = serializers.PrimaryKeyRelatedField(read_only=True, many=True) # 新增
        def validate_btitle(self, value):
        	"""单个字段验证"""
        	if 'django' not in value.lower():
        		raise serializers.ValidationError("图书不是关于Django的")
        	return value
        def validate(self, attrs):
        	"""多字段联合验证"""
        	bread = attrs['bread']
        	bcomment = attrs['bcomment']
        	if bread < bcomment:
        		raise serializers.ValidationError('阅读量⼩于评论量')
        	return attrs
        def create(self, validated_data):
        	"""新建"""
        	return BookInfo.objects.create(**validated_data)
        def update(self, instance, validated_data):
        	"""更新，instance为要更新的对象实例"""
        	instance.btitle = validated_data.get('btitle', instance.btitle)
        	instance.bpub_date = validated_data.get('bpub_date', instance.bpub_date)
        	instance.bread = validated_data.get('bread', instance.bread)
            instance.bcomment = validated_data.get('bcomment', instance.bcomment)
        	instance.save()
        	return instance
   ```

   ```python
   # 验证后新增
   # 说明：
   # data中的数据，要让验证通过
   # 没有传⼊模型对象实例，所以是新增数据
   >>> from booktest.serializers import BookInfoSerializer
   >>> data = {'btitle':'django', 'bread':30, 'bcomment':10, 'bpub_date':'1989-11-11'}
   >>> s = BookInfoSerializer(data=data)
   >>> s.is_valid()
   True
   >>> book = s.save()
   >>> book
   <BookInfo: django>
       
   # 验证后更新
   >>> data = {'btitle':'django python', 'bread':30, 'bcomment':10, 'bpub_date':'1989-11-11'}
   >>> s = BookInfoSerializer(book, data=data)
   >>> s.is_valid()
   True
   >>> s.save()
   <BookInfo: django python>
   ```


