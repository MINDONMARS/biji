# Flask注册登录模块

[TOC]

## 数据库

##### 数据库表结构和关系分析 

![1535189601888](C:\Users\ADMINI~1\AppData\Local\Temp\1535189601888.png)

##### 数据库迁移建表

	导⼊模型⽂件  

```python
  constants.py 和 models.py ⽂件拷⻉到项⽬的 info ⽬录下
```

	模型迁移建表

```python
1.创建迁移⽂件夹，存储迁移版本 python manage.py db init 
2.创建当前版本的迁移⽂件 python manage.py db migrate -m 'initial' 
3.执⾏当前版本的迁移⽂件 python manage.py db upgrade
```

	可能出现的问题 

```python
NO change in schema detected  # model没被引用

alembic.util.exc.CommandError: Can't locate revision identified by '8d743b5f56c0'  # 删除字段数据
```

##### 添加测试数据

	准备sql⽂件 
	
	information_info_category.sql
	
	information_info_news.sql
	
	导⼊sql⽂件 
	
	source information_info_category.sql
	
	source information_info_news.sql

## 导入静态文件

##### 主页渲染

```python
@index_blue.route('/')
def index():
    """主⻚"""
    # 渲染主⻚
    return render_template('news/index.html')
```

##### ⽹站title标题favicon.ico 

```python
# http://127.0.0.1:5000/favicon.ico
@index_blue.route('/favicon.ico')
def favicon():
    """加载⽹站title图标"""
    return current_app.send_static_file('news/favicon.ico')

```

	Mac上清除favicon.ico在⾕歌浏览器的缓存  

```
cd /Users/zhangjie/Library/Application\ Support/Google/Chrome/Default/
rm Favicons Favicons-journal
```

## 图⽚验证码

##### 图⽚验证码显示逻辑 

浏览器请求验证码 > > 服务器接收请求 > > 生成图片验证码 > >返回图片验证码 >>浏览器展示验证码

##### 图⽚验证码使⽤逻辑 

浏览器传递参数UUID请求验证码 > > 服务器接收UUID生成验证码 > > 使用UUID存储验证码内容(redis) > > 返回验证码 > > 用户输入验证码点击向服务器请求获取短信验证码 > > 服务器使用UUID在redis中拿到验证码与用户输入对比 > > 对比成功发送短信

##### 后端逻辑

1.创建passport模块 实现登录注册 

2.准备蓝图 

```python
from flask import Blueprint

passport_blue = Blueprint('passport', __name__, url_prefix='/passport')

from . import views
```

3.导⼊框架 captcha 

	⽤于⽣成图⽚验证码 
	
	在info⽬录下，创建utils⼯具包，导⼊框架 captcha 到utils⼯具包中 

```python
@passport_blue.route('/image_code')
def get_image_code():
    # 接收图片uuid
    image_code_id = request.args.get("imageCodeId")

    # 校验uuid
    if not image_code_id:
        abort(400)
    # 生成验证码的图片和文字信息
    name, text, image = captcha.generate_captcha()

    # 讲uuid和图片验证码文字绑定到redis
    try:
        redis_store.set("imageCode:" + image_code_id, text, constants.IMAGE_CODE_REDIS_EXPIRES)
    except Exception as e:
        logging.error(e)
        abort(500)
    # 响应
    response = make_response(image)
    response.headers['Content-Type'] = 'image/jpg'
    return response
```

**注*:** 1.参数需要校验 

	2.操作redis数据库需要try 
	
	3.需要在响应中设置验证码的数据类型为image/jpg 

## 短信验证码 

##### 第三⽅云通讯集成 

⽹址 云通讯：http://www.yuntongxun.com/ 

1.下载云通讯平台的SDK 

	⽂档及Demo地址：http://www.yuntongxun.com/doc/ready/demo/1_4_1_2.html 
	
	SDK是⽤于调⽤API的 

2.集成SDK到项⽬中 

	在info⽬录下，创建 libs ⼯具包，并将下载的 yuntongxun SDK拷⻉到libs⼯具包
	
	 注册云通信账号后配置sms.py:

```python
# 说明：主账号，登陆云通讯网站后，可在"控制台-应用"中看到开发者主账号ACCOUNT SID
_accountSid = '8aaf07086521916801652d2332ee0734'

# 说明：主账号Token，登陆云通讯网站后，可在控制台-应用中看到开发者主账号AUTH TOKEN
_accountToken = '1d8d4a5bdc71430aa7767bd951ff025e'

# 请使用管理控制台首页的APPID或自己创建应用的APPID
_appId = '8aaf07086521916801652d233340073a'
```

3.集成测试代码，并发送测试短信 

