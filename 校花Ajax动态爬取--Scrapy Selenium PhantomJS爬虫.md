## 头条美女/校花图片Ajax动态爬取--Scrapy Selenium PhantomJS爬虫

在前一篇中，通过分析Ajax动态加载的内容，构造相似的url请求获取微博图片信息。
这次爬取则将PhantomJS和Selenium集成到Scrapy中，PhantomJS是一个headless browser，
模拟浏览器向服务器发送请求。Selenium是一个web自动化框架，它主要用途是来测试web应用，不过也有很多人用它来做爬虫。

```
1. 利用Selenium PhantomJS动态加载JS
2. 爬取头条原始图片
```

### 1. 首先还是对头条进行分析：

首先打开今日头条首页，在搜索框中搜索要下载的内容，这里搜索“校花”，如下图：

<img src="http://img.meinvce.com/tech/toutiao1.png" height="500px" hspace="50px">

用Chrome Developer Tool分析网页可以看到加载了很多内容。
其中有一个XHR文件，查看文件Headers如下，该文件是浏览器向服务器发送如下请求后

```
Request URL: https://www.toutiao.com/search_content/?offset=0&format=json&keyword=%E6%A0%A1%E8%8A%B1&autoload=true&count=20&cur_tab=1&from=search_tab
```

返回的一个JSON文件，JSON文件里存放了页面加载的图片信息。

<img src="http://img.meinvce.com/tech/toutiao2.png" width="700px" hspace="50px">

### 2. 通过Selenium和PhantomJS模拟我们查看搜索结果

