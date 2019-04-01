# 一.项目简介
该爬虫是使用Python框架Scrapy开发，用来爬取新浪网首页分类的爬虫项目，适合新手用来学习Scrapy框架的使用及开发流程。
爬虫的目标网站地址：http://news.sina.com.cn/guide/
项目一共要爬取三级内容，分别是大类，小类，小类中的资讯文章。如下图所示，新闻，体育是一个大类，新闻大类下有国内，国际，社会等几个小类
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190401163415192.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2VhZ2xldW5pdmVyc2l0eWV5ZQ==,size_16,color_FFFFFF,t_70)
在国际小类中，有很多资讯文章，该爬虫的最终目标就是爬取这些资讯文章的内容。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190401164259466.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2VhZ2xldW5pdmVyc2l0eWV5ZQ==,size_16,color_FFFFFF,t_70)
我们需要做的工作是为每个大类创建一个文件夹，每个大类的文件夹下为其对应的小类再创建一个子文件夹，然后将资讯文件存储到对应的小类文件夹中
最终效果展示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019040116511419.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2VhZ2xldW5pdmVyc2l0eWV5ZQ==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190401165122571.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2VhZ2xldW5pdmVyc2l0eWV5ZQ==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019040116512921.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2VhZ2xldW5pdmVyc2l0eWV5ZQ==,size_16,color_FFFFFF,t_70)
# 二.项目过程
### 1.使用命令创建Scrapy爬虫项目
首先要安装Scrapy框架，Scrapy的安装在我的另一篇博客：https://blog.csdn.net/eagleuniversityeye/article/details/80644804
```
scrapy startproject mySpider
```

### 2.使用命令创建sina爬虫

```
scrapy genspider sina
```

### 3.完善sina爬虫
```python
class SinaSpider(scrapy.Spider):
    name = "sina"
    allowed_domains = ["sina.com.cn"]
    start_urls = [
        "http://news.sina.com.cn/guide/"
    ]

    def parse(self, response):
        items = []
        # 所有大类的url 和 标题
        parentUrls = response.xpath('//div[@id=\"tab01\"]/div/h3/a/@href').extract()
        parentTitle = response.xpath("//div[@id=\"tab01\"]/div/h3/a/text()").extract()

        # 所有小类的ur 和 标题
        subUrls = response.xpath('//div[@id=\"tab01\"]/div/ul/li/a/@href').extract()
        subTitle = response.xpath('//div[@id=\"tab01\"]/div/ul/li/a/text()').extract()

        # 爬取所有大类
        for i in range(0, len(parentTitle)):
            # 指定大类目录的路径和目录名
            parentFilename = "./Data/" + parentTitle[i]

            # 如果目录不存在，则创建目录
            if not os.path.exists(parentFilename):
                os.makedirs(parentFilename)

            # 爬取所有小类
            for j in range(0, len(subUrls)):
                item = MySpiderItem()

                # 保存大类的title和urls
                item['parentTitle'] = parentTitle[i]
                item['parentUrls'] = parentUrls[i]

                # 检查小类的url是否以同类别大类url开头，如果是返回True (sports.sina.com.cn 和 sports.sina.com.cn/nba)
                if_belong = subUrls[j].startswith(item['parentUrls'])

                # 如果属于本大类，将存储目录放在本大类目录下
                if if_belong:
                    subFilename = parentFilename + '/' + subTitle[j]
                    # 如果目录不存在，则创建目录
                    if not os.path.exists(subFilename):
                        os.makedirs(subFilename)

                    # 存储 小类url、title和filename字段数据
                    item['subUrls'] = subUrls[j]
                    item['subTitle'] = subTitle[j]
                    item['subFilename'] = subFilename

                    items.append(item)

        # 发送每个小类url的Request请求，得到Response连同包含meta数据 一同交给回调函数 second_parse 方法处理
        for item in items:
            yield scrapy.Request(url=item['subUrls'], meta={'meta_1': item}, callback=self.second_parse)

    # 对于返回的小类的url，再进行递归请求
    def second_parse(self, response):
        # 提取每次Response的meta数据
        meta_1 = response.meta['meta_1']

        # 取出小类里所有子链接
        sonUrls = response.xpath('//a/@href').extract()

        items = []
        for i in range(0, len(sonUrls)):
            # 检查每个链接是否以大类url开头、以.shtml结尾，如果是返回True
            if_belong = sonUrls[i].endswith('.shtml') and sonUrls[i].startswith(meta_1['parentUrls'])

            # 如果属于本大类，获取字段值放在同一个item下便于传输
            if if_belong:
                item = MySpiderItem()
                item['parentTitle'] = meta_1['parentTitle']
                item['parentUrls'] = meta_1['parentUrls']
                item['subUrls'] = meta_1['subUrls']
                item['subTitle'] = meta_1['subTitle']
                item['subFilename'] = meta_1['subFilename']
                item['sonUrls'] = sonUrls[i]
                items.append(item)

        # 发送每个小类下子链接url的Request请求，得到Response后连同包含meta数据 一同交给回调函数 detail_parse 方法处理
        for item in items:
            yield scrapy.Request(url=item['sonUrls'], meta={'meta_2': item}, callback=self.detail_parse)

    # 数据解析方法，获取文章标题和内容
    def detail_parse(self, response):
        item = response.meta['meta_2']
        content = ""
        head = response.xpath('//h1[@id=\"main_title\"]/text()')
        content_list = response.xpath('//div[@id=\"artibody\"]/p/text()').extract()

        # 将p标签里的文本内容合并到一起
        for content_one in content_list:
            content += content_one

        item['head'] = head
        item['content'] = content

        yield item
```

