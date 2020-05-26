---
title: 用机器学习判断ins内容是否能上热门
date: 2020-05-21 23:21:46
tags: [算法]
---

## 背景

假设现在要使用爬虫从ins抓取内容,在ins的网页版上每个#tag标签下都有'热门'和'最新'两部分tab页,因为热门内容的质量更好,所以希望能抓取更多热门内容,但是网页版每个#tag只显示最近的9条热门内容,不过'最新'tab下可以一直向前翻页,所以如果能从'最新'下的内容中过滤出热门内容就可以满足内容抓取质量的要求,需要找到一个能将内容分类为热门和非热门的算法.

<!--more-->


<div align=center> {% asset_img ins.png image %} </div>



那么现在我来介绍一下如何使用机器学习来实现这个算法,本文主要介绍从零开始使用机器学习并解决问题的过程.不会介绍理论知识(刚好我也不懂),速战速决就完事儿了.

{% asset_img gan.jpg image %}

### 开始之前要了解

- **python**:一种蛇
- **instagram**: 著名被墙网站之一
- **pandas**:数据处理类库
- **matplotlib&seaborn**: 画图类库
- **机器学习**: 一种对计算机算法的研究方式，算法会根据经验自动优化效果
- **分类问题**: 把一组数据分成两类或者多类,要划分类型已提前定义
- **sklearn**: 封装了多种机器学习算法的类库,开箱即用

## 数据处理

ins内容有很多字段信息,首先选择出可能影响内容上热门的字段,包含点赞/曝光/评论/发布时间/抓取时间/标题包含的tag数和当前页面tag下的内容总数等,这些内容用json格式保存,示例如下:

```python
{
    "id":"CAA-r0SjtF3",
    "sourceTag":"dharmaproductions",
    "publishTime":1589130910000,
    "addTime":1589134125000,
    "likes":4,
    "comments":1,
    "views":18,
    "tagNumber":15,
    "description":"",
    "totalMedia":57149,
    "hot":1
}
```

### 数据探索

这一步是为了进一步了解原始数据,检查数据真实性和字段的分布

1. 用seaborn画直方图,查看热门/非热门内容的比例:

```python
    self.df = pd.read_json('debug/train_video.jl', lines=True)
    sns.countplot(self.df['hot'], label="Count")
```

<div align=center> {% asset_img hot.png image %} </div>

图中热门内容占比还是比较高的,有一点脱离真实情况,实际对于ins上被活跃使用的tag,本身内容数量就很多又更新频繁,所以只有很少一部分内容能上热门.

2. 用seaborn画热力图,这一步是为了查看字段之间的关系:
```
    self.df = pd.read_json('debug/train_video.jl', lines=True)
    corr = self.df.corr()
    plt.figure(figsize=(14, 14))
    sns.heatmap(corr, annot=True)
```
corr()函数会计算数据集df中各字段的相关关系,图中的颜色越浅代表越相关,可以看到like/view之前相关性比较高,如果两个字段之间相关性接近1,可以考虑去掉其中一个字段.

<div align=left> {% asset_img heatmap.png image %} </div>

3. 用matplotlib画频率分布直方图

```python
    def draw_histo(self):
        plt.figure(1, figsize=(20, 20))
        i = 810
        for field in self.df.keys():
            i += 1
            # 去掉区间上数量小于10的记录
            items = self.df.groupby(field).filter(lambda x: len(x) > 10)[field]
            self.histogram(items=items, index=i, field=field)
        plt.savefig('histogram.png')
        
        def histogram(self, items=None, index=None, field=None, y_label='Probability'):
        """
        画频率直方图（带正态分布曲线）
        :param index: 图片位置
        :param field: 字段名
        :param y_label:
        :return:
        """
        title = field + ' distribution'
        if items is None:
            items = self.df[field]
        try:
            plt.subplot(index)
            mean = items.mean()
            std = items.std()
            x = np.arange(items.min(), items.max())
            y = self.normfun(x=x, mu=mean, sigma=std)
            plt.plot(x, y)
            plt.hist(items, bins='auto', density=True, rwidth=0.9, stacked=True)
            plt.title(title)
            plt.xlabel(field)
            plt.ylabel(y_label)
            plt.tight_layout()
        except Exception as e:
            print(field)
            print(e)

    @staticmethod
    def normfun(x, mu, sigma):
        """
        正态分布的概率密度函数。可以理解成 x 是 mu（均值）和 sigma（标准差）的函数
        :param x:
        :param mu:
        :param sigma:
        :return:
        """
        pdf = np.exp(-((x - mu) ** 2) / (2 * sigma ** 2)) / (sigma * np.sqrt(2 * np.pi))
        return pdf    
```
<div align=center> {% asset_img histogram.png image %} </div>

