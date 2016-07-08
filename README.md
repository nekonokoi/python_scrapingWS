# Scraping WS

## 全体観
- HTMLの取得:urllib,Scrapy
- HTML上のデータの取得:BeautifulSoup
- ファイルごとのモジュール:feedparserなど

## BeautifulSoup
- HTMLのタグを操作する
- タグの構造やタグの情報を調査
- タグの情報を修正

### HTMLの取得

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

### Soupオブジェクトの作成
```python
soup = bs(dat,"lxml")
```

### タグ構造の取得
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

### タグ情報の取得
```python
soup.body.div.attrs
soup.body.div.get("<attrib>")
soup.body.text
soup.body.div.prettify
```

### その他複雑なやつ
- 子ども、親、兄弟などを取ってくることも可能

## Scrapy
- リクエストの制御を行う
- クローリングのための便利機能の集合体

```bash
scrapy startproject <pjname>
```

```scrapy
<pjname>/
├scrapy.cfg            # deploy configuration file
├<pjname>/             # project's Python module, you'll import your code from here
    ├__init__.py
    ├items.py          # project items file
    ├pipelines.py      # project pipelines file
    ├settings.py       # project settings file
    ├spiders/          # a directory where you'll later put your spiders
        ├__init__.py
```

### 1. settings.py
```
REDIRECT_MAX_TIMES = 6
RETRY_ENABLED = False
DOWNLOAD_DELAY=10
COOKIES_ENABLED=False
```

### 2. items.py
- 取得するデータの構造を定義する
- scrapy 1.0から必須ではなくなった
- 保存対象のデータをクラスとして扱いたいときに

```python
class TestSpiderItem(scrapy.Item):
    # define the fields for your item here like:
    # name = scrapy.Field()
    title = scrapy.Field()
    body = scrapy.Field()
    time = scrapy.Field()

```

### 3. spiders
- クローリングの挙動を決定する
- 事前に用意されたスパイダーがあり、それをカスタムする
- HTMLから情報を取得し、そこから新しい取得対象を発見した場合、さらに取得に行く

```
scrapy genspider [options] <name> <domain(巡回するサイトドメイン)>
```

#### スパイダーの基本情報
- name:スパイダーの名前。実行時に使用
- allowed_domain:対象のドメイン。ここから外れる場合は取得しない
- start_urls:開始されるURL

#### スパイダーの動きのルール
- rules:ルールを格納しておくリスト
- Rule:ルールオブジェクト
- LinkExtractor:次の対象とするかどうかのURLルール
- follow:対象となったリストを次に実行するか
- allow:対象とするURL（の正規表現)をリストで与える
- deny:対象としないURL(の正規表現)をリストで与える
- callback:対象だった場合に呼び出される関数名を指定

#### セレクター
- Selector():textやresponseに対象を渡す
- Selector().xpath():xpathで対象を抽出
- Selector().css():Cssセレクタで対象を抽出
- Selector().extract():対象オブジェクトをリストで返す(extract_firstで一つだけ返す)


#### レスポンス
- url
- status
- headers
- body


#### 補足
- yield:関数をジェネレータ化する

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

        # こういうデータ操作はitems.pyに書くべき
        item["body"] = "body"
        item["time"] = datetime.datetime.now()

        yield item
```

### 4.実行コマンド
- `-t`や`-o`の拡張子でフォーマット指定可能
- csv,json,xmlなど

```bash
cd pj name
scrapy crawl my_spider -o result.json -t json
```

### 5.piplines
- クローリング時のデータ保存のルールを書く
- open_spider:クローラが動き始めた時。ファイルの指定やデータベースとの接続など
- close_spider:クローリングが完了した時に実行される
- process_item:itemの取得が発生したとき:データベースへの保存など

```python
import pickle

