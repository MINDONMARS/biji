# Django多账号登陆(用户名/手机号)

[TOC]

## 理论依据

1. JWT扩展的登录视图，在收到⽤户名与密码时，也是调⽤Django的认证系统中提供的authenticate()来检查⽤户名与密码是否正确。
2. 修改Django认证系统的认证后端需要继承django.contrib.auth.backends.ModelBackend，并重写authenticate⽅法。

## 代码实现（users.utils.py）

```python
def get_user_by_account(account):
    """
    根据用户名或者手机号查询用户
    :param account: 用户名或者手机号
    :return: user/None
    """
    try:
        if re.match(r'^1[3-9]\d{9}$', account):
            # 使用手机号查询
            user = User.objects.get(mobile=account)
        else:
            # 使用用户名查询
            user = User.objects.get(username=account)
    except User.DoesNotExists:
        return None
    else:
        return user


class UsernameMobileAuthBackend(ModelBackend):
    """自定义用户认证后端"""

    def authenticate(self, request, username=None, password=None, **kwargs):
        """
        根据username(此时不仅仅表示用户名，主要是账号，可以是用户名也可以是手机号)
        :param request: 本次登录请求
        :param username: 可以是用户名也可以是手机号
        :param password: 密码明文
        :param kwargs: 额外参数
        :return: user
        """
        # 使用账号或者手机号查询用户
        user = get_user_by_account(username)

        # 如果用户存在，就校验密码，密码正确就返回user
        if user and user.check_password(password):
            return user
```

## 设置配置文件默认的用户认证后端

```python
# 修改默认的认证后端
AUTHENTICATION_BACKENDS = [
    'users.utils.UsernameMobileAuthBackend',
]
```

## 登录时next参数的作⽤

### 作⽤

1. next可以⽤于标记⽤户是从哪个界⾯跳转到登录⻚
2. 登录完成后前端控制next参数，跳转回来时的界⾯

### 提示：

属于前端约定的⽤户交互效果，主要由前端程序员实现

## 举例

1. ⽤户尝试进⼊user_center_info.html界⾯

2. 发现登录状态是未登录

3. 引导到登录界⾯(此时在引导到登录界⾯时，可以追加next参数)

   http://127.0.0.1:8080/login.html?next=user_center_info.html

4. 登录完成后，前端判断是否有next参数，如果有，登录完成后跳转回user_center_info