---
layout: post
title: 利用Python对链家网北京主城区二手房进行数据分析
---

* 本文主要讲述如何通过pandas对爬虫下来的链家数据进行相应的二手房数据分析，主要分析内容包括各个行政区，各个小区的房源信息情况。
* 数据来源:  该[repo](https://github.com/XuefengHuang/lianjia-scrawler)提供了python程序进行链家网爬虫，并从中提取二手房价格、面积、户型和二手房关注度等数据。
* 本文所用到的代码放在本人的[Github](https://github.com/XuefengHuang/lianjia-scrawler/blob/master/data/lianjia.ipynb)上，便于下载学习。
* 分析方法参考 http://www.jianshu.com/p/44f261a62c0f

## 导入链家网二手房在售房源的文件（数据更新时间2017-11-29）


```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from sklearn.cluster import KMeans

import sys

stdout = sys.stdout
reload(sys)
sys.setdefaultencoding('utf-8')
sys.stdout = stdout

plt.rcParams['font.sans-serif'] = ['SimHei']    
plt.rcParams['axes.unicode_minus'] = False

#所有在售房源信息
house=pd.read_csv('houseinfo.csv')

# 所有小区信息
community=pd.read_csv('community.csv')

# 合并小区信息和房源信息表，可以获得房源更详细的地理位置
community['community'] = community['title']
house_detail = pd.merge(house, community, on='community')
```

## 将数据从字符串提取出来


```python
# 将字符串转换成数字
def data_adj(area_data, str):       
    if str in area_data :        
        return float(area_data[0 : area_data.find(str)])    
    else :        
        return None
# 处理房屋面积数据
house['square'] = house['square'].apply(data_adj,str = '平米')
```

## 删除车位信息


```python
car=house[house.housetype.str.contains('车位')]
print '记录中共有车位%d个'%car.shape[0]
house.drop(car.index,inplace=True)
print '现在还剩下%d条记录'%house.shape[0]
```

    记录中共有车位32个
    现在还剩下16076条记录


## 价格最高的5个别墅


```python
bieshu=house[house.housetype.str.contains('别墅')]
print '记录中共有别墅%d栋'%bieshu.shape[0]
bieshu.sort_values('totalPrice',ascending=False)['title'].head()
```

    记录中共有别墅50栋
    
    8020       香山清琴二期独栋别墅，毛坯房原始户型，花园1200平米
    102               千尺独栋 北入户 红顶商人金融界入住社区
    2729    临湖独栋别墅 花园半亩 观景湖面和绿化 满五年有车库房主自荐
    3141           银湖别墅 独栋 望京公园旁 五环里 封闭式社区
    4112           首排别墅 位置好 全景小区绿化和人工湖 有车库
    Name: title, dtype: object



## 删除别墅信息


```python
house.drop(bieshu.index,inplace=True)
print '现在还剩下%d条记录'%house.shape[0]
```

    现在还剩下16026条记录


## 获取总价前五的房源信息


```python
house.sort_values('totalPrice',ascending=False)['title'].head(5)
```




    8571      中关村创业大街对过 有名的公司入驻其中正规写字楼
    11758    中关村创业大街对过 有名的公司入驻其中正规一层底商
    2480       西山别墅区拥有900平大花园纯独栋社区房主自荐
    14492         盘古大观 大平层 观景房 格局可塑性强！
    10154       朝阳公园内建筑，视野好，可以俯视朝阳公园美景
    Name: title, dtype: object



## 获取户型数量分布信息


```python
housetype = house['housetype'].value_counts()
housetype.head(8).plot(kind='bar',x='housetype',y='size', title='户型数量分布')
plt.legend(['数量']) 
plt.show()
```

![获取户型数量分布信息]({{ site.url }}/assets/9144104-4132b1de864912b2.png)

## 关注人数最多5套房子


```python
house['guanzhu'] = house['followInfo'].apply(data_adj,str = '人关注')
house.sort_values('guanzhu',ascending=False)['title'].head(5)
```




    47                   弘善家园南向开间，满两年，免增值税
    2313         四惠东 康家园 南向一居室 地铁1号线出行房主自荐
    990     远见名苑  东南两居  满五年家庭唯一住房 诚心出售房主自荐
    2331               荣丰二期朝南复式无遮挡全天采光房主自荐
    915             通州万达北苑地铁站 天时名苑 大两居可改3居
    Name: title, dtype: object



## 户型和关注人数分布


```python
fig, ax1 = plt.subplots(1,1)    
type_interest_group = house['guanzhu'].groupby(house['housetype']).agg([('户型', 'count'), ('关注人数', 'sum')])    
#取户型>50的数据进行可视化
ti_sort = type_interest_group[type_interest_group['户型'] > 50].sort_values(by='户型')    
ti_sort.plot(kind='barh', alpha=0.7, grid=True, ax=ax1)    
plt.title('二手房户型和关注人数分布')    
plt.ylabel('户型') 
plt.show()
```

![户型和关注人数分布]({{ site.url }}/assets/9144104-adfeaae925b049c8.png)

## 面积分布


```python
fig,ax2 = plt.subplots(1,1)    
area_level = [0, 50, 100, 150, 200, 250, 300, 500]    
label_level = ['小于50', '50-100', '100-150', '150-200', '200-250', '250-300', '300-350']    
area_cut = pd.cut(house['square'], area_level, labels=label_level)        
area_cut.value_counts().plot(kind='bar', rot=30, alpha=0.4, grid=True, fontsize='small', ax=ax2)    
plt.title('二手房面积分布')    
plt.xlabel('面积')    
plt.legend(['数量'])    
plt.show()
```

![面积分布]({{ site.url }}/assets/9144104-3c7317acab94caa0.png)

## 聚类分析


```python
# 缺失值处理:直接将缺失值去掉    
cluster_data = house[['guanzhu','square','totalPrice']].dropna()    
#将簇数设为3    
K_model = KMeans(n_clusters=3)    
alg = K_model.fit(cluster_data)    
'------聚类中心------'   
center = pd.DataFrame(alg.cluster_centers_, columns=['关注人数','面积','房价'])    
cluster_data['label'] = alg.labels_ 
center
```

id |关注人数 | 面积 | 房价
--- | --- | --- | ---
0 | 49.787599|134.952934|1138.906738
1 | 48.573579|256.357676|2549.974916
2 | 61.485727|74.484049|515.354447


## 北京市在售面积最小二手房


```python
house.sort_values('square').iloc[0,:]
```




    houseID                                            101102324602
    title                                      智德北巷（北河沿大街）+小户型一居+南向
    link          https://bj.lianjia.com/ershoufang/101102324602...
    community                                                  智德北巷
    years                                          中楼层(共6层)1985年建板楼
    housetype                                                  1室0厅
    square                                                    15.29
    direction                                                     南
    floor                                          中楼层(共6层)1985年建板楼
    taxtype                                          距离5号线灯市口站1113米
    totalPrice                                                  220
    unitPrice                                                143885
    followInfo                               56人关注 / 共2次带看 / 8天以前发布
    validdate                                   2017-11-29 13:23:16
    guanzhu                                                      56
    Name: 15260, dtype: object



## 北京市在售面积最大二手房


```python
house.sort_values('square',ascending=False).iloc[0,:]
```




    houseID                                            101102105035
    title                                  中关村创业大街对过 有名的公司入驻其中正规写字楼
    link          https://bj.lianjia.com/ershoufang/101102105035...
    community                                                  银科大厦
    years                                         低楼层(共22层)2004年建塔楼
    housetype                                                 1房间0卫
    square                                                  2623.28
    direction                                               东 南 西 北
    floor                                         低楼层(共22层)2004年建塔楼
    taxtype                                     距离10号线苏州街站898米房本满五年
    totalPrice                                                12000
    unitPrice                                                 45745
    followInfo                               1人关注 / 共0次带看 / 2个月以前发布
    validdate                                   2017-11-29 14:07:32
    guanzhu                                                       1
    Name: 8571, dtype: object



## 各个行政区房源均价


```python
house_unitprice_perdistrict = house_detail.groupby('district').mean()['unitPrice']
house_unitprice_perdistrict.plot(kind='bar',x='district',y='unitPrice', title='各个行政区房源均价')
plt.legend(['均价']) 
plt.show()
```


![各个行政区房源均价]({{ site.url }}/assets/9144104-63b37e70c992f2e2.png)

## 各个区域房源数量排序


```python
bizcircle_count=house_detail.groupby('bizcircle').size().sort_values(ascending=False)
bizcircle_count.head(20).plot(kind='bar',x='bizcircle',y='size', title='各个区域房源数量分布')
plt.legend(['数量']) 
plt.show()
```

![各个区域房源数量排序]({{ site.url }}/assets/9144104-bf6e4bb4f573c47b.png)

## 各个区域均价排序


```python
bizcircle_unitprice=house_detail.groupby('bizcircle').mean()['unitPrice'].sort_values(ascending=False)
bizcircle_unitprice.head(20).plot(kind='bar',x='bizcircle',y='unitPrice', title='各个区域均价分布')
plt.legend(['均价']) 
plt.show()
```


![各个区域均价排序]({{ site.url }}/assets/9144104-b7bb31b1bdd871d8.png)

## 各个区域小区数量


```python
bizcircle_community=community.groupby('bizcircle')['title'].size().sort_values(ascending=False)
bizcircle_community.head(20).plot(kind='bar', x='bizcircle',y='size', title='各个区域小区数量分布')
plt.legend(['数量']) 
plt.show()
```


![各个区域小区数量]({{ site.url }}/assets/9144104-4c5d9fb264c93624.png)


## 按小区均价排序


```python
community_unitprice = house.groupby('community').mean()['unitPrice'].sort_values(ascending=False)
community_unitprice.head(15).plot(kind='bar',x='community',y='unitPrice', title='各个小区均价分布')
plt.legend(['均价']) 
plt.show()
```

![小区均价排序]({{ site.url }}/assets/9144104-684e09b1779ebc35.png)