# Django第三方登陆(QQ)

[TOC]

## QQ登录⽂档说明

### 1.QQ互联开发者申请步骤

运营
http://wiki.connect.qq.com/%E6%88%90%E4%B8%BA%E5%BC%80%E5%8F%91%E8%80%85

### 2.QQ互联应⽤申请步骤

运营
http://wiki.connect.qq.com/__trashed-2

### 3.⽹站对接QQ登录步骤

后端程序员
http://wiki.connect.qq.com/%E5%87%86%E5%A4%87%E5%B7%A5%E4%BD%9C_oauth2-0

## QQ登录模型类

### 0.创建oauth应⽤

注册oauth应⽤

### 1.BaseMode

全局utils文件夹新建models.py文件

```python
from django.db import models


class BaseModel(models.Model):
    """为模型类补充字段"""
    create_time = models.DateTimeField(auto_now_add=True, verbose_name="创建时间")
    update_time = models.DateTimeField(auto_now=True, verbose_name="更新时间")

    class Meta:
        abstract = True  # 说明是抽象模型类, 用于继承使用，数据库迁移时不会创建BaseModel的表
```

### 2.OauthQQUser

oauth应用创建模型类继承自BaseMode

```python
from django.db import models

from meiduo_mall.utils.models import BaseModel
# Create your models here.


class OAuthQQUser(BaseModel):
    """
    QQ登录用户数据：需要使用QQ账户绑定美多账户
    openid : 表示QQ账户
    user ：表示美多账户
    """
    user = models.ForeignKey('users.User', on_delete=models.CASCADE, verbose_name='用户')
    openid = models.CharField(max_length=64, verbose_name='openid', db_index=True)

    class Meta:
        db_table = 'tb_oauth_qq'
        verbose_name = 'QQ登录用户数据'
        verbose_name_plural = verbose_name
```

### 3.迁移 

```python
python manage.py makemigrations
python manage.py migrate
```

### 4.存储数据效果

![1536074680010](C:\Users\ADMINI~1\AppData\Local\Temp\1536074680010.png)

## QQ登录OAuth2.0认证

### 1.后端接⼝设计

```python
# url(r'^qq/user/$', views.QQAuthUserView.as_view()),
class QQAuthUserView(APIView):
    """⽤户扫码登录的回调处理"""
    def get(self, request):
    pass
```

### 2.待处理逻辑分析

```python
# url(r'^qq/user/$', views.QQAuthUserView.as_view()),
class QQAuthUserView(APIView):
    """⽤户扫码登录的回调处理"""
    def get(self, request):
    # 提取code请求参数
    # 使⽤code向QQ服务器请求access_token
    # 使⽤access_token向QQ服务器请求openid
    # 使⽤openid查询该QQ⽤户是否在美多商城中绑定过⽤户
    # 如果openid已绑定美多商城⽤户，直接⽣成JWT token，并返回
    # 如果openid没绑定美多商城⽤户，创建⽤户并绑定到openid
    pass
```

### 3.获取code+access_token+open_id

```python
# url(r'^qq/user/$', views.QQAuthUserView.as_view()),
class QQAuthUserView(GenericAPIView):
    """处理QQ扫码登录的回调：完成oauth2.0认证过程"""

    # 指定序列化器
    serializer_class = QQAuthUserSerializer

    def get(self, request):
        # 提取code请求参数
        code = request.query_params.get('code')
        if not code:
            return Response({'message':'缺少code'}, status=status.HTTP_400_BAD_REQUEST)

        # 创建OAuthQQ对象
        oauth = OAuthQQ(client_id=settings.QQ_CLIENT_ID, client_secret=settings.QQ_CLIENT_SECRET,
                        redirect_uri=settings.QQ_REDIRECT_URI)

        try:
            # 使用code向QQ服务器请求access_token
            access_token = oauth.get_access_token(code)

            # 使用access_token向QQ服务器请求openid
            open_id = oauth.get_open_id(access_token)
        except Exception as e:
            logger.info(e)
            return Response({'message':'QQ服务器内部错误'}, status=status.HTTP_503_SERVICE_UNAVAILABLE)
```

