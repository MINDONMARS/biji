# Flask后台管理

[TOC]

## 后台⽤户列表 

- 后端业务逻辑 (分页查询)

  ```python
  @admin_blue.route('/user_list')
  def user_list():
      page = request.args.get('p', 1)
      try:
          page = int(page)
      except Exception as e:
          logging.error(e)
          page = 1
      users = []
      total_page = 1
      current_page = 1
      try:
          paginate = User.query.filter(User.is_admin == 0).paginate(page, constants.ADMIN_USER_PAGE_MAX_COUNT, False)
          users = paginate.items
          total_page = paginate.pages
          current_page = paginate.page
      except Exception as e:
          logging.error(e)
      user_dict_list = []
      for user in users:
          user_dict_list.append(user.to_admin_dict())
      context = {
          'users': user_dict_list,
          'total_page': total_page,
          'current_page': current_page
      }
      return render_template('admin/user_list.html', context=context)
  ```

## 后台新闻审核 (分页)

1. 展示待审核新闻列表 

   ```python
   @admin_blue.route('/news_review')
   def news_review():
       page = request.args.get('p', 1)
       try:
           page = int(page)
       except Exception as e:
           logging.error(e)
           page = 1
       news_list = []
       current_page = 1
       total_page = 1
       try:
           paginate = News.query.filter(News.status != 0).order_by(News.create_time.desc()).paginate(page, constants.ADMIN_NEWS_PAGE_MAX_COUNT, False)
           
           current_page = paginate.page
           total_page = paginate.pages
           news_list = paginate.items
           
       except Exception as e:
           logging.error(e)
           
       news_dict_list = []
       for news in news_list:
           news_dict_list.append(news.to_review_dict())
       context = {
           'current_page': current_page,
           'total_page': total_page,
           'news_list': news_dict_list
       }
       return render_template('admin/news_review.html', context=context)
   ```

2. 搜索待审核新闻 

   ```html
   准备搜索表单
   <form class="news_filter_form">
   	<input type="text" name="keyword" placeholder="请输⼊关键字" class="input_txt">
   	<input type="submit" value="搜 索" class="input_sub">
   </form>
   ```

   ```python
   # 接受关键字参数
   keyword = request.args.get('keyword', None)
   
   ####################################################################
   try:
       if keyword:
           paginate = News.query.filter(News.status!=0,News.title.contains(keyword)).order_by(News.create_time.desc()).paginate(page,constants.ADMIN_NEWS_PAGE_MAX_COUNT,False)
       else:
           paginate = News.query.filter(News.status!=0).order_by(News.create_time.desc()).paginate(page, constants.ADMIN_NEWS_PAGE_MAX_COUNT, False)
           news_list = paginate.items
           total_page = paginate.pages
           current_page = paginate.page
   except Exception as e:
       current_app.logger.error(e)
   ```

3. 展示待审核新闻详情 

   1.定义视图函数 

   ```python
   @admin_blue.route('/news_review_detail/<int:news_id>')
   def news_review_detail(news_id):
       """待审核新闻详情"""
   
       # 响应待审核新闻详情界⾯
       return render_template('admin/news_review_detail.html')
   ```

   2.点击跳转到待审核新闻详情  

   ```html
   <a href="{{ url_for('admin.news_review_detail', news_id=news.id) }}" class="review">审核</a>
   ```

   3.查询待审核新闻详情 

   ```python
   # 查询新闻详情信息
   news = None
   try:
       news = News.query.get(news_id)
   except Exception as e:
       logging.error(e)
       abort(404)
   if not news:
       abort(404)
   context = {
    'news': news.to_dict()
   }
   # 响应待审核新闻详情界⾯
   return render_template('admin/news_review_detail.html', context=context)
   ```

   4.渲染 

4. 后台新闻审核实现 

   后端逻辑 

   ```python
   @admin_blue.route('/news_review_action', methods=['post'])
   def news_review_action():
       news_id = request.json.get('news_id')
       action = request.json.get('action')
       if not all([news_id, action]):
           return jsonify(errno=response_code.RET.PARAMERR, errmsg='参数不全')
       if action not in ['accept', 'reject']:
           return jsonify(errno=response_code.RET.PARAMERR, errmsg='参数错误')
       try:
           news = News.query.get(news_id)
       except Exception as e:
           logging.error(e)
           return jsonify(errno=response_code.RET.DBERR, errmsg='查询新闻失败')
       if not news:
           return jsonify(errno=response_code.RET.NODATA, errmsg='新闻不存在')
       if action == 'accept':
           news.status = 0
       else:
           reason = request.json.get('reason')
           if not reason:
               return jsonify(errno=response_code.RET.PARAMERR, errmsg='参数不全')
           news.status = -1
       try:
           db.session.commit()
       except Exception as e:
           logging.error(e)
           return jsonify(errno=response_code.RET.DBERR, errmsg='同步数据库失败')
       return jsonify(errno=response_code.RET.OK, errmsg='OK')
   ```

5. 测试 

   1.先修改前台主⻚新闻列表查询语句 

   ```python
   try:
       if cid == 1:
           # 查询'最新'分类的新闻
           paginate = News.query.filter(News.status==0).order_by(News.create_time.desc()).paginate(page, per_page, False)
       else:
    # 查询指定分类的新闻
   		paginate = News.query.filter(News.status==0,News.category_id==cid).order_by(News.create_time.desc()).paginate(page, per_page, False)
       
   except Exception as e:
       current_app.logger.error(e)
   	return jsonify(errno=response_code.RET.DBERR, errmsg='查询新闻数据失败')
   ```

   2.审核之后，刷新主⻚新闻列表 

## 后台新闻板式编辑 

