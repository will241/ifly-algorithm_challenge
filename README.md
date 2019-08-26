# ifly-algorithm_challenge
讯飞移动反欺诈算法竞赛,目前分数只有94.48

讯飞移动反欺诈算法数据竞赛网址： http://challenge.xfyun.cn/2019/gamedetail?type=detail/mobileAD

### 总体流程
```
  | EDA
  | 数据预处理
  | 数据特征构造
  | 模型搭建
  | 模型参数的调优以及特征筛选
```
#### 一、EDA

 在做数据竞赛的时候，当我们拿到数据集的时候，我们首先要做的，也是最重要的事情那就是进行数据探索分析，这部分做的好，对接下来的每个部分的工作都是有力的参考，而不会盲目的去进行数据预处理，以及特征构造。数据探索分析主要做以下几个部分的工作:
   - 查看每个数据特征的缺失情况、特征中是否含有异常点，错误值
   - 查看每个特征的分布情况，主要包括连续特征是否存在偏移，也就是不服从正态分布的情况；离散特征的值具体分布如何
   - 查看一般特征之间的相关性如何
   - 查看一般特征与目标特征之间的相关性


#### 二、数据预处理

通过EDA,我们对数据进行了初步分析，接下来就针对EDA部分得出的结果，来进行数据的预处理工作,主要做了以下工作:
  
  - 先是对特征中不符合类型的数据进行类别转换
  - 利用众数来对缺失值填充，异常值的剔除，以及错误值进行修正
  - 对于连续特征，如果存在偏移现象，使用```对数```或者```Box-Cox```进行转换，使其满足正太分布，降低拖尾带来的影响
  - 对于离散值，一般是进行```独热编码```，但由于本身数据集离散特征的取值类别数较多，如果使独热编码(OneHotEncoder)，会使得数据得维度变得很高，会降低模型运行的速度；这里我使用的是```类别编码(LabelEncoder)```来对离散特征进行编码
  - 对部分连续特征进行分箱操作，将其离散化。
  - 对离散特征进行更加细粒度的划分，如make,model,osv这种特征，可以进行更加详细的划分，比如```osv : 10.0.3  --- > 10 0 3```,对于make制造商，为了降低噪声数据带来的影星，将make特征进行了预处理，如```huawei/HUAWEI/honor/...``` 统一用```HUAWEI```表示。 

#### 补充一点(比较重要)
```
面对数据量很大时候，如何解决大规模数据建模问题？一般会有三种基本方法:
  1. 对原始样本进行抽样，不过这样会存在一个问题，那就是会导致正负样本可能失衡，影响模型最终的效果
  2. 对数据结构或者类型进行优化来降低内存的消耗：
     (1) 当特征取值没有负数的时候，我们可以将int32类型的数据转换为uint8。
     (2) 将float64类型的数据转换为float32
     (3) 在不影响模型效果的前提下，可以将object类型的数据转化为category类型的数据，这种适合离散特征取值较少的情况，一般特征的的取值类别数占特征总的数量的比例小于5%。这次采用的是这种方法，具体实现见代码.
  3. 利用online learning等相关方法。
```
#### 数据特征构造

```
特征主要分为：
   1. 原始特征  
   2. 统计特征 ： count, max, min,std, nunique,mean,...
   3. 组合特征 : 将重要性较高的特征与其他的特征进行组合
   4. 叶子节点特征：利用lgb和xgb模型来生成部分叶子节点特征
```

#### 模型的选择
```
  这里主要是选用了四个模型，三个机器学习模型，一个深度学习模型
    1. lightgbm
    2. xgboost
    3. catboost
    4. MLP 
  最后发现catboost效果最好，catboost有一个参数cat_features,通过这个参数，我们可以指定离散特征的索引.效果的确很好，但也很吃内存，最后因为没有机器跑模型，到后面基本上很多想法都不能实现。。。。
```
#### 模型参数的寻优
可直接参考：  https://www.cnblogs.com/pinard/p/11114748.html

