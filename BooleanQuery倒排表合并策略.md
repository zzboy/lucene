Lucene中基本的Query类型有以下几种：
```
TermQuery,WildcardQuery,PhraseQuery,PrefixQuery,MultiPhraseQuery,FuzzyQuery,RegexpQuery,TermRangeQuery,NumericRangeQuery,ConstantScoreQuery,DisjunctionMaxQuery,MatchAllDocsQuery
```
除此以外还有一种特殊的BooleanQuery，用户可以通过该Query将查询组装成树状结构。举例如下：<br/>
+title:lucene +(+title:lucene title:luc* +title:lucene~2)<br/>
上面的查询语句表示成树的形式：<br/>
```
|__+ title:lucene(TermQuery)
|__+|__+ title:lucene(TermQuery)
    |__  title:luc*( PrefixQuery)
    |__+ title:lucene~2(FuzzyQuery)
```
+表示MUST，没有前缀表示SHOULD，-表示MUST_NOT，对应到BooleanQuery中，MUST表示该Term在Document中必须出现，SHOULD表示可以出现也可以不(影响打分)，MUST_NOT表示必须不出现。

对于PrefixQuery，查询之前，需要将其重写成TermQuery的组合，但是当满足Prefix的Term数量大于阀值或者满足条件的倒排文档数量大于阀值时，就不会将其重写成TermQuery的组合，
此时会使用PrefixQuery构造一个QueryFilter，使用此过滤器对Term进行过滤(term.StartWith(prefix))，同理FuzzyQuery也将被重写。<br/>
在我测试的例子中，ReWrite结束之后，查询树变成：
```
|__+ title:lucene(TermQuery)
|__+|__+ title:lucene(TermQuery)
    |__  title:luc*( PrefixQueryFilter) 
    |__+ | title:lucek(TermQuery)
         | title:lucene(TermQuery)
         | title:lucewe(TermQuery)
```
ReWrite从顶向下重写，可以看成是对Query树的遍历过程。<br/>
忽略打分过程，接下来就是合并倒排表了。<br/>
Lucene首先自底向上将同一父亲下的子节点归类，所有MUST的放在一起，SHOULD的放在一起，MUST_NOT的放在一起，这样做的目的是为了方便合并倒排表。<br/>
对于MUST，倒排表取交集，SHOULD取并集，MUST_NOT内部取交集，之后再和其它的取差集。<br/>
这里需要注意的是取出来的doc需要保序，对于单独的Term，其倒排表本身是有序的，因此lucene实现并集用了小顶堆，交集就是一次遍历。<br/>
MUST、SHOULD、MUST_NOT不同情况的组合会影响查询结果，(MUST|MUST_NOT)&SHOULD时SHOULD只会影响打分，其它的组合没什么好说的。<br/>

Lucene实现倒排表没有使用bitmap，为了效率，lucene使用了一些策略，具体如下：<br/>
1.	使用FST保存词典，FST可以实现快速的Seek，这种结构在当查询可以表达成自动机时(PrefixQuery、FuzzyQuery、RegexpQuery等)效率很高。(可以理解成自动机取交集)<br/>
此种场景主要用在对Query进行rewrite的时候。<br/>
2.	FST可以表达出Term倒排表所在的文件偏移。<br/>
3.	倒排表使用SkipList结构。从上面的讨论可知，求倒排表的交集、并集、差集需要各种SeekTo(docId)，SkipList能对Seek进行加速。<br/>
