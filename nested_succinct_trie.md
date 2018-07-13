# 模糊查询
模糊查询是全文检索中一类查询的统称，包括通配符查询、前缀查询、正则表达式查询等等，比如ap*e这一通配符查询，含义是查询以ap开头、以e结尾的所有的词。
要实现该功能，系统必须对所有见过的词构建词典，词典就等价于一个有限状态自动机，根据输入字符，跳转到下一个状态，结合查询条件以及该状态是否是terminal，判断该词是否满足查询条件。
同样模糊查询也可以等价看作一个有限状态自动机，这样模糊查询的问题就变成两个DFA求交集的过程。
常用于构建词典的数据结构是Prefix trie(前缀树)，相同前缀的词在树中将共享前缀，从而达到压缩辞典的目的。本文讨论一种Prefix trie的表示形式，Nested succinct trie，该结构有如下优点：

* 压缩率高，可以达到原始**distinct数据**的40%。
* 无需解码就可以直接使用，降低cpu开销。

下面分成四部分讨论。

* **编码Tree**：主要讨论一种succinct数据结构DFUDS，用于对树形进行编码。
* **编码Prefix trie**：这一节主要讨论扩展DFUDS，使其可以用于编码前缀树。
* **嵌套Prefix trie**：讨论进一步压缩Prefix trie编码后大小的方法。
* **Prefix trie的合并**：主要讨论多棵Prefix trie在merge过程中减少内存占用的方法，解释树形编码使用DFUDS的原因。

## 编码Tree
**DFUDS(Depth First Unary Degree Sequence)** 是一种树形结构的编码方法，是[succinct数据结构](https://en.wikipedia.org/wiki/Succinct_data_structure)的一种，如果树中有n个节点，那么使用DFUDS编码这棵树只需要2*n bit。

**树的DFUDS编码构建过程如下：**

----
从根节点深度优先遍历树，对遍历到的每个节点N，将 **(<sup>d</sup>)** 插入序列，d表示节点N的度，最后在序列的头部加上 **(**，保持序列的平衡性。

```
void Encode(string& unary, TreeNode& node)
{
	for(int i = 0; i < node.degree; ++i) {
		unary.push_back('(');
	}
	unary.push_back(')');
	for(TreeNode child: node.childs) {
		Encode(unary, child);
	}
}
TreeNode treeRoot;
string unary = "(";
Encode(unary, treeRoot);
println("DFUDS: " + unary);
```
----
**下图是树编码的例子：**

![tree.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/b1854fb94361c9f09860da285cf0d219.png)

**DFUDS编码：**

```
((() ((() ) ) ) (() ) ) 
 ——— ———— - - - ——— - -
 1   2    3 4 5 6   7 8
```
下划线对应的部分表示树中各个节点编码。

DFUDS编码方法介绍完了，接下来就是树的各种操作怎么在DFUDS编码上实现，基础的树操作有：

* **degree(v)**: v所在序列的位置表示的树节点的度。
* **child(v,i)**: v表示的节点的第i个孩子在编码中的偏移，v从0记，i从1记。
* **parent(v)**: 获取v的父亲节点在编码中的偏移。

基于上面三个操作，就可以组装出更多复杂的算法，比如树的先序遍历：

```
void preorder(string& unary, int v)
{
	println("node: " + v);
	if(unary[v] == ')') return; //has no child
	else {
		for(in i = 1; i <= degree(v); ++i) preorder(unary, child(v, i));
	}
}
string unary = "((()((())))(()))";
preorder(unary, 1);
```
接下来，我们来看上面三种树的基本操作怎么高效的计算出来。
### degree、child、parent的实现
下表列出了各种基本操作的计算方法：

|操作|计算公式|说明|
|------|------|------|
|v | select<sub>)</sub>(i - 1) + 1|表示DFUDS编码中的一个具体偏移，从0计|
|i | rank<sub>)</sub>(v - 1) + 1|表示节点编号，从1计|
|degree(v) | select<sub>)</sub>(rank<sub>)</sub>(v) + 1) - v|v所在序列的位置表示的树节点的度|
|child(v,i) | findclose(select<sub>)</sub>(rank<sub>)</sub>(v) + 1) - i) + 1|v表示的节点的第i个孩子在编码中的偏移|
|parent(v) | select<sub>)</sub>(rank<sub>)</sub>(findopen(v-1))) + 1|获取v的父亲节点在编码中的偏移|

