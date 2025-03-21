# 生成html详情页面

我们通过celery异步触发来生成HTML页面

在celery\_tasks中添加任务

![](/assets/detail_html_generate.png)

tasks中的代码

```
import os
from django.template import loader
from django.conf import settings
from apps.goods import models
from apps.contents.utils import get_categories
from apps.goods.utils import get_breadcrumb
from celery_tasks.main import app

@app.task
def generate_static_sku_detail_html(sku_id):
    """
    生成静态商品详情页面
    :param sku_id: 商品sku id
    """
    # 获取当前sku的信息
    sku = models.SKU.objects.get(id=sku_id)

    # 查询商品频道分类
    categories = get_categories()
    # 查询面包屑导航
    breadcrumb = get_breadcrumb(sku.category)

    # 构建当前商品的规格键
    sku_specs = sku.specs.order_by('spec_id')
    sku_key = []
    for spec in sku_specs:
        sku_key.append(spec.option.id)
    # 获取当前商品的所有SKU
    skus = sku.spu.sku_set.all()
    # 构建不同规格参数（选项）的sku字典
    spec_sku_map = {}
    for s in skus:
        # 获取sku的规格参数
        s_specs = s.specs.order_by('spec_id')
        # 用于形成规格参数-sku字典的键
        key = []
        for spec in s_specs:
            key.append(spec.option.id)
        # 向规格参数-sku字典添加记录
        spec_sku_map[tuple(key)] = s.id
    # 获取当前商品的规格信息
    goods_specs = sku.spu.specs.order_by('id')
    # 若当前sku的规格信息不完整，则不再继续
    if len(sku_key) < len(goods_specs):
        return
    for index, spec in enumerate(goods_specs):
        # 复制当前sku的规格键
        key = sku_key[:]
        # 该规格的选项
        spec_options = spec.options.all()
        for option in spec_options:
            # 在规格参数sku字典中查找符合当前规格的sku
            key[index] = option.id
            option.sku_id = spec_sku_map.get(tuple(key))
        spec.spec_options = spec_options

    # 上下文
    context = {
        'categories': categories,
        'breadcrumb': breadcrumb,
        'sku': sku,
        'specs': goods_specs,
    }

    template = loader.get_template('detail.html')
    html_text = template.render(context)
    file_path = os.path.join(settings.STATICFILES_DIRS[0], 'detail/'+str(sku_id)+'.html')
    with open(file_path, 'w') as f:
        f.write(html_text)
```

在create方法中触发

```
class SKUSerializer(serializers.ModelSerializer):

    ...

    def create(self, validated_data):
        ##1.获取specs的数据
        data=self.context['request'].data
        specs=data.get('specs')
        #2.把validated_data中的 specs 删除掉
        if 'specs' in validated_data:
            del  validated_data['specs']
        #3.保存sku数据
        sku=SKU.objects.create(**validated_data)
        #4.自己手动实现specs的数据入库
        # 因为 spces 是列表数据
        #[{spec_id: "4", option_id: 8}, {spec_id: "5", option_id: 11}]
        for item in specs:
            # item = {spec_id: "4", option_id: 8}
            SKUSpecification.objects.create(
                sku=sku,
                spec_id=item.get('spec_id'),
                option_id=item.get('option_id')
            )

        #生成详情页面
        from celery_tasks.html.tasks import generate_static_sku_detail_html
        generate_static_sku_detail_html.delay(sku.id)

        return sku
```

> 注意： 给商品设置一个默认图片，否则会报错
>
> ![](/assets/sku_default_image.png)