### 4.判断QQ⽤户是否绑定过美多商城

```python
# 使⽤openid查询该QQ⽤户是否在美多商城中绑定过⽤户
try:
   oauthqquser_model = OAuthQQUser.objects.get(openid=open_id)
except OAuthQQUser.DoesNotExist:
 # 如果openid没绑定美多商城⽤户，创建⽤户并绑定到openid
	pass
else:
    # 如果openid已绑定美多商城⽤户，直接⽣成JWT token，并返回
    pass
```

## openid绑定美多商城⽤户

### 1.如果⽤户身份已绑定

⽣成JWT token 并响应给前端

```python
        # 使用openid查询该QQ用户是否在美多商城中绑定过用户
        try:
            oauthqquser_model = OAuthQQUser.objects.get(openid=open_id)
        except OAuthQQUser.DoesNotExist:
            pass
        
        else:
            # 如果openid已绑定美多商城用户，直接生成JWT token，并返回，user_id,username,token
            jwt_payload_handler = api_settings.JWT_PAYLOAD_HANDLER
            jwt_encode_handler = api_settings.JWT_ENCODE_HANDLER

            # 获取oauth_user关联的user
            # user = models.ForeignKey('users.User', on_delete=models.CASCADE, verbose_name='用户')
            user = oauthqquser_model.user

            payload = jwt_payload_handler(user)
            token = jwt_encode_handler(payload)

            return Response({
                'user_id':user.id,
                'username':user.username,
                'token':token
            })
```

### 2.如果⽤户身份未绑定

#### 1.将openid签名后响应到前端(直接返回不安全)

##### 1.itsdangerous的使⽤

使⽤TimedJSONWebSignatureSerializer可以⽣成带有有效期的token

```python
from itsdangerous import TimedJSONWebSignatureSerializer as Serializer
from django.conf import settings
# serializer = Serializer(秘钥, 有效期秒)
serializer = Serializer(settings.SECRET_KEY, 300)
# serializer.dumps(数据), 返回bytes类型
token = serializer.dumps({'mobile': '18512345678'})
token = token.decode()

# 检验token
# 验证失败，会抛出itsdangerous.BadData异常
serializer = Serializer(settings.SECRET_KEY, 300)
try:
	data = serializer.loads(token)
except BadData:
	return None
```



##### 2.使⽤itsdangerous签名openid

新建应用内utils.py工具文件

```python
from itsdangerous import TimedJSONWebSignatureSerializer as TJSSerializer  # 起个别名,太长了...
from django.conf import settings


def generate_save_user_token(openid):
    """
    使用itsdangerous对原始的openid进行签名
    :param openid: 原始的openid
    :return: 签名后的openid
    """
    # 创建序列化器对象，指定秘钥和过期时间（10分钟）
    serializer = TJSSerializer(settings.SECRET_KEY, 600)
    # 准备原始的openid
    data = {'openid':openid}
    # 对openid进行签名,返回签名之后的bytes类型的字符串
    token = serializer.dumps(data)
    # 将bytes类型的字符串转成标准的字符串，并返回
    return token.decode()
```

##### 3.响应签名后的openid给前端

```python
class QQAuthUserView(GenericAPIView):
    ......
    except OAuthQQUser.DoesNotExist:
    # 如果openid没绑定美多商城用户，创建用户并绑定到openid
  	# 为了方便后续使用openid去绑定美多商城用户，现在需要将openid响应给用户
    # return Response({'openid':open_id}) # 不能将openid以明文的形式发送出去，需要混淆，让外界不知道原始数据
        openid_access_token = generate_save_user_token(open_id)
        return Response({'access_token':openid_access_token})
    ......
```

