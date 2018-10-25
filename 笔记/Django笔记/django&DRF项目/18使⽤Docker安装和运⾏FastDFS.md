# 使⽤Docker安装和运⾏FastDFS

[TOC]

## 1.获取fastfds镜像

```
sudo docker image pull delron/fastdfs
sudo docker load -i ⽂件路径/fastdfs_docker.tar
```

## 2.运⾏tracker

```
sudo docker run -dti --network=host --name tracker -v /var/fdfs/tracker:/var/fdfs delron/fastdfs tracker
sudo docker container start tracker
```

## 3.运⾏storage

```
sudo docker run -dti --network=host --name storage -e TRACKER_SERVER=192.168.73.129:22122 -v /var/fdfs/storage:/var/fdfs delron/fastdfs storage
sudo docker container start storage
```

## 4.运⾏FastDFS客户端

### 1.安装依赖包

进⼊虚拟环境：`pip install fdfs_client-py-master.zip`
源码下载地址 https://github.com/jefforeilly/fdfs_client-py
如果不使⽤源码安装，有可能安装的不全
如果需要还要安装 pip install mutagen
`pip isntall requests`

### 2.项⽬中准备客户端配置⽂件

```
client.conf ⽂件在项⽬路径：'meiduo_mall/utils/fastdfs/client.conf
base_path=FastDFS客户端存放⽇志⽂件的⽬录
tracker_server=运⾏tracker服务的机器ip:22122
```

### 3.终端shell演练

```python
# 开启shell python manage.py shell
# 运⾏
In [1]: from fdfs_client.client import Fdfs_client
In [2]: client = Fdfs_client('meiduo_mall/utils/fastdfs/client.conf')
In [3]: ret = client.upload_by_filename('/Users/zhangjie/Desktop/01.jpeg')
getting connection

# 结果
In [4]: ret
Out[4]:
{'Group name': 'group1',
'Remote file_id': 'group1/M00/00/00/wKhnhluSuYeAIc39AAC4j90Tziw56.jpeg',
'Status': 'Upload successed.',
'Local file name': '/Users/zhangjie/Desktop/01.jpeg',
'Uploaded size': '46.00KB',
'Storage IP': '192.168.103.132'}
```

### 4.下载

拼接Ngix路径

例如:http://192.168.103.132:8888/group1/M00/00/00/wKhnhluSuYeAIc39AAC4j90Tziw56.jpeg

### 5.可能会出现的错误

解决⽅案-->减少tracker预留空间

```
sudo docker exec -it tracker /bin/bash
cd /etc/fdfs/
vi tracker.conf  减少tracker预留空间  :wp保存退出
exit
sudo service docker restart
容器ID 重启tracker和storage(pid会冲突)
重启storage前要删除 cd /var/fdfs/storage/data/...pid
rm -rf ...pid
```

