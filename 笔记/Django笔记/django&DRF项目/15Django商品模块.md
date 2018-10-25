# Django商品模块

[TOC]

## 1.SPU和SKU概念

### SPU

1. SPU = Standard Product Unit （标准产品单位）
2. SPU是商品信息聚合的最⼩单位，是⼀组可服⽤、易检索的标准化信息的集合，该集合描述了⼀个产品的特性。
3. iPhone X 就是⼀个SPU，与商家、颜⾊、款式、规格、套餐等都⽆关。

### SKU

1. SKU = Stock Keeping Unit （库存量单位）
2. SKU即库存进出计量的单位，可以是以件、盒、托盘等为单位，是物理上不可分割的最⼩存货单元。
3. iPhone X 全⽹通⿊⾊256G 就是⼀个SKU，表示了具体的规格、颜⾊等信息

## 2.主⻚⼴告表分析

![首页广告数据表结构](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC06%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/%E9%A6%96%E9%A1%B5%E5%B9%BF%E5%91%8A%E6%95%B0%E6%8D%AE%E5%BA%93%E7%BB%93%E6%9E%84.png)

## 3.商品数据表结构

![商品数据库表结构](file:///F:/python%E5%B0%B1%E4%B8%9A%E7%8F%AD%E8%AF%BE%E4%BB%B6/Django%E9%A1%B9%E7%9B%AE/%E7%BE%8E%E5%A4%9A%E5%95%86%E5%9F%8E%E9%A1%B9%E7%9B%AE-%E7%AC%AC06%E5%A4%A9/1-%E6%95%99%E5%AD%A6%E8%B5%84%E6%96%99/02-%E6%95%99%E5%AD%A6%E7%AC%94%E8%AE%B0/images/%E5%95%86%E5%93%81%E6%95%B0%E6%8D%AE%E5%BA%93%E8%A1%A8%E7%BB%93%E6%9E%84.png)

## 4.数据库模型类

创建商品应用goods，商品数据模型类

```python
class GoodsCategory(BaseModel):
    """
    商品类别
    """
    name = models.CharField(max_length=10, verbose_name='名称')
    parent = models.ForeignKey('self', null=True, blank=True, on_delete=models.CASCADE, verbose_name='父类别')

    class Meta:
        db_table = 'tb_goods_category'
        verbose_name = '商品类别'
        verbose_name_plural = verbose_name

    def __str__(self):
        return self.name


class GoodsChannel(BaseModel):
    """
    商品频道
    """
    group_id = models.IntegerField(verbose_name='组号')
    category = models.ForeignKey(GoodsCategory, on_delete=models.CASCADE, verbose_name='顶级商品类别')
    url = models.CharField(max_length=50, verbose_name='频道页面链接')
    sequence = models.IntegerField(verbose_name='组内顺序')

    class Meta:
        db_table = 'tb_goods_channel'
        verbose_name = '商品频道'
        verbose_name_plural = verbose_name

    def __str__(self):
        return self.category.name


class Brand(BaseModel):
    """
    品牌
    """
    name = models.CharField(max_length=20, verbose_name='名称')
    logo = models.ImageField(verbose_name='Logo图片')
    first_letter = models.CharField(max_length=1, verbose_name='品牌首字母')

    class Meta:
        db_table = 'tb_brand'
        verbose_name = '品牌'
        verbose_name_plural = verbose_name

    def __str__(self):
        return self.name


class Goods(BaseModel):
    """
    商品SPU
    """
    name = models.CharField(max_length=50, verbose_name='名称')
    brand = models.ForeignKey(Brand, on_delete=models.PROTECT, verbose_name='品牌')
    category1 = models.ForeignKey(GoodsCategory, on_delete=models.PROTECT, related_name='cat1_goods', verbose_name='一级类别')
    category2 = models.ForeignKey(GoodsCategory, on_delete=models.PROTECT, related_name='cat2_goods', verbose_name='二级类别')
    category3 = models.ForeignKey(GoodsCategory, on_delete=models.PROTECT, related_name='cat3_goods', verbose_name='三级类别')
    sales = models.IntegerField(default=0, verbose_name='销量')
    comments = models.IntegerField(default=0, verbose_name='评价数')

    class Meta:
        db_table = 'tb_goods'
        verbose_name = '商品'
        verbose_name_plural = verbose_name

    def __str__(self):
        return self.name


class GoodsSpecification(BaseModel):
    """
    商品规格
    """
    goods = models.ForeignKey(Goods, on_delete=models.CASCADE, verbose_name='商品')
    name = models.CharField(max_length=20, verbose_name='规格名称')

    class Meta:
        db_table = 'tb_goods_specification'
        verbose_name = '商品规格'
        verbose_name_plural = verbose_name

    def __str__(self):
        return '%s: %s' % (self.goods.name, self.name)


class SpecificationOption(BaseModel):
    """
    规格选项
    """
    spec = models.ForeignKey(GoodsSpecification, on_delete=models.CASCADE, verbose_name='规格')
    value = models.CharField(max_length=20, verbose_name='选项值')

    class Meta:
        db_table = 'tb_specification_option'
        verbose_name = '规格选项'
        verbose_name_plural = verbose_name

    def __str__(self):
        return '%s - %s' % (self.spec, self.value)


class SKU(BaseModel):
    """
    商品SKU
    """
    name = models.CharField(max_length=50, verbose_name='名称')
    caption = models.CharField(max_length=100, verbose_name='副标题')
    goods = models.ForeignKey(Goods, on_delete=models.CASCADE, verbose_name='商品')
    category = models.ForeignKey(GoodsCategory, on_delete=models.PROTECT, verbose_name='从属类别')
    price = models.DecimalField(max_digits=10, decimal_places=2, verbose_name='单价')
    cost_price = models.DecimalField(max_digits=10, decimal_places=2, verbose_name='进价')
    market_price = models.DecimalField(max_digits=10, decimal_places=2, verbose_name='市场价')
    stock = models.IntegerField(default=0, verbose_name='库存')
    sales = models.IntegerField(default=0, verbose_name='销量')
    comments = models.IntegerField(default=0, verbose_name='评价数')
    is_launched = models.BooleanField(default=True, verbose_name='是否上架销售')
    default_image_url = models.CharField(max_length=200, default='', null=True, blank=True, verbose_name='默认图片')

    class Meta:
        db_table = 'tb_sku'
        verbose_name = '商品SKU'
        verbose_name_plural = verbose_name

    def __str__(self):
        return '%s: %s' % (self.id, self.name)


class SKUImage(BaseModel):
    """
    SKU图片
    """
    sku = models.ForeignKey(SKU, on_delete=models.CASCADE, verbose_name='sku')
    image = models.ImageField(verbose_name='图片')

    class Meta:
        db_table = 'tb_sku_image'
        verbose_name = 'SKU图片'
        verbose_name_plural = verbose_name

    def __str__(self):
        return '%s %s' % (self.sku.name, self.id)


class SKUSpecification(BaseModel):
    """
    SKU具体规格
    """
    sku = models.ForeignKey(SKU, on_delete=models.CASCADE, verbose_name='sku')
    spec = models.ForeignKey(GoodsSpecification, on_delete=models.PROTECT, verbose_name='规格名称')
    option = models.ForeignKey(SpecificationOption, on_delete=models.PROTECT, verbose_name='规格值')

    class Meta:
        db_table = 'tb_sku_specification'
        verbose_name = 'SKU规格'
        verbose_name_plural = verbose_name

    def __str__(self):
        return '%s: %s - %s' % (self.sku, self.spec.name, self.option.value)
```

创建广告内容应用contents，广告数据模型类

```python
class ContentCategory(BaseModel):
    """
    广告内容类别
    """
    name = models.CharField(max_length=50, verbose_name='名称')
    key = models.CharField(max_length=50, verbose_name='类别键名')

    class Meta:
        db_table = 'tb_content_category'
        verbose_name = '广告内容类别'
        verbose_name_plural = verbose_name

    def __str__(self):
        return self.name


class Content(BaseModel):
    """
    广告内容
    """
    category = models.ForeignKey(ContentCategory, on_delete=models.PROTECT, verbose_name='类别')
    title = models.CharField(max_length=100, verbose_name='标题')
    url = models.CharField(max_length=300, verbose_name='内容链接')
    image = models.ImageField(null=True, blank=True, verbose_name='图片')
    text = models.TextField(null=True, blank=True, verbose_name='内容')
    sequence = models.IntegerField(verbose_name='排序')
    status = models.BooleanField(default=True, verbose_name='是否展示')

    class Meta:
        db_table = 'tb_content'
        verbose_name = '广告内容'
        verbose_name_plural = verbose_name

    def __str__(self):
        return self.category.name + ': ' + self.title
```