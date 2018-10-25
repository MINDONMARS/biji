# Django异步发送邮件(验证邮箱)

[TOC]

## 1.定义发送邮件异步任务

配置邮箱服务器

```python
# 配置邮件服务器
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend' # 导入邮件模块
EMAIL_HOST = 'smtp.163.com' # 发邮件主机
EMAIL_PORT = 25 # 发邮件端口
EMAIL_HOST_USER = 'chi_xu_1016@163.com' # 授权的邮箱
EMAIL_HOST_PASSWORD = 'CH1230' # 邮箱授权时获得的密码，非注册登录密码
EMAIL_FROM = '美多商城<chi_xu_1016@163.com>' # 发件人抬头
```

```python
    def generate_email_verify_url(self):
        """
        生成邮箱验证时的激活链接
        :return: 激活链接
        提示：self，就是该实例方法的调用者，此处是当前的更新邮件的登录用户
        """
        # 创建序列化器对象
        serializer = TJSSerializer(settings.SECRET_KEY, expires_in=constants.VERIFY_EMAIL_TOKEN_EXPIRES)
        # 准备原始数据
        data = {
            'user_id':self.id,
            'email':self.email
        }
        # 进行签名
        token = serializer.dumps(data).decode()

        # 使用token拼接完整的激活链接
        verify_url = 'http://www.meiduo.site:8080/success_verify_email.html?token=' + token

        return verify_url
```

文件路径:celery_tasks.email.tasks.py

```python
# 注册celery任务
celery_app.autodiscover_tasks(['celery_tasks.sms','celery_tasks.email'])
```

```python
from celery_tasks.main import celery_app
from django.core.mail import send_mail
from django.conf import settings


@celery_app.task(name='send_verify_email')
def send_verify_email(to_email, verify_url):
    """
    定义发送验证邮件的方法
    :param to_email: 收件人
    :param verify_url: 邮箱激活链接
    :return: None
    """
    subject = "美多商城邮箱验证"
    html_message = '<p>尊敬的用户您好！</p>' \
                   '<p>感谢您使用美多商城。</p>' \
                   '<p>您的邮箱为：%s 。请点击此链接激活您的邮箱：</p>' \
                   '<p><a href="%s">%s<a></p>' % (to_email, verify_url, verify_url)
    send_mail(subject, "", settings.EMAIL_FROM, [to_email], html_message=html_message)
```

## 2.生成激活链接

文件路径:users.models.User(这样在发送邮件时可以直接使用当前用户模型对象的方法)

```python
    def generate_email_verify_url(self):
        """
        生成邮箱验证时的激活链接
        :return: 激活链接
        提示：self，就是该实例方法的调用者，此处是当前的更新邮件的登录用户
        """
        # 创建序列化器对象
        serializer = TJSSerializer(settings.SECRET_KEY, expires_in=constants.VERIFY_EMAIL_TOKEN_EXPIRES)
        # 准备原始数据
        data = {
            'user_id':self.id,
            'email':self.email
        }
        # 进行签名
        token = serializer.dumps(data).decode()

        # 使用token拼接完整的激活链接
        verify_url = 'http://www.meiduo.site:8080/success_verify_email.html?token=' + token

        return verify_url
```

## 3.触发异步任务

在序列化器中保存完邮箱,响应之前触发异步任务发送邮件

```python
def update(self, instance, validated_data):
        """重写该方法：
        我们只对email进行存储
        后续还会在这里发送邮件：需要在更新数据成功之后，返回响应之前发送邮件
        instance ： 表示当前登录用户对象
        """
        instance.email = validated_data['email']
        instance.save() # save()是ORM的方法

        # 获取邮箱激活链接
        # verify_url = generate_email_verify_url(user)
        verify_url = instance.generate_email_verify_url()

        # 更新完邮箱之后，处理响应之前，异步发送邮箱验证的邮件
        # send_verify_email() # 不会触发异步任务，相当于调用普通包中的普通函数而已
        send_verify_email.delay(instance.email, verify_url)

        return instance
```

## 4.验证邮箱的后端业务实现

解出url中token中的信息(user_id, email)

文件路径:users.models.User(这样在验证邮件时可以直接使用当前用户模型类的静态方法)

```python
    @staticmethod
    def check_email_verify_token(token):
        """
        验证token并提取用户信息
        :param token: 签名后的token
        :return: user、None
        """

        # 创建序列化器
        serializer = TJSSerializer(settings.SECRET_KEY, expires_in=constants.VERIFY_EMAIL_TOKEN_EXPIRES)

        try:
            # 解出签名后的token
            data = serializer.loads(token)
        except BadData:
            return None
        else:
            user_id = data.get('user_id')
            email = data.get('email')
            # 使用user_id和email作为条件，出现出user
            try:
                user = User.objects.get(id=user_id, email=email)
            except User.DoesNotExist:
                return None
            else:
                return user
```

验证视图(调用上方读取user逻辑)

```python
# url(r'^emails/verification/$', views.VerifyEmailView.as_view()),
class VerifyEmailView(APIView):
    """实现邮箱的验证激活"""

    def get(self, request):
        # 从查询字符串中读取token
        token = request.query_params.get('token')
        if not token:
            return Response({'message':'缺少token'}, status=status.HTTP_400_BAD_REQUEST)

        # 再从token中读取用户信息,直接读取token中的user
        user = User.check_email_verify_token(token)
        if not user:
            return Response({'message':'无效token'}, status=status.HTTP_400_BAD_REQUEST)

        # 最后修改用户的email_active字段为True
        user.email_active = True
        user.save()

        # 响应结果
        return Response({'message': 'OK'})
```

