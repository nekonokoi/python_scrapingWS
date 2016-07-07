# Scraping

## BeautifulSoup

```
import urllib
from bs4 import BeautifulSoup as bs
```

```python
url = ""
html = urllib.urlopen(url).read()

with open("test.html","wb") as f:
  f.write(html)

with open("test.html","rb") as f:
  dat = f.read()
  print dat
```

```python
soup = bs(dat,"lxml")
```

```python
soup.body

soup.body.children#iterator
soup.body.findChildren()#list
soup.body.findChild()#first

```

```python
soup.findAll("div")
soup.find("div")

```

```python
soup.body.div.attrs
soup.body.text
soup.body.div.prettify
```



```


from bs4 import BeautifulSoup
import urllib
from bottle import *

def get_dom(url):

    url = url

    html = urllib.urlopen(url).read()
    return BeautifulSoup(html)

def scraper(articles,url_func,desc_func):
    data = []
    for art in articles:
        try:
            tmp = {}
            tmp["url"] = url_func(art)
            tmp["desc"] = desc_func(art)
            data.append(tmp)
        except:
            pass

    return data


def hateblo():
    url = "http://hatenablog.com/k/keywordblog/Python"
    soup =get_dom(url)
    article_list = soup.findAll("article")

    url_func = lambda x : x.a["href"]
    desc_func = lambda x :x.text


    return scraper(article_list,url_func,desc_func)


def qiita():
    url = "https://qiita.com/tags/Python/items"
    soup =get_dom(url)

    article_list = soup.body.findAll("div",attrs = {"class":"publicItem_body"})

    url_func = lambda x: "http://qiita.com" + x.a.get("href")
    desc_func = lambda x :x.a.text

    return scraper(article_list,url_func,desc_func)

def hatenabook():
    url = "http://b.hatena.ne.jp/search/text?q=python"
    soup = get_dom(url)

    article_list = soup.findAll("div",attrs={"class":"entryinfo"})

    url_func = lambda x :x.findParent().h3.a.get("href")
    desc_func = lambda x:x.findParent().h3.text

    return scraper(article_list,url_func,desc_func)

def main():
    services = {
        "hatena blog":hateblo,
        "qiita":qiita,
        "hatena bookmark":hatenabook
    }
    data = []
    for name,func in services.items():

        try:
            tmp = {}
            tmp["service"] = name
            tmp["content"] = func()

            data.append(tmp)
        except:
            pass

    return data



@route("/")
def server():
    data = main()
    html = """
    <html>
        <header>
        </header>
        <body>
            %for d in data:
                <h1>{{d["service"]}}</h1>
                    <ul>
                    %for c in d["content"]:
                        <li>
                            <a href = "{{c["url"]}}">
                                {{c["url"]}}
                            </a>:
                            {{c["desc"]}}
                        </li>
                    %end
                    </ul>
        </body>
    </html>
    """

    return template(html,data = data)

run(host = "localhost",port=8080)
```


## Scrapy

### 1. settings.py

### 2. items.py
```python
class TestSpiderItem(scrapy.Item):
    # define the fields for your item here like:
    # name = scrapy.Field()
    title = scrapy.Field()
    body = scrapy.Field()
    time = scrapy.Field()

```

### 3. spiders
- spiderの種類
- name/allowed_domain/start_urls
- rules/follow/callback(and event-driven)
- cssセレクター/xpath
- yield

```python
from scrapy.contrib.spiders import CrawlSpider,Rule
from scrapy.linkextractors import LinkExtractor

import datetime

class TestSpider(CrawlSpider):
    name = "my_spider"

    allowed_domain = ["b.hatena.ne.jp"]
    start_urls = ["http://b.hatena.ne.jp"]
    rules = [
        Rule(
            LinkExtractor(
                allow = r"/hotentry/[^/]*"
            ),
            follow=False,
            callback="parse_test"
        )
    ]

    def parse_test(self,response):

        item = {}

        item["title"] = response.css("title").extract()
        print item["title"]
        item["body"] = "body"

        item["time"] = datetime.datetime.now()

        yield item
```

### 4.実行コマンド
```bash
cd meetpy
scrapy crawl my_spider -o result.json -t json
```

### 5.piplines
- open_spider
- process_item

```python
import pickle

class TestSpiderPipeline(object):
    def open_spider(self,spider):

        print "----start----"
        print "spider name:" + spider.name
        with open("test.pickle","wb") as f:
            dat = []
            pickle.dump(dat,f)


    def process_item(self, item, spider):
        with open("test.pickle","rb") as f:
            dat = pickle.load(f)

        dat.append(item)

        with open("test.pickle","wb") as f:
            pickle.dump(dat,f)

        return item
```
settings
ITEM_PIPELINES = {
    'meetpy.pipelines.TestSpiderPipeline': 300,
}
