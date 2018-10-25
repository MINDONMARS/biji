# Django注册之短信验证码

[TOC]

## 1.短信验证码需求分析

	0.新建应⽤ verifications
	1.接受⼿机号码，并校验
	2.⽣成短信验证码
	3.保存短信验证码到Redis
	4.集成容联云通讯发送短信验证码
	5.响应结果

## 2.后端接⼝设计和定义

### 后端接⼝设计

1.请求⽅法 GET

2.请求地址 /sms_codes/(?P1[3-9]\d{9})/

3.请求参数

|  参数  | 类型 | 是否必须 |  说明  |
| :----: | :--: | :------: | :----: |
| mobile | str  |    是    | 手机号 |

4.响应数据(JSON)

| 返回值  | 类型 | 是否毕传 |    说明     |
| :-----: | :--: | :------: | :---------: |
| message | str  |    否    | OK,发送成功 |

### 后端接⼝定义

总路由

```python
# 验证模块
url(r'^', include('verifications.urls')),
```

⼦路由

```python
# 短信验证码
url(r'/sms_codes/(?P<mobile>1[3-9]\d{9})/', views.SMSCodeView.as_view()),
```

视图

```python
class SMSCodeView(APIView):
    # 获取参数：mobile,image_code_id,text(暂时省略)
    # 校验参数(暂时省略)
    # ⽐较客户端传⼊的和服务器存储的图⽚验证码是否⼀致(暂时省略)
    # 如果⼀致，⽣成短信验证码
    sms_code = '%06d' % random.randint(0,999999)

    # 存储短信验证码内容到redis数据库
    redis_conn = get_redis_connection('verify_codes')
    redis_conn.setex("sms_%s" % mobile, constants.SMS_CODE_REDIS_EXPIRES, sms_code)
    # 发送短信验证码
    CCP().send_template_sms(mobile,[sms_code, constants.SMS_CODE_REDIS_EXPIRES // 60], 1)
    # 响应发送短信验证码结果
    return Response({"message": "OK"})
```

## 3.跨域CORS

1. #### 跨域CORS介绍

   域：协议 + IP(域名) + 端⼝

   前端：域 http://127.0.0.1:8080
   后端：域 http://127.0.0.1:8000

   说明：

   1. js 进⾏跨域请求 同源策略
   2. 浏览器会尝试向后端发送options请求 —> 向后端询问是否⽀持前端这个域名发起的请求
   3. 后端返回allow，就可以进⾏跨域请求

2. #### 跨域CORS解决

   1.⽅案 在中间件处理js的跨域请求

   2.安装cors-headers⼯具 pip install django-cors-headers

   3.安装cors-headers应⽤

   ```python
   INSTALLED_APPS = [
       ...
       'corsheaders', # cors
       ...
   ]
   ```

   4.配置中间件(放最前面)

   ```python
   MIDDLEWARE = [
       'corsheaders.middleware.CorsMiddleware', # 最外层的中间件
       'django.middleware.security.SecurityMiddleware',
       'django.contrib.sessions.middleware.SessionMiddleware',
       'django.middleware.common.CommonMiddleware',
       'django.middleware.csrf.CsrfViewMiddleware',
       'django.contrib.auth.middleware.AuthenticationMiddleware',
       'django.contrib.messages.middleware.MessageMiddleware',
       'django.middleware.clickjacking.XFrameOptionsMiddleware',
   ]
   ```

   5.添加⽩名单

   ```python
   # CORS
   CORS_ORIGIN_WHITELIST = (
       '127.0.0.1:8080',
       'localhost:8080',
   )
   CORS_ALLOW_CREDENTIALS = True # 允许携带cookie
   
   ALLOWED_HOSTS = ['127.0.0.1', 'localhost']
   ```

3. #### 设置域名

```
1.域名设计
前端 127.0.0.1 www.meiduo.site
后端 127.0.0.1 api.meiduo.site
```

域名配置:`sudo vim /etc/hosts`

windows C:\Windows\System32\drivers\etc

追加⽩名单

```python
# CORS
CORS_ORIGIN_WHITELIST = (
    '127.0.0.1:8080',
    'localhost:8080',
    'www.meiduo.site:8080',
    'api.meiduo.site:8000'
)
CORS_ALLOW_CREDENTIALS = True # 允许携带cookie
ALLOWED_HOSTS = ['127.0.0.1', 'localhost', 'www.meiduo.site', 'api.meiduo.site']
```

## 4.发送短信验证码优化

