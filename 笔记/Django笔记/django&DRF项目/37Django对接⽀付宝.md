# Django对接⽀付宝

[TOC]

## 1.相关⽂档

### 1.⽀付宝开发平台登录

- https://open.alipay.com/platform/home.htm

### 2.沙箱环境

- 是⽀付宝提供给开发者的模拟⽀付的环境

- 跟真实环境是分开的

- 沙箱应⽤：https://docs.open.alipay.com/200/105311

- 沙箱账号：https://openhome.alipay.com/platform/appDaily.htm?tab=account

  ![沙箱环境](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/%E8%BF%9B%E5%85%A5%E6%B2%99%E7%AE%B1%E7%8E%AF%E5%A2%83.png)

### 3.⽀付宝开发者⽂档

- ⽂档主⻚：https://openhome.alipay.com/developmentDocument.htm
- 产品介绍：https://docs.open.alipay.com/270
- 快速接⼊：https://docs.open.alipay.com/270/105899/
- SDK：https://docs.open.alipay.com/270/106291/
  - python对接⽀付宝SDK：https://github.com/fzlee/alipay/blob/master/README.zh-hans.md
  - python对接⽀付宝SDK安装：pip install python-alipay-sdk --upgrade
- API列表：https://docs.open.alipay.com/270/105900/



## 2.电脑网站支付流程

![支付流程](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/%E7%94%B5%E8%84%91%E7%BD%91%E7%AB%99%E6%94%AF%E4%BB%98%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

## 3.接入步骤

1. 创建应用
2. 配置密钥
3. 搭建和配置开发环境
4. 接口调用

#### 1. 生成应用的私钥和公钥

```shell
openssl
OpenSSL> genrsa -out app_private_key.pem 2048  # 私钥RSA2
OpenSSL> rsa -in app_private_key.pem -pubout -out app_public_key.pem # 导出公钥

OpenSSL> exit
```

#### 2. 保存应用私钥文件

在payment应用中新建keys目录，用来保存秘钥文件。

将应用私钥文件app_private_key.pem复制到payment/keys目录下。

#### 3. 查看公钥

```shell
cat app_publict_key.pem
```

将公钥内容复制给支付宝

![配置公钥](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/%E9%85%8D%E7%BD%AE%E5%85%AC%E9%92%A5.png)

#### 4. 保存支付宝公钥

在payment/keys目录下新建alipay_public_key.pem文件，用于保存支付宝的公钥文件。

将支付宝的公钥内容复制到alipay_public_key.pem文件中![支付宝公钥](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC12%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/%E6%94%AF%E4%BB%98%E5%AE%9D%E5%85%AC%E9%92%A5.png)

注意，还需要在公钥文件中补充开始与结束标志

```shell
-----BEGIN PUBLIC KEY-----
此处是公钥内容
-----END PUBLIC KEY-----
```