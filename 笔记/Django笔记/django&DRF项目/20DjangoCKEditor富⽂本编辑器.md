# DjangoCKEditor富⽂本编辑器

[TOC]

## 1.安装包 

```python
pip install django-ckeditor
```

## 2.注册应⽤

```python
INSTALLED_APPS = [
    ......
    'ckeditor', # 富⽂本编辑器
    'ckeditor_uploader', # 富⽂本编辑器上传图⽚模块

    ......
]
```

## 3.加载配置

```python
# 富⽂本编辑器ckeditor配置
CKEDITOR_CONFIGS = {
    'default': {
        'toolbar': 'full', # ⼯具条功能
        'height': 300, # 编辑器⾼度
        # 'width': 300, # 编辑器宽
    },
}
CKEDITOR_UPLOAD_PATH = '' # 上传图⽚保存路径，使⽤了FastDFS，所以此处设为''

```

## 4.添加总路由

```python
# 富⽂本编辑器
url(r'^ckeditor/', include('ckeditor_uploader.urls')),
```

## 5.模型类补充富⽂本字段

```python
from ckeditor.fields import RichTextField
from ckeditor_uploader.fields import RichTextUploadingField


class Goods(BaseModel):
    """
    商品SPU
    """
    ......
    desc_detail = RichTextUploadingField(default='', verbose_name='详细介绍')
    desc_pack = RichTextField(default='', verbose_name='包装信息')
    desc_service = RichTextUploadingField(default='', verbose_name='售后服务')
```

## 6.修复bug适配FastDFS

~/.virtualenvs/meiduo/lib/python3.5/site-packages/ckeditor_uploader/views.py

95 行(2进制上传文件没有后缀)

```python
if len(str(saved_path).split('.')) > 1:
    if(str(saved_path).split('.')[1].lower() != 'gif'):
    self._create_thumbnail_if_needed(backend, saved_path)
```

## 7.迁移 

```
python manage.py makemigrations
python manage.py migrate
```

