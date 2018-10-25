# Django中间件

[TOC]

类似flask中的请求钩子

- Django中的中间件是一个轻量级、底层的插件系统，可以<!--介入-->Django的请求和响应处理过程，<!--修改-->Django的<!--输入或输出-->。中间件的设计为开发者提供了一种<!--无侵入式-->的开发方式，<!--增强了Django框架的健壮性-->。 

- 在工程中定义一个中间件工厂函数,返回一个可以调用的中间件(装饰器)

  中间件工厂函数需要接收一个可以调用的get_response对象。

  返回的中间件也是一个可以被调用的对象，并且像视图一样需要接收一个request对象参数，返回一个response对象。

  ```python
  # 要在setting中注册中间件(跟位置中间件文件路径有关)
  """
  定义在应用外层,在MIDDLEWARE中添加如下:
  MIDDLEWARE = [
      'middleware.my_middle1',
      'middleware.my_middle2'
  ]
  
  定义在应用中,在MIDDLEWARE中添加如下:
  MIDDLEWARE = [,
      'users.middleware.my_middleware',  # 添加中间件
  ]
  """
  def my_middle1(get_response):
      # print('__init__')
  
      def middle(request):
          print(111111)
          response = get_response(request)
          print(111111)
          return response
      return middle
  
  
  def my_middle2(get_response):
  
      def middle(request):
          print(222222)
          response = get_response(request)
          print(222222)
          return response
      return middle
  ```