1. ### 校验60s内是否重复发送短信

   ⽬的 防⽌恶意⽤户暴⼒测试短信接⼝，频繁发送短信

   逻辑分析 

   	1.每次发送短信验证码时，在Redis数据库中记录⼀个值，并设置该值的		有效期为60s
   	2.在⽣成短信验证码之前，尝试从Redis数据库中读取该值
   	3.读取结果
   		存在 说明在60s内重复发送，响应400
   		不存在 说明在60s内没有重复发送，继续执⾏后续逻辑

   ```python
   class SMSCodeView(APIView):
       """短信验证码"""
       
       def get(self, request, mobile):
           """
           GET /sms_codes/(?P<mobile>1[3-9]\d{9})/
           """
           # 创建连接到redis对象
           redis_conn = get_redis_connection('verify_codes')
           # 判断⽤户是否是在60s内发送短信
           send_flag = redis_conn.get('send_flag_%s' % mobile)
           if send_flag:
           	return Response({'message':'发送短信频繁'}, status=status.HTTP_400_BAD_REQUEST)
           # ⽣成短信验证码
           sms_code = '%06d' % random.randint(0, 999999)
           logger.info(sms_code)
           # 保存短信验证码
           redis_conn.setex('sms_%s' % mobile, constants.SMS_CODE_REDIS_EXPIRES, sms_code)
           # 保存发送短信验证码的标记
           redis_conn.setex('send_flag_%s' % mobile, constants.SEND_SMS_CODE_INTERVAL, 1)
           # 集成容联云通讯发送短信验证码
           CCP().send_template_sms(mobile, [sms_code, constants.SMS_CODE_REDIS_EXPIRES // 60], constants.SMS_CODE_TEMPLATE_ID)
           # 响应结果
           return Response({'message':'OK'})
   ```

2. ### 2.redis pipeline的使⽤

   ⽬的 ⼀次执⾏多条Redis命令，避免多条命令多次访问Redis数据库

   逻辑实现

   ```python
   # 创建管道
   pl = redis_conn.pipeline()
   # 保存短信验证码
   # redis_conn.setex('sms_%s' % mobile, constants.SMS_CODE_REDIS_EXPIRES, sms_code)
   pl.setex('sms_%s' % mobile, constants.SMS_CODE_REDIS_EXPIRES, sms_code)
   # 保存发送短信验证码的标记
   # redis_conn.setex('send_flag_%s' % mobile, constants.SEND_SMS_CODE_INTERVAL, 1)
   pl.setex('send_flag_%s' % mobile, constants.SEND_SMS_CODE_INTERVAL, 1)
   # 执⾏
   pl.execute()
   ```

   **注意点** 在使⽤pipeline时，记得调⽤ execute() 执⾏⼀下

3. ### Celery异步发送短信

   Celery介绍

   ![1535961618834](C:\Users\ADMINI~1\AppData\Local\Temp\1535961618834.png)

   客户端 

   	Django、定义和发起异步任务

   任务队列(broker) 存储异步任务
   	redis
   	RabbitMQ

   任务处理着(worker)

   	执⾏异步任务

   安装和启动Celery

```
# 进⼊虚拟环境
pip install celery
celery -A 应⽤的包路径 worker -l info
```

Celery定义

	代码结构

![1535961791304](C:\Users\ADMINI~1\AppData\Local\Temp\1535961791304.png)

main.py

```python
# celery启动⽂件
from celery import Celery


# 创建celery实例
celery_app = Celery('meiduo')
# 加载celery配置
celery_app.config_from_object('celery_tasks.config')
# ⾃动注册celery任务
celery_app.autodiscover_tasks(['celery_tasks.sms'])
```

config.py	

```python
# celery配置⽂件
# 指定任务队列的位置
broker_url = "redis://192.168.73.128/7"
```

Celery异步任务的定义(发短信)

sms.tasks.py

```python
from .yuntongxun.sms import CCP
from . import constants
from celery_tasks.main import celery_app


# 装饰器将send_sms_code装饰为异步任务,并设置别名
@celery_app.task(name='send_sms_code')
def send_sms_code(mobile, sms_code):
    """
    发送短信异步任务
    :param mobile: ⼿机号
    :param sms_code: 短信验证码
    :return: None
    """
    CCP().send_template_sms(mobile, [sms_code, constants.SMS_CODE_REDIS_EXPIRES // 60], constants.SEND_SMS_TEMPLATE_ID)
```

开启Celery

```
celery -A 应⽤包路径 worker -l info
celery -A celery_tasks.main worker -l info
```

执⾏Celery异步任务

```python
# ⽣成和发送短信验证码
sms_code = '%06d' % random.randint(0,999999)
# CCP().send_template_sms(mobile,[sms_code, constants.SMS_CODE_REDIS_EXPIRES // 60], 1)
# celery异步发送短信
send_sms_code.delay(mobile,sms_code)
```