画直方图的目的是查看字段的分布类型,图中蓝色的线是正态分布函数曲线,明细like/view/comment的分布不符合正态分布,更像是长尾分布(大多数值分布在头部)


### 数据清洗&特征选择
因为原始数据中并没有缺失值或异常值得情况,所以忽略数据清洗的步骤,直接进行特征选择.
首先我们不需要具体的时间戳,把发布时间publistTime和抓取时间addTime的间隔计算出来,单位是天和小时(ins热门tab中通常是最近发布的内容)

```python
    df['days'] = (pd.to_datetime(df['addTime'], unit='ms') - pd.to_datetime(df['publishTime'], unit='ms')).dt.days
    # 不满一天的用1天替换
    df['days'] = df['days'].replace(0, 1)
    df['hours'] = ((pd.to_datetime(df['addTime'], unit='ms') - pd.to_datetime(df['publishTime'], unit='ms'))
                   .dt.total_seconds() / 3600).astype(int)
    df['hours'] = df['hours'].replace(0, 1)
```

然后因为一般内容发布越早曝光也越高,所以添加like_per_hour/view_per_hour两个特征,减弱时间影响

```python
    df['like_per_hour'] = (df['likes'] / df['hours']).astype(int)
    df['view_per_hour'] = (df['views'] / df['hours']).astype(int)
```

最后选择的特征如下:

```python
    features = ['views', 'likes', 'comments', 'tagNumber', 'totalMedia', 'days', 'like_per_hour', 'view_per_hour']
```

## 评分标准

在选择分类算法前,先来了解一下评估分类算法的常用标准

### 准确率,精确率,召回率

假设你开发了一款检测新冠病毒的试剂盒,那么每次检测结果有一下四种(阳性表示被检测人携带病毒):

TP(True Prosivite): 真阳性,说明正确检测出病毒

FP(False Prosivite): 假阳性

TN(True Negative): 真阴性

FN(False Negative): 假阴性,携带病毒却没有检测出来

对于病毒检测来说FN的危害显然要比FP更大,而根据这几种情况的样本数量就可以计算出准确率,精确率和召回率:

准确率(accuracy): ${TP+TN}\over{TP+FP+TN+FN}$,代表全部样本的正确率

精确率(precision): ${TP}\over{TP+FP}$,代表检测为阳性时的正确率

召回率(recall): ${TP}\over{TP+FN}$,代表所有病毒携带者被检测为阳性的覆盖率,也叫查全率


![举个例子:][1]

有三个样本,检测结果为y_predict,而实际值为y_true,1代表阳性,sklearn的score函数默认返回的是阳性分类的分数

```python
from sklearn.metrics import accuracy_score,precision_score,recall_score

if __name__ == '__main__':
    y_predict = [1,0,1]
    y_true = [0,0,1]
    print(f'accuracy:{accuracy_score(y_true,y_predict)}')
    print(f'precision:{precision_score(y_true,y_predict)}')
    print(f'recall:{recall_score(y_true,y_predict)}')
```

输出结果为:
accuracy:0.6666666666666666
precision:0.5
recall:1.0

