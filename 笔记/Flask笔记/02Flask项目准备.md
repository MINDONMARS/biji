# Flask项目准备

[TOC]

## 立项

1. ##### 装备虚拟环境

   ```python
   mkvirtualenv -p python3 py3_flask
   pip install flask
   ```

2. ##### 使用git管理源代码(项目托管在码云)

3. ##### 创建项目入口文件manage.py

   ```python
   from flask import Flask
   
   
   app = Flask(__name__)
   
   @app.route('/index')
   def index():
   	return 'index'
   
   if __name__ == '__main__':
    app.run()
   ```

4. ##### 设置配置加载配置(配置类)

   ```python
   # app = Flask(__name__)
   
   class Config(object):
    """封装配置的类"""
   	DEBUG = True
   # 加载项⽬配置
   app.config.from_object(Config)
   ```

5. ##### 集成Mysql和Redis

   mysql

   安装flask_sqlalchemy 

   `pip install flask-sqlalchemy `

    `pip install flask-mysqldb `

   ```python
   # 导入
   from flask_sqlalchemy import SQLAlchemy
   
   # 写入配置类
   # mysql连接配置
   SQLALCHEMY_DATABASE_URI = 'mysql://root:mysql@127.0.0.1:3306/information(数据库名)'
   # mysql禁⽌追踪数据库增删改
   SQLALCHEMY_TRACK_MODIFICATIONS = False
   
   # 创建连接到mysql数据库的对象
   db = SQLAlchemy(app)
   ```

   redis

   `pip install redis `

   `启动redis:  sudo redis-server /etc/redis/redis.conf `

   ```python
   # 导入
   from redis import StrictRedis
   
   # redis数据库配置(写入配置类)
   REDIS_HOST = '192.168.73.128'
   REDIS_PORT = 6379
   
   # 创建连接到redis数据库的对象
   redis_store = StrictRedis(host=Config.REDIS_HOST, port=Config.REDIS_PORT)
   
   ```

6. ##### 开启csrf保护

   安装flask_wrf

   ```python
   from flask_wtf.csrf import CSRFProtect
   # ⽣成秘钥SECRET_KEY
   import os, base64
   base64.b64encode(os.urandom(48))
   SECRET_KEY = ''
   # 开启CSRF保护
   CSRFProtect(app)
   # postman测试看是否开启
   # 在每一次相应中, 都写入一个cookie 值为csrf_token
   @app.after_request
   def after_request_set_csrf_token(response):
       """
      	    监听每一个请求之后逻辑的请求勾子
           给每一个响应中，都写入一个cookie,值为csrf_token
           (表单隐藏input或请求头要带上csrf_token)
       """
       # 生成csrf_token
       # generate_csrf() : 生成一个签名后的csrf_token,并写入到session
       csrf_token = csrf.generate_csrf()
       # 将csrf_token写入到cookie
       response.set_cookie('csrf_token', csrf_token)
       return response
   ```

7. ##### session配置(状态保持的session数据写入redis)

   安装flask_session

   `pip install flask_session `

   ```python
   # 导入
   from flask_session import Session
   # 使⽤flask_session扩展存储session到Redis数据库
   Session(app)
   
   # 写入配置类
   SESSION_TYPE = 'redis'
   SESSION_USE_SIGNER = True
   SESSION_REDIS = StrictRedis(host=REDIS_HOST, port=REDIS_PORT)
   PERMANENT_SESSION_LIFETIME = 60 * 60 * 24
   # 测试 
   from flask import session
   # 写入
   session['name'] = 'zxc'
   # 终端查看redis
   ```

8. ##### Flask_script和数据库迁移集成 

   Flask_script集成 

   `pip install flask_script `

   ```python
   from flask_script import Manager
   
   manager = Manager(app)
   
   if __name__ == '__main__':
   	# app.run()
   	manager.run()  # 改用manage
   ```

   数据库迁移集成 

   安装flask_migrate 

   `pip install flask_migrate `

   ```python
   from flask_migrate import Migrate, MigrateCommand
   # 在迁移时让app和db建⽴关联
   Migrate(app, db)
   # 将迁移脚本添加到脚本管理器
   manager.add_command('db', MigrateCommand)
   ```

