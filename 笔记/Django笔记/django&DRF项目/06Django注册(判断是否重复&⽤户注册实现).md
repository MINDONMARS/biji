# Django注册(判断是否重复&⽤户注册实现)

[TOC]

## 判断是否重复

用户名:

```python
# url(r'^usernames/(?P<username>\w{5,20})/count/$', views.UsernameCountView.as_view()),
class UsernameCountView(APIView):
    """
    ⽤户名数量
    """
    def get(self, request, username):
        """
        获取指定⽤户名数量
        """
        count = User.objects.filter(username=username).count()
        data = {
        	'username': username,
        	'count': count
        }
        return Response(data)
```

手机号:

```python
# url(r'^mobiles/(?P<mobile>1[3-9]\d{9})/count/$', views.MobileCountView.as_view()),
class MobileCountView(APIView):
    """
    ⼿机号数量
    """
    def get(self, request, mobile):
        """
        获取指定⼿机号数量
        """
        count = User.objects.filter(mobile=mobile).count()
        data = {
        	'mobile': mobile,
        	'count': count
        }
        return Response(data)
```

## ⽤户注册实现

### 视图逻辑

```python
# url(r'^users/$', views.UserView.as_view()),
class UserView(CreateAPIView):
    """注册"""
    serializer_class = serializers.CreateUserSerializer
```

### 序列化器

```python
class CreateUserSerializer(serializers.ModelSerializer):
    """注册序列化器"""
    password2 = serializers.CharField(label='确认密码', write_only=True)
    sms_code = serializers.CharField(label='短信验证码', write_only=True)
    allow = serializers.CharField(label='同意协议', write_only=True)
    class Meta:
    model = User
    fields = ('id', 'username', 'password', 'password2', 'sms_code', 'mobile', 'allow')
    extra_kwargs = {
        'username': {
        	'min_length': 5,
        	'max_length': 20,
        'error_messages': {
        	'min_length': '仅允许5-20个字符的⽤户名',
        	'max_length': '仅允许5-20个字符的⽤户名',
    	}
    },
    	'password': {
            'write_only': True,
                'min_length': 8,
                'max_length': 20,
            'error_messages': {
                'min_length': '仅允许8-20个字符的密码',
                'max_length': '仅允许8-20个字符的密码',
            }
        }
    }
    def validate_mobile(self, value):
        """验证⼿机号"""
        if not re.match(r'^1[3-9]\d{9}$', value):
        	raise serializers.ValidationError('⼿机号格式错误')
        return value
    def validate_allow(self, value):
        """检验⽤户是否同意协议"""
        if value != 'true':
        	raise serializers.ValidationError('请同意⽤户协议')
        return value
    def validate(self, data):
        # 判断两次密码
        if data['password'] != data['password2']:
        	raise serializers.ValidationError('两次密码不⼀致')
        # 判断短信验证码
        redis_conn = get_redis_connection('verify_codes')
        mobile = data['mobile']
        real_sms_code = redis_conn.get('sms_%s' % mobile)
        if real_sms_code is None:
        	raise serializers.ValidationError('⽆效的短信验证码')
        if data['sms_code'] != real_sms_code.decode():
        	raise serializers.ValidationError('短信验证码错误')
        return data
    
    # 重写序列化器的create方法去掉外部字段
    def create(self, validated_data):
        """
        创建⽤户
        """
        # 移除数据库模型类中不存在的属性
        del validated_data['password2']
        del validated_data['sms_code']
        del validated_data['allow']
        user = super().create(validated_data)
        # 调⽤django的认证系统加密密码
        user.set_password(validated_data['password'])
        user.save()
        return user
```

