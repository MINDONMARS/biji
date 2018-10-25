# Flask新闻详情

[TOC]

##### 新建news模块和蓝图, 在app中注册蓝图

```python
news_blue = Blueprint('news', __name__, url_prefix='/news')
```

## 新闻详情&点击排行(包含评, 是否收藏论, 是否点赞, 是否关注的渲染)

```python
@news_blue.route('/detail/<int:news_id>')
@user_login_data
def news_detail(news_id):
    """
    :param news_id: 新闻id
    :return: 新闻详情
    """
    # 查from info.utils.comment import get_user_info询用户基本信息

    # user = get_user_info()
    user = g.user
    # 查询点击排行信息
    news_clicks = None
    try:
        news_clicks = News.query.order_by(News.clicks.desc()).limit(constants.CLICK_RANK_MAX_NEWS)
    except Exception as e:
        logging.error(e)
    # 查询新闻详情
    news = None
    try:
        news = News.query.get(news_id)
    except Exception as e:
        logging.error(e)
    if not news:
        # 抛出404 对404 统一处理
        abort(404)
    # 重置新闻点击量
    news.clicks += 1
    try:
        db.session.commit()
    except Exception as e:
        logging.error(e)
        db.session.rollback()
    # 判断用户是否收藏过该新闻
    is_collected = False
    if user:
        if news in user.collection_news:
            is_collected = True
    comments = None
    try:
        comments = Comment.query.filter(Comment.news_id == news_id).order_by(Comment.create_time.desc()).all()
    except Exception as e:
        logging.error(e)
    # comment_dict_list = []
    # for comment in comments:
    #     comment_dict_list.append(comment.to_dict())
    # 判断当前用户为哪些评论点了赞
    comment_like_ids = []
    if user:
        try:
            # 拿到当前用户模型对象 点赞的所有 评论的模型对象
            comment_likes = CommentLike.query.filter(CommentLike.user_id == user.id).all()
            # 把 这些评论的id放到列表里
            comment_like_ids = [comment_like.comment_id for comment_like in comment_likes]
        except Exception as e:
            logging.error(e)
    # 准备渲染的数据
    comment_dict_list = []
    for comment in comments:
        comment_dict = comment.to_dict()
        # 给字典加个键值对
        comment_dict['is_like'] = False
        # 如果评论的id在用户点赞的列表里
        if comment.id in comment_like_ids:
            comment_dict['is_like'] = True
        comment_dict_list.append(comment_dict)
    is_followed = False
    if user and news.user:
        if news.user in user.followed:
            is_followed = True
    # 构造渲染|详情页|上下文
    context = {
        'user': user.to_dict() if user else None,
        'news_clicks': news_clicks,
        'news': news.to_dict(),
        'is_collected': is_collected,
        'comments': comment_dict_list,
        'is_followed': is_followed
    }
    return render_template('news/detail.html', context=context)

```

### 装饰器查询登录⽤户信息 

1.问题 获取登录⽤户信息的地⽅很多，如果每次都编写相同的代码，⾮常麻烦 

2.需求 使⽤装饰器的形式提供登录⽤户的信息，哪个视图要⽤就装饰哪个视图

 使用应用上下文g变量存储user对象

3.注意点 : 

装饰器会将被装饰的函数的__name__属性修改成装饰器内层个函数的名字 

如果相同的装饰器装饰同⼀个模块中的多个视图函数时，会出现多个路由对应⼀个视图函数的情况 

所以：在定义视图函数的装饰器时，需要使⽤ @wraps(view_func) 装饰视图函数 

```python
# 装饰器实现查询用户信息
def user_login_data(view_func):
    """
    获取用户基本信息的装饰器
    :param view_func: 被装饰的视图函数
    :return:
    """
    @wraps(view_func)  # 不让装饰器修改被装饰函数的__name__属性
    def wrapper(*args, **kwargs):
        user_id = session.get('user_id', None)
        user = None
        if user_id:
            try:
                user = User.query.get(user_id)
            except Exception as e:
                logging.error(e)
        g.user = user
        return view_func(*args, **kwargs)
    return wrapper
```