1. 展示新闻板式编辑列表 

   提示： 这⾥的查询条件是通过审核的新闻 

   后端逻辑 :   

   ```python
   @admin_blue.route('/news_edit')
   def news_edit():
       page = request.args.get('p', 1)
       try:
           page = int(page)
       except Exception as e:
           logging.error(e)
           page = 1
       current_page = 1
       total_page = 1
       news_list = []
       try:
           paginate = News.query.filter(News.status == 0).order_by(News.create_time.desc()).paginate(page, constants.ADMIN_NEWS_PAGE_MAX_COUNT, False)
           current_page = paginate.page
           total_page = paginate.pages
           news_list = paginate.items
       except Exception as e:
           logging.error(e)
           abort(404)
       news_dict_list = []
       for news in news_list:
           news_dict_list.append(news.to_basic_dict())
       context = {
           'current_page': current_page,
           'total_page': total_page,
           'news_list': news_dict_list
       }
   
   
       return render_template('admin/news_edit.html', context=context)
   ```

2. 搜索板式编辑列表新闻 

   1.准备搜索表单 

   ```html
   <form class="news_filter_form">
       <input type="text" name="keyword" placeholder="请输⼊关键字" class="input_txt">
       <input type="submit" value="搜 索" class="input_sub">
   </form>
   ```

   2.接受关键字参数 

   ```python
   keyword = request.args.get('keyword', None)
   ```

   3.调整查询语句 

   ```python
   try:
       if keyword:
           paginate = News.query.filter(News.status == 0, News.title.contains(keyword)).order_by(News.create_time.desc()).paginate(page, constants.ADMIN_NEWS_PAGE_MAX_COUNT, False)
       else:
           paginate = News.query.filter(News.status == 0).order_by(News.create_time.desc()).paginate(page, constants.ADMIN_NEWS_PAGE_MAX_COUNT, False)
           news_list = paginate.items
           total_page = paginate.pages
           current_page = paginate.page
   except Exception as e:
       logging.error(e)
   ```

3. 展示新闻板式编辑详情 

   1.定义视图函数 

   2.点击跳转到新闻板式编辑详情 

   3.查询新闻板式编辑详情 

   4.渲染 

4. 后台新闻板式编辑实现 

   ```python
   @admin_blue.route('/news_edit_detail/<int:news_id>', methods=['GET', 'POST'])
   def news_edit_detail(news_id):
       if request.method == 'GET':
           news = None
           categories = []
           try:
               news = News.query.get(news_id)
               categories = Category.query.all()
               categories.pop(0)
           except Exception as e:
               logging.error(e)
               abort(404)
           if not news:
               abort(404)
   
           context = {
               'news': news.to_dict(),
               'categories': categories
           }
           return render_template('admin/news_edit_detail.html', context=context)
       
       if request.method == 'POST':
           # 接受参数
           title = request.form.get("title")
           digest = request.form.get("digest")
           content = request.form.get("content")
           index_image = request.files.get("index_image")
           category_id = request.form.get("category_id")
           # 校验参数
           if not all([title, digest, content, category_id]):
               return jsonify(errno=response_code.RET.PARAMERR, errmsg='缺少参数')
           try:
               news = News.query.get(news_id)
           except Exception as e:
               logging.error(e)
               return jsonify(errno=response_code.RET.DBERR, errmsg='查询数据库失败')
           if not news:
               return jsonify(errno=response_code.RET.NODATA, errmsg='新闻不存在')
           # 读取和上传图⽚
           if index_image:
               try:
                   index_image_data = index_image.read()
               except Exception as e:
                   logging.error(e)
                   return jsonify(errno=response_code.RET.PARAMERR, errmsg='图片参数错误')
               # 将标题图⽚上传到七⽜
               try:
                   key = upload_file(index_image_data)
               except Exception as e:
                   logging.error(e)
                   return jsonify(errno=response_code.RET.THIRDERR, errmsg='上传图片到七牛云失败')
               news.index_image_url = constants.QINIU_DOMIN_PREFIX + key
           # 保存数据并同步到数据库
           news.title = title
           news.digest = digest
           news.content = content
           news.category_id = category_id
           try:
               db.session.commit()
           except Exception as e:
               logging.error(e)
               db.session.rollback()
               return jsonify(errno=response_code.RET.DBERR, errmsg='同步到数据库失败')
           return jsonify(errno=response_code.RET.OK, errmsg='ok')
   ```

## 后台新闻分类管理 

1. 展示新闻分类

2. 修改和新增新闻分类 

   ```python
   # 此处为展示新闻分类
   @admin_blue.route('/news_type', methods=['GET', 'POST'])
   def news_type():
       if request.method == 'GET':
           categories = []
           try:
               categories = Category.query.all()
           except Exception as e:
               logging.error(e)
               abort(404)
           categories.pop(0)
   
           context = {
               'categories': categories
           }
           return render_template('admin/news_type.html', context=context)
       
       # 此处修改和新增新闻分类
       if request.method == 'POST':
           cid = request.json.get('id')
           cname = request.json.get('name')
           if not cname:
               return jsonify(errno=response_code.RET.PARAMERR, errmsg='缺少参数')
           if not cid:
               category = Category()
               category.name = cname
               db.session.add(category)
           else:
               try:
                   categories = Category.query.get(cid)
               except Exception as e:
                   logging.error(e)
                   return jsonify(errno=response_code.RET.DBERR, errmsg='查询数据库失败')
               if not categories:
                   return jsonify(errno=response_code.RET.NODATA, errmsg='分类不存在')
               categories.name = cname
           try:
               db.session.commit()
           except Exception as e:
               logging.error(e)
               db.session.rollback()
               return jsonify(errno=response_code.RET.DBERR, errmsg='同步数据库失败')
           return jsonify(errno=response_code.RET.OK, errmsg='OK')
   ```
