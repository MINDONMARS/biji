# DRF视图(Request和Response)

[TOC]

## Request和Response

**request**

- Request对象的数据是⾃动根据前端发送数据的格式进⾏解析之后的结果。	
- 1）.data
  request.data 返回解析之后的请求体数据。类似于Django中标准的request.POST和request.FILES属性，但提供如下特性：
  • 包含了解析之后的⽂件和⾮⽂件数据
  • 包含了对POST、PUT、PATCH请求⽅式解析后的数据
  • 利⽤了REST framework的parsers解析器，不仅⽀持表单类型数据，也⽀持JSON数据
  2）.query_params
  request.query_params与Django标准的request.GET相同，只是更换了更正确的名称**⽽**已。

**response**

**介绍**:

- rest_framework.response.Response
- REST framework提供了⼀个响应类Response，使⽤该类构造响应对象时，响应的具体数据内容会被转换（render渲染）成符合前端需求的类型。

**渲染器**:

- REST framework提供了Renderer 渲染器，⽤来根据请求头中的Accept（接收数据类型声明）来⾃动转换响应数据到对应格式。
- 如果前端请求中未进⾏Accept声明，则会采⽤默认⽅式处理响应数据，我们可以通过配置来修改默认响应格式。

```python
REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': ( # 默认响应渲染类
    'rest_framework.renderers.JSONRenderer', # json渲染器
    'rest_framework.renderers.BrowsableAPIRenderer', # 浏览API渲染器
    )
}
```

	**构造⽅法**

```python
Response(data, status=None, template_name=None, headers=None, content_type=None)
```

	参数说明：
	• data: 为响应准备的序列化处理后的数据；
	• status: 状态码，默认200；
	• template_name: 模板名称，如果使⽤HTMLRenderer 时需指明；
	• headers: ⽤于存放响应头信息的字典；
	• content_type: 响应数据的Content-Type，通常此参数⽆需传递，REST 		framework会根据前端所需类型数据来设置该参数。	

	**属性**

	1）.data:	传给response对象的序列化后，但尚未render处理的数据
	2）.status_code:		状态码的数字
	3）.content		经过render处理后的响应数据

	**状态码**

	为了⽅便设置状态码，REST framewrok在rest_framework.status模块中提供	了常⽤状态码常量。

```
HTTP_200_OK
HTTP_201_CREATED
HTTP_202_ACCEPTED
HTTP_203_NON_AUTHORITATIVE_INFORMATION
HTTP_204_NO_CONTENT
......
```