### 4.完善items文件

```python
class MySpiderItem(scrapy.Item):
    # 大类的标题 和 url
    parentTitle = scrapy.Field()
    parentUrls = scrapy.Field()

    # 小类的标题 和 子url
    subTitle = scrapy.Field()
    subUrls = scrapy.Field()

    # 小类目录存储路径
    subFilename = scrapy.Field()

    # 小类下的子链接
    sonUrls = scrapy.Field()

    # 文章标题和内容
    head = scrapy.Field()
    content = scrapy.Field()
```

### 5.完善pipelines文件

```python
class SinaPipeline(object):
    def process_item(self, item, spider):
        sonUrls = item['sonUrls']

        # 文件名为子链接url中间部分，并将 / 替换为 _，保存为 .txt格式
        filename = sonUrls[7:-6].replace('/', '_')
        filename += ".txt"

        fp = open(item['subFilename'] + '/' + filename, 'w')
        fp.write(item['content'])
        fp.close()

        return item
```

### 6.在setting添加设置

```python
BOT_NAME = 'mySpider'

SPIDER_MODULES = ['mySpider.spiders']
NEWSPIDER_MODULE = 'mySpider.spiders'

ITEM_PIPELINES = {
    'mySpider.pipelines.SinaPipeline': 300,
}

LOG_LEVEL = 'DEBUG'
```

### 7.创建爬虫启动文件
在爬虫项目的根目录下创建一个文件名为main.py的文件
使用该文件设置命令启动爬虫，注意启动命令为：`scrapy crawl 爬虫名`，爬虫名就是你在第二步创建的爬虫的名字，注意两两个名称要对应

```python
from scrapy import cmdline
cmdline.execute('scrapy crawl sina'.split())
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/201904011809241.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2VhZ2xldW5pdmVyc2l0eWV5ZQ==,size_16,color_FFFFFF,t_70)
### 8.设置启动项
在PyCharm中设置项目启动方式，如下图所示，主要配置这两点。该项目使用的是Python3，如果使用Python2还需要进行编码转换，这里不再赘述
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190401181315146.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2VhZ2xldW5pdmVyc2l0eWV5ZQ==,size_16,color_FFFFFF,t_70)






































































