## 收藏和取消收藏后端逻辑实现 

只有登录⽤户才可以收藏

```python
@news_blue.route('/news_collect', methods=['post'])
@user_login_data
def news_collect():
    user = g.user
    if not user:
        return jsonify(errno=response_code.RET.SESSIONERR, errmsg='用户未登录')
    # 接收参数
    json_dict = request.json
    news_id = json_dict.get('news_id')
    action = json_dict.get('action')
    # 校验参数
    if not all([news_id, action]):
        return jsonify(errno=response_code.RET.PARAMERR, errmsg='缺少必传参数')
    if action not in ['collect', 'cancel_collect']:
        return jsonify(errno=response_code.RET.PARAMERR, errmsg='参数错误')
    # 查询新闻数据
    try:
        news = News.query.get(news_id)
    except Exception as e:
        logging.error(e)
        return jsonify(errno=response_code.RET.DBERR, errmsg='查询失败')
    # 如果没有新闻
    if not news:
        return jsonify(errno=response_code.RET.NODATA, errmsg='数据不存在')
    # 根据action的值实现收藏/取消收藏
    #  收藏

    if action == 'collect':
        if news not in user.collection_news:
            user.collection_news.append(news)
    # 取消收藏
    else:
        if news in user.collection_news:
            user.collection_news.remove(news)
    # 提交数据到数据库
    try:
        db.session.commit()
    except Exception as e:
        logging.error(e)
        db.session.rollback()
        return jsonify(errno=response_code.RET.DBERR, errmsg="操作失败")
    # 操作成功响应
    return jsonify(errno=response_code.RET.OK, errmsg="操作成功")
```

## 新闻评论 

##### 新闻评论和回复别⼈的评论后端逻辑 

只有登录⽤户才可以评论 

注意点：需要将评论内容响应给⽤户渲染出来(否则刷新就后看不见效果) 

```python
@news_blue.route('/news_comment',  methods=['POST'])
@user_login_data
def news_comment():
    """
    1.判断用户是否登录
    2.接收参数, news_id, comment, parent_id
    3.校验参数: 是否齐全, news_id, parent_id是否为整数
    4.判断新闻是否存在, 存在才可以评论
    5.根据参数创建Comment模型对象, 给属性赋值
    6.同步到数据库,
    7.响应前端, 评论内容也发过去显示在前端页面
    """
    user = g.user
    # 接收参数
    json_dict = request.json
    news_id = json_dict.get('news_id')
    comment_content = json_dict.get('comment')
    parent_id = json_dict.get('parent_id')
    # 校验参数
    if not all([news_id, comment_content]):
        return jsonify(errno=response_code.RET.PARAMERR, errmsg='缺少必传参数')
    # 是否为整数
    try:
        news_id = int(news_id)
        if parent_id:
            parent_id = int(parent_id)
    except Exception as e:
        logging.error(e)
        return jsonify(errno=response_code.RET.PARAMERR, errmsg='参数错误')
    # 数据库是否有该新闻
    try:
        news = News.query.get(news_id)
    except Exception as e:
        logging.error(e)
        return jsonify(errno=response_code.RET.DBERR, errmsg='查询数据失败')
    if not news:
        return jsonify(errno=response_code.RET.NODATA, errmsg='新闻不存在')
    # 创建comment对象
    comment = Comment()
    comment.user_id = user.id
    comment.news_id = news.id
    comment.content = comment_content
    if parent_id:
        comment.parent_id = parent_id
    # 同步数据库
    try:
        db.session.add(comment)
        db.session.commit()
    except Exception as e:
        logging.error(e)
        db.session.rollback()
        return jsonify(errno=response_code.RET.DBERR, errmsg='评论失败')
    return jsonify(errno=response_code.RET.OK, errmsg='评论成功', data=comment.to_dict())
```

## ⽤户关注和取消关注 

