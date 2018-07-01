---
layout: post
title: 使用python抓取并用kibana分析数据—链家网
date: 2018-06-22 13:32:20 +0300
description: copied from:https://github.com/XuefengHuang # Add post description (optional)
img: software.jpg # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [Python, lian jia,scrawler]
---

copied from: https://github.com/XuefengHuang

本篇文章介绍如何使用python(requests+BeautifulSoup)的方法对页面进行抓取和数据提取。通过使用requests库对链家网二手房列表页进行抓取，通过BeautifulSoup对页面进行解析，并从中获取房源价格，面积，户型和关注度的数据。

# 准备工作
inital idea comes from:
http://lanbing510.info/2016/03/15/Lianjia-Spider.html

codes: 
https://github.com/lanbing510/LianJiaSpider

* 为了方便数据库建表读写，我们使用python ORM工具[Peewee](https://github.com/coleifer/peewee)。并且使用BeautifulSoup+Requests对页面进行抓取和数据提取。具体所需依赖包如下

```
appdirs==1.4.3
beautifulsoup4==4.5.3
bs4==0.0.1
lxml==3.7.3
MySQL-python==1.2.5
packaging==16.8
peewee==2.8.0
psycopg2==2.7.1
pyparsing==2.2.0
requests==2.13.0
six==1.10.0
```
* 该工具使用python对链家网页面进行数据抓取，然后存到数据库中，目前支持三种数据库类型（MYSQL，Postgres，SQLite）。由于我们是中文存储，所以在配置数据库时注意设置UTF-8格式。可以参考此[问题](https://github.com/XuefengHuang/lianjia-scrawler/issues/1)。
数据库配置如下([config.ini](https://github.com/XuefengHuang/lianjia-scrawler/blob/master/config.ini)):

```
[Mysql]
enable = True
scheme = test
host = 127.0.0.1
port = 3306
user = root
password = 

[Sqlite]
enable = False
dbname = lianjia.db

[Postgresql]
enable = False
scheme = database_name
host = db.mysite.com
user = postgres
password = secret
```

# 页面爬虫
* 开始抓取前先观察下目标页面或网站的结构，其中比较重要的是URL的结构。链家网的二手房列表页面共有100个，URL结构为http://bj.lianjia.com/ershoufang/pg9/，其中bj表示城市，/ershoufang/是频道名称，pg9是页面码。我们要抓取的是北京的二手房频道，所以前面的部分不会变，属于固定部分，后面的页面码需要在1-100间变化，属于可变部分。将URL分为两部分，前面的固定部分赋值给url，后面的可变部分使用for循环。我们以根据小区名字搜索二手房出售情况为例：

```
BASE_URL = u"http://bj.lianjia.com/"
url = BASE_URL + u"ershoufang/rs" + urllib2.quote(communityname.encode('utf8')) + "/"
total_pages = misc.get_total_pages(url) //获取总页数信息
for page in range(total_pages):
    if page > 0:
        url_page = BASE_URL + u"ershoufang/pg%drs%s/" % (page+1, urllib2.quote(communityname.encode('utf8')))

//获取总页数信息代码
def get_total_pages(url):
    source_code = get_source_code(url)
    soup = BeautifulSoup(source_code, 'lxml')
    total_pages = 0
    try:
        page_info = soup.find('div',{'class':'page-box house-lst-page-box'})
    except AttributeError as e:
        page_info = None

    if page_info == None:
        return None
    page_info_str = page_info.get('page-data').split(',')[0]  #'{"totalPage":5,"curPage":1}'
    total_pages = int(page_info_str.split(':')[1])
    return total_pages
```
* 页面抓取完成后无法直接阅读和进行数据提取，还需要进行页面解析。我们使用BeautifulSoup对页面进行解析。

```
soup = BeautifulSoup(source_code, 'lxml')
nameList = soup.findAll("li", {"class":"clear"})
```

* 完成页面解析后就可以对页面中的关键信息进行提取了。下面我们分别对房源各个信息进行提取。

```
for name in nameList: # per house loop
    i = i + 1
    info_dict = {}
    try:
        housetitle = name.find("div", {"class":"title"})
        info_dict.update({u'title':housetitle.get_text().strip()})
        info_dict.update({u'link':housetitle.a.get('href')})

        houseaddr = name.find("div", {"class":"address"})
        info = houseaddr.div.get_text().split('|')
        info_dict.update({u'community':info[0].strip()})
        info_dict.update({u'housetype':info[1].strip()})
        info_dict.update({u'square':info[2].strip()})
        info_dict.update({u'direction':info[3].strip()})

        housefloor = name.find("div", {"class":"flood"})
        floor_all = housefloor.div.get_text().split('-')[0].strip().split(' ')
        info_dict.update({u'floor':floor_all[0].strip()})
        info_dict.update({u'years':floor_all[-1].strip()})

        followInfo = name.find("div", {"class":"followInfo"})
        info_dict.update({u'followInfo':followInfo.get_text()})

        tax = name.find("div", {"class":"tag"})
        info_dict.update({u'taxtype':tax.get_text().strip()})

        totalPrice = name.find("div", {"class":"totalPrice"})
        info_dict.update({u'totalPrice':int(totalPrice.span.get_text())})

        unitPrice = name.find("div", {"class":"unitPrice"})
        info_dict.update({u'unitPrice':int(unitPrice.get('data-price'))})
        info_dict.update({u'houseID':unitPrice.get('data-hid')})
    except:
        continue
```

* 提取完后，为了之后数据分析，要存进之前配置的数据库中。


```
model.Houseinfo.insert(**info_dict).upsert().execute()
model.Hisprice.insert(houseID=info_dict['houseID'], totalPrice=info_dict['totalPrice']).upsert().execute()
```

# 数据分析
* 什么是[Kibana](https://www.elastic.co/products/kibana)：

Kibana是一个开源的分析与可视化平台，设计出来用于和Elasticsearch一起使用的。你可以用kibana搜索、查看、交互存放在Elasticsearch索引里的数据，使用各种不同的图表、表格、地图等kibana能够很轻易地展示高级数据分析与可视化。

Kibana让我们理解大量数据变得很容易。它简单、基于浏览器的接口使你能快速创建和分享实时展现Elasticsearch查询变化的动态仪表盘。安装Kibana非常快，你可以在几分钟之内安装和开始探索你的Elasticsearch索引数据—-—-不需要写任何代码，没有其他基础软件依赖。

* Mysql数据同步到Elasticsearch：
正如上面所介绍，如果想用kibana分析数据，必须把数据导入Elasticsearch。此[工具](https://github.com/siddontang/go-mysql-elasticsearch)可以帮助你轻松进行mysql到es的数据同步。

* 样例：
![alt text1]({{ site.url }}/assets/lianjia1.png)
![alt text2]({{ site.url }}/assets/lianjia2.png)
![alt text3]({{ site.url }}/assets/lianjia3.png)

更多信息请参考github[链家爬虫源代码](https://github.com/XuefengHuang/lianjia-scrawler)。
