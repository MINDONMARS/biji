# Django请求

[TOC]

## request

- url路径参数(路由中正则获取, 视图中接收参数)

  ```python
  # 未命名参数按定义顺序传递
  url(r'^weather/([a-z]+)/(\d{4})/$', views.weather)
  def weather(request, a, b):
      print('city=%s' % a)
      print('year=%s' % b)
      return HttpResponse('OK')
  # 命名参数按名字传递
  url(r'^weather/(?P<city>[a-z]+)/(?P<year>\d{4})/$', views.weather),
  def weather(request, city, year):
      print('city=%s' % city)
      print('year=%s' % year)
      return HttpResponse('OK')
  ```

- QueryDict对象(request.GET/POST.get()/get(list))

  可处理一键多值(getlist())

  get()

- 查询字符串

  ```python
  # /qs/?a=1&b=2&a=3
  request.GET.get('a')  # 得到的是最后的a
  request.GET.getlist('a')  # a列表
  ```

- 请求体(POST / PUT / PATCH / DELETE)

  默认开启csrf会对上述请求方式进行保护

  ```python
  # 表单类型 Form Data
  # 重要：request.POST只能用来获取POST方式的请求体表单数据。
  def get_body(request):
      a = request.POST.get('a')
      b = request.POST.get('b')
      alist = request.POST.getlist('a')
      print(a)
      print(b)
      print(alist)
      return HttpResponse('OK')
  # 非表单类型 Non-Form 非表单类型的请求体数据，Django无法自动解析，可以通过request.body属性获取最原始的请求体数据，自己按照请求体格式（JSON、XML等）进行解析。request.body返回bytes类型。
  # 导入json
  import json
  def get_body_json(request):
      json_bytes = request.body
      json_str = json_bytes.decode()  # python3.6 无需执行此步(代码健壮性统一设置)
      req_data = json.loads(json_str)
      print(req_data['a'])
      print(req_data['b'])
      return HttpResponse('OK')
  ```

- 请求头

  request.META['大写请求头']

  ```python
  def get_headers(request):
      print(request.META['CONTENT_TYPE'])
      return HttpResponse('OK')
  ```

  - `CONTENT_LENGTH` – The length of the request body (as a string).
  - `CONTENT_TYPE` – The MIME type of the request body.
  - `HTTP_ACCEPT` – Acceptable content types for the response.
  - `HTTP_ACCEPT_ENCODING` – Acceptable encodings for the response.
  - `HTTP_ACCEPT_LANGUAGE` – Acceptable languages for the response.
  - `HTTP_HOST` – The HTTP Host header sent by the client.
  - `HTTP_REFERER` – The referring page, if any.
  - `HTTP_USER_AGENT` – The client’s user-agent string.
  - `QUERY_STRING` – The query string, as a single (unparsed) string.
  - `REMOTE_ADDR` – The IP address of the client.
  - `REMOTE_HOST` – The hostname of the client.
  - `REMOTE_USER` – The user authenticated by the Web server, if any.
  - `REQUEST_METHOD` – A string such as `"GET"` or `"POST"`.
  - `SERVER_NAME` – The hostname of the server.
  - `SERVER_PORT` – The port of the server (as a string).

- method  请求方式
- user 请求的用户(没有:匿名用户)
- path 请求的路径
- encoding 提交数据的编码方式(可写)
  - 如果为None则表示使用浏览器的默认设置，一般为utf-8。
  - 这个属性是可写的，可以通过修改它来修改访问表单数据使用的编码，接下来对属性的任何访问将使用新的encoding值。

- FILES：一个类似于字典的对象，包含所有的上传文件 



