# Django用户中心(页面展示)

[TOC]

## 1.给User模型类新增email_active字段

为了展示邮箱的认证状态, 标识该⽤户的邮件是否通过验证

⽤于个⼈中⼼界⾯展示邮箱时的判断条件:

email_active=True 展示⽤户邮箱

email_active=False 展示设置按钮

```python
class User(AbstractUser):
    """⽤户模型类"""
    mobile = models.CharField(max_length=11, unique=True, verbose_name='⼿机号')
    email_active = models.BooleanField(default=False, verbose_name='邮箱验证状态')
```

数据库迁移

```python
python manage.py makemigrations
python manage.py migrate
```

## 2.个⼈信息接⼝

### 1.判断⽤户是否登录

**提示**: 根据前端请求头中传⼊的JWT token信息，使⽤DRF中追加的JWT认证判断是否登录

认证结果

True 查询和响应该⽤户的详情

Fasle 权限验证失败

```python
# DRF配置
REST_FRAMEWORK = {
    # 异常处理
    'EXCEPTION_HANDLER': 'meiduo_mall.utils.exceptions.exception_handler',
    # 认证
    'DEFAULT_AUTHENTICATION_CLASSES': (
    'rest_framework_jwt.authentication.JSONWebTokenAuthentication',
    'rest_framework.authentication.SessionAuthentication',
    'rest_framework.authentication.BasicAuthentication',
    ),
}
```

### 2.实现逻辑

#### 1.视图

```python
from django.shortcuts import render
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.generics import CreateAPIView, RetrieveAPIView, UpdateAPIView
from rest_framework.permissions import IsAuthenticated

from .models import User
from . import serializers
# Create your views here.


# url(r'^user/$', views.UserDetailView.as_view()),
class UserDetailView(RetrieveAPIView):
    """用户基本信息"""

    # 指定序列化器
    serializer_class = serializers.UserDetailSerializer
    # 指定权限：只有登录用户才能访问该接口
    permission_classes = [IsAuthenticated]

    def get_object(self):
        """重写该方法，告诉序列化器要序列化的登录用户数据
        user是经过JSONWebTokenAuthentication认证之后的user
        经过permission_classes的权限之后，能够进入这里一定是登录用户
        """
        return self.request.user
```

#### 2.序列化器

```python
from rest_framework import serializers
#import re
#from django_redis import get_redis_connection
#from rest_framework_jwt.settings import api_settings

from .models import User


class UserDetailSerializer(serializers.ModelSerializer):
    """对用户基本信息进行序列化"""

    class Meta:
        model = User
        fields = ['id', 'username', 'mobile', 'email', 'email_active']
```

