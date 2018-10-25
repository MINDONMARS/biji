# Flask个⼈中⼼ 

[TOC]

##### 0.⼊⼝准备 

	1.个⼈中⼼⽤户信息蓝图准备 

```python
from flask import Blueprint
user_blue = Blueprint('user', __name__, url_prefix='/user')
from . import views
from info.modules.user import user_blue
app.register_blueprint(user_blue)
```

	2.展示user.html模板 (render_template())

```python
@user_blue.route('/info')
@user_login_data
def user_info():
    """⽤户中⼼⽤户信息"""
    user = g.user
    if not user:
        # 如果⽤户未登录，重定向到index蓝图中的index视图
        return redirect(url_for('index.index'))
    context = {
        'user':user.to_dict()
    }
    return render_template('news/user.html', context=context
```

##### 1.基本资料 

	1.准备视图和模板 

```python
@user_blue.route('/base_info', methods=['GET','POST'])
@user_login_data
def base_info():
    """基本资料"""
    # 1.获取user信息
    user = g.user
    # 2.渲染⽤户基本界⾯
    if request.method == 'GET':
        context = {
            'user':user.to_dict()
        }
        return render_template('news/user_base_info.html', context=context)
```

	2.后端逻辑实现 

 	修改⽤户基本资料：签名，昵称，性别 

	接受参数 

	校验参数 

	更新基本资料数据 

	**修改session中的nick_name** **!!!**

	响应更新数据的结果 

```python
# @user_blue.route('/base_info', methods=['get', 'post'])
# @user_login_data
# def base_info():
#    user = g.user
#    if not user:
#        return redirect(url_for('index.index'))
#    if request.method == 'GET':
#        context = {
#            'user': user.to_dict()
#       }
#        return render_template('news/user_base_info.html', context=context)
    
    if request.method == 'POST':
        # 获取参数(nick_name/signature)
        nick_name = request.json.get('nick_name')
        signature = request.json.get('signature')
        gender = request.json.get('gender')
        # 校验
        if not all([nick_name, signature, gender]):
            return jsonify(errno=response_code.RET.PARAMERR, errmsg='缺少参数')
        if gender not in ['MAN', 'WOMAN']:
            return jsonify(errno=response_code.RET.PARAMERR, errmsg='参数错误')

        user.nick_name = nick_name
        user.signature = signature
        user.gender = gender
        try:
            db.session.commit()
        except Exception as e:
            logging.error(e)
            db.session.rollback()
            return jsonify(errno=response_code.RET.DBERR, errmsg='修改失败')
        session['nick_name'] = nick_name
        return jsonify(errno=response_code.RET.OK, errmsg='ok')
```

##### 2.头像设置 

	1.准备视图和模板渲染 

```python
@user_blue.route('/pic_info', methods=['GET','POST'])
@user_login_data
def pic_info():
    """⽤户头像上传"""
    # 1.获取user信息
    user = g.user
    # 2.渲染⽤户头像界⾯
    if request.method == 'GET':
        context = {
            'user': user.to_dict()
        }
        return render_template('news/user_pic_info.html', context=context)
```

	2.七⽜云介绍 :

	官⽹地址 https://www.qiniu.com/ 

	SDK地址 https://developer.qiniu.com/sdk#official-sdk 

	3.封装七⽜云上传⽂件⼯具⽅法 :

		1.在utils包中创建file_storage.py⽂件 

		2.说明 : 

		上传图⽚到七⽜云后，返回图⽚标识和状态码 

		图⽚标识需要存储到数据库，⽤于将来做下载 

		状态码⽤于判断上传是否成功 

		3.封装⼯具⽅法 :

