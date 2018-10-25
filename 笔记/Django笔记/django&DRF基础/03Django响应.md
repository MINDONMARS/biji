# Django响应

[TOC]

## HttpResponse

```python
# 可以使用django.http.HttpResponse来构造响应对象。
HttpResponse(content=响应体, content_type=响应体数据类型, status=状态码)

# 也可通过HttpResponse对象属性来设置响应体、状态码：
from django.http import HttpResponse

def demo_view(request):
    return HttpResponse('itcast python', status=400)
    # 或者
    response = HttpResponse('itcast python')
    response.status_code = 400
    response['Itcast'] = 'Python'
    return response
```

## HttpResponse子类(快速设置状态码 )

- HttpResponseRedirect 301
- HttpResponsePermanentRedirect 302
- HttpResponseNotModified 304
- HttpResponseBadRequest 400
- HttpResponseNotFound 404
- HttpResponseForbidden 403
- HttpResponseNotAllowed 405
- HttpResponseGone 410
- HttpResponseServerError 500

## JsonResponse

若要返回json数据，可以使用JsonResponse来构造响应对象

作用： 

- 帮助我们将数据转换为json字符串
- 设置响应头**Content-Type**为 **application/json**

```python
def demo_view(request):
    return JsonResponse({'city': 'beijing', 'subject': 'python'})
def demo_view(request):
    json = {'city': 'beijing', 'subject': 'python'}
    return JsonResponse(json)
```

## redirect重定向

```python
from django.shortcuts import redirect

def demo_view(request):
    return redirect('/index.html')
```

## Cookie

- Cookie以键值对的格式进行信息的存储。
- Cookie基于域名安全，不同域名的Cookie是不能互相访问的，如访问itcast.cn时向浏览器中写了Cookie信息，使用同一浏览器访问baidu.com时，无法访问到itcast.cn写的Cookie信息。
- 当浏览器请求某网站时，会将浏览器存储的跟网站相关的所有Cookie信息提交给网站服务器。

可以通过**HttpResponse**对象中的**set_cookie**方法来设置cookie。 

```python
# 设置
# max_age 单位为秒，默认为None。如果是临时cookie，可将max_age设置为None。
HttpResponse.set_cookie(cookie名, value=cookie值, max_age=cookie有效期)
# 例:
def demo_view(request):
    response = HttpResponse('ok')
    response.set_cookie('itcast1', 'python1')  # 临时cookie
    response.set_cookie('itcast2', 'python2', max_age=3600)  # 有效期一小时
    return response
# 读取
# 可以通过HttpRequest对象的COOKIES属性来读取本次请求携带的cookie值。request.COOKIES为字典类型。
def demo_view(request):
    cookie1 = request.COOKIES.get('itcast1')
    print(cookie1)
    return HttpResponse('OK')
```

## Session

通过HttpRequest对象的session属性进行会话的读写操作。

1） 以键值对的格式写session。

```python
request.session['键']=值
```

2）根据键读取值。

```python
request.session.get('键',默认值)
```

3）清除所有session，在存储中删除值部分。

```python
request.session.clear()
```

4）清除session数据，在存储中删除session的整条数据。

```python
request.session.flush()
```

5）删除session中的指定键及值，在存储中只删除某个键及对应的值。

```python
del request.session['键']
```

6）设置session的有效期

```python
request.session.set_expiry(value)
```

- 如果value是一个整数，session将在value秒没有活动后过期。
- 如果value为0，那么用户session的Cookie将在用户的浏览器关闭时过期。
- 如果value为None，那么session有效期将采用系统默认值，**默认为两周**，可以通过在settings.py中设置**SESSION_COOKIE_AGE**来设置全局默认值。