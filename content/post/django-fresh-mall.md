---
title: 用 Django 搭出「天天生鲜」电商站：从 MTV 模型到购物车结算
date: 2026-10-28T22:05:00.000+08:00
categories:
  - 项目日志
tags:
  - Django
  - Python
  - MySQL
  - URL路由
  - 模板渲染
---

> I built a "Tiantian Fresh" e-commerce site on Django with MySQL: product listing, category pages, product detail, a live cart with price totaling, an order flow, plus a separate admin backend for managing products and users. This post covers the MTV layout, model definitions, routing and the two most confusing beginner pitfalls — 404 page-not-found and data that never renders to the template.

## 一、前言：刚从 Servlet 过来，Django 让我"不太适应"

写 JavaWeb 那会儿，一个请求要手写 Servlet、Service、DAO 三层，连接池、编码过滤器样样自己配。换到 Django，第一感觉是"脚手架太全了"——路由、ORM、后台管理几乎开箱即用。但脚手架全也意味着：**很多事情 Django 替你做了，一旦出错你都不知道它卡在哪一层**。

这个"天天生鲜"项目就是我系统过一遍 Django 的练手：前台六个页面（首页、分类、详情、购物车、下单、下单成功），外加一个自带的后台管理（商品和用户的增删改）。目标不是多炫，而是把 Django 的 MTV、路由、模板渲染这条主线走通。

## 二、环境与架构

技术栈干净利落：

| 组件 | 作用 | 说明 |
|---|---|---|
| Django | Web 框架 | MTV：Model 取数据、View 编排、Template 渲染 |
| MySQL | 数据库 | 四张核心表：用户/商品/分类/订单 |
| Python | 语言 | 3.x，`manage.py` 起服务 |
| 浏览器 | 访问端 | `127.0.0.1:8000` 本地访问 |

按功能拆，整站是一棵功能树：前台面向购物，后台面向管理，最终都落到 MySQL：

![天天生鲜系统总体功能结构](/images/django-fresh-func-arch.png)

*商品展示/分类/详情/购物车/订单提交构成前台，后台管理独立处理商品与用户*

数据库我建了四张表，关键字段直接对应 ORM 模型：

| 表 | 关键字段 | 说明 |
|---|---|---|
| 用户表 | uname / upwd / uemail / uaddress / uphone | 收货人与联系方式 |
| 商品表 | gtitle / gprice / gunit / gclick / gkucun / gtype_id | 价格用 decimal，库存单独存 |
| 商品类别表 | title / IsDelete | 软删除标记 |
| 订单表 | odate(默认当前时间) / olsPay / ototal / oaddress / User_id | 下单时间自动填 |

## 三、核心实现：模型 → 路由 → 视图 → 模板

**Step 1：先把数据库和模型接上**

`settings.py` 里配好 MySQL：

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'tiantianfresh',
        'USER': 'root',
        'PASSWORD': '123456',
        'HOST': '127.0.0.1',
        'PORT': '3306',
    }
}
```

商品模型用 ORM 定义，字段类型直接对应表里那几列：

```python
class Goods(models.Model):
    gtitle  = models.CharField(max_length=200)
    gprice  = models.DecimalField(max_digits=10, decimal_places=2)
    gunit   = models.CharField(max_length=200)
    gclick  = models.IntegerField(default=0)   # 阅读量
    gkucun  = models.IntegerField(default=0)    # 库存
    gcontent= models.CharField(max_length=200)
    gtype   = models.ForeignKey('GoodsType', on_delete=models.CASCADE)
```

然后两条命令建表、起服务：

```bash
python manage.py makemigrations
python manage.py migrate
python manage.py runserver 0.0.0.0:8000
```

**Step 2：路由 + 视图把页面串起来**

根路由把请求分发到对应视图：

```python
from django.urls import path
from df import views

urlpatterns = [
    path('admin/', admin.site.urls),
    path('',       views.index,  name='index'),
    path('cart/',  views.cart,   name='cart'),
]
```

购物车视图做的事就是：查出当前选中的商品、算个总价，塞进 context 传给模板：

```python
def cart(request):
    ids = request.session.get('selected_ids', [])
    goods_list = Goods.objects.filter(id__in=ids)
    total = sum(float(g.gprice) for g in goods_list)
    return render(request, 'cart.html',
                  {'goods': goods_list, 'total': total})
```

模板里再用 `{% for %}` 把商品列表循环渲染出来，`{{ total }}` 显示合计。整条链路就是 Django 最经典的一次请求生命周期。

## 四、实际跑起来长什么样

首页是一个典型生鲜电商首页——左侧分类导航、中间轮播 banner、下面按"新鲜水果/海鲜水产"分区展示商品，每个商品带图带价：

![天天生鲜商城首页](/images/django-fresh-home.png)

*127.0.0.1:8000 首页：分类导航 + 轮播 + 商品推荐分区*

点进购物车，三件商品、单价、数量、小计一目了然，底部自动算出合计：

![购物车页面](/images/django-fresh-cart.png)

*新鲜草莓 ¥20、海南香蕉 ¥89、濒危刀鱼 ¥100，共 3 件，合计 ¥209.00*

后台则是另一套界面——商品列表直接从数据库读出来，库存、点击量都能在线改：

![后台商品管理](/images/django-fresh-admin-goods.png)

*后台商品管理页：17 条商品记录，价格、库存、简介可直接编辑*

## 五、踩坑实录

**坑 1：页面永远 404，找不到**

现象：访问 `127.0.0.1:8000/cart/` 直接报 Page not found。新手在 Django 里最常漏的就是**路由没接上**：要么 `urls.py` 里没写对应 `path`，要么写了却忘了在根路由里 `include`，要么视图函数名对不上。解决：从访问 URL 倒推，先确认根路由分发到了哪个 app、再确认那个 app 的 `urls.py` 里有没有这一条。

**坑 2：数据库里有数据，页面就是渲染不出来**

现象：后台能查到商品，但前台模板死活是空的。原因是**视图 `render` 传的 context 变量名和模板里 `{{ }}` 用的名字对不上**——视图传的是 `goods`，模板里写成了 `goods_list`，自然啥也取不到。这是 Django 新手最高频的 bug：ORM 没问题、SQL 没问题，就是名字对不上。逐字核对 context key 和模板变量即可。

**坑 3：连不上 MySQL**

Django 默认连 SQLite，要切 MySQL 还得装数据库驱动（`mysqlclient` 或在 `__init__.py` 里 `pymysql.install_as_MySQLdb()`），否则 `migrate` 时直接报引擎错误。驱动 + `settings.py` 的 `ENGINE`/`NAME` 配对，缺一不可。

## 六、总结与思考

这一遍做下来，我最大的体会是：**Django 把"路由—视图—ORM—模板"这条链封装得很爽，但封装也意味着你必须清楚数据是在哪一环丢的**。404 是路由层的事，空白页是 context/模板层的事，连不上库是配置层的事——学会分层定位，比死记 API 有用得多。

和之前手写 Servlet 那版电商比，Django 几乎不用写重复的 CRUD，后台管理更是白送。但它也不是银弹：现在这个版本没有用户登录鉴权、购物车存在 session 里一刷新就乱、订单也没接真实支付。要往能用的方向走，下一步该补的是用户认证、订单与商品的真实关联，以及把现在写死在视图里的查询拆成更清晰的服务层——这些就是下一轮迭代的事了。

---

*本文整理自一次 Django Web 开发课程设计，页面与后台均为本地 127.0.0.1:8000 实测截图。*
