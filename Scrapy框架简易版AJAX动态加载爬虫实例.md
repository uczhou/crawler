
## 微博图片/相册爬虫：Scrapy框架简易版AJAX动态加载爬虫实例

最近在爬取网络上的图片资源，今天用scrapy框架搭建了一个简单的爬虫，尝试着爬取微博上某个大V的美女图片。
首先是搜索网络上现有的爬取微博图片的code，结合网上最近几篇有关如何爬取微博图片的文章，以及其中的一些被踩过的坑，自己码了几行代码，爬取了图片。

<img src="http://img.meinvce.com/tech/p1.png" height="500px" hspace="50px">

### 1. 首先还是对微博相册进行分析
本次微博相册爬虫爬取的对象是某大V，按照其他文章所写，现在爬取微博图片（或者说社交网络数据）首先考虑移动端。
所以这次就直接用Chrome的Device mode加载微博页，然后用Chrome自带的Developer Tools抓取网页加载。

<img src="http://img.meinvce.com/tech/p3.png" width ="500px" hspace="50px">

其中有一个向m.weibo.cn/api/container发送的请求。

<img src="http://img.meinvce.com/tech/p4.png" width="500px" hspace="50px">

### 2. 获取微博信息和图片资源
点击该请求返回的资源，有以下信息：

<img src="http://img.meinvce.com/tech/p5.png" width="500px" hspace="50px">

<img src="http://img.meinvce.com/tech/p7.png" width="500px" hspace="50px">

将Request URL复制到浏览器中打开，发现返回的是一个JSON文件，文件内容包含该页的微博条数，以及每条微博的详细信息，结构如下：

<img src="http://img.meinvce.com/tech/p9.png" width="500px" hspace="50px">

data包含cards，cardlistInfo等信息，微博图片有关的信息都在cards中。获取cards中图片的方式如下：

```
result_json = json.loads(response.body_as_unicode())
cards = result_json['data']['cards']
for i in range(len(cards)):
    if 'mblog' in cards[i]:
        mblog = cards[i]['mblog']
        pic_exist = cards[i]['mblog'].get('pics')
        if pic_exist:
            pics = cards[i]['mblog']['pics']
            for j in range(len(pics)):
                pic_large = pics[j]['large']['url']        
                        
```

### 3. 用Scrapy框架写爬虫来爬取图片
分析完微博信息和图片资源，最后就是利用scrapy框架来爬取图片了。scrapy框架的安装和使用就不在这里叙述了，直接上代码。

最后爬取的过程截图如下：

<img src="http://img.meinvce.com/tech/p10.png" height="500px" hspace="50px">

爬取的文件：

<img src="http://img.meinvce.com/tech/p11.png" height="500px" hspace="50px">

<img src="http://img.meinvce.com/tech/p12.png" height="500px" hspace="50px">

### 4. 总结
这次爬虫的主要时间是花在了阅读其他博文，以及自己分析微博页面的过程中，写代码用了不到半个小时，调试了十几分钟。
以防微博账号被封，这次爬取速度是每10秒一次请求，暂时没有遇到账号被封的情况。

### 5. 代码

### a. image_spider.py

