# CCF-BDCI2019_Entity_Discovery
CCF-BDCI大数据与计算智能大赛-互联网金融新实体发现-9th
![guize](https://github.com/gaozhanfire/CCF-BDCI2019_Entity_Discovery/raw/master/beijing.png) 
---
# 赛题描述：
提取出每篇文章中所有的金融新实体，  
例如某篇文章的文本如下（此处只做示例，实际每篇文章文本长度远远大于以下示例）：  
度小满金融在首次发布的《科创板基金》中提到，今年前5月，京东金融的股票迅速升值。  
那么该篇文章对应的金融新实体列表为：度小满金融;科创板基金。  
***由于京东金融是知名的（金融）实体，所以“京东金融”并不算新实体。***  

# 个人理解：
结合出题方是“国家互联网应急响应中心”来看，这个赛题的目的实际上是找那种类似校园贷的实体（其实就是不知名的金融实体）。  
**整个赛题要把握住的重点是：**  
①这个赛题实际上偏向于“领域词抽取”（主要针对的是金融领域），但实际上却又不失关键词抽取的要素。  
②注意把握赛题要求中“新实体发现”的“新”，题目中说明了：持有金融牌照的银行、证券、保险、基金等机构、知名的互联网企业如腾讯、淘宝、京东等认为是已知实体（即不能让这些已知实体出现在最终预测结果中）。  

---
# 方案总框架：
使用了深度模型+传统模型  
深度模型：受限于显存的限制，我们只使用了bert_base，并且搭配的下游模型有bilstm+crf（复赛主力模型）（固定参数）以及全连接+crf（初赛主力模型）  
传统模型：只使用了lgb，共100维特征。虽然特征量大，但由于数据量不大，我们造特征所消耗的时间实际上是并不多的。  

---
# 具体解决方案：

## 数据清洗：
因为本赛题的文本数据大都是由爬虫对html抓取的互联网金融文本，故原始数据集的文本非常脏，主要发现的问题如下：  
①文本中有很多超链接    
②表情、特殊符号   
③多个问号（可能是爬取过程中乱码）   
④无意义文本，比如邮箱、电话之类  
我们通过编写脚本对这些文本进行了清洗。  
***特别重要的是，文本不能清洗的特别干净，比如比赛初期我将停用词、除了常用标点（就是回车问号等等一系列的）之外的标点、网页标签都去掉了，把大写的句号全都改成小写的句号（还有大写的逗号，大写的顿号等等），还把所有繁体都转成简体了，这样的预处理所带来的是效果不升反降。因为这些改动实际上都可能会破坏文本原有的结构以及信息，且bert对于停用词、特殊符号等本身就不敏感。***    
  
## 数据预处理：  

- 本赛题提供的文本中，有很多文本的长度甚至都超出了2000个字，***而bert读入文本时，都会把文本截断到maxlen长度以下***，所以bert读入超过maxlen长度的文本后，后面的那些文本就直接“抛弃”了。这样就会有一部分数据得不到利用。  
      + 我的解决方法是将长度超过maxlen的文本切成长度都小于maxlen的几个小段。  
      具体做法请参考：roberta预训练中FULL-SENTENCE的数据预处理方法，  
      即用几个连续的句子拼接，直到最大长度512。   
      （这个是我误打误撞想到的，甚至不知道是出自某篇论文。好像前5中也有人用了这个方案。其余很多人都是暴力的直接把句子截到512，但我觉得那种方法可能  
      不能保证语义上的连续性）。  
      + 我队友的方法是使用了句子滑窗（也很有借鉴意义），就是例如有句子序列ABCDEF……，若ABCD组成的段能正好小于maxlen，接下来就按固定的句子数滑窗，
      比如向前滑两句，即句子的开头由A滑到C，从C开始填充512，即看看CD、CDE、CDEF……直到填充满512为止。 
      这个方法是一种既保留了句子原连续语义，又在一定程度上进行了数据增广。然而在我自己的方案（固定参数bert-bilstm-crf）中，并没有得到效果提升。
  
## 深度模型选择：
 - 我们的语言模型选择了bert-base，主要原因还是我们的机器跑不动roberta-large的512长度。而哈工大的bert-wwm我尝试过，效果并不比bert-base要好  
 这就导致了我们的深度模型与top队伍深度模型的差距（top队伍赛后总结中基本都写到用roberta-large或者bert-wwm来搞。）  
  
## 初赛方案（bert）：
### 正负样本均衡：
原始数据中，存在一部分没有被标注出新实体的样本，这一部分样本我在最一开始的时候是直接去掉不要的，且我将文本都切分到maxlen以下后，也会产生不少这样的样本。
后来观察了一下，发现那部分样本存在很多的知名实体（已知实体），这也是这些样本没有被标注的原因。  
- 我的方案是试做下采样，即只取一部分负样本，经过试验，发现使正负样本比例保持在8：1左右。结果使线上效果又上升了0.5个百分点。  
- 我队友的方案：由于他的maxlen取的比较小，这也导致他负样本非常多，正负样本比例达到了1：2，他是做了上采样，将正样例加了3倍，使正负样本比例到了3：2.  

### 数据标注格式：
- 最初采用了bio格式，但观察模型输出结果，我发现有很多 BOIIIO……这样的情况，后期换成了BIOES标注，这也带来了2~3个千分点提升。  

### 模型预训练：
- 我的方案：下载谷歌官方的预训练模型，并在此基础上按照roberta训练方式进行预训练（没有全词mask，去掉NSP（next sentence predict）任务的loss，使用FULL-SENTENCE（上面有提到）而不是谷歌官方提供的“将文章分为一个个句子”）。效果出奇的好，单模效果非常稳定（原来提交结果线上波动非常大），在f1 32左右（线上排名前5）。  
- 我队友的方案：直接使用谷歌官方的预训练方式，但是也去掉NSP任务。  

### 下游模型选择:
- 我的方案是：bilstm+crf，固定bert参数。该方案识别出来的实体非常多，但是却并不是那么精准（比如有“广州市创鑫软件科技有限公司是金融公司”）。  
- 我队友的方案：全连接+crf，不固定bert参数。该方案识别出来的实体比我的少很多，但是却非常精准，没有上面说的那种杂乱的实体。（我曾经使用过相同的样本采样方式，用我队友的模型来训练，得到的样本数一样非常少，所以可以排除数据处理、样本采样带来的影响）  

## 初赛方案（lgb）：
- 第一种：我构建了一个知名实体集，例如{京东，百度，阿里，腾讯，平安保险……}，这些实体都是一定不会出现在预测实体中的，我把带有这些实体的文本都拿出来用来提取特征（从训练集和测试集都可以拿），这些样本的标签全部设为0。用所有的原始训练集的每篇文本对应的实体，送入模型训练。  
  - 词在文本中的tfidf  
  - 词在文本中所有词的tfidf的排名    
用该模型来对bert-crf的结果进行后处理，把  
该模型在复赛后期甚至都能提升1个千分点，在前期提升的最多，甚至能提升1~2个百分点。  
- 第二种：按照关键词抽取的思路，做一个大的lgb。  
  - 正样本：仍是每篇文本对应的实体。  
  - 负样本（此处只说初赛的提取方案）：  
    - 对标题进行ngram，统计ngram的词在正文中的词频，出现次数大于两次的并入每篇文本所对应的候选词集     
    - 将训练完后的bert模型对训练集重新预测一遍，把每篇文本预测出来的词全部并入到文本对应的候选词集      
    - 把每篇文本对应的候选词集中存在的正样本都去掉，这样剩下的词就全都是“不是金融新实体的样本”了    
  - 特征提取：    
    这里实在是太多了，代码中都有注释，有兴趣的可以去看看，我只说一下特征提取的思考维度和比较典型的特征：  
    - **文本特征**：例如文本长度，标题长度……  
    - **单词特征**：比如单词的互信息、左右熵、单词的长度、单词的词性组成、单词中字母数量、汉字数量……  
    - **业务特征**：是否以“贷”、“宝”为结尾（主要针对金融新实体提取的特征）  
    - **文本-单词特征**：单词在文本中的位置、单词是否在文本中、是否在标题中、在文本中的平均位置（比如京东可能会在文本中不同位置出现多次，统计这些位置，计算均值）……  
    - **词向量特征**：单词的词向量和整个文本的doc2vec向量的欧式距离、cos距离、文章词向量矩阵的均值、方差、标准差、偏度以及峰度……  
    - **主题特征**：kmeans_tfidf、tsvd_tfidf……  
    - **单词共现矩阵的统计特征**，以及共现矩阵一阶差分的统计特征，包括均值、方差、标准差、峰度、偏度等（复现的某篇论文）   
### 创新型特征：  
- 组内特征***（我们把一篇文章对应的所有实体称为“一组实体”，以文章为轴来做特征）***   
  例如一篇文章对应有一组实体ABCDEF  
  - 考虑相对性：例如A的词频m相对于这一组实体中的最大词频n的特征m/n，相对于组内所有实体平均词频a的特征m/a……。还有就是我们发现预测结果有很多如：霸屏天
  下、霸屏天下ababu这样具有字面上包含关系的实体，进而我们可以提取出具有包含关系的词，词长之比、位置之比。。
  - 考虑组内全局性：依据特征重要性，取几个特征（比如tfidf特征重要性高），就拿tfidf特征举例，算组内ABCDEF这6个实体的tfidf排名。
- 用整体来描述个体：  
  例如组内实体的个数（实际上就是候选实体集中实体的个数），组内候选词的平均词长……这些特征都可以刻画“文章”这个 个体。
- 将规则转化为lgb特征：
  主要是针对“保留公司”那条规则做的。有：组内是否有带“公司”的实体、是否有带“公司”的实体包含了不带公司的实体、带公司实体的数量、被带公司实体包含的那个不带公司的实体相对于带公司实体的词频只差、词频之除、长度之差（还有很多）…………
***非常遗憾的就是初赛由于我没有特别关注lgb的正负样本比例，导致lgb效果很差，我以为按关键词提取来做lgb是不适用与本赛题的，初赛就没有太多使用lgb。***  
***遗憾的还有没有尽早使用ctr特征***  
## 初赛方案（规则）：
这个主要是队友做的，初赛提升非常大，但是B榜严重过拟合，于是便有了我们复赛将所有规则转为模型特征的做法  
简单的来说就是多个bert模型的结果投票，但是这个投票更加的细粒度。  
![guize](https://github.com/gaozhanfire/CCF-BDCI2019_Entity_Discovery/raw/master/guize.png)  
***初赛的时候队友的模型识别以“公司”、“有限公司”等为结尾的实体效果并不好，规则便将这些以公司为后缀的尽可能保留了。***  

---
# 复赛方案
***复赛大都是基于初赛的改动，这里只说一下复赛方案相对于初赛方案的改进***

## 数据分析方面：
- 初赛时原始数据的标注少而精，每篇文章对应的实体数大多集中在2个。  
- 复赛每篇文章对应的实体数有很多是大于两个的，最多的一篇文章甚至能搞出50多个实体，有的文章甚至把“小米金融” “黄金” “期货”都算成了实体。  
***初赛和复赛的数据分布差异非常大，而我在刚进入复赛时，使用初赛+复赛的数据来训练模型，效果非常差，这也进一步说明了不同任务下对数据分布分析的重要性***   

## 数据分布分析及处理方面（复赛提分点之1）：
- 最初使用了初赛和复赛全量训练数据   
- 效果非常不好，便尝试只使用复赛数据（提升了3百分点）  
- 尝试使用***回标***数据标注方式，即将训练集中所有的实体（即所有文章对应的实体）收集到一个set里，然后用这个set中的实体对每篇文章进行标注。（提升了3个百分点）！！！！  
- 我们认为复赛数据更加注重语义（比如词出现在文中的位置（比如分号隔开的实体））所以我们将一些知名实体（比如京东、阿里）也加入到上面提到的那个set集合中进行，与原数据中的实体一起对训练集文章进行标注，对结果进行后处理时将知名实体进行去除。（带来了5个千左右的提升）  

## 下游模型方面：
- 初赛我们的主力模型是队友的全连接+crf（不固定bert后两层参数），该模型预测结果少而精，预测结果比较接近初赛的结果分布。
- 复赛我们的处理模型换成了我的bilstm+crf（固定bert后两层参数），该模型预测出来的实体非常多，比较接近复赛的结果分布。  
主力模型的更换给我们带来了3个百分点的提升。

## lgb方面（复赛提分点之2）：  
***我们分析到了lgb效果非常差的原因，那就是我们的负样本有10W条，而正样本（也就是训练集中实体）只有5000条。正负比例20：1，正样本实在太少了***   
我的解决方案：进行重采样，将正样本加了5倍，再将f1的阈值（预测概率大于这个阈值的认为是新实体，小于这个阈值的认为不是新实体）从原来的0.5调到了0.15，效果终于变好了起来，并且在我将lgb和bert的结果取交集后，线上成绩提升了3个百分点，稳定在了f1 46左右。  
加入了提升比较大的三个特征：
- **图特征**：例如一篇文章对应有三个候选实体ABC，那么（A,B）(A,C)(B,C)共可以建立三条边。依次类推，对每篇文章对应的实体连边，这样就可以得到一个“基于实体共现关系”的图  
  + **无向图特征——边连接数max_degrees**：统计每个实体的边连接数
  + **无向图特征——pagerank值**：根据Google的pagerank算法计算每个节点的pagerank值，这个虽然我们使是使用无向图进行计算的，但是pagerank是通过迭代计算每个节点的入度来评判每个节点的重要程度。所以在运行pagerank算法的时候会默认把无向边连接转化成双向的边连接。
  + **无向图特征——HITS算法A值和H值**：HITS类似pagerank模型，在HIST算法中，分为Hub页面和Authority页面，Authority页面是指与某个领域或者某个话题相关的高质量页面，Hub页面则是包含很多指向高质量Authority页面链接的网页。HITS算法模型会给出每个节点的A值和H值用来评估节点的重要程度。
  + **无向图特征——neighbour**：每个实体的邻居数  

这几个图特征在比赛后期带来了一个百分点的提升！！！

- **全局交互特征** （一个百分位的提升） 
借鉴lda抽取领域词的方法，我们认为不单单是文章，词也应该拥有它自己的主题。刻画词主题的方法具体如下:  
用某个实体A为例，如有a、b、c、d、e五篇文章包含实体A，a的主题为1号，b主题为5号，c主题为2号，d主题为1号，e主题为5号，  
设主题种类数为8，那么就可得：A实体在1号主题的文章中共出现过2次，2号主题的文章中出现过1次，3号为0次，4号为0次，5号为2次，6号为0次……  
可得词主题向量为（2，1，0，0，2，0，0，0），用该向量作为特征，送入模型训练。  

- **转化率特征**（3个百分位提升）：
统计一个实体出现在多少条记录中（即实体记录数n），在这些出现次数中有多少次被当成新实体（即实体正确次数m），用m/n，选择n大于5的实体（十分重要，如果用全部实体的话会发生严重的过拟合现象）进行贝叶斯平滑。  

- lgb的数据采样：
  -初赛的时候我们使用了标题ngram+bert模型向回预测。但我们发现我们的候选词集质量仍然不高，比如：“小米钱包的”。  
  -我们的图特征和组特征对候选词集的质量和数量的平衡是具有一定的要求的，所以为了提升候选集的质量，同时不降低召回率，  我们设计了一个“五折交叉预测训练
  集”的方法。**即训练集分为5折，取其中的4折用来训练，去预测剩下的一折，对测试集也是这样。**（这样做的好处是：不会用到训练集的信息。在初赛的时候，我
  用整个训练集训练bert，并重新预测训练集，得到的预测出来的词非常少，这大概是因为模型已经知道了训练集的正确答案，预测结果不会向外发散。）

## 模型融合：
取线上结果最好的几个bert结果做并集，然后同lgb的结果（f1阈值设为0.15）取交集，得到最终结果。  
---
