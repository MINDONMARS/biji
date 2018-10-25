# Django用户中心(添加邮箱)

## 1.添加邮箱后端逻辑

### 1.视图

```python
from django.shortcuts import render
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.generics import CreateAPIView, RetrieveAPIView, UpdateAPIView
from rest_framework.permissions import IsAuthenticated

from .models import User
from . import serializers
# Create your views here.


class EmailView(UpdateAPIView):
    """添加邮箱"""

    # 指定序列化器
    serializer_class = serializers.EmailSerializer
    # 指定权限：只有登录用户才能访问该接口
    permission_classes = [IsAuthenticated]

    def get_object(self):
        """重写该方法，告诉序列化器要序列化的登录用户数据
        user是经过JSONWebTokenAuthentication认证之后的user
        经过permission_classes的权限之后，能够进入这里一定是登录用户
        """
        return self.request.user
```

### 2.序列化器

```python
from rest_framework import serializers
import re
from django_redis import get_redis_connection
from rest_framework_jwt.settings import api_settings

from .models import User


class EmailSerializer(serializers.ModelSerializer):
    """添加邮箱序列化器"""

    class Meta:
        model = User
        fields = ['id', 'email']

        # 指定email为必传字段
        extra_kwargs = {
            'email':{
                'required':True
            }
        }

    def update(self, instance, validated_data):
        """重写该方法：
        我们只对email进行存储
        后续还会在这里发送邮件：需要在更新数据成功之后，返回响应之前发送邮件
        instance ： 表示当前登录用户对象
        """
        instance.email = validated_data['email']
        instance.save() # save()是ORM的方法
        return instance
```