1. 详情⻚展示新闻作者 

   ```html
   {% if context.news.author %}
   	<div class="author_card">
           <a href="#" class="author_pic"><img src="{% if context.news.author.avatar_url %}
    {{ context.news.author.avatar_url }}
    {% else %}
    ../../static/news/images/user_pic.png
    {% endif %}" alt="author_pic"></a>
           <a href="#" class="author_name">{{ context.news.author.nick_name }}</a>
           <div class="author_resume">签名：{{ context.news.author.signature }}</div>
           <div class="writings"><span>总篇数</span><b>{{ context.news.author.news_count }}</b></div>
           <div class="follows"><span>粉丝</span><b>{{ context.news.author.followers_count }}</b></div>
           <a href="javascript:;" class="focus fr" data-userid="{{ context.news.author.id }}">关注</a>
           <a href="javascript:;" class="focused fr" data-userid="{{ context.news.author.id }}"><span class="out">已关注</span><span class="over">取消关注</span></a>
    </div>
   {% endif %}
   ```

2. 关注和取消关注实现 

   1.显示逻辑分析 

   ```python
   # 7.关注和取消显示逻辑
   is_followed = False
   # 如果⽤户已登录且该新闻有作者，并且该新闻的作者被该登录⽤户关注
   if user and news.user:
       if news.user in user.followed:
           is_followed = True
   ```

   ```html
   <a href="javascript:;" class="focus fr" data-userid="{{ context.news.author.id }}" style="display: {% if context.is_followed %}none{% else %}block{% endif %}">关注</a>
   <a href="javascript:;" class="focused fr" data-userid="{{ context.news.author.id }}" style="display: {% if context.is_followed %}block{% else %}none{% endif %}"><span class="out">已关注</span><span class="over">取消关注</span></a>
   ```

   2.后端逻辑实现 

   ```python
   @news_blue.route('/followed_user', methods=['POST'])
   @user_login_data
   def followed_user():
       user = g.user
       if not user:
           return jsonify(errno=response_code.RET.SESSIONERR, errmsg='用户未登录')
       user_id = request.json.get('user_id')
       action = request.json.get('action')
       if not all([user_id, action]):
           return jsonify(errno=response_code.RET.PARAMERR, errmsg='参数不全')
       if action not in ['follow', 'unfollow']:
           return jsonify(errno=response_code.RET.PARAMERR, errmsg='参数错误')
       try:
           author = User.query.get(user_id)
       except Exception as e:
           logging.error(e)
           return jsonify(errno=response_code.RET.DBERR, errmsg='查询数据失败')
       if not author:
           return jsonify(errno=response_code.RET.NODATA, errmsg='用户不存在')
       if action == 'follow':
           if author not in user.followed:
               user.followed.append(author)
       else:
           if author in user.followed:
               user.followed.remove(author)
       try:
           db.session.commit()
       except Exception as e:
           db.session.rollback()
           logging.error(e)
           return jsonify(errno=response_code.RET.DBERR, errmsg='操作失败')
       return jsonify(errno=response_code.RET.OK, errmsg='OK')
   ```

3. 个⼈中⼼我的关注 (分页)

   ```python
   @user_blue.route('/user_follow')
   @user_login_data
   def user_follow():
       user = g.user
       if not user:
           return redirect(url_for('index.index'))
       page = request.args.get('p', 1)
       try:
           page = int(page)
       except Exception as e:
           logging.error(e)
           page = 1
       current_page = 1
       total_page = 1
       followed_users = []
       try:
           paginate = user.followed.paginate(page, constants.USER_FOLLOWED_MAX_COUNT, False)
           current_page = paginate.page
           total_page = paginate.pages
           followed_users = paginate.items
       except Exception as e:
           logging.error(e)
       followed_user_dict_list = []
       for followed_user in followed_users:
           followed_user_dict_list.append(followed_user.to_dict())
       context = {
           'users': followed_user_dict_list,
           'current_page': current_page,
           'total_page': total_page
       }
       return render_template('news/user_follow.html', context=context)
   ```
