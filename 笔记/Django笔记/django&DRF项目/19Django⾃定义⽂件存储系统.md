# Django⾃定义⽂件存储系统

## 1.⾃定义⽂件存储系统的类

相关⽂档 https://segmentfault.com/a/1190000003708436

meiduo_mall.utils.fastdfs.fdfs_storage.py

## 2.内部实现

```python
class FastDFSStorage(Storage):
    """⾃定义Django⽂件存储系统"""
    def __init__(self, client_conf=None, base_url=None):
        self.client_conf = client_conf or settings.FDFS_CLIENT_CONF
        self.base_url = base_url or settings.FDFS_BASE_URL
    
    def _open(self, name, mode='rb'):
        """打开⽂件时调⽤的，⽬前⽤不到，但是必须实现，所以pass"""
        pass
   
    def _save(self, name, content):
    """
    保存⽂件时调⽤的
    :param name: 要保存的⽂件名字
    :param content: 要保存的⽂件内容
    :return: ⽂件在fdfs唯⼀标识（file_id）
    """
        client = Fdfs_client(self.client_conf)
        ret = client.upload_by_buffer(content.read())
        # 判断上传是否成功
        if ret.get('Status') != 'Upload successed.':
        	raise Exception('fastfds upload failed')
        # 返回结果
        file_id = ret.get('Remote file_id')
        return file_id
	def exists(self, name):
        """判断⽂件是否存在时调⽤的，返回Fasle告诉Django每次都是新的⽂件"""
        return False
        def url(self, name):
        """返回⽂件全路径"""
        return self.base_url + name
```

## 3.修改默认的存储后端

```python
# django⽂件存储
DEFAULT_FILE_STORAGE = 'meiduo_mall.utils.fastdfs.fdfs_storage.FastDFSStorage'
# FastDFS
FDFS_BASE_URL = 'http://192.168.73.129:8888/'
FDFS_CLIENT_CONF = os.path.join(BASE_DIR, 'utils/fastdfs/client.conf')

```