对于判断ins是否属于热门内容的算法,可以允许ta把非热门内容分到热门但是要尽量不遗漏热门内容，即FN越小越好，FP 可以大一些，所以我们要求**召回率越高越好,精确率次之**

### 宏平均和微平均

对于求准确率还有宏平均和微平均两种方式,宏平均是对直接每个分类的准确率求平均值,而微平均要先对所有分类的预测结果求和再计算平均值,举个例子:

```
    Class A: 1 TP and 1 FP
    Class B: 10 TP and 90 FP
    Class C: 1 TP and 1 FP
    Class D: 1 TP and 1 FP
```
对于上面的分类结果:

准确率 $pA=pC=pD=0.5$, $pB=0.1$

$宏平均准确率(macro-avg)=$${0.5+0.1+0.5+0.5}\over{4}$$=0.4$

$微平均准确率(micro-avg)=$${1+10+1+1}\over{2+100+2+2}$$=0.123$


这个例子体现了宏平均把所有分类的**权重都视为1**的问题，在进行C分类时只有0.1的准确率，并且分类C的样本数占整体的90%以上，却没有影响宏平均的结果，所以一般把宏平均改进为加权的宏平均(权重是分类样本占总数的比例):
$macro-weight-avg=$$0.0189\times0.5+0.943\times0.1+0.0189\times0.5+0.0189\times0.5=0.123$

sklearn中求宏平均/微平均的精确率,猜下结果是多少?
```python
y_predict = [1,0,1]
y_true = [0,0,1]
macro=precision_score(y_true, y_predict,average="macro")
micro=precision_score(y_true, y_predict,average="micro")
```

## 算法选择
### sklearn常用分类算法

**决策树**: 使用树形结构,把特征作为决策树上的节点,叶节点就是分类结果,构造决策树时追求最纯净的分类结果(越纯净则分类的不确定性越低)

**朴素贝叶斯**: 在你不知道事件全貌的情况下,先根据一点人生经验得到一个主观判断,然后根据后续观察结果进行修正.根据概率大小判断最后的分类,常用于文本分类

**SVM**: 几何解法,先把所有样本用多维空间的向量表示,然后求一个平面将不同分类的样本分隔开

**KNN**: K-Nearest Neighbor,几何解法,因为"近朱者赤,近墨者黑",所以对于节点A,相邻最近的K个节点是什么类型,A大概就是什么类型

**集成算法**: 本着"人多力量大"的原则,使用多个分类器一起工作,按分类器的协作方式分为两种,bagging:分类器一起投票,看哪个分类票多;boosting:再学习,通过多次迭代强化整体的分类效果

