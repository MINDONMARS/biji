# Django用户中心⽤户地址管理(增删改查)

[TOC]

## 1.准备⽤户地址模型类

```python
class Address(BaseModel):
    """
    用户地址
    """
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='addresses', verbose_name='用户')
    title = models.CharField(max_length=20, verbose_name='地址名称')
    receiver = models.CharField(max_length=20, verbose_name='收货人')
    province = models.ForeignKey('areas.Area', on_delete=models.PROTECT, related_name='province_addresses', verbose_name='省')
    city = models.ForeignKey('areas.Area', on_delete=models.PROTECT, related_name='city_addresses', verbose_name='市')
    district = models.ForeignKey('areas.Area', on_delete=models.PROTECT, related_name='district_addresses', verbose_name='区')
    place = models.CharField(max_length=50, verbose_name='地址')
    mobile = models.CharField(max_length=11, verbose_name='手机')
    tel = models.CharField(max_length=20, null=True, blank=True, default='', verbose_name='固定电话')
    email = models.CharField(max_length=30, null=True, blank=True, default='', verbose_name='电子邮箱')
    is_deleted = models.BooleanField(default=False, verbose_name='逻辑删除')

    class Meta:
        db_table = 'tb_address'
        verbose_name = '用户地址'
        verbose_name_plural = verbose_name
        ordering = ['-update_time']
```

用户模型类添加默认收货地址字段

```python
class User(AbstractUser):
    """自定义用户模型类"""
    mobile = models.CharField(max_length=11, unique=True, verbose_name='手机号')
    # 记录邮件的状态（是否被激活）
    email_active = models.BooleanField(default=False, verbose_name='邮箱验证状态')
    
    
    default_address = models.ForeignKey('Address', related_name='users', null=True, blank=True,
                                        on_delete=models.SET_NULL, verbose_name='默认地址')
```

## 2.迁移建表

`python manage.py makemigrations`
`python manage.py migrate`

## 3.视图集

```python
class AddressViewSet(mixins.CreateModelMixin, mixins.UpdateModelMixin, GenericViewSet):
    """用户地址管理视图集：增删改查"""

    # 指定序列化器
    serializer_class = serializers.UserAddressSerializer
    # 指定用户登录才能增删改查地址
    permission_classes = [IsAuthenticated]

    # POST /address/
    def create(self, request, *args, **kwargs):
        """重写视图集的create行为，用于新增地址"""

        # 判断用户的地址总数是否大于上限（20个）
        # count = Address.objects.filter(user=request.user).count()
        count = request.user.addresses.count()
        if count > constants.USER_ADDRESS_COUNTS_LIMIT:
            return Response({'message':'超出地址上限'}, status=status.HTTP_400_BAD_REQUEST)

        # 补充父类之后的写法
        return super().create(request, *args, **kwargs)

        # # 获取序列化器对象
        # serializer = self.get_serializer(data=request.data)
        # # 校验
        # serializer.is_valid(raise_exception=True)
        # # 执行保存
        # serializer.save()
        # # 响应序列化之后的数据
        # return Response(serializer.data, status=status.HTTP_201_CREATED)

    def get_queryset(self):
        return self.request.user.addresses.filter(is_deleted=False)

    # GET /addresses/
    def list(self, request, *args, **kwargs):
        """
        用户地址列表数据
        """
        queryset = self.get_queryset()
        serializer = self.get_serializer(queryset, many=True)
        user = self.request.user
        return Response({
            'user_id': user.id,
            'default_address_id': user.default_address_id,
            'limit': constants.USER_ADDRESS_COUNTS_LIMIT,
            'addresses': serializer.data,
        })

    # delete /addresses/<pk>/
    def destroy(self, request, *args, **kwargs):
        """
        处理删除
        """
        address = self.get_object()

        # 进行逻辑删除
        address.is_deleted = True
        address.save()

        return Response(status=status.HTTP_204_NO_CONTENT)

    # put /addresses/pk/status/
    @action(methods=['put'], detail=True)
    def status(self, request, pk=None):
        """
        设置默认地址
        """
        address = self.get_object()
        request.user.default_address = address
        request.user.save()
        return Response({'message': 'OK'}, status=status.HTTP_200_OK)

    # put /addresses/pk/title/
    # 需要请求体参数 title
    @action(methods=['put'], detail=True)
    def title(self, request, pk=None):
        """
        修改标题
        """
        address = self.get_object()
        serializer = serializers.AddressTitleSerializer(instance=address, data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        return Response(serializer.data)
```

## 4.序列化器

```python
class UserAddressSerializer(serializers.ModelSerializer):
    """
    用户地址序列化器
    """
    province = serializers.StringRelatedField(read_only=True)
    city = serializers.StringRelatedField(read_only=True)
    district = serializers.StringRelatedField(read_only=True)
    province_id = serializers.IntegerField(label='省ID', required=True)
    city_id = serializers.IntegerField(label='市ID', required=True)
    district_id = serializers.IntegerField(label='区ID', required=True)

    class Meta:
        model = Address
        exclude = ('user', 'is_deleted', 'create_time', 'update_time')

    def validate_mobile(self, value):
        """
        验证手机号
        """
        if not re.match(r'^1[3-9]\d{9}$', value):
            raise serializers.ValidationError('手机号格式错误')
        return value

    # 重写create方法，追加地址的外键的存储
    def create(self, validated_data):
        # user = validated_data['user']
        # 读取当前经过认证和权限的登录用户
        user = self.context['request'].user
        validated_data['user'] = user

        address = Address.objects.create(**validated_data)

        return address


class AddressTitleSerializer(serializers.ModelSerializer):
    """
    地址标题
    """
    class Meta:
        model = Address
        fields = ('title',)
```

## 5.视图集路由

```python
router = routers.DefaultRouter()
router.register(r'addresses', views.AddressViewSet, base_name='addresses')
urlpatterns += router.urls
```