```python
# 向mobile发送短信，验证码为666，请于5分钟内输⼊ '1' 表示默认的短信模板
sendTemplateSMS('mobile',['666', '5'], '1')
```

##### 封装发短信的单例类 

防止大量用户请求发送短信而创建大量对象占用资源

```python
class CCP(object):
    """封装单例类发送短信验证码"""
    def __new__(cls, *args, **kwargs):
        # 判断_instance对象是否存在, 如果存在,就不再重复的创建对象, 如果不存就创建一个_instanc对象
        if not hasattr(cls, '_instance'):
            cls._instansce = super(CCP, cls).__new__(cls, *args, **kwargs)

            # 搭配单例让, rest对象只实例化一次
            rest = REST(_serverIP, _serverPort, _softVersion)
            # 给_instance单例动态的绑定一个rest属性, 属性保存到rest对象
            cls._instansce.rest = rest

            cls._instansce.rest.setAccount(_accountSid, _accountToken)
            cls._instansce.rest.setAppId(_appId)

        return cls._instansce

    def send_sms_code(self, to, datas, tempId):
        """
        单例发送短信的方法:需要单例对象才能调用的
        :param to: 发送短信的手机
        :param datas: ['sms_code', 短信提示过期的时间]
        :param tempId: 默认免费的是1
        :return: 如果发送短信成功，返回0；反之，返回-1
        """
        # self ： 代表CCP()初始化出来的单例_instance
        # sendTemplateSMS : 是容联云通讯提供的发送短信的底层方法，我只借用了一下，在外面套了壳子（send_sms_code）
        result = self.rest.sendTemplateSMS(to, datas, tempId)

        if result.get('statusCode') != '000000':
            return -1
        else:
            return 0
```

##### 发送短信验证码后端逻辑 

```python
@passport_blue.route('/sms_code', methods=['POST'])
def send_sms_code():
    """发送短信验证码
    1.接受参数：mobile, image_code, image_code_id
    2.校验参数：判断参数是否齐全，判断手机号格式是否正确
    3.使用image_code_id从redis中读取出服务器存储的图片验证码
    4.使用用户输入的图片验证码 对比 服务器存储的图片验证码
    5.如果对比成功生成短信验证码，长度是6位的随机数字
    6.再使用CCP()单例发送短信验证码
    7.发送短信验证码成功，就把短信验证码存储到redis()
    8.返回短信验证码发送的结果
    """
    # 接收参数: mobile, image_code, image_code_id
    json_dict = request.json
    mobile = json_dict.get('mobile')
    image_code_client = json_dict.get('image_code')
    image_code_id = json_dict.get('image_code_id')
    # 校验参数: 是否齐全, 手机号格式是否正确
    if not all([mobile, image_code_client, image_code_id]):
        return jsonify(errno=response_code.RET.PARAMERR, errmsg='缺少必传参数')
    if not re.match('^1[345678][0-9]{9}$', mobile):
        return jsonify(errno=response_code.RET.PARAMERR, errmsg='手机号格式错误')
    # 使用image_code_id从redis中读取出服务器存储的图片验证码
    try:
        image_code_server = redis_store.get(

            "imageCode:" + image_code_id
        )
    except Exception as e:
        logging.error(e)
        return jsonify(errno=response_code.RET.PARAMERR, errmsg='查询验证码失败')
    # 判断image_code_id 是否存在
    if not image_code_server:
        return jsonify(errno=response_code.RET.NODATA, errmsg='图片验证码不存在')
    # 4.使用用户输入的验证码对比服务器存储的验证码
    if image_code_client.lower() != image_code_server.lower():
        return jsonify(errno=response_code.RET.PARAMERR, errmsg='验证码输入有误')
    # 成功 : 生成6位验证码
    sms_code = "%06d" % random.randint(0, 999999)
    logging.debug(sms_code)
    # CCP()单例发送验证码

    # 不发短信

    # result = CCP().send_sms_code(mobile, [sms_code, 5], 1)
    # if result != 0:
    #     return jsonify(errno=response_code.RET.THIRDERR, errmsg='发送短信失败')
    # 发送成功, 将验证码存储到redis
    try:
        redis_store.set('SMS:' + mobile, sms_code, constants.SMS_CODE_REDIS_EXPIRES)
    except Exception as e:
        logging.error(e)
        return jsonify(errno=response_code.RET.DATAERR, errmsg='存储短信失败')

    # 返回结果
    return jsonify(errno=response_code.RET.OK, errmsg='发送短信成功')


@passport_blue.route('/image_code')
```

## 注册 

##### 后端逻辑注意点 :