### 效果对比
准备了训练数据24w条,测试数据2.7w条,训练数据来自上百个tag,非热门内容占比较少,测试来自两个tag,非热门数据占比更接近整体比例.
```python
import pandas as pd
from sklearn import naive_bayes
from sklearn.ensemble import AdaBoostClassifier, RandomForestClassifier
from sklearn.metrics import classification_report
from sklearn.neighbors import KNeighborsClassifier
from sklearn.tree import DecisionTreeClassifier
from sklearn.utils import shuffle

clf_list = {
    'cart': DecisionTreeClassifier(max_depth=5),
    # 'svm': svm.SVC(),  # 太慢了，放弃
    # 'nb_gauss': naive_bayes.GaussianNB(), # 太差了，放弃
    'nb_multi': naive_bayes.MultinomialNB(),
    'k_neighbors': KNeighborsClassifier(),
    'adaptive_boost': AdaBoostClassifier(),
    'random_forest': RandomForestClassifier(max_depth=2)
}

def train():
    train_data = pd.read_json('debug/train_video.jl', lines=True)
    train_data = shuffle(train_data)

    # 特征选择
    features = ['views', 'likes', 'comments', 'tagNumber', 'totalMedia', 'days', 'like_per_hour', 'view_per_hour']
    train_features = train_data[features]
    train_labels = train_data['hot']

    # 测试数据
    test_data = pd.read_json('debug/test_video.jl', lines=True)
    test_data = shuffle(test_data)
    test_features = test_data[features]
    test_labels = test_data['hot']

    for k, clf in clf_list.items():
        print(f'-----{k} result-------')
        # 决策树训练
        clf.fit(train_features, train_labels)
        target_names = ['normal', 'hot']
        test_predict = clf.predict(test_features)
        sample_weight = test_labels.replace(1, 100).replace(0, 1)
        # 打印测试报告
        print(classification_report(test_labels, test_predict, target_names=target_names, sample_weight=sample_weight))
```
在测试报告中,f1是precision/recall的综合平均分,support是分类的样本权重(默认按样本数量计算权重).因为ins内容热门数量比较少,所以我用sample_weight参数把hot分类的权重调整为normal分类的100倍,macro/weighted avg就是上面说的宏平均和加权宏平均.
观察测试报告可以发现,CART决策树的对热门内容的召回率最高,达到92%,朴素贝叶斯次之
```
-----cart result-------
              precision    recall  f1-score   support

      normal       0.44      0.90      0.59   24222.0
         hot       0.99      0.92      0.95  344800.0

    accuracy                           0.92  369022.0
   macro avg       0.72      0.91      0.77  369022.0
weighted avg       0.96      0.92      0.93  369022.0

-----nb_multi result-------
              precision    recall  f1-score   support

      normal       0.17      0.63      0.27   24222.0
         hot       0.97      0.79      0.87  344800.0

    accuracy                           0.78  369022.0
   macro avg       0.57      0.71      0.57  369022.0
weighted avg       0.92      0.78      0.83  369022.0

-----k_neighbors result-------
              precision    recall  f1-score   support

      normal       0.11      0.69      0.20   24222.0
         hot       0.97      0.63      0.76  344800.0

    accuracy                           0.63  369022.0
   macro avg       0.54      0.66      0.48  369022.0
weighted avg       0.91      0.63      0.72  369022.0

-----adaptive_boost result-------
              precision    recall  f1-score   support

      normal       0.08      0.99      0.15   24222.0
         hot       1.00      0.20      0.34  344800.0

    accuracy                           0.25  369022.0
   macro avg       0.54      0.60      0.24  369022.0
weighted avg       0.94      0.25      0.32  369022.0

-----random_forest result-------
              precision    recall  f1-score   support

      normal       0.10      0.96      0.18   24222.0
         hot       0.99      0.37      0.54  344800.0

    accuracy                           0.41  369022.0
   macro avg       0.54      0.67      0.36  369022.0
weighted avg       0.93      0.41      0.52  369022.0

```
为了进一步验证，将测试集分为14个大小2000的子集，然后分别记录每次的测试结果，画出折线图，选择的标准是热门内容的召回率&精确率和整体的准确率，在14次测试中CART算法都表现最好。
重复测试的代码如下：
```python
def train():
    train_data = pd.read_json('debug/train_video.jl', lines=True)
    train_data = shuffle(train_data)

    # 特征选择
    features = ['views', 'likes', 'comments', 'tagNumber', 'totalMedia', 'days', 'like_per_hour', 'view_per_hour']
    train_features = train_data[features]
    train_labels = train_data['hot']

    # 测试数据
    test_data = pd.read_json('debug/test_video.jl', lines=True)
    test_data = shuffle(test_data)

    # 切分为每组大小为2000的集合
    test_size = 2000
    test_list = [test_data[i:i + test_size] for i in range(0, test_data.shape[0], test_size)]
    reports = {}
    for k, clf in clf_list.items():
        print(f'-----{k} result-------')
        # 决策树训练
        clf.fit(train_features, train_labels)
        target_names = ['normal', 'hot']
        for test_set in test_list:
            test_features = test_set[features]
            test_labels = test_set['hot']
            test_predict = clf.predict(test_features)
            sample_weight = test_labels.replace(1, 100).replace(0, 1)
            report = classification_report(test_labels, test_predict, target_names=target_names,
                                           sample_weight=sample_weight, output_dict=True)
            reports.setdefault(f'{k}_hot_recall', []).append(report['hot']['recall'])
            reports.setdefault(f'{k}_hot_precision', []).append(report['hot']['precision'])
            reports.setdefault(f'{k}_accuracy', []).append(report['accuracy'])

    df = pd.DataFrame(reports)
    metrics = ['hot_recall', 'hot_precision', '_accuracy']
    fig, axes = plt.subplots(nrows=3, figsize=(16, 20))
    for i, m in enumerate(metrics):
        m_df = df.filter(regex=m)
        m_df.plot(ax=axes[i], title=m, xticks=range(1, len(test_list) + 1))
    fig.savefig(f"debug/report.png")
```
测试结果:
<div align=center> {% asset_img report.png image %} </div>