#### 特征的筛选
##### 主要是从以下几个方面进行考虑：
     - 高百分比的缺失值特征选择法
     - 高度相关特征选择方法(如果一般变量之间的相关性很高，则可去掉其中的某些特征)
     - 树型结构中的重要性特征选择方法
     - 低重要性特征选择方法
     - 唯一值特征选择方法

在我们的建模过程中，主要是使用了以下方法进行特征选取:

  - [x] 使用了随机森林，lgb来进行特征重要性的输出
  - [x] 使用了基于过滤器的卡方，基于包装器的递归特征消除，以及皮尔逊相关系数来进行特征选择
  ```
  参考代码:
  基于卡方进行特征选择的方法:
        chi_selector = SelectKBest(chi2, k = self.k)
        chi_selector.fit(self.X.values, self.y)
        chi_support = chi_selector.get_support(indices=True)  #返回被选择的特征所在的列
        _ = chi_selector.get_support()
  ```
  
#### 模型的融合
在打数据类竞赛的时候，最后都会进行多模型的融合，来进行提分，模型融合使用的是集成学习的思想，主要分为了四种类型的集成学习，分别是bagging,boosting,stacking,blending.大概说下它们的思想和不同点:
##### 1. bagging：有放回采样，弱学习器之间没有联系，相互独立，可以进行并行拟合
   - [x] ```有放回采样```: 对数据进行随机采样(bootstrap),就是从训练集中采集固定数量的样本，但是每采集一个样本之后，都将样本放回，并且随机采样的而样本数量与训练集数量大小一致。
   - [x] ```袋外数据(oob)```： 由于m个样本的训练集，在每次随机采样中，被采集到的概率为1/m,不被采集到的数据概率为(1-1/m),那么经过m次采集之后，没被采集到的概率为```(1-1/m)^m```, 当m -> 无穷大的时候，前式是趋向于1/e, 约等于36.8%,即有36.8%的数据在m次采样后未被采集到，我们称这样的数据为袋外数据，一般用袋外数据来检测模型的泛化性能, 在sklarn库,```randomclassifier()```类中的```oob```参数就表示这个意思，默认是```false```,如若需要可以设置为```true```.
   - [x] ```弱学习器之间是相互独立```: tbagging认为弱学习器之间的地位是相同的，是相互独立的，互相并不影响
   - [x] ```并行拟合```: 由于各个弱学习器之间是相互独立的，因此不每个弱学习器训练不受其他弱学习器的约束，即没有前后依赖关系，所以可以并行训练模型。
   - [x] bagging对弱学习器的选择没有限制，一般常用决策树或者神经网络
   - [x] ```集合策略```: 对于分类任务，一般采用的是投票选择；对于回归任务，采用加权平均方法
   - [x] bagging学习，由于每次采样不同的数据集来训练不同的弱学习器，因此泛化性能比较好，即方差较小，但是对于训练集的拟合程度会差一点，即偏差较大。
   - [x] 代表性算法: ```随机森林(RF)```  --- 在bagging随机采样的基础之上，还进行了```特征属性的随机采样```，大大提高了模型训练的速度，以及使用```CART```作为弱学习器。
   
##### 2. boosting: 无放回采样，弱学习器之间有联系，并不是相互独立，串行拟合
   - [x] ```无放回采样```
   - [x] Boosting思想: 采用的是加法模型，前向分步算法
   - [x] 损失函数: 对于回归问题，使用平方误差损失函数，对于分类问题，使用指数函数作为损失函数
   - [x] 当前强学习器是上一轮强学习器与当前弱学习器的组合，所以各个弱学习器之间并不是相互独立的，是相互影响的，所以只能串行拟合，这也是Boosting方法的缺点，在代表算法中，例如Xgboost算法就利用不同的线程实现了局部并行拟合，来提高模型训练的速度。
   - [x] Boosting 可以降低模型对训练集的拟合误差，但是训练方差较大。
   - [x] 代表算法: Adaboost, GBDT, LightGBM, Xgboost,CatBoost

