<head><font face=”黑体“ size=7>Mysql索引（一）</head>
<h2><font face="宋体" size =5>1️⃣为什么要加索引<h2>
<a>一句简单的话来讲，<font color ="red">索引的出现是为了提高数据查询的效率，就像书的目录一样，索引就是数据库的目录。</font></a>

<h2><font face="宋体" size =5>2️⃣索引的常见模型<h2>
<a>常见的索引模型有以下三种：</br></a>
<a>1.哈希表<br/>
<a>哈希表是一种以（key-value）存储的数据结构，输入待查询的的值key，即可找到相应的值value。<br>
哈希的思路很简单，把值放在数组里，用一个哈希函数把key换算成一个确定的位置，然后把value放在数组的这个位置。但是会出现多个key值经过计算换算出同一个确定的位置。处理这种情况的方法是，在该位置下面拉出一个链表然后依次存储各个值。<br>
<font color="red">哈希表这种结构适用于只用等值查询的场景，即 A=？<br><br>
<a>2.有序数组<br/>
<font color="red">有序数组在等值查询和范围查询场景中的性能就都非常优秀。</font>
但是在需要更新数据时，由于其数据结构的原因，插入一条数据（非头插和尾插）
就需要移动所有数据，成本相当之高，所以有序数组只适用于静态存储引擎。<br>
<br>
<a>3.搜索树<br/>
......


<h2>3️⃣InnoDB引擎中基于主键索引和普通索引查询的区别</h2>

主键索引：即以主键作为索引，<font color="red">其叶子节点存储的是整行数据</font>。也被叫做聚簇索引。<br>

<br>
非主键索引：即不以主键作为索引的索引，<font color="red">非主键索引的叶子节点内容是主键的值</font>，也被叫做二级索引。

<br>

案例：
<br>SELECT * FROM T WHERE ID = 500 （ID为主键，且ID为主键索引）<br> SELECT * FROM T WHERE k = 5 (k为非主键索引)<br>

二者执行的区别:<br>

1.主键查询方式，只需要搜索ID这课B+树即可得到结果（因为ID的叶子节点中存储的是整行数据）<br>

2.非主键索引方式，先搜索k这课搜索树得到主键ID，然后再根据ID搜索ID这棵B+树（该过程叫做回表），得到结果。<br>

结论:

基于非主键索引的查询需要多扫描一颗索引树，因此，我们在应用中应该尽量使用主键查询。

<h2>4️⃣索引维护</h2>
B+树为了维护索引的<font color="red">有序性</font>,在<font color="red">插入新值</font>的时候需要做必要的维护（页分裂和页合并）。

页分裂：当某一个数据页已经满了，根据B+树的算法，这时候需要申请一个新的数据页，然后挪动部分数据过去。（这种情况性能会受到影响，除此之外，页分裂操作还影响数据也的利用率。原本放在一个数据页的数据，现在分到两个数据页中，整体空间利用率降低大约50%）

页合并：当两个相邻的数据页由于删除了数据，利用率很低之后，会将数据页做合并。合并的过程可以认为是分页过程的逆过程。


B+树索引由于进行页分裂操作，可能是出现碎片化索引的情况，碎片化的索引可能会以很差或者无序的方式存储在磁盘上，这会降低查询效率。

针对索引的碎片化，新版的InnoDB中添加了在线添加和删除索引的功能，可以通过先删除，然后再重建新索引的方式来消除索引的碎片化。（当然还有数据碎片化的问题，后期笔记会进行详细记录）。也可以通过开启expand_fast_index_creation参数的percona server，按照这种方式重建表，则会同时消除表和索引的碎片化。

总结：<br>
索引碎片的解决办法：<br>
&ensp;&ensp;1.先删除旧索引，然后添加新的索引<br>
&ensp;&ensp;2.开启expand_fast_index_creation参数的percona server 重建表

<h2>基于索引维护的自增主键的选用场景的讨论</h2>

&ensp;&ensp;自增主键：值自增列上定义的主键。<br>
&ensp;&ensp;例子语句： NOT NULL PRIMARY KEY AUTO_INCREMENT<br>

&ensp;&ensp;自增主键在插入新纪录的时候可以不指定该列的ID值，系统会获取当前ID值的最大值加1作为下一条记录的ID值。
也就是说，自增主键的插入数据的模式是一种递增插入的场景（每插入一条新纪录，都是追加操作，都不挪动其他记录，这样也就不会触发叶子节点的分裂）(注：B+树的分裂策略：按照原页面中50%的数据量进行分裂)。<br>
&ensp;&ensp;由业务逻辑的字段做主键，则往往不容易保证<font color="red">有序插入</font>,这样写数据的成本相对比较高。<br>
&ensp;&ensp;从存储空间来看，主键长度越小，普通索引的叶子节点就越小，普通索引占的空间也就越小。<br>
&ensp;&ensp;综上所述：从性能和存储空间方面考量，自增主键往往是更合理的选择。


<h2>实战推荐</h2>
&ensp;&ensp;1.除在特定情境外，设立自增主键，并使用主键索引，<font color="red">尽量使用主键查询</font>,这样可以避免每次查询需要搜索两颗树；<br>
&ensp;&ensp;2. 在业务场景为KV场景，即只有一个索引，该索引必须为唯一索引，此时使用业务字段直接做主键，由于没有其它索引，所以也不用考虑其它索引的叶子节点大小的问题。