**rank<sub>)</sub>(v)** 表示括号序列中v位置及其前面)的数量。
**select<sub>)</sub>(i)** 表示括号序列中第i个)的偏移。
rank和select操作都可以通过在序列上构建O(n)bit的辅助索引实现O(1)时间的查询，实现方案参考[RRR – A Succinct Rank/Select Index for Bit Vectors](http://alexbowe.com/rrr/)。
**findopen(v)** 表示括号序列中v位置的)配对的(的位置。
**findclose(v)** 表示括号序列中v位置的(配对的)的位置。
实现findopen、findclose需要一个辅助的索引结构**Range min-max tree** ，构建方法如下：

----
1. 基于DFUDS序列，构建一个新的序列，DFUDS序列的每个值对应新序列的一个值，DFUDS中v位置对应新序列的值为(rank<sub>(</sub>(v)-rank<sub>)</sub>(v))。
	- findclose(v)等价于在新序列中v右侧比当前值小1的最近的位置。
	- findopen(v)等价于在新序列中v左侧比当前值大1的最近的位置。
2. 将新序列中每s个元素分成一个block，block取名为med block，记录每个med block中的最大最小值。
3. 以第2步中的最大最小值作为叶子结点，构建完全二叉树，树中每个中间节点也是最大最小值节点，最大值是所有孩子节点中的最大值，最小值是所有孩子节点中的最小值。

----

下图是构建 **Range min-max tree** 的示例:
![range_min_max_tree.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/7f994146e33cbe1995ce6594d82db238.png)

最下面一层是DFUDS序列，倒数第二层是构建的新序列，每三个是一个block，倒数第三层往上就是构建的**Range min-max tree**，取值的例子如下：

```
findclose(0) = 15, findopen(15) = 0
findclose(2) = 3, findopen(3) = 2
findclose(11) = 14, findopen(14) = 11
```
findclose(v)算法如下：

```
// 判断是否在当前的med block中，s是med block元素数量。
if(exist in med_block[v % s]) return pos;
// 计算当前med block在完全二叉树中的偏移，完全二叉树以数组形式存储,
// tree_inner_nodes_count表示Range min-max tree中中间节点的数量
idx = tree_inner_nodes_count + (v % s)
// 自底向上回溯，查找最大最小值满足需求的最近的树节点
while(!is_root(idx)){
	if(is_left_child(idx)){
		idx = right_sibling(idx);
		if(tree[idx].min <= desired_value && desired_value  <= tree[idx].max)
			break;
	}
	idx = parent(idx);
}
// 自顶向下查找，为满足最近原则，先找左孩子，不在左孩子中则查找右孩子
if(!is_root(idx)){
	while(!is_leaf(idx)){
		idx = left_child(idx);
		if(!(tree[idx].min <= desired_value && desired_value  <= tree[idx].max))
			idx = right_sibling(idx);
	}
	if(exist in med_block[idx - tree_inner_nodes_count]) return pos;
}
return not found;
```
findopen(v)算法是findclose的翻转，如下：

```
if(exist in med_block[v % s]) return pos;
idx = tree_inner_nodes_count + (v % s)
while(!is_root(idx)){
	if(is_right_child(idx)){
		idx = left_sibling(idx);
		if(tree[idx].min <= desired_value && desired_value  <= tree[idx].max)
			break;
	}
	idx = parent(idx);
}
if(!is_root(idx)){
	while(!is_leaf(idx)){
		idx = right_child(idx);
		if(!(tree[idx].min <= desired_value && desired_value  <= tree[idx].max))
			idx = left_sibling(idx);
	}
	if(exist in med_block[idx - tree_inner_nodes_count]) return pos;
}
return not found;
```
### 总结
本节主要介绍DFUDS的编码方法，以及树的三个基本操作degree、child、parent在DFUDS上的实现，基于这三个基本操作可以组装成更加复杂的树算法。另外，本节还详细介绍了实现三种基本操作依赖的rank、select、findclose、findopen的实现，重点介绍了实现findopen、findclose依赖的Range min-max tree结构。

下面我们介绍使用DFUDS来编码前缀树(Prefix trie)。

## 编码Prefix trie
Prefix trie的概念就不介绍了，直接看一个例子。

```
a, after、apple、as、dance、day
```
这几个单词构建的前缀树如下：

![prefix_trie.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/1cc3fda4f0d8f27e1ff82f942a94f4cb.png)

使用DFUDS可以将前缀树的树形编码下来，另外，每条边上对应的标签也需要记录下来，本节就是要讨论三点：

1. 怎么记录边上的标签。
2. 怎么将DFUDS编码和标签对应起来。
3. 怎么记录终态结点，比如节点2、3、4、5、7、8。

其实很简单，我们对DFUDS构建过程稍加改动，便得到如下Prefix trie的编码算法：

```
void Encode(string& unary, IntArray& offsets, string& labels, BitVector& terminals, TreeNode& node)
{
	for(TreeNode child: node.childs) {
		unary.push_back('(');
		offsets.push_back(labels.length);
		labels.append(child.label);
	}
	unary.push_back(')');
	terminals.push_back(node.is_terminal());
	for(TreeNode child: node.childs) {
		Encode(unary, offsets, labels, terminals, child);
	}
}
TreeNode treeRoot;
string unary;
IntArray offsets;
string labels;
BitVector terminals;
unary.push_back('(');
offsets.push_back(0);
labels.append("_");
Encode(unary, offsets, labels, terminals, treeRoot);
```
示例的Prefix trie编码结果如下：

```
DFUDS: ((()((())))(()))
offsets: 0 1 2 4 8 12 13 16
labels: _adafterpplesncey
terminals: 01111011
```
offsets表示边上的标签在labels中的偏移。
labels表示边上的标签组装起来的字符串。
terminals表示结点是否是终态，0表示非终态，1表示终态。

我们通过一个算法来认识上面几个变量之间的关系，计算节点v的第i个孩子对应的标签，方法如下：

```
string label(int v, int i)
{
	if(DFUDS[v] == ')') return ""; // 叶子结点没有孩子
	else{
		int offset = rank_left(v) + i - 2;
		int label_start = offsets[offset];
		int label_end = 0;
		if(offset == offsets.length - 1) label_end = labels.length;
		else label_end = offsets[offset + 1];
		return labels.sub_str(label_start, label_end - label_start);
	}
}
```
label函数结合DFUDS的几个基本操作就可以很容易将整个树还原出来，先序打印出所有label算法如下：

```
DFUDS dfuds = "((()((())))(()))";
int offsets[] = {0,1,2,4,8,12,13,16};
string labels = "_adafterpplesncey";
BitVector terminals = {0,1,1,1,1,0,1,1};
void preorder_print(int v, int i, string stack)
{
	string label_vi = label(v, i);
	if(terminals[dfuds.rank_right(child(v, i) - 1) - 1] == 1)
		println(stack + label_vi);
	else{
		int vc = child(v,i);
		if(dfuds[vc] == ')') return;
		else {
			string cstack = stack + label_vi;
			for(int j = 0; dfuds[vc + j] != ')'; ++j)
				preorder_print(vc, j + 1, cstack);
		}
	}	
}
void preorder_print(int v)
{
	string stack;
	for(int i = 0; dfuds[v + i] != ")"; ++i){
		preorder_print(v, i, stack);
	}
}
pre_order_print(1);
```
### 总结
本节介绍了Prefix trie的DFUDS编码方法，除了DFUDS外，还需IntArray offsets、string labels以及BitVector terminals辅助，offsets是一个严格递增的整型数组，非常容易压缩，labels是一个长字符串，也可以使用一些常见的字符串压缩方法压缩，terminals是bit数组，压缩方法也很多，下一节我们讨论一种另辟蹊径的压缩方法，嵌套Prefix trie。

## Prefix trie的压缩(嵌套Prefix trie)
通过前面的讨论我们知道影响Prefix trie编码之后大小的因素有四个，分别是DFUDS、offsets、labels、terminals，DFUDS和terminals的压缩收益不大，我们想在offsets、labels上做做文章，看是否还有压缩的空间。

我做了一个实验，将系统的日志，拉下来构建前缀树，并统计distinct term的数量，以及前缀树上标签长度分别等于2、3...的，再使用长度大于等于2的这些labels构造下一棵前缀树。 distinct term count是前缀树中包含的不同term的数量。

|level|distinct term count|2|3|4|5|>=6|
|---|---|---|---|---|---|---|
|0|1804170|629051|738463|250149|46740|248804|
|1|194077|3725|88081|9047|1163|499|
|2|14380|1043|165|55|97|183|
|3|765|60|110|73|15|96|
|4|336|95|36|12|8|87|
|5|219|32|9|13|16|67|
|6|132|15|12|17|7|59|

分析：

1. level 0用于构造level 1前缀树的labels数量为629051+738463+250149+46740+248804=1913207.
2. level 1 distinct term count是194077，悬殊巨大，可以看出很多都是重复的labels。
3. 这些重复尤其在第0层的前缀树中特别明显。

通过上面的观察，我们对第一次构建的前缀树中边上labels长度大于等于2的标签，再构建一次前缀树，形成嵌套结构，实验结果表明，嵌套两层的树大小是单层的65%左右，构建时间是原来的2.1倍，查询时间无明显变化。

嵌套Prefix trie的构建算法实现起来有点复杂，难点在于怎么将第二层的Prefix trie和第一层trie的边对应起来，这里就不介绍具体的构建算法了，所有需要的操作都在前面介绍了，感兴趣可以找我交流。

## Prefix trie的合并
多棵Prefix trie合并时，只需要对每一棵Prefix trie进行深度优先遍历，得到有序的各个term，对所有树的term进行归并，同时构建一棵新的Prefix trie即可。
合并过程中，如果用于构建Prefix trie的term都是有序的，可以通过优化Prefix trie构建算法，大大减少字符串比较次数，从而减少构建时间。获取有序的term，就要求能够对Prefix trie进行深度遍历，这也是我们使用DFUDS作为树形编码而非LOUDS等广度序的编码算法的原因。
另外，如果参与合并的树的数量很多，有可能导致构建的新的Prefix trie内存中放不下。
解决的思路是利用构建新的Prefix trie的term都是有序的这一特点，在构建过程中，边构建，边将不会再被访问到的子树序列化到磁盘上。
举个例子：

![merge.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/4e6a86abf873d24654d997b72999e767.png)

上图中下一个要被插入的term是dance，那么此时以2为根结点的子树就可以冻结起来，因为接下来插入的所有term都比a*大，子树2不会再被操作到，所谓冻结就是序列化到磁盘上。
插入dance之后树的形态如下：

![frozen_tree.png](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/3c0e69dbbb7356bdc0b083cc86bc9c07.png)

frozen 2节点是一个虚拟的节点，其记录了一些序列化到磁盘上的子树的meta信息。
随着merge的进行，这类冻结节点可能会越来越多，是否冻结一棵子树，除了接下来不会被访问到这一必要条件外，子树的大小也是要考虑的因素，这个可以根据实际情况设置一个阈值。
在包含冻结节点的树中查询时，冻结节点只有在必要时才被从磁盘中load起来，使用完后，可以从内存中删除，也可以放到cache中，通过这种lazy load的方式可以节省内存的开销。
## 总结
本文分四部分讨论了模糊查询的设计方案，对比了多种开源技术，包括lucene、marisa-trie、cedar等等，最终设计了Nested succinct trie的方案，压缩率上可以达到原始 **distinct数据** 的40%，构建速度可以达到35MB/s，和std::unordered_set相当。

在具体实现时，还有很多优化的地方，比如升序整型数据的压缩、01序列rank、select操作的优化等等，大家有疑惑或者建议欢迎一起讨论。
## 参考文档
1. [succinct数据结构](https://en.wikipedia.org/wiki/Succinct_data_structure)
2. [RRR – A Succinct Rank/Select Index for Bit Vectors](http://alexbowe.com/rrr/)
3. [Succinct Trees:Theory and Practice](http://www-erato.ist.hokudai.ac.jp/alsip2011/ALSIP2011_sadakane.pdf)
4. [Succinct Data Structure Library](https://github.com/simongog/sdsl-lite)
5. [marisa-trie](https://code.google.com/archive/p/marisa-trie/)
6. [lucene](http://lucene.apache.org/)