### 未解决的问题

这里我们缺失了机器学习最关键的一步,那就是调参(手动狗头),总感觉使用默认参数的模型没有灵魂.因为按直觉来说随机森林和AdaBoost应该优于决策树才对,结果准确率和召回率相差都很大
另外测试结果容易受测试集影响,当我使用只有39个样本，其中只有一个非热门内容的测试集时，决策树的召回率反而最低，所以选择测试集时要注意样本数量和真实性.
```
-----cart result-------
              precision    recall  f1-score   support

      normal       0.00      1.00      0.00       1.0
         hot       1.00      0.79      0.88    3800.0

    accuracy                           0.79    3801.0
   macro avg       0.50      0.89      0.44    3801.0
weighted avg       1.00      0.79      0.88    3801.0

-----nb_multi result-------
              precision    recall  f1-score   support

      normal       0.01      1.00      0.02       1.0
         hot       1.00      0.97      0.99    3800.0

    accuracy                           0.97    3801.0
   macro avg       0.50      0.99      0.50    3801.0
weighted avg       1.00      0.97      0.99    3801.0

```


### 保存分类器

把表现最好的CART分类器模型保存到代码文件中，然后当需要对ins内容进行分类就可以直接用模型进行判断了.保存模型时我使用的是cPickle(
除了cPickle还可以使用joblib,python3.6直接import _pickle即可，而joblib还需要安装)
保存非常简单,调用dump()即可
```
import _pickle as cPickle
    # 保存
    with open('ins_hot_model.clf','wb') as f:
        cPickle.dump(clf,f)
```
使用时需要从文件中加载分类器:
```
def use_model():
    # 未分类的ins文档信息
    doc = {"likes": 107, "comments": 2, "views": 435.0, "tagNumber": 25, "totalMedia": 22000, "days": 1343,
           "like_per_hour": 0, "view_per_hour": 0}
    # 读取分类器       
    clf = cPickle.load(open('ins_hot.clf', "rb"))
    # 把文档转换为DataFrame
    source = pd.DataFrame([doc], columns=list(doc.keys()))
    features = ['views', 'likes', 'comments', 'tagNumber', 'totalMedia', 'days', 'like_per_hour', 'view_per_hour']
    df = source[features]
    # 为文档分类
    print(clf.predict(df))
```

## 总结
使用机器学习进行数据分析时,需要经过数据采集,清洗,特征选择,模型训练几个过程.借助sklearn库可以让我们在不了解算法原理的情况下也能轻松地使用机器学习进行数据分析,而对于在生产环境中用机器学习解决实际问题,还是需要丰富的经验和大量的优化验证才可以,看来要成为高薪的算法工程师也不是那么容易.
所以,亲爱的朋友,你想成为算法大佬吗?你想学习数据分析吗?现在扫码即可8折购买参加极客时间的数据分析课程哦😯

<div align=center> {% asset_img jike_ad.jpeg image %} </div>




  [1]: https://throwsnew.com/images/example.jpg
  [2]: https://ss0.bdstatic.com/70cFuHSh_Q1YnxGkpoWK1HF6hhy/it/u=2143959536,453965174&fm=26&gp=0.jpg