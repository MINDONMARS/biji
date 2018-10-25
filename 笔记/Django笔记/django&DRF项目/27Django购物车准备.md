# Django购物车准备

[TOC]

## 1.购物⻋业务需求说明

1. 在⽤户登录与未登录状态下，都可以保存⽤户的购物⻋数据
2. ⽤户可以对购物⻋数据进⾏增、删、改、查
3. ⽤户对于购物⻋数据的勾选也要保存，在订单结算⻚⾯会使⽤勾选数据
4. ⽤户登录时，合并cookie中的购物⻋数据到redis中
5. 技术实现
   1. 对于未登录的⽤户，购物⻋数据使⽤浏览器cookie缓存
   2. .对于已登录的⽤户，购物⻋数据在后端使⽤Redis缓存

## 2.购物⻋数据存储设计

### 1.⽤户已登录操作Redis数据

![1536978671225](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1536978671225.png)

添加redis配置

```python
"cart": {
    "BACKEND": "django_redis.cache.RedisCache",
    "LOCATION": "redis://192.168.103.132:6379/4",
    "OPTIONS": {
        "CLIENT_CLASS": "django_redis.client.DefaultClient",
    }
}
```

### 2.⽤户未登录操作Cookie数据

```python
{
    sku_id1: {
   		"count": 10, // 数量
    	"selected": True // 是否勾选
    },
    sku_id2: {
    	"count": 20,
    	"selected": False
    },
    ...
}
```

## 3.pickle模块与base64模块的介绍

在cookie中只能保存字符串数据，所以将上述数据使用pickle进行序列化转换 ，并使用base64编码为字符串，保存到cookie中。

### 1.pickle模块

#### 1.概念介绍

- pickle模块是python的标准模块，提供了对于python数据的序列化操作，可以将数据转换为bytes类型，其序列化速度⽐json模块要⾼。
- pickle.dumps() 将python数据序列化为bytes类型
- pickle.loads() 将bytes类型数据反序列化为python的数据类型

#### 2.代码演练

```python
>>> import pickle
>>> d = {'1': {'count': 10, 'selected': True}, '2': {'count': 20, 'selected': False}}
>>> s = pickle.dumps(d)
>>> s
b'\x80\x03}q\x00(X\x01\x00\x00\x001q\x01}q\x02(X\x05\x00\x00\x00countq\x03K\nX\x08\x00\x00\x00selectedq\x04\x88uX\x01\x00\x00\x002q\x05}q\x06(h\x03K\x14h\x04\x89uu.'
>>> pickle.loads(s)
{'1': {'count': 10, 'selected': True}, '2': {'count': 20, 'selected': False}}
```

### 2.base64模块

#### 1.概念介绍

- python标准库中提供了base64模块
- base64.b64encode() 将bytes类型数据进⾏base64编码，返回编码后的bytes类型字符串
- base64.b64deocde() 将base64编码的bytes类型进⾏解码，返回解码后的bytes类型

#### 2.代码演练

```python
>>> import base64
>>> s
b'\x80\x03}q\x00(X\x01\x00\x00\x001q\x01}q\x02(X\x05\x00\x00\x00countq\x03K\nX\x08\x00\x00\x00selectedq\x04\x88uX\x01\x00\x00\x002q\x05}q\x06(h\x03K\x14h\x04\x89uu.'
>>> b = base64.b64encode(s)
>>> b
b'gAN9cQAoWAEAAAAxcQF9cQIoWAUAAABjb3VudHEDSwpYCAAAAHNlbGVjdGVkcQSIdVgBAAAAMnEFfXEGKGgDSxRoBIl1dS4='
>>> base64.b64decode(b)
b'\x80\x03}q\x00(X\x01\x00\x00\x001q\x01}q\x02(X\x05\x00\x00\x00countq\x03K\nX\x08\x00\x00\x00selectedq\x04\x88uX\x01\x00\x00\x002q\x05}q\x06(h\x03K\x14h\x04\x89uu.'
```