#### 2.绑定⽤户到openid后端逻辑

##### 1.视图逻辑准备

```python
# url(r'^qq/user/$', views.QQAuthUserView.as_view()),
class QQAuthUserView(GenericAPIView):
	"""⽤户扫码登录的回调处理"""
    # 指定序列化器
    serializer_class = serializers.QQAuthUserSerializer
    def get(self, request):
 		# 提取code请求参数
		 .......
 	def post(self, request):
 		"""使⽤open_id绑定美多⽤户"""
		 pass
```

##### 2.序列化器逻辑

```python
from rest_framework import serializers
from django_redis import get_redis_connection

from .utils import check_save_user_token
from users.models import User
from .models import OAuthQQUser


class QQAuthUserSerializer(serializers.Serializer):
    """
    QQ登录创建用户序列化器
    """
    access_token = serializers.CharField(label='操作凭证')
    mobile = serializers.RegexField(label='手机号', regex=r'^1[3-9]\d{9}$')
    password = serializers.CharField(label='密码', max_length=20, min_length=8)
    sms_code = serializers.CharField(label='短信验证码')

    def validate(self, data):
        # 检验access_token
        access_token = data['access_token']
        # 获取身份凭证
        openid = check_save_user_token(access_token)
        if not openid:
            raise serializers.ValidationError('无效的access_token')

        # 将openid放在校验字典中，后面会使用
        data['openid'] = openid

        # 检验短信验证码
        mobile = data['mobile']
        sms_code = data['sms_code']
        redis_conn = get_redis_connection('verify_codes')
        real_sms_code = redis_conn.get('sms_%s' % mobile)
        if real_sms_code.decode() != sms_code:
            raise serializers.ValidationError('短信验证码错误')

        # 如果用户存在，检查用户密码
        try:
            user = User.objects.get(mobile=mobile)
        except User.DoesNotExist:
            pass
        else:
            password = data['password']
            if not user.check_password(password):
                raise serializers.ValidationError('密码错误')

            # 将认证后的user放进校验字典中，后续会使用
            data['user'] = user
        return data

    def create(self, validated_data):
        # 获取校验的用户
        user = validated_data.get('user')

        if not user:
            # 用户不存在,新建用户
            user = User.objects.create_user(
                username=validated_data['mobile'],
                password=validated_data['password'],
                mobile=validated_data['mobile'],
            )

        # 将用户绑定openid
        OAuthQQUser.objects.create(
            openid=validated_data['openid'],
            user=user
        )
        # 返回用户数据
        return user
```

##### 3.读取itsdangerous处理后的openid

oauth.utils.py

```python
def check_save_user_token(token):
    """
    检验保存⽤户数据的token
    :param token: token
    :return: openid or None
    """
    serializer = Serializer(settings.SECRET_KEY, expires_in=constants.SAVE_QQ_USER_TOKEN_EXPIRES)
 	try:
 		data = serializer.loads(token)
 	except BadData:
 		return None
 	else:
 		return data.get('openid')
```

##### 4.视图逻辑

```python
class QQAuthUserView(GenericAPIView):
    ......
    def post(self, request):
        """绑定openid到美多用户
        """
        # 初始化序列化器对象
        serializer = self.get_serializer(data=request.data)
        # 调用校验的方法
        serializer.is_valid(raise_exception=True)
        # 调用save(),保存数据
        user = serializer.save()

        # 生成JWT token，并响应
        jwt_payload_handler = api_settings.JWT_PAYLOAD_HANDLER
        jwt_encode_handler = api_settings.JWT_ENCODE_HANDLER
        payload = jwt_payload_handler(user)
        token = jwt_encode_handler(payload)

        # 保存结束表示QQ登录成功，需要生成JWT token，并返回 token,user_id,username
        return Response({
            'token': token,
            'user_id': user.id,
            'username': user.username
        })
```