```
class WeiboSpider(Spider):
    name = 'weibospider'

    logging.basicConfig(
        filename=log_path,
        format='%(levelname)s: %(message)s',
        level=logging.INFO
    )

    cookie = '查看你浏览器的cookie内容'

    referer = 'https://m.weibo.cn/u/5834408758?uid=5834408758&luicode=10000011&lfid=100103type%3D1%26q%3Dv%E5%A5%B3%E7%A5%9E' 

    default_headers = {
        "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8",
        'Accept-Encoding': 'gzip,deflate',
        "Accept-Language": "en-US,en;q=0.9,ja;q=0.8,zh-CN;q=0.7,zh;q=0.6,zh-TW;q=0.5,de;q=0.4",
        'Cache-Control': 'max-age=0',
        "Connection": "keep-alive",
        "Cookie": cookie,
        "Host": 'm.weibo.cn',
        'Referer': referer,
        "Upgrade-Insecure-Requests": "1",
        "User-Agent": "Mozilla/5.0 (Linux; Android 6.0; Nexus 5 Build/MRA58N) "
                      "AppleWebKit/537.36 (KHTML, like Gecko) Chrome/68.0.3440.106 Mobile Safari/537.36"
    }

    logger = logging.getLogger(__name__)

    allowed_domains = ["m.weibo.cn"]

    user_id = '5834408758'

    parsed = set()

    def start_requests(self):
        urls = ['https://m.weibo.cn/api/container/getIndex?containerid=2304135834408758_-_WEIBO_SECOND_PROFILE_WEIBO'
                '&luicode=10000011&lfid=2302835834408758&type=uid&value=5834408758&page_type=03&page=1']
        for url in urls:
            yield Request(url, headers=self.default_headers, meta={"dont_retry": False, "max_retry_times": 5})

    def get_url(self, n):
        # 构建新的url
        data = {
            'containerid': '2304135834408758_-_WEIBO_SECOND_PROFILE_WEIBO',
            'luicode': '10000011',
            'lfid': '2302835834408758',
            'type': 'uid',
            'value': '5834408758',
            'page_type': '03',
            'page': n
        }

        url = 'https://m.weibo.cn/api/container/getIndex?' + urlencode(data)

        return url

    def parse(self, response):

        result_json = json.loads(response.body_as_unicode())

        cards = result_json['data']['cards']

        for i in range(len(cards)):
            
            # imagedownloadpipeline 处理的item
            image_item = ImageItem()
            image_item['image_urls'] = []
            image_item['referer'] = response.url
            image_item['pid'] = ''

            if 'mblog' in cards[i]:
                mblog = cards[i]['mblog']
                pic_exist = cards[i]['mblog'].get('pics')
                if pic_exist:

                    pics = cards[i]['mblog']['pics']
                    for j in range(len(pics)):
                        pic_large = pics[j]['large']['url']
                        
                        # 记录每张图片的blog id和该图片的url地址，用于后续图片归类，存档
                        src_item = SrcItem()
                        src_item['pid'] = mblog['id']
                        src_item['referer'] = response.url
                        src_item['image_url'] = pic_large

                        yield src_item

                        if image_item['pid'] == '':
                            image_item['image_urls'].append(pic_large)
                            image_item['pid'] = mblog['id']
                            
                            # 图片的一些文字信息，用于后续图片处理，存档
                            meta_item = MetaItem()
                            meta_item['pid'] = mblog['id']
                            meta_item['info'] = mblog['text']                           

                            yield meta_item

                        elif image_item['pid'] == mblog['id']:
                            image_item['image_urls'].append(pic_large)

                        else:
                            yield image_item

                            image_item = ImageItem()
                            image_item['image_urls'] = [pic_large]
                            image_item['referer'] = response.url
                            image_item['pid'] = mblog['id']
                yield image_item
        pageno = result_json['data']['cardlistInfo'].get('page')
        yield Request(self.get_url(pageno), headers=self.default_headers,
                      meta={"dont_retry": False, "dont_redirect": True, "max_retry_times": 3})

```