9. ##### 代码抽取和封装 

   封装代码步骤 

   1.确定哪些代码需要封装 

   2.确定代码要封装到哪⾥ 

   3.将代码移动到指定位置 

   4.检查代码移动后是否有错

   5.如果有错就改错 

   6.改错后，直接运⾏ 

   7.注意点： 

   切记不要⼀次性封装太多代码，⼩步⼩步的封装 

   如果逻辑复杂，逐步的封装 

   **开始:**

   抽取配置到配置⽂件 

   1.在manage.py⽂件同级⽬录新建 config.py ⽂件 

   2.将类 class Config() 拷⻉到 config.py ⽂件 

   ```python
   class Config(object):
       DEBUG = True
       # 配置MySQL:指定数据库位置
       SQLALCHEMY_DATABASE_URI = 'mysql://root:mysql@127.0.0.1:3306/information'
       # 禁用追踪mysql:因为mysql的性能差，如果再去追踪mysql的所有的修改，会再次浪费性能
       SQLALCHEMY_TRACK_MODIFICATIONS = False
   
       REDIS_HOST = '192.168.73.128'
       REDIS_PORT = 6379
   	# 用上面方法生成随机字符串
       SECRET_KEY = ''
       
       SESSION_TYPE = 'redis'
       SESSION_USE_SIGNER = True
       SESSION_REDIS = StrictRedis(host=REDIS_HOST, port=REDIS_PORT)
       PERMANENT_SESSION_LIFETIME = 60 * 60 * 24
   ```

   抽取app到业务逻辑模块 

   1.在项⽬根路径下，创建名为info的包 

   2.将app创建和配置的代码拷⻉到info的包的(--init--.py)⽂件 

   ```python
   from flask import Flask
   from flask_sqlalchemy import SQLAlchemy
   from redis import StrictRedis
   from flask_wtf.csrf import CSRFProtect
   from flask_session import Session
   from config import Config
   app = Flask(__name__)
   # 加载项⽬配置
   app.config.from_object(Config)
   # 配置MYSQL
   db = SQLAlchemy(app)
   # 配置redis
   redis_store = StrictRedis(host=Config.REDIS_HOST, port=Config.REDIS_PORT)
   # 开启CSRF保护
   CSRFProtect(app)
   # 使⽤Session扩展
   Session(app)
   ```

   抽取不同环境的配置 

   ```python
   class Config(object):
       DEBUG = True
       # 配置MySQL:指定数据库位置
       SQLALCHEMY_DATABASE_URI = 'mysql://root:mysql@127.0.0.1:3306/information'
       # 禁用追踪mysql:因为mysql的性能差，如果再去追踪mysql的所有的修改，会再次浪费性能
       SQLALCHEMY_TRACK_MODIFICATIONS = False
   
       REDIS_HOST = '192.168.73.128'
       REDIS_PORT = 6379
   	# 用上面方法生成随机字符串
       SECRET_KEY = ''
       
       SESSION_TYPE = 'redis'
       SESSION_USE_SIGNER = True
       SESSION_REDIS = StrictRedis(host=REDIS_HOST, port=REDIS_PORT)
       PERMANENT_SESSION_LIFETIME = 60 * 60 * 24
       
   class DevelopmentConfig(Config):
    """开发模式下的"""
   	DEBUG = True
   class ProductionConfig(Config):
    """⽣产模式下的"""
   	DEBUG = False
   # ⼯⼚原材料
   configs = {
    'dev':DevelopmentConfig,
    'prod':ProductionConfig
   }
   ```

   ⼯⼚⽅法创建app 

   1.方法

   ```python
   def create_app(config_name):
       """
       根据config_name获取配置类
       :param config_name: 配置类别名
       :return: app
       """
   	app = Flask(__name__)
   	return app
   ```

   2.原料在config.py

   3.成品

   ```python
   def create_app(config_name):
       """
        根据config_name获取配置类
        :param config_name: 配置类别名
        :return: app
       """
       app = Flask(__name__)
       # 根据配置名字，获取配置类
       Config = configs[config_name]
       # 加载项⽬配置
       app.config.from_object(Config)
       ......
       return app
   ```

   4.工厂方法   > > 类⽐路由正则转换器读取的⽅式 

   ```python
   def get_converter(self, variable_name, converter_name, args, kwargs):
       """Looks up the converter for the given parameter.
    .. versionadded:: 0.9
   	"""
       if converter_name not in self.map.converters:
           raise LookupError('the converter %r does not exist' % converter_name)
           return self.map.converters[converter_name](self.map, *args, **kwargs)
       #: the default converter mapping for the map.
       DEFAULT_CONVERTERS = {
           'default': UnicodeConverter,
           'string': UnicodeConverter,
           'any': AnyConverter,
           'path': PathConverter,
           'int': IntegerConverter,
           'float': FloatConverter,
           'uuid': UUIDConverter,
       }
   ```

   全局db的处理 

   ```python
   # 全局db
   db = SQLAlchemy()
   # 全局db
   def create_app(config_name):
       """  根据config_name获取配置类
        :param config_name: 配置类别名
        :return: app
       """
       app = Flask(__name__)
       # 根据配置名字，获取配置类
       Config = configs[config_name]
       # 加载项⽬配置
       app.config.from_object(Config)
       
       # 配置MYSQL
       # db = SQLAlchemy(app)
       db.init_app(app)
   
       ......
       return app
   ```

10. ⽇志 log

    1.概念 

    - ⽇志是⼀种可以追踪某些软件运⾏时所发⽣事件的⽅法 

    - 软件开发⼈员可以向他们的代码中调⽤⽇志记录相关的⽅法来表明发⽣了某些事情

    - ⼀个事件可以⽤⼀个可包含可选变量数据的消息来描述 

    - 事件也有重要性的概念，这个重要性也可以被称为严重性级别（level） 

    2.作⽤ 

    - 1.程序调试 

    - 2.了解软件程序运⾏情况，是否正常 

    - 3.软件程序运⾏故障分析与问题定位 

    - 4.如果应⽤的⽇志信息⾜够详细和丰富，还可以⽤来做⽤户⾏为分析 

    - 5.⽐较print 打印完就没有了，⽆法记录，⽽且频繁的I/O操作浪费服务器资源 

    3.等级 

    - 1.NOTSET = 没有设置 最低等级 

    - 2.DEBUG = 调试 开发环境从这⼀等级开始 

    - 3.INFO = 信息 

    - 4.WARNING = 警告 ⽣产环境从这⼀等级开始 

    - 5.ERROR = 错误 

    - 6.FATAL/CRITICAL = 重⼤的，危险的 最⾼等级

    4.实现 

    python标准库⾥⾯的logging模块 import logging 

    重要: 要在git的忽略文件中写入.idea/ 和 logs/log

    ```python
    # 1.配置不同环境下的⽇志等级(config.py)
    
    class DevelopmentConfig(Config):
        # 开发环境的⽇志等级为调试模式
        LOGGING_LEVEL = logging.DEBUG
    class ProductionConfig(Config):
        # ⽣产环境的⽇志等级为警告
        LOGGING_LEVEL = logging.WARNING
    ```

    ```python
    # info/__init__.py
    def create_app(config_name):
        ......
        # 3调用日志的配置方法
        setup_log(config_class.LOGGING_LEVEL)
    	......
        return app
    
    # 2封装设置⽇志等级的⽅法
    def setup_log(level):
        """
        配置日志等级
        :param level: 日志等级：根据开发环境而变（dev--DEBUG; prod--WARNING）
        :return: None
        """
        # 设置日志的记录等级。
        logging.basicConfig(level=level)  # 调试debug级
        # 创建日志记录器，指明日志保存的路径、每个日志文件的最大大小、保存的日志文件个数上限
        file_log_handler = RotatingFileHandler("logs/log", maxBytes=1024 * 1024 * 100, backupCount=10)
        # 创建日志记录的格式                 日志等级    输入日志信息的文件名 行数    日志信息
        formatter = logging.Formatter('%(levelname)s %(filename)s:%(lineno)d %(message)s')
        # 为刚创建的日志记录器设置日志记录格式
        file_log_handler.setFormatter(formatter)
        # 为全局的日志工具对象（flask app使用的）添加日志记录器
        logging.getLogger().addHandler(file_log_handler)
    
    ```

11. ##### 蓝图

    **提示： 为了避免蓝图在使⽤时的导⼊问题，哪⾥注册蓝图就在哪⾥导⼊蓝图，不要提前导⼊蓝图**

    蓝图模块化管理项⽬ 

    ![1535187639638](C:\Users\ADMINI~1\AppData\Local\Temp\1535187639638.png)

12. ##### 变量注释 

    redis_store = None  # type: StrictRedis 或

    redis_store: StrictRedis = None 

    其他文件导入后使用有智能提示 :-)

