# Flask点赞

[TOC]

##### 评论点赞和取消点赞 

后端逻辑 

0.准备后端接⼝ 

1.获取登录⽤户信息：只有登录⽤户才能点赞 

2.接受参数：评论id,点赞事件 

3.校验参数：判断参数是否⻬全，点赞事件是否在范围内 

4.查询被点赞的评论是否存在 

5.点赞或取消点赞 

6.响应结果 

```python
@news_blue.route('/comment_like', methods=['POST'])
@user_login_data
def comment_like():
    """
    1.用户是否登录
    2.comment_id, action
    3.校验参数是否齐全, action是否在范围内(add/remove)
    4.查询被点赞的数据是否存在
    5.根据action的值实现点赞/取消点赞
    6.数据同步到数据库.
    7.返回点赞/取消点赞结果
    """
    user = g.user
    if not user:
        return jsonify(errno=response_code.RET.SESSIONERR, errmsg='用户未登录')
    json_dict = request.json
    comment_id = json_dict.get('comment_id')
    action = json_dict.get('action')
    if not all([comment_id, action]):
        return jsonify(errno=response_code.RET.PARAMERR, errmsg='缺少必传参数')
    if action not in ['add', 'remove']:
        return jsonify(errno=response_code.RET.PARAMERR, errmsg='缺少必传参数')
    try:
        comment = Comment.query.get(comment_id)
    except Exception as e:
        logging.error(e)
        return jsonify(errno=response_code.RET.DBERR, errmsg='查询失败')
    if not comment:
        return jsonify(errno=response_code.RET.NODATA, errmsg='评论不存在')
    comment_like_model = CommentLike.query.filter(CommentLike.user_id == user.id, CommentLike.comment_id == comment.id).first()
    if action == 'add':
        if not comment_like_model:
            comment_like_model = CommentLike()
            comment_like_model.user_id = user.id
            comment_like_model.comment_id = comment.id
            # 点赞数 + 1
            comment.like_count += 1
            db.session.add(comment_like_model)
    else:
        if comment_like_model:
            comment.like_count -= 1
            db.session.delete(comment_like_model)
    # 提交数据库
    try:
        db.session.commit()
    except Exception as e:
        logging.error(e)
        return jsonify(errno=response_code.RET.DBERR, errmsg='操作失败')
    return jsonify(errno=response_code.RET.OK, errmsg='操作成功')
```

##### 展示是否点赞的后端逻辑(模板渲染部分) 

逻辑分析 :

**注意点*** 查询时要求⽤户是登录状态 

```python
comment_dict_list = []
for comment in comments:
    # 给评论字典增加⼀个key,记录该评论是否被该⽤户点赞
    comment_dict = comment.to_dict()
    # 根据is_like判断是否点赞
    comment_dict['is_like'] = False # 默认未点赞
    if comment.id in comment_like_ids:
        comment_dict['is_like'] = True
        comment_dict_list.append(comment_dict)
        context = {
            'comments':comment_dict_list
        }
```

后端逻辑 

```python
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
   # 构造渲染详情页上下文
   context = {
        'comments': comment_dict_list,
    }
```