### b. pipelines.py
```
import json
from scrapy.pipelines.images import ImagesPipeline
from scrapy.exceptions import DropItem
from scrapy import Request
from .items import MetaItem, SrcItem, TagItem
from .rules import meta_path, src_path, image_path, tag_path, imgdir_path
import os
import hashlib
from urllib.parse import urlparse

crawled = set()
if os.path.exists('{}/full'.format(imgdir_path)):
    files = os.listdir('{}/full'.format(imgdir_path))
    for file in files:
        crawled.add(file.split('.')[0])


class KatiebotPipeline(object):

    def __init__(self):
        # meta_path = 'desc/avnvyou_meta.json'
        self.meta_file = open(meta_path, 'a+')
        # src_path = 'desc/avnvyou_src.json'
        self.src_file = open(src_path, 'a+')
        self.tag_file = open(tag_path, 'a+')

    def close_spider(self, spider):
        self.meta_file.close()
        self.src_file.close()
        self.tag_file.close()

    def process_item(self, item, spider):
        if isinstance(item, MetaItem):
            line = json.dumps(dict(item)) + "\n"
            print(line)
            self.meta_file.write(line)
            self.meta_file.flush()

            raise DropItem("Finished Processing Item: {}".format(line))

        elif isinstance(item, SrcItem):
            line = json.dumps(dict(item)) + "\n"
            print(line)
            self.src_file.write(line)
            self.src_file.flush()

            raise DropItem("Finished Processing Item: {}".format(line))
        elif isinstance(item, TagItem):
            line = json.dumps(dict(item)) + "\n"
            print(line)
            self.tag_file.write(line)
            self.tag_file.flush()

            raise DropItem("Finished Processing Item: {}".format(line))
        else:
            return item


class ImageDownloadPipeline(ImagesPipeline):
    cookie = '你的cookie'

    referer = 'https://m.weibo.cn/u/5834408758?uid=5834408758&luicode=10000011&lfid=100103type%3D1%26q%3Dv%E5%A5%B3%E7%A5%9E'

    default_headers = {
        "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8",
        'Accept-Encoding': 'gzip,deflate',
        "Accept-Language": "en-US,en;q=0.9,ja;q=0.8,zh-CN;q=0.7,zh;q=0.6,zh-TW;q=0.5,de;q=0.4",
        'Cache-Control': 'max-age=0',
        "Connection": "keep-alive",
        "Cookie": cookie,
        "Host": '',
        'Referer': referer,
        "Upgrade-Insecure-Requests": "1",
        "User-Agent": "Mozilla/5.0 (Linux; Android 6.0; Nexus 5 Build/MRA58N) "
                      "AppleWebKit/537.36 (KHTML, like Gecko) Chrome/68.0.3440.106 Mobile Safari/537.36"
    }

    def get_media_requests(self, item, info):
        print(item)
        for image_url in item['image_urls']:
            sha1_name = hashlib.sha1(image_url.encode('utf-8')).hexdigest()
            if sha1_name not in crawled:
                # self.default_headers['Referer'] = item['referer']
                # print(self.default_headers)
                self.default_headers['Host'] = urlparse(image_url).netloc
                yield Request(image_url, headers=self.default_headers,
                              meta={"dont_retry": False, "dont_redirect": True, "max_retry_times": 3})

    def item_completed(self, results, item, info):
        image_paths = [{x['url']: x['path'].split('/')[-1]} for ok, x in results if ok]
        if not image_paths:
            raise DropItem("Item contains no images")
        item['image_paths'] = image_paths

        return item


class ImagePathPipeline(object):

    def __init__(self):

        self.imagepath_file = open(image_path, 'a+')

    def close_spider(self, spider):
        self.imagepath_file.close()

    def process_item(self, item, spider):
        if not item['image_paths']:
            raise DropItem("Item contains no image path")

        for path in item['image_paths']:
            line = json.dumps(dict(path)) + "\n"
            print(line)
            self.imagepath_file.write(line)
            self.imagepath_file.flush()

```

### c. items.py
```
import scrapy
from scrapy import Field


class KatiebotItem(scrapy.Item):
    # define the fields for your item here like:
    # name = scrapy.Field()
    pass


class TagItem(scrapy.Item):
    number = Field()
    tags = Field()
    model = Field()
    co = Field()
    link = Field()
    title = Field()


class MetaItem(scrapy.Item):
    name = Field()
    pid = Field()
    info = Field()
    album = Field()
    time = Field()


class ImageItem(scrapy.Item):
    image_urls = Field()
    images = Field()
    image_paths = Field()
    referer = Field()
    pid = Field()
    retry = Field()


class SrcItem(scrapy.Item):
    pid = Field()
    referer = Field()
    image_url = Field()

```

使用scrapy框架确实省去了很多事情，框架本身的一些网络错误处理，以及pipeline处理让写爬虫变得非常简单。
