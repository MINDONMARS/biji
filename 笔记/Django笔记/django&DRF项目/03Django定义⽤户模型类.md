# Django定义⽤户模型类

[TOC]

## 1.Django提供⽤户认证系统介绍

相关⽂档 https://yiyibooks.cn/xx/Django_1.11.6/topics/auth/index.html

Django的认证系统包含

1. ⽤户的数据模型

   提供⽤户模型类(但是默认字段可能不全，需要⾃定义 继承⾃ AbstractUser)

   ```python
   from django.contrib.auth.models import AbstractUser
   ```

2. .⽤户密码的加密与验证 

   提供密码加密和验证的⽅法

   ```python
   set_password(self, raw_password)
   check_password(self, raw_password)
   ```

3. ⽤户的权限系统 （认证和权限）

   Django认证系统同时处理认证和授权。

   即Django的认证系统同时提供了认证机制和权限机制。

## 2.⾃定义⽤户模型类

新建apps目录, 新建users应⽤, 在models文件中创建

```python
class User(AbstractUser):
    """⾃定义⽤户模型类"""
    mobile = models.CharField(max_length=11, unique=True, verbose_name='⼿机号')
    
    
	class Meta:
        db_table = 'tb_users'
        verbose_name = '⽤户'
        verbose_name_plural = verbose_name
```

### 重新指定默认⽤户模型类

**提示**

默认的⽤户模型类是 AbstractUser
现在需要替换成我们⾃定义的⽤户模型类 User

```python
# 指定默认的⽤户模型类
AUTH_USER_MODEL = 'users.User'
```

## 3.注册users应⽤

```python
'users.apps.UsersConfig', # ⽤户模块
```

**注意** 迁移模型类之前，需要注册应⽤

## 4.追加导包路径

**提示** 上⾯注册users应⽤的⽅式的导包路径，默认是不会被识别的，需要追加导包路径

查看默认的导包路径

```python
import sys
print(sys.path)
```

追加导包路径 

```python
sys.path.insert(0, os.path.join(BASE_DIR, 'apps'))
```

## 5.实现迁移

```python
python manage.py makemigrations
python manage.py migrate
```

