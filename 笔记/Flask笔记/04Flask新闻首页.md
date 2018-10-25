# Flask新闻首页

[TOC]

## 首页新闻分类点击排行右上角用户信息

##### 后端逻辑 

⾃定义过滤器实现点击排⾏特殊序号展示 (info.utils.comment.py(公用) )

```python
def do_rank(index):
    """
    自定义过滤器: 根据index返回first, second, third, ''
    :return:first, second, third
    """
    if index == 1:
        return 'first'
    elif index == 2:
        return 'second'
    elif index == 3:
        return 'third'
    else:
        return ''
```

app中注册

```python
# 添加⾃定义的过滤器
from info.utils.comment import do_rank
app.add_template_filter(do_rank, 'rank')
```

```python
@index_blue.route('/')
@user_login_data
def index():
    """
    添加点击排行(右侧1-6)
    """
    # 改用装饰器
    # 从session查询user_id
    # from info.utils.comment import get_user_info
    # user = get_user_info()
    user = g.user
    # 点击排行
    news_clicks = None
    categories = None
    try:
        news_clicks = News.query.order_by(News.clicks.desc()).limit(constants.CLICK_RANK_MAX_NEWS)
        # 新闻分类
        categories = Category.query.all()
    except Exception as e:
        logging.error(e)
    # 构造渲染模板的上下文
    context = {
        'user': user.to_dict() if user else None,
        'news_clicks': news_clicks,
        'categories': categories
    }

    return render_template('news/index.html', context=context, categories=categories)
```

##### ⾸⻚新闻列表数据展示后端逻辑 (分页查询)(上拉刷新:前端根据当前页和总页的关系)

(<=就刷新, 不=刷不到最后一页)

```python
@index_blue.route('/news_list')
def index_news_list():
    """
    1. 接收参数: 当前页, 每页条数, 当前分类id
    2. 校验参数是否齐全
    3. 校验参数是否为整数
    4. 根据参数分页查询
    5. 生成响应
    6. 返回响应
    """
    # 1
    cid = request.args.get('cid', '1')
    page = request.args.get('page', '1')
    per_page = request.args.get('per_page', '10')
    # 2
    if not all([cid, page, per_page]):
        return jsonify(errno=response_code.RET.PARAMERR, errmsg='缺少参数')
    # 3
    try:
        cid = int(cid)
        page = int(page)
        per_page = int(per_page)
    except Exception as e:
        logging.error(e)
        return jsonify(errno=response_code.RET.PARAMERR, errmsg='参数错误')
    # 4. 根据参数分页查询
    try:
        if cid == 1:
            # 最新分类, 查询所有新闻按时间倒序排序
            paginate = News.query.filter(News.status == 0).order_by(News.create_time.desc()).paginate(page, per_page, False)
        else:
            paginate = News.query.filter(News.category_id == cid, News.status == 0).order_by(News.create_time.desc()).paginate(page, per_page, False)
    except Exception as e:
        logging.error(e)
        return jsonify(errno=response_code.RET.DBERR, errmsg='查询数据失败')
    # 生成响应数据
    total_page = paginate.pages  # 总页数
    current_page = paginate.page  # 当前页
    news_list = paginate.items  # 当前页的数据列表
    news_dict_list = []
    for news in news_list:
        news_dict_list.append(news.to_basic_dict())
    # 返回响应数据
    return jsonify(errno=response_code.RET.OK, errmsg='OK', cid=cid, current_page=current_page, total_page=total_page,
                   news_dict_list=news_dict_list)
```