```python
import qiniu
access_key = 'yV4GmNBLOgQK-1Sn3o4jktGLFdFSrlywR2C-hvsW'
secret_key = 'bixMURPL6tHjrb8QKVg2tm7n9k8C7vaOeQ4MEoeW'
bucket_name = 'ihome'
def upload_file(data):
    """
 	上传、存储图⽚到七⽜云
 	:param data: ⽂件⼆进制数据
 	:return: FtEAyyPRhUT8SU3f5DNPeejBjMV5
	"""
    q = qiniu.Auth(access_key, secret_key)
    token = q.upload_token(bucket_name)
    ret, info = qiniu.put_data(token, None, data)
    if info.status_code != 200:
        # 如果上传失败就抛出异常
        raise Exception('七⽜上传图⽚失败')
        # 如果上传成功就返回key
        return ret.get('key')
```

		4.配置七⽜云空间域名 :

		在constants.py⽂件中配置 

		QINIU_DOMIN_PREFIX = "http://oyucyko3w.bkt.clouddn.com/" 

	4.上传头像后端逻辑 

		上传⽤户头像 

		获取上传的图⽚ 

		上传头像 

		保存图⽚上传后的唯⼀标识 

		响应上传结果 

```python
@user_blue.route('/pic_info', methods=['get', 'post'])
@user_login_data
def pic_info():
    user = g.user
    if not user:
        return redirect(url_for('index.index'))
    if request.method == 'GET':
        context = {
            'user': user.to_dict()
        }
        return render_template('news/user_pic_info.html', context=context)
    if request.method == 'POST':
        # 接收参数(文件二进制)
        # 拿到文件对象
        avatar_file = request.files.get('avatar')
        try:
            avatar_data = avatar_file.read()
        except Exception as e:
            logging.error(e)
            return jsonify(errno=response_code.RET.PARAMERR, errmsg='参数错误')
        try:
            key = upload_file(avatar_data)
        except Exception as e:
            logging.error(e)
            return jsonify(errno=response_code.RET.THIRDERR, errmsg='上传文件到七牛云失败')
        user.avatar_url = key
        try:
            db.session.commit()
        except Exception as e:
            db.session.rollback()
            logging.error(e)
            return jsonify(errno=response_code.RET.DBERR, errmsg='存储头像失败')
        data = {
            'avatar_url': constants.QINIU_DOMIN_PREFIX + key
        }
        return jsonify(errno=response_code.RET.OK, errmsg='存储头像成功', data=data)
```

##### 3.密码修改 

	1.准备视图和模板 

```python
@user_blue.route('/pass_info', methods=['GET','POST'])
@user_login_data
def pass_info():
    """密码修改"""
    # 1.渲染密码修改界⾯
    if request.method == 'GET':
        return render_template('news/user_pass_info.html')
```

	2.后端逻辑

```python
# 接上
if request.method == 'POST':
        # 接收参数
        old_password = request.json.get('old_password')
        new_password = request.json.get('new_password')
        # 校验
        if not all([old_password, new_password]):
            return jsonify(errno=response_code.RET.PARAMERR, errmsg='缺少参数')
        if not user.check_password(old_password):
            return jsonify(errno=response_code.RET.PARAMERR, errmsg='旧密码输入错误')
        user.password = new_password
        try:
            db.session.commit()
        except Exception as e:
            logging.error(e)
            db.session.rollback()
            return jsonify(errno=response_code.RET.DBERR, errmsg='修改密码失败')
        return jsonify(errno=response_code.RET.OK, errmsg='修改密码成功')

    context = {
        'user': user
    }
    return render_template('news/user_pass_info.html', context=context)
```

##### 4.我的收藏 

	1.准备视图和模板 

```python
@user_blue.route('/user_collection')
@user_login_data
def user_collection():
    """⽤户收藏的新闻"""
    return render_template('news/user_collection.html', context=context)
```

	2.后端逻辑 (分页查询)

```python
@user_blue.route('/user_collection')
@user_login_data
def user_collection():
    user = g.user
    if not user:
        return redirect(url_for('index.index'))
    page = request.args.get('p', 1)
    try:
        page = int(page)
    except Exception as e:
        logging.error(e)
        page = 1
    user_news_list = []
    current_page = 1
    total_page = 1
    try:
        paginate = user.collection_news.paginate(page, constants.USER_COLLECTION_MAX_NEWS, False)
        user_news_list = paginate.items
        current_page = paginate.page
        total_page = paginate.pages
    except Exception as e:
        logging.error(e)
    news_dict_list = []
    for news in user_news_list:
        news_dict_list.append(news.to_basic_dict())
    context = {
        'news_dict_list': news_dict_list,
        'current_page': current_page,
        'total_page': total_page
    }
    return render_template('news/user_collection.html', context=context)
```

##### 5.新闻发布 

	1.准备新闻发布界⾯ 

```python
@user_blue.route('/news_release', methods=['GET','POST'])
@user_login_data
def news_release():
    """新闻发布"""
    return render_template('news/user_news_release.html')
```

	2.界面展示

```python
@user_blue.route('/news_release', methods=['GET','POST'])
@user_login_data
def user_news_release():
    """新闻发布"""
    # 1.获取登录⽤户信息
    user = g.user
    if not user:
        # 如果⽤户未登录，重定向到index蓝图中的index视图
        return redirect(url_for('index.index'))

    # 2.GET请求逻辑
    if request.method == 'GET':
        # 2.1查询和渲染新闻分类数据
        categories = []
        try:
            categories = Category.query.all()
        except Exception as e:
            current_app.logger.error(e)

        # 2.2去掉'最新'标签
      	categories.pop(0)

        # 2.3构造渲染数据
        context = {
            'categories':categories
           }

                # 2.4渲染界⾯
    	return render_template('news/user_news_release.html', 
                           context=context) 
```

	3.发布新闻后端逻辑 

```python
@user_blue.route('/news_release', methods=['get', 'post'])
@user_login_data
def news_release():
    user = g.user
    if not user:
        return redirect(url_for('index.index'))
    if request.method == 'GET':
        categories = None
        try:
            categories = Category.query.all()
            categories.pop(0)
        except Exception as e:
            logging.error(e)
        context = {
            "categories": categories
        }
        return render_template('news/user_news_release.html', context=context)
    if request.method == 'POST':
        title = request.form.get('title')  # 新闻标题
        source = '个人发布'
        category_id = request.form.get("category_id")
        digest = request.form.get('digest')
        content = request.form.get('content')
        index_image = request.files.get('index_image')
        if not all([title, source, category_id, digest, content, index_image]):
            return jsonify(errno=response_code.RET.PARAMERR, errmsg='缺少参数')
        try:
            index_image_data = index_image.read()
        except Exception as e:
            logging.error(e)
            return jsonify(errno=response_code.RET.PARAMERR, errmsg='参数错误')
        try:
            key = upload_file(index_image_data)
        except Exception as e:
            logging.error(e)
            return jsonify(errno=response_code.RET.THIRDERR, errmsg='上传七牛云失败')
        news = News()
        news.title = title
        news.source = source
        news.category_id = category_id
        news.content = content
        news.digest = digest
        news.index_image_url = constants.QINIU_DOMIN_PREFIX + key
        news.user_id = user.id
        news.status = 1
        try:
            db.session.add(news)
            db.session.commit()
        except Exception as e:
            logging.error(e)
            return jsonify(errno=response_code.RET.DBERR, errmsg='添加新闻到数据库失败')
        return jsonify(errno=response_code.RET.OK, errmsg='添加新闻到数据库成功')
```

##### 6.我发布的新闻 

	后端逻辑 

```python
@user_blue.route('/news_list')
@user_login_data
def news_list():
    user = g.user
    if not user:
        return redirect(url_for('index.index'))
    page = request.args.get('p', 1)
    try:
        page = int(page)
    except Exception as e:
        logging.error(e)
        return jsonify(errno=response_code.RET.PARAMERR, errmsg='参数错误')
    user_news_list = []
    total_page = 1
    current_page = 1
    try:
        paginate = user.news_list.paginate(page, constants.USER_NEWS_PRE_PAGE, False)
        total_page = paginate.pages
        current_page = paginate.page
        user_news_list = paginate.items
    except Exception as e:
        logging.error(e)

    news_dict_list = []
    for news in user_news_list:
        news_dict_list.append(news.to_review_dict())
    context = {
        'news_dict_list': news_dict_list,
        'total_page': total_page,
        'current_page': current_page
    }
    return render_template('news/user_news_list.html', context=context)
```