当然，分析到这里，如果自己根据这个请求手动构建请求，一样也能够获取图片信息。
具体做法参照之前的博客： [微博图片/相册爬虫](http://www.meinvce.com/tech/2018-09-19/6594.html)

但是现在我们不用自己构造这个请求，而是通过Selenium和PhantomJS模拟我们查看搜索结果，
然后将以加载内容的页面返回给Scrapy进行解析。


### 3. 先观察页面，不停的将页面往下翻
发现页面到最底部时，会自动加载新的页面，根据这个特性，在Selenium中有一个方法scrollTo可以模拟往下翻的动作。

<img src="http://img.meinvce.com/tech/toutiao3.png" height="500px" hspace="50px">

```
driver = webdriver.PhantomJS()
print('PhantomJS is starting...')
driver.get(request.url)  #加载页面
print("页面渲染中····开始自动下拉页面")
driver.execute_script("window.scrollTo(0, height)")
```

通过改变height值，可以模拟滚动上下翻的功能。
下面的代码模拟不停翻，直到没有新内容加载为止：

```
driver = webdriver.PhantomJS()
print('PhantomJS is starting...')
driver.get(request.url)
print("页面渲染中····开始自动下拉页面")
lastHeight = driver.execute_script("return document.body.scrollHeight")
indexPage = 0
driver.get_screenshot_as_file("{}/{}.png".format(screenshot_path, indexPage))  # 快照功能，将页面加载内容快照保存
while True:
    driver.execute_script("window.scrollTo(0, document.body.scrollHeight)")  # 开始翻页
    driver.implicitly_wait(15)   # 等待15秒
    newHeight = driver.execute_script("return document.body.scrollHeight")   # 获取新的高度
    if lastHeight == newHeight:    # 已经翻到最后了
        break
    lastHeight = newHeight
    indexPage += 1
    driver.get_screenshot_as_file("{}/{}.png".format(screenshot_path, indexPage))
    time.sleep(3)
    print('PhantomJS is visiting'+request.url)
body = driver.page_source
```

翻页快照内容如下：

<img src="http://img.meinvce.com/tech/toutiao4.png" height="500px" hspace="50px">

### 3. 解析内容
内容解析跟普通爬虫一样，用scrapy的xpath获取需要的内容，再传给peipeline进行下载。代码见最后。
下面是下载的一些图片内容：

<img src="http://img.meinvce.com/tech/toutiao5.png" height="500px" hspace="50px">

### 代码：
### spider.py
```
# -*- coding: utf-8 -*-
from scrapy.spiders import CrawlSpider
from scrapy.http import Request
from scrapy.selector import Selector
from ..items import ImageItem, SrcItem, MetaItem
from ..rules import default_headers
import re
from urllib.parse import urljoin

"""
设置页面编码
"""


class LindabotSpider(CrawlSpider):
    name = "ajaxspider"
    allowed_domains = ["toutiao.com", "pstatp.com"]
    parsed = set()

    def start_requests(self):
        urls = ['https://www.toutiao.com/search/?keyword=%E6%A0%A1%E8%8A%B1']
        for url in urls:
            yield Request(url, headers=default_headers, meta={"dont_retry": False, "max_retry_times": 5, "selenium": True},
                          callback=self.parse)

    def parse(self, response):
        print(response.url)

        hxs = Selector(response)

        if re.match('https://www.toutiao.com/search/', response.url):

            links = hxs.xpath('//div[@name="feedBox"]/div[@class="sections"]/div')

            print(len(links))

            for link in links:
                # print(link)
                href = link.xpath('//a[@class="img-wrap"]/@href').extract()
                # print(href)
                for src in href:
                    src = urljoin(response.url, src)
                    if src not in self.parsed:
                        print(src)
                        yield Request(url=src, headers=default_headers, meta={"dont_retry": False, "max_retry_times": 5,
                                      "selenium": False}, callback=self.parse)
                    self.parsed.add(src)
        elif re.match('https://www.toutiao.com/a', response.url):

            links = hxs.xpath('//div[@class="imageList"]/ul/li')

            if len(links) > 0:

                title = ' '.join(hxs.xpath('//title/text()').extract())

                print(title)

                pid = response.url.split('/')[-1]
                image_item = ImageItem()
                image_item['image_urls'] = []
                image_item['referer'] = response.url
                image_item['pid'] = pid

                meta_item = MetaItem()
                meta_item['pid'] = pid
                meta_item['info'] = title

                for link in links:
                    print(link)
                    href = link.xpath('.//div[@class="image-item-inner"]/a/@href').extract_first()

                    image_item['image_urls'].append(href)

                    src_item = SrcItem()
                    src_item['pid'] = pid
                    src_item['referer'] = response.url

                    src_item['image_url'] = href

                    yield src_item

                yield meta_item
                yield image_item
            else:
                print('Dropped url: {}'.format(response.url))
```

### middleware.py

```
class LindabotJSMiddleware(object):

    def process_request(self, request, spider):

        if request.meta.get('selenium') is None:
            return
        if request.meta.get('selenium'):
            driver = webdriver.PhantomJS()
            print('PhantomJS is starting...')

            driver.get(request.url)

            print("页面渲染中····开始自动下拉页面")

            lastHeight = driver.execute_script("return document.body.scrollHeight")

            indexPage = 0

            driver.get_screenshot_as_file("{}/{}.png".format(screenshot_path, indexPage))

            while True:
                driver.execute_script("window.scrollTo(0, document.body.scrollHeight)")

                driver.implicitly_wait(15)

                newHeight = driver.execute_script("return document.body.scrollHeight")

                if lastHeight == newHeight:
                    break

                lastHeight = newHeight

                indexPage += 1

                driver.get_screenshot_as_file("{}/{}.png".format(screenshot_path, indexPage))

                time.sleep(3)

                print('PhantomJS is visiting'+request.url)
            body = driver.page_source

            return HtmlResponse(driver.current_url, body=body, encoding='utf-8', request=request)
        else:
            driver = webdriver.PhantomJS()
            print('{} 加载中...'.format(request.url))

            driver.implicitly_wait(15)

            driver.get(request.url)

            driver.get_screenshot_as_file("{}/{}.png".format(screenshot_path,
                                                             hashlib.sha1(request.url.encode('utf-8')).hexdigest()))

            body = driver.page_source

            return HtmlResponse(driver.current_url, body=body, encoding='utf-8', request=request)
```