##### 3. stacking: 初级学习器，次级学习器
直接借用别人的图，![参考] (https://www.cnblogs.com/gczr/p/7144508.html)
![img](https://github.com/jiangzhongkai/ifly-algorithm_challenge/blob/master/img/stacking.jpg)
  
根据上图分析一下stacking具体步骤：
  
　　(1) TrainingData进行5-fold分割，正好生成5个model，每个model预测训练数据的1/5部分，最后合起来正好是一个完整的训练集Predictions，行数与TrainingData一致。
  
　　(2) TestData数据，model1-model5每次都对TestData进行预测，形成5份完整的Predict（绿色部分），最后对这个5个Predict取平均值，得到测试集Predictions。
  
　　(3) 上面的1）与2）步骤只是用了一种算法，如果用三种算法，就是三份“训练集Predictions与测试集Predictions”，可以认为就是形成了三列新的特征，训练集Predictions与测试集Predictions各三列。
  
　　(4) 3列训练集```Predictions+TrainingData```的y值，就形成了新的训练样本数据；测试集Predictions的三列就是新的测试数据。
  
　　(5) 利用```meta model```（模型上的模型），其实就是再找一种算法对上述新数据进行建模预测，预测出来的数据就是提交的最终数据
  
##### 4. blending
算法简单思想就是: 假如总的数据集为12500条，其中训练集training data的条数为10000，测试集testing data的条数为2500；然后将traing data 分为7000和3000，然后使用m个模型来训练7000条数据，然后再将m个模型在3000条数据上进行预测，将m个模型预测的结果进行拼接，则会得到3000 x m的数据，同时对testing data进行预测，将预测结果与3000 x m 合并共同作为第二层training data.最后再使用一个模型来进行预测即可。(和stacking思想有点像，但是区别还是挺大的。)，主要优缺点如下：
 优点:
     1. 比stacking简单(因为没有进行交叉验证来获取新的feature)
     2. 避开了信息泄露问题
 缺点：
     1. 使用了很少的数据
     2. blender可能会过拟合
     3. stacking使用了多次的CV会比较稳健。


在模型的输出时，我们输出的是概率，这里采用的加权融合的方法,```catboost*0.5 + lgb*0.2 + xgb*0.2+ MLP * 0.1```,然后再进行最终类别的输出.

#### 最后再补充一下

我们在做分类任务时，如果正负样本分布均衡，我们如何处理呢？

Q: 首先要说下，为什么正负样本分布不均衡，会影向模型的最终效果呢？

A: 本质原因是在训练优化的目标寒素与测试时使用的评价标准不一致，主要分为样本分布不一致，类别权重不同。

Q: 如何处理?

A: 一般会进行```随机采样```操作，具体分```过采样```和```欠采样```操作。但是如果单纯的增加类别少的样本，增加模型的复杂度的同时，也容易造成过拟合，一般我们选择```SMOTE```方法来生成新样本。在进行欠采样的时候，如果单纯的从类别多的样本中选取数量和样本少的样本进行训练，这样容易丢失部分有用信息，一般我们使用```Easy Ensamble``` 和```Balance Cascade```方法来进行采样，其中```Easy Ensemble```思想就是对数据集进行多次欠采样，训练多个基模型，然后将这些模型的输出进行有效结合作为最的结果。```Balance Cascade```是一个级联结构，在每一级中从多数类S1中随机抽取子集E,用E+S2(少数类)训练该级分类器，然后将S1中能够被当前分类器正确判别的样本剔除掉，继续下一级的操作，重复若干次得到级联结构(有点像Adaboost思想降低被正确分类样本的权重，只不过是直接将权重设为0)，最终的结果也是各级分类器结果的融合。

### 参考资料xiang

- [x] GBDT、xgboost对比分析：

https://wenku.baidu.com/view/f3da60b4951ea76e58fafab069dc5022aaea463e.html

- [x] xgboost论文：

https://arxiv.org/pdf/1603.02754.pdf

- [x] lightgbm论文:

http://papers.nips.cc/paper/6907-lightgbm-a-highly-efficient-gradient-boosting-decision-tree.pdf

- [x] catboost论文：

  - https://arxiv.org/pdf/1706.09516.pdf

  - http://learningsys.org/nips17/assets/papers/paper_11.pdf
