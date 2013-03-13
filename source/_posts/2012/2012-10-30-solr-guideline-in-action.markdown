---
layout: post
title: "深入Solr实战 - 实例篇"
date: 2012-10-30 10:29
comments: true
categories:
---

{% img right http://0.gravatar.com/avatar/084c149bb5a1769082ef794c6dcd1a91 %}

之前我们有提到[Solr的使用的一些规范][1]，这次我们深入实例。

我们从access log中找出一些查询耗时比较高的查询,跟去"深入solr实战"中提到的修改方式,做了一些更改,查询结果没有变化但是速度出现下降

<!-- more -->

测试环境:在cloud中不对外服务的solr instance,关闭filtercache，测试以下查询：

96ms左右

```
http://10.10.6.34:7731/solr/select/?
start=0&
q=commid:51087&
qt=standard&
fq=city_id:14&
fq=hpstarttime:[0+TO+1347292898]&
fq=hpendtime:[1347292898+TO++2147483647+]&
fq=%28chpratio:3897.99871+AND+hpplanid:[0+TO+689944]%29+OR+one_chpratio:[3897.99872+TO+2147483647]&
rows=0&
version=2.2
```

改成

40ms左右

```
http://10.10.6.34:7731/solr/select/?
start=0&
q=\*:\*&
qt=standard&
fq=commid:51087&
fq=hpstarttime:[0+TO+1347292898]%20AND%20hpendtime:[1347292898+TO++2147483647+]%20AND%20%28%28chpratio:3897.99871+AND+hpplanid:[0+TO+689944]%29+OR+one_chpratio:[3897.99872+TO+2147483647+]%29&
rows=0&
version=2.2
```

解析:

1. 这个instance存放的都是北京房源,所以拿掉了city_id:14的fq
2. hp的一系列fq首先不是term查询,其次相关的参数几乎时刻都在变化无法吃出cache,所以进行了合并
3. 不需要计算score,所以让q=*:*
4. commid相对固定,容易吃住cache,所以放在fq而没有合并到hp的查询条件中
5. hp的fq过滤后没有文档,所以commid是否合并如hp的fq对速度没有影响


169ms左右

```
http://10.10.6.34:7731/solr/select/?
fl=score,*&
sort=order_string+desc,rank_level+asc,score+desc,rank_sub_level+asc,rank_score+desc&start=0&
q=_val_:%22map%28rank_score2,4683.3,10000,0,1%29%22&
qt=standard&
hl=false&
fq=city_id:14&
fq=commid:50815&
fq=islist:1&
fq=hpendtime:[0+TO+1347292823]+OR+hpstarttime:[1347292823+TO+*]
```

改成

35ms左右

```
http://10.10.6.34:7731/solr/select/?
fl=score,*&
sort=order_string+desc,rank_level+asc,score+desc,rank_sub_level+asc,rank_score+desc&
start=0&
q=_val_:%22map%28rank_score2,4683.3,10000,0,1%29%22&
qt=standard&
hl=false&
fq=commid:50815%20AND%20islist:1%20AND%20hpendtime:[0+TO+1347292823]+OR+hpstarttime:[1347292823+TO+2147483647]
```

解析

1. 同上1,去掉city_id:14的fq
2. 同上2,合并fq
3. hp的fq过滤后有大量文档,所以合并commid可以明显提升速度

[1]: /blog/2012/10/15/solr-guildeline/