class TestSpiderPipeline(object):
    def open_spider(self,spider):

        print "----start----"
        print "spider name:" + spider.name
        import os

        if os.path.exists("test.pickle"):
          with open("test.pickle","wb") as f:

              dat = []
              pickle.dump(dat,f)


    def process_item(self, item, spider):
       # 本来はバッチ化してメモリに格納しておくべき

        with open("test.pickle","rb") as f:
            dat = pickle.load(f)

        dat.append(item)

        with open("test.pickle","wb") as f:
            pickle.dump(dat,f)

        return item
```

```
settings
ITEM_PIPELINES = {
    'meetpy.pipelines.TestSpiderPipeline': 300,
}
```

## 参考文献
[BeautifulSoup](https://www.crummy.com/software/BeautifulSoup/bs4/doc/)
[scrapy](http://scrapy.org/)
[rubyによるクローラ開発技法](https://www.amazon.co.jp/Rubyによるクローラー開発技法-巡回・解析機能の実装と21の運用例-佐々木-拓郎/dp/4797380357/)
[python webスクレイピング](https://www.amazon.co.jp/PythonによるWebスクレイピング-Ryan-Mitchell/dp/4873117615/)

## おまけ
### formで最初にログインしたい
spiderで`start_requests`を書いておく
```python
def start_requests(self):
        return [scrapy.FormRequest("http://www.example.com/login",
                                   formdata={'user': 'john', 'pass': 'secret'},
                                   callback=self.logged_in)]

    def logged_in(self, response):
        # here you would extract links to follow and return Requests for
        # each of them, with another callback
        pass
```
※対象をDBなどからとってきて順にやりたい場合も使える

### logを書きたい
```
self.logger.<status>("<ログ内容>")
```

spiderに`log`を書いておく
```python
def log(self):
```

### spiderの種類
- CrawlSpider:巡回ルールが付いている
- XMLFeedParser:XMLを巡回する
- CSVFeedParser:csvを操作する(delimiter,quotechar,headers,row)
- SitemapSpider

```
<order>
  <date>2016-07-08</date>
  <order_user>nekonokoi</order_user>
  <order_detail>
    <detail_no>0</detail_no>
    <item_no>1111</item_no>
    <cnt>2</cnt>
  </order_detail>
  <order_detail>
    <detail_no>1</detail_no>
    <item_no>1112</item_no>
    <cnt>2</cnt>
  </order_detail>
  <order_detail>
    <detail_no>2</detail_no>
    <item_no>3112</item_no>
    <cnt>1</cnt>
  </order_detail>
</order>
```
```
itertag="order_detail"
```
### リクエストとレスポンスにふっくしたい
- middlewareを書いてsettingで取り込む

### 設定ファイルに情報を書いておきたい
- piplineにクラスメソッドでfrom_crawler(cls,crawler)を書いておく
- clsをリターンする

### spiderをAPI化する
```python
process = CrawlerProcess({
    'USER_AGENT': 'Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 5.1)'
})

process.crawl(MySpider)
process.start() # the script will block here until the crawling is finished

```

### spiderを並列化する
```
runner = CrawlerRunner()
runner.crawl(MySpider1)
runner.crawl(MySpider2)
d = runner.join()
d.addBoth(lambda _: reactor.stop())

reactor.run()
```

### サーバー上でデーモン化したい
- scrapydを入れる
- scrapy.cfgのurlをscrapydのurlに変える
- `scrapy deploy`
- httpでapiを叩く
[参考](http://orangain.hatenablog.com/entry/scrapyd)


### インタラクティブにスクレイピングする
```
scrapy shell http://doc.scrapy.org/en/latest/_static/selectors-sample1.html
```

### webAPI化する

[jsonrpc](https://github.com/scrapy-plugins/scrapy-jsonrpc)


### ファイルを扱う
[参考](http://doc.scrapy.org/en/latest/topics/media-pipeline.html)

### メールを送信する
```
from scrapy.mail import MailSender
mailer = MailSender()
mailer.send(to=["someone@example.com"], subject="Some subject", body="Some body", cc=["another@example.com"])
```

### リクエストを操作したい
- `make_requests_from_url(url)`をスパイダー内に追加する
- `Request`オブジェクトをリターンする

#### リクエストオブジェクト
- url
- cookies
- method:http method
- body,headers
