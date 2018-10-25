# DRF安装配置

[TOC]

##### DRF需要以下依赖：

- Python (2.7, 3.2, 3.3, 3.4, 3.5, 3.6)
- Django (1.10, 1.11, 2.0)

DRF是以Django扩展应用的方式提供的，所以我们可以直接利用已有的Django环境而无需从新创建。（若没有Django环境，需要先创建环境安装Django

## 安装配置

### 1. 安装DRF

```
pip install djangorestframework
```

### 2. 添加rest_framework应用

我们利用在Django框架学习中创建的demo工程，在**settings.py**的**INSTALLED_APPS**中添加'rest_framework'。

```
INSTALLED_APPS = [
    ...
    'rest_framework',
]
```



