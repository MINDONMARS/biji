# Django注册登陆(JWT状态保持)

[TOC]

## 1.JWT登录后端逻辑实现

### 配置认证

```python
# DRF配置
REST_FRAMEWORK = {
    # 异常处理
    'EXCEPTION_HANDLER': 'meiduo_mall.utils.exceptions.exception_handler',
    
    ###########################################
    
    # 认证
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_jwt.authentication.JSONWebTokenAuthentication', # JWT认证，在前面的认证方案优先
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.BasicAuthentication',
    ),
}
```

### 配置过期时间

```python
# JWT配置
JWT_AUTH = {
    # WT的有效期
    'JWT_EXPIRATION_DELTA': datetime.timedelta(days=1),
}
```

### 后端代码

```python
from rest_framework_jwt.views import obtain_jwt_token
urlpatterns = [
    。。。。。。
    # JWT登录
    url(r'^authorizations/$', obtain_jwt_token),
]

# obtain_jwt_token是一个变量里面封装ObtainJSONWebToken.as_view()
```

### 补充JWT登录请求后返回的字段

```python
# 重写ObtainJSONWebToken视图的父类中post中以下方法{jwt_response_payload_handler}
response_data = jwt_response_payload_handler(token, user, request)
```

创建应用内utils.py文件, 重写, 源代码可追踪断点进去看, 只return{'token':token} 

```python
def jwt_response_payload_handler(token, user=None, request=None):
    """
    自定义jwt认证成功返回数据
    """
    return {
        'token': token,
        'user_id': user.id,
        'username': user.username
    }
```

在配置文件中重新制定自定义的方法

```python
# JWT配置
JWT_AUTH = {
    # WT的有效期
    'JWT_EXPIRATION_DELTA': datetime.timedelta(days=1),
    
   ################################################
    
    # 为JWT登录视图补充返回值
    'JWT_RESPONSE_PAYLOAD_HANDLER': 'users.utils.jwt_response_payload_handler',
}
```