- 保存session数据，实现注册即登录 
- 注册时记录最后⼀次登录时间 
- 密码加密后再保存 

```python
@passport_blue.route('/register', methods=['post'])
def register():
    # 接收参数 mobile, password
    json_dict = request.json
    mobile = json_dict.get('mobile')
    sms_code_client = json_dict.get('sms_code')
    password = json_dict.get('password')
    # 校验数据会否存在
    if not all([mobile, password]):
        return jsonify(errno=response_code.RET.PARAMERR, errmsg='缺少必传参数')
    # 校验手机号
    if not re.match('^1[345678][0-9]{9}$', mobile):
        return jsonify(errno=response_code.RET.PARAMERR, errmsg='手机号格式错误')
    # 用手机号去redis取短信验证码
    try:
        sms_code_server = redis_store.get('SMS:' + mobile)
    except Exception as e:
        logging.error(e)
        return jsonify(errno=response_code.RET.DBERR, errmsg='查询短信验证码失败')
    # 短信验证码是否过期
    if not sms_code_server:
        return jsonify(errno=response_code.RET.NODATA, errmsg='短信验证码不存在')
    # 短信验证码是否正确
    if sms_code_server != sms_code_client:
        return jsonify(errno=response_code.RET.PARAMERR, errmsg='验证码输入错误')
    # 正确, 创建User模型对象
    user = User()
    user.nick_name = mobile  # 用户昵称(默认手机号)
    
    # 明文密码
    # user.password_hash = password  # 加密的密码(明文...)
    
    # 密文密码(@property)
    user.password = password

    user.mobile = mobile  # 手机号
    # 获取登录时间
    user.last_login = datetime.datetime.now()
    # 存进mysql
    try:
        db.session.add(user)
        db.session.commit()
    except Exception as e:
        db.session.rollback()
        logging.error(e)
        return jsonify(errno=response_code.RET.DBERR, errmsg='存储用户信息失败')
    # 状态保持,user.id写入ssession
    session['user_id'] = user.id
    session['nick_name'] = user.nick_name
    session['mobile'] = mobile
    # 响应请求结果
    return jsonify(errno=response_code.RET.OK, errmsg='注册成功')
```

##### 密码加密保存部分(重写属性方法)

1.在模型类中给User类声明⼀个password属性 

2.并实现set和get⽅法 

```python
@property
def password(self):
    """
    password属性的getter方法
    :return: 不让读, 抛异常
    """
    return AttributeError('can not read')

@password.setter
def password(self, value):
    """
    password属性的setter方法
    :param value: 传入的明文密码
    :return: 密文密码
    """
    self.password_hash = generate_password_hash(value)

    
    
    
# 密码校验
def check_password(self, password):
        return check_password_hash(self.password_hash, password)
```

## 登录 

##### 登录后端逻辑 

**注*:  需要保存⽤户登录的时间** 

```python
@passport_blue.route('/login', methods=['post'])
def login():
    # 接收参数
    json_dict = request.json
    mobile = json_dict.get('mobile')
    password = json_dict.get('password')
    # 校验参数
    if not all([mobile, password]):
        return jsonify(errno=response_code.RET.PARAMERR, errmsg='缺少参数')
    # 校验手机号
    if not re.match('^1[345678][0-9]{9}$', mobile):
        return jsonify(errno=response_code.RET.PARAMERR, errmsg='手机号格式错误')
    # 看mysql中有没有这个手机号
    try:
        user = User.query.filter(User.mobile == mobile).first()
    except Exception as e:
        logging.error(e)
        return jsonify(errno=response_code.RET.DBERR, errmsg='用户名或密码错误')
    # 用户是否存在
    if not user:
        return jsonify(errno=response_code.RET.NODATA, errmsg='用户名或密码错误')
    # 校验密码
    if not user.check_password(password):
        return jsonify(errno=response_code.RET.PARAMERR, errmsg='用户名或密码错误')
    # 记录最后登录时间
    user.last_login = datetime.datetime.now()
    # 写入数据库
    try:
        db.session.commit()
    except Exception as e:
        logging.error(e)
        return jsonify(errno=response_code.RET.DATAERR, errmsg='登录失败')
    # 状态保持
    session['user_id'] = user.id
    session['nick_name'] = user.nick_name
    session['mobile'] = user.mobile
    return jsonify(errno=response_code.RET.OK, errmsg='登录成功')
```

##### 退出登录(弹出状态保持)

```python
@passport_blue.route('/logout')
def logout():
    session.pop('user_id', None)
    session.pop('mobile', None)
    session.pop('nick_name', None)
    return jsonify(errno=response_code.RET.OK, errmsg='退出成功')
```