---
layout: post
title: "深入Solr实战"
date: 2012-10-15 15:03
comments: true
categories:
---

{% img right http://0.gravatar.com/avatar/084c149bb5a1769082ef794c6dcd1a91 %}

solr为我们提供便捷的全文检索服务,查询标准,更新容易,部署方便.我们来介绍一下如何更好的使用solr来为我们服务

<!-- more -->

新建solr服务
============

### 新建solr服务时,需要大家提供以下信息:

1. servicename

2. 预计查询、更新频率

3. 多少文档数，每个文档有多少field,field的类型，有多少field是store的

4. 常用查询语句

servicename是提供solr服务的url的一部分

预计查询频率是指访问高峰期solr每分钟的请求次数

更新频率是指全天solr更新最频繁的时段,每分钟会提交多少个文档更新

每个文档有多少个field,每个field的类型,是否是store的,可以预估solr服务的磁盘及内存消耗

提供常用的查询语句,可以纠正一些不正确的写法,预估cpu的消耗

### 一些建议:

所有业务字段除去id以外都不要store

为了调试和排查错误方便,可以有一个updatetime的store字段,用于记录当前solr数据被update的时间

#### 旁白:

关于字段store的测试,我们使用prop00服务,当时有245w的文档.我们将所有字段,除ID外进行了store和非store的调整,得到了下面的测试结果.

<table>
<thead>
<tr>
<th></th>
<th>非store</th>
<th>store</th>
</tr>
</thead>
<tbody>
<tr>
<td>索引文件大小</td>
<td>767m</td>
<td>1552m</td>
</tr>
<tr>
<td>VSZ</td>
<td>2527m</td>
<td>2608m</td>
</tr>
<tr>
<td>RSS</td>
<td>2185m</td>
<td>2194m</td>
</tr>
<tr>
<td>10-11点90%时间</td>
<td>119ms</td>
<td>179ms</td>
</tr>
</tbody>
</table>

**想速度快一点吗?非store吧**

使用solr服务
============

solr服务对于应用来说,就是更新和查询.而更新大家都知道通过包含unique的id的xml文件进行单个或者批量更新,这里说得更多的是如何写一个快一点的查询

## 理论知识(可以跳过)

### 1. start

和数据库一样，使用大的start会对solr造成很大的压力，因此，start一定要限制在很小的范围内。

如 ajk-qa 有以下查询：

```text
fl=id&sort=quetime+asc&start=557550&q=*:*&wt=json&rows=50
```

这句话查询时间要1.5s左右。

<u>**结论:类似db的游标翻页,通过quetime>某个值来过滤，减少start的值。**</u>

### 2. 关键词查询时间复杂度

#### 2.1 单个term查询:

假设 n为命中的doc数量，k为寻找前面k条记录，对应solr的start+rows

最糟糕的情形下，时间复杂度为 n * log(k) 其中log(k)主要为调整最小堆所花费的时间。

最好的情形下：为n，命中的doc的score都一样，无需调整堆。

<u>**结论：单个term性能主要决定于k，若k很大，会对性能有较多影响。**</u>

#### 2.2 多个term or查询：

假设 n为单个term平均命中doc数量，m为term个数，k为寻找前面k条记录

最糟糕的情形下：

```text
n*m + n*m*log(m) + n*m + n * m * log(k) = n * m (log(m) + log(k))
```

较前面的单个term查询，多出来log(m)，其为调整各个term对应的scorer最小堆所需的时间。

最好的情形下：

```text
n*m， 一样的道理，两个堆都无需调整。
```

<u>**结论：多个term or时多了log(m)，因此term的数量不宜过大。**</u>

#### 2.3 多个term and查询：

假设 n为单个term平均命中doc数量，ni为第i个term命中的文档数，m为term个数，N 为最终命中的doc数量，k为寻找前面k条记录

最糟糕的情形下：

```text
N*log(k) + max(ni)
```

最好的情形下：

```text
N + min(ni)
```

找到N条记录后，选择k条最前面的记录时，堆也不用调整， 各个term的对应的doc都差不多分布集中在一起，即

```text
min(docend-docbegin) = n
```

<u>**结论：多个term and时和单个term基本一样。**</u>

### 3.facet query

#### 3.1 facet.field: n 为命中的doc数量

基本流程是：solr里面遍历q命中的n个文档，然后统计出各个field value对的数量,

#### 3.2 facet.query: 同搜索，取决于facet的查询，将facet.query查出来的文档集合和q查出来的文档集合做交集，然后统计个数。

注：由于这个facet.field需要从索引中获得field对应的term信息，因此使用facet.field并不一定比facet.query快。

<u>**结论：facet query和q一样需要注意性能问题。**</u>

### 4.function boost

和一般查询一样，只是修改了scorer，没有特别增加的时间复杂度

<u>**结论：不是复杂的function boost对性能影响不大。**</u>

### 5.fq的查询过程

#### 5.1 查询过程

solr这边fq会先从索引中查出对应的文档集合,每个fq都有对应的filtercache，然后做交集,然后从这些文档中去检索满足q条件的doc.

#### 5.2 fq 特点：

fq有filtercache。q没有cache

对最终的结果来说，fq不计算分数。q会计算

fq在单个term查询时效率优于q，因为不需要计算分数, solr直接调用lucene的接口，获得倒排表命中的文档.fq在除去单个term查询的情况以外, 则调用lucene的search接口，效率都和q一样

多个非term查询会导致solr调用lucene的多次search，会非常慢，应该合并。

#### 5.3 多个fq查询和fq里面多个查询AND的区别如下：

多个fq 进行 and时，使用的lucene的OpenBitSet(java中的BitSet)对fq结果集做并集.

多个fq是每个fq查出来的结果后进行交集

fq里面多个查询是一边查询一边做交集，如果fq有一个查询对应的文档数很少，建议使用fq多个查询AND

#### 5.4 使用fq时的建议：

如果fq里面有非term查询，建议将非term查询合并在一起

如果是仅仅根据单个term过滤结果，又不需要排序，建议使用`fq，q=*:*`

合理精简条件，solr不会对fq优化，去除一个fq可以少一次查询. fq如果并不能起到过滤文档的作用,请不要放在url中.例如:查询ajk-prop11的url中fq=city_id:11就可以拿掉

如果查询中有出现非term查询,可以将非term查询合并到当前查询中命中文档数最少的fq中

### 6. q里面的and查询过程

#### 6.1查询时间复杂度的计算

lucene在score nextdoc的时候，就对根据term找到的doc做并集处理，该项动作的时间复杂度最好为O(N) N为满足所有条件的文档数，其实就是 docend-docbegin

选择k个文档的时间复杂度是一样的： `N * log(k)`， `N`为满足所有条件的文档数

#### 6.2 使用q时的建议:

功能上，fq没有score的功能，而q里面可以score。

效率上，fq 和 q查询的差异在于找到满足条件文档的时间复杂度，一个是`O(min(n)) * m + O(Nf)`，一个是`O(N)`，如果fq的结果是经常变的，建议使用q里面的and条件。

### 7. 关于dismax的mm

`mm=2`表示q里面的字句任意两个匹配就返回,造成大量的term查询

### 8. 范围查询

范围查询`fq=hpstarttime:[* TO 1351354156] OR hpendtime:[1351354156 TO *]`,范围越大越耗时,这种查询对速度影响非常大,且从查询语句上没有任何办法优化.

lucene存的都是针对term的倒排表,即不是数值类型而是多个字符串，不像数据库那样会有某个字段的index.

schema中可以调整`precisionStep=4`的值,默认是4或者没有设置,这个值越大则索引小性能差，越小则索引大性能理论上的好,精度高(目前还未测试)

建议:

> 缩小范围，这是最直接最管用的方法

> 不要使用`*`而是根据业务使用一个具体的数值,比如当前时间

#### 9. 其他建议

solr的机器的内存使用率不能超过80%

optimize虽然官方声明是deprecated的,但是实际用下来还是需要继续

通过`solr/admin/luke?fl=normalanswerstr&numTerms=100`分析term,过滤掉没有意义的term


附：什么是单个term查询：就是指定field 以及 value，且没有二元逻辑运算符，如 `city_id:14`， 注意：`field:(val1 OR val2)` 不是单个term查询。

## 结论(精华)

<u>**结论:**</u> start+rows过大会严重影响solr的查询速度.

解决方案:类似db的游标翻页,通过加入其他过滤条件(如`id>某个值`)`quetime>`，减少start的值.

建议:start值`+rows`的值不要超过2000

---

<u>**结论:**</u> term数量在or查询中严重影响solr的查询数度

解决方案:减少term的个数

建议:term数量控制不超过10个

---

<u>**结论**</u>:每个fq都会有filtercache,对于更新太频繁的solr没有效果

解决方案:减少update频率,减少fq的查询条件数目

建议:daily build,减少自定义条件查询,或者将fq放到q的and条件中

---

<u>**结论:**</u> q可以计算分数,而fq不会,对于不需要score排序的查询,可以不使用q

解决方案:非score排序,可以使用q=*:*&fq=field1:value1而不是q=field1:value1

---

<u>**结论:**</u>多个fq查询,先对每个fq进行查询再做交集.而在fq里做多个and查询,会先优化查询顺序然后再边查询边交集查询

建议:如果fq中有一个可以快速过滤文档的查询,可以将其他fq查询合并入此fq中

---

<u>**结论:**</u> 每次非term查询,会使solr调用lucene的search

建议:如果有多个fq进行非term查询,建议合并(但是合并会导致fq的filtercache命中率低,需要平衡,另外可以结合上面fq and查询结论,合并到一个可以快速过滤文档的fq查询中)

---

<u>**结论:**</u> solr不会对多个fq进行优化,只会对单个fq内部优化

建议:对不过滤文档的fq可以去除

---

<u>**结论:**</u> dismax的mm参数,是从q中取出制定数据的关键字进行组合然后进行or查询,会造成大量的term查询

解决方案:`mm=1%`和`mm=99%`是一样的情况,只有拿掉mm使成为and查询

---

<u>**结论:**</u> solr的数值在存储是依然使用的是若干字符串进行存储,所以范围查询中范围越大越耗时

解决方案:缩小查询范围

建议:不要使用`*`,而是用一个根据业务使用一个具体的值

---

参考资料

+ [http://stackoverflow.com/questions/6462350/is-filtering-faster-than-querying-in-lucene](http://stackoverflow.com/questions/6462350/is-filtering-faster-than-querying-in-lucene)

+ [http://xangqun.iteye.com/blog/686840](http://xangqun.iteye.com/blog/686840)

+ [http://www.wanghd.com/blog/2012/09/21/solrde-shu-zhi-cun-chu-he-fan-wei-cha-xun/](http://www.wanghd.com/blog/2012/09/21/solrde-shu-zhi-cun-chu-he-fan-wei-cha-xun/)

+ [http://hadoopcn.iteye.com/blog/1550402](http://hadoopcn.iteye.com/blog/1550402)

