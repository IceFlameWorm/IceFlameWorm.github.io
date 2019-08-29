---
title: Partial Dependence Plots —— 部分依赖图
date: 2019-08-28 23:33:26
tags:
    - machine learning
    - data mining
    - 可解释性
categories: 可解释性
description: 部分依赖图可以用来展示一个特征是怎样影响模型预测的。可以用部分依赖图回答一些与下面这些类似的问题：1. 假如保持其它所有的特征不变，经纬度对房价有什么影响？换句话说，相同大小的房子，在不同的地方价格会有什么差别？2. 在两组不同的人群上，模型预测出的健康水平差异是由他们的负债水平引起的，还是另有原因？
copyright: true
---

这是第三节：[Partial Dependence Plots](https://www.kaggle.com/dansbecker/partial-plots)

特征重要性展示的是哪些变量对预测的影响最大，而部分依赖图展示的是特征如何影响模型预测的。

可以用部分依赖图回答一些与下面这些类似的问题：

- 假如保持其它所有的特征不变，经纬度对房价有什么影响？换句话说，相同大小的房子，在不同的地方价格会有什么差别？
- 在两组不同的人群上，模型预测出的健康水平差异是由他们的负债水平引起的，还是另有原因？

如果你对线性回归或逻辑回归比较熟悉的话，部分依赖图起到的效果跟这些模型里面的参数差不多。但是，与简单模型中的参数相比，复杂模型上的依赖图可以捕捉到更复杂的模式。


# 工作原理

跟排列重要性一样，**部分依赖图也是要在拟合出模型之后才能进行计算。** 模型是在真实的未加修改的真实数据上进行拟合的。

以足球比赛为例，球队间可能在很多方面都存在着不同。比如传球次数，射门次数，得分数等等。乍看上去，似乎很难梳理出这些特征产生的影响。

为了搞清楚部分依赖图是怎样把每个特征的影响分离出来的，首先我们只看一行数据。比如，这行数据显示的可能是一支占有50%的控球时间，传了100次球，射门了10次，得了1分的球队。

接下来，利用训练好的模型和上面的这一行数据去预测该队斩获最佳球员的概率。但是，我们会**多次改变某一特征的数值**，从而产生一系列预测结果。比如我们会在把控球时间设成40%的时候，得到一个预测结果，设成50%的时候，得到一个预测结果，设成60%的时候，也得到一个结果，以此类推。以从小到大设定的控球时间为横坐标，以相应的预测输出为纵坐标，我们可以把实验的结果画出来。

# 代码示例

在这里，重点不是建模过程，所以在下面的代码中，不会有过多数据探索和建模的内容。

训练模型

```python
import numpy as np
import pandas as pd
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
from sklearn.tree import DecisionTreeClassifier

data = pd.read_csv('../input/fifa-2018-match-statistics/FIFA 2018 Statistics.csv')
y = (data['Man of the Match'] == "Yes")  # Convert from string "Yes"/"No" to binary
feature_names = [i for i in data.columns if data[i].dtype in [np.int64]]
X = data[feature_names]
train_X, val_X, train_y, val_y = train_test_split(X, y, random_state=1)
tree_model = DecisionTreeClassifier(random_state=0, max_depth=5, min_samples_split=5).fit(train_X, train_y)
```

为了节省解释的时间，第一个例子用决策树，如下所示。在实际应用中，你可能会用到更复杂的模型。

决策树结构可视化

```python
from sklearn import tree
import graphviz

tree_graph = tree.export_graphviz(tree_model, out_file=None, feature_names=feature_names)
graphviz.Source(tree_graph)
```

![img](tree_structure.jpg)

怎么理解得到的决策树：
- 非叶子节点顶部的数值表示分支标准
- 节点最底部的一对数字表示当前节点上正负类样本的数目

可以用[**PDPBox库**](https://pdpbox.readthedocs.io/en/latest/)来生成部分依赖图。示例如下：

```python
from matplotlib import pyplot as plt
from pdpbox import pdp, get_dataset, info_plots

# Create the data that we will plot
pdp_goals = pdp.pdp_isolate(model=tree_model, dataset=val_X, model_features=feature_names, feature='Goal Scored')

# plot it
pdp.pdp_plot(pdp_goals, 'Goal Scored')
plt.show()
```

![img](pdp_1.jpg)

在看上面的部分依赖图的时候，有两点值得注意的地方：
- y轴表示的是**模型预测**相较于基线值或最左边的值的**变化**。
- 蓝色阴影部分表示置信区间。

从这幅图可以看出，进一个球会显著地增加获得最佳球员称号地机会，但是进更多的球似乎对预测的影响不大。

下面是另外一个示例图：

```python
feature_to_plot = 'Distance Covered (Kms)'
pdp_dist = pdp.pdp_isolate(model=tree_model, dataset=val_X, model_features=feature_names, feature=feature_to_plot)

pdp.pdp_plot(pdp_dist, feature_to_plot)
plt.show()
```

![img](pdp_2.jpg)

这种图似乎太简单了，并不能代表现实情况。其实这是因为模型太简单了。从上面的的决策树结构图可以看出，上面两幅部分依赖图展示的结果正是决策树的结构。

通过部分依赖图，可以比较轻松地比较不同模型的结构或含义。下面是一个随机森林的例子：

```python
# Build Random Forest model
rf_model = RandomForestClassifier(random_state=0).fit(train_X, train_y)

pdp_dist = pdp.pdp_isolate(model=rf_model, dataset=val_X, model_features=feature_names, feature=feature_to_plot)

pdp.pdp_plot(pdp_dist, feature_to_plot)
plt.show()
```

![img](pdp_3.jpg)

这个模型认为在比赛过程中，如果所有球员一共跑动了100km的话，球队会更有可能斩获最佳球员。但是跑动得更多的话，可能性就会下降一些。

一般来说，这条曲线的光滑形态看上去比决策树的阶跃函数更可信。但是因为例子中用的数据集太小了，所以在对任意一个模型进行解释的时候，要特别注意选用的方式。


# 2D 部分依赖图

如果你对特征之间的相互作用感兴趣的话，2D部分依赖图就能排得上用场了。举个例子看下。

仍然以决策树模型为例，下面会生成一个非常简单的图，但是你仍然能够把你从图中看出的东西跟决策树本身的结果匹配到一起。

```python
# Similar to previous PDP plot except we use pdp_interact instead of pdp_isolate and pdp_interact_plot instead of pdp_isolate_plot
features_to_plot = ['Goal Scored', 'Distance Covered (Kms)']
inter1  =  pdp.pdp_interact(model=tree_model, dataset=val_X, model_features=feature_names, features=features_to_plot)

pdp.pdp_interact_plot(pdp_interact_out=inter1, feature_names=features_to_plot, plot_type='contour', plot_pdp=True)
plt.show()
```

![img](2d_pdp.jpg)

上面这幅图展示了在`Goals Scored`和`Distance coverd`两个特征的任意组合上的预测结果。

比如，当一个球队得了至少一分，并且跑通的总距离接近100km的时候，模型预测的结果最高。如果得了0分，跑动的距离就没什么用了。你能从决策树中看出这一结果吗？

但是如果球队得分了的话，跑动的距离是会影响到预测的。确定你可以从2D的部分依赖图中看出这一个结果。你能从决策树中看出同样的结果吗?

# 练习

[kaggle小练习](https://www.kaggle.com/kernels/fork/1637380)