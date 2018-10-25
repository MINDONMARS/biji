# Flask后台管理 (登录&首页)

[TOC]

## 1.创建管理员账户 

- 本质是创建⼀个新的⽤户，is_admin属性为True 

```python
@manager.option('-u', '-username', dest='username')
@manager.option('-p', '-password', dest='password')
@manager.option('-m', '-mobile', dest='mobile')
def createsuperuser(username, password, mobile):
    """
     创建管理员的函数
     :param name: 管理员名字
     :param password: 管理员密码
    """
    if not all([username, password, mobile]):
        print('缺少参数')
    else:
        user = User()
        user.nick_name = username
        user.password = password
        user.mobile = mobile
        user.is_admin = True
    try:
        db.session.add(user)
        db.session.commit()
    except Exception as e:
        db.session.rollback()
        print(e)

```

## 2.登录管理员账户 

   0.准备后台管理模块 

	新建admin模块, 导入蓝图, 注册蓝图

1. 准备登录界⾯ 

2. 实现登录逻辑 

3. 站点主⻚展示 

   ```python
   @admin_blue.route('/')
   @user_login_data
   def admin_index():
       user = g.user
       if not user:
           return redirect(url_for('admin.admin_login'))
       context = {
           'user': user.to_dict()
       }
       return render_template('admin/index.html', context=context)
   ```

4. 已登录⽤户直接进⼊主⻚ 

5. 管理员身份登录到新经资讯前台 

6. 优化退出登录逻辑 

   ```python
   @admin_blue.route('/admin_login', methods=['GET', 'POST'])
   def admin_login():
       if request.method == 'GET':
           # 已登录⽤户直接进⼊主⻚ 
           is_admin = session.get('is_admin', False)
           user_id = session.get('user_id', None)
           if user_id and is_admin:
               return redirect(url_for('admin.admin_index'))
   
           return render_template('admin/login.html')
   
       if request.method == 'POST':
           username = request.form.get('username')
           password = request.form.get('password')
           if not all([username, password]):
               return render_template('admin/login.html', errmsg='缺少参数')
           try:
               user = User.query.filter(User.nick_name == username).first()
           except Exception as e:
               logging.error(e)
               return render_template('admin/login.html', errmsg='查询用户失败')
           if not user:
               return render_template('admin/login.html', errmsg='用户不存在')
           if not user.check_password(password):
               return render_template('admin/login.html', errmsg='用户名或密码错误')
           # 管理员身份登录到新经资讯前台
           session['user_id'] = user.id
           session['mobile'] = user.mobile
           session['nick_name'] = user.nick_name
           session['is_admin'] = user.is_admin
           return redirect(url_for('admin.admin_index'))
   
   ```

7. 优化退出登录逻辑 

   ```python
   session.pop('user_id',None)
   session.pop('mobile', None)
   session.pop('nick_name', None)
   # 为了防⽌admin登录到新经资讯后，留下管理员的权限，退出登录时需要将此标记也清除
   session.pop('is_admin', False)
   ```

## 3.管理后台访问权限 

1. 需求 : (1)管理员⽤户可以进⼊到所有界⾯ 

   	   (2)⾮管理员⽤户只能进⼊到后台的登录界⾯ (<u>如果⾮管理员访问的是⾮登录界⾯，直接引导到新经资讯前台主⻚</u> )

2. 实现 

   路径:

   ```
    admin.__init__.py
   ```

   ```python
   @admin_blue.before_request
   def admin_authentication():
       if not request.url.endswith('/admin/login'):
           user_id = session['user_id']
           is_admin = session['is_admin']
           if not user_id or not is_admin:
               return redirect(url_for('index.index'))
   ```

## 4.添加测试数据 

运行:

```python
import datetime, random
from info.models import User
from manager import app
from info import db


def add_test_users():
    users = []
    now = datetime.datetime.now()
    for num in range(0, 10000):
        try:
            user = User()
            user.nick_name = "%011d" % num
            user.mobile = "%011d" % num
            user.password_hash = "pbkdf2:sha256:50000$SgZPAbEj$a253b9220b7a916e03bf27119d401c48ff4a1c81d7e00644e0aaf6f3a8c55829"
            user.last_login = now - datetime.timedelta(seconds=random.randint(0, 2678400))
            users.append(user)
            print(user.mobile)
        except Exception as e:
            print(e)
    with app.app_context():
        db.session.add_all(users)
        db.session.commit()
    print ('OK')

if __name__ == '__main__':
    add_test_users()
```

## 5.⽤户统计 

1. ⽤户总⼈数 

   ```python
   try:
       total_count = User.query.filter(User.is_admin==False).count()
   except Exception as e:
       current_app.logger.error(e)
   ```

2. ⽤户⽉新增⼈数 

   ```python
    # 月新增
       t = time.localtime()
       month_begin_str = "%d-%02d-1" % (t.tm_year, t.tm_mon)
       month_begin_date = datetime.datetime.strptime(month_begin_str, "%Y-%m-%d")
       try:
           month_count = User.query.filter(User.create_time >= month_begin_date, User.is_admin == 0).count()
       except Exception as e:
           logging.error(e)
   ```

3. ⽤户⽇新增⼈数 

   ```python
   # 日新增
       t = time.localtime()
       day_begin_str = "%d-%02d-%02d" % (t.tm_year, t.tm_mon, t.tm_mday)
       day_begin_date = datetime.datetime.strptime(day_begin_str, "%Y-%m-%d")
       try:
           day_count = User.query.filter(User.is_admin == 0, User.create_time >= day_begin_date).count()
       except Exception as e:
           logging.error(e)
       today_begin = "%d-%02d-%02d" % (t.tm_year, t.tm_mon, t.tm_mday)
       today_begin_date = datetime.datetime.strptime(today_begin, "%Y-%m-%d")
   ```

4. 渲染

   ```python
   context = {
           'total_count': total_count,
           'month_count': month_count,
           'day_count': day_count,
       }
   return render_template('admin/user_count.html', context=context)
   ```


## 6.活跃⽤户数量 

```python
# 日活
    active_date = []
    active_count = []
    for i in range(15):
        # 计算一天开始
        begin_date = today_begin_date - datetime.timedelta(days=i)
        # 计算一天结束
        end_date = today_begin_date - datetime.timedelta(days=(i - 1))

        # 查询每一天的日活跃量
        count = User.query.filter(User.is_admin == 0, User.last_login >= begin_date,
                                  User.last_login < end_date).count()

        active_count.append(count)
        # 需要将datetime类型的begin_date，转成时间字符串，因为前段的折线图框架需要的是字符串
        active_date.append(datetime.datetime.strftime(begin_date, '%Y-%m-%d'))

    # 为了保证今天在X轴的最左侧，需要反转列表
    active_date.reverse()
    active_count.reverse()
```

