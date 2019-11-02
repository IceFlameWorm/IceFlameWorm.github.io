---
title: SHAP VALUES —— 什么影响了你的决定？
date: 2019-09-01 10:59:02
tags:
    - machine learning
    - data mining
    - 可解释性
categories: 可解释性
description: 很多指标都是在总体样本上衡量特征的影响，但是针对某一特定样本，该如何表示各个特征对其预测结果的影响呢？针对某一样本的预测结果，SHAP值通过跟基线结果作比较，得出各个特征的取值分别对预测结果的影响程度。
copyright: true
---

这是第四节：[SHAP VALUES](https://www.kaggle.com/dansbecker/shap-values)

# 用途

SHAP值（SHapley Additive exPlanations的缩写）从预测中把每一个特征的影响分解出来。可以把它应用到类似于下面的场景当中:
- 模型认为银行不应该给某人放贷，但是法律上需要银行给出每一笔拒绝放贷的原因。
- 医务人员想要确定对不同的病人而言，分别是哪些因素导致他们有患某种疾病的风险，这样就可以因人而异地采取针对性的卫生干预措施，直接处理这些风险因素。

# 工作原理

SHAP值通过与某一特征取基线值时的预测做对比，来解释该特征取某一特定值的影响。

可以继续用[排列重要性](http://iceflameworm.github.io/2019/08/17/permutaion-importance/)和[部分依赖图](http://iceflameworm.github.io/2019/08/28/partial-plots/)中用到的例子进行解释。

我们对一个球队会不会赢得“最佳球员”称号进行了预测。

我们可能会有以下疑问：
- 预测的结果有多大的程度是由球队进了3个球这一事实影响的？

但是，如果我们像下面这样重新表述一下的话，那么给出具体、定量的答案还是比较容易的：
- 预测的结果由多大的程度时由球队进了3个球这一事实影响的，而不是某些基线进球数？

当然，每个球队都由很多特征，所以，如果我们能回答“进球数”的问题，那么我们也能对其它特征重复这一过程。

SHAP值用一种保证良好性质的的方式做这件事。具体而言，用如下等式对预测进行分解：

```
sum(SHAP values for all features) = pred_for_team - pred_for_baseline_values
```

即，用所有特征的SHAP值的加和来解释为什么预测结果与基线不同。这就允许我们用像下面这样的一幅图来对预测进行分解：

![img](shap_graph_1.jpg)

该怎么解释这幅图呢？

我们预测的结果时0.7，而基准值是0.4979。引起预测增加的特征值是粉色的，它们的长度表示特征影响的程度。引起预测降低的特征值是蓝色的。最大的影源自`Goal Scored`等于2的时候。但`ball possesion`的值则对降低预测的值具有比较有意义的影响。

如果把粉色条状图的长度与蓝色条状图的长度相减，差值就等于基准值到预测值之间的距离。

要保证基线值加上每个特征各自影响的和等于预测值的话，在技术上还是有一些复杂度的（这并不像听上去那么直接）。我们不会研究这些细节，因为对于使用这项技术来说，这并不是很关键。[这篇博客](https://towardsdatascience.com/one-feature-attribution-method-to-supposedly-rule-them-all-shapley-values-f3e04534983d)对此做了比较长篇幅的技术解释。

# 代码示例

这里，我们用[Shap库](https://github.com/slundberg/shap)计算SHAP值。

沿用[部分依赖图](http://iceflameworm.github.io/2019/08/28/partial-plots/)中用到的足球数据。

input:
```python
import numpy as np
import pandas as pd
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier

data = pd.read_csv('../input/fifa-2018-match-statistics/FIFA 2018 Statistics.csv')
y = (data['Man of the Match'] == "Yes")  # Convert from string "Yes"/"No" to binary
feature_names = [i for i in data.columns if data[i].dtype in [np.int64, np.int64]]
X = data[feature_names]
train_X, val_X, train_y, val_y = train_test_split(X, y, random_state=1)
my_model = RandomForestClassifier(random_state=0).fit(train_X, train_y)
```

看一下数据集某一行数据上的SHAP值（就随意第5行吧）。在查看SHAP值之前，先看一下最原始的预测值。

input:
```python
row_to_show = 5
data_for_prediction = val_X.iloc[row_to_show]  # use 1 row of data here. Could use multiple rows if desired
data_for_prediction_array = data_for_prediction.values.reshape(1, -1)


my_model.predict_proba(data_for_prediction_array)
```

output:
```
array([[0.3, 0.7]])
```

该球队有70%的可能性赢得这一奖项。

接下来是给上面那条预测计算SHAP值的代码。

```python
import shap  # package used to calculate Shap values

# Create object that can calculate shap values
explainer = shap.TreeExplainer(my_model)

# Calculate Shap values
shap_values = explainer.shap_values(data_for_prediction)
```

上面的`shap_values`对象是一个包含两个array的list。第一个array是负向结果（不会获奖）的SHAP值，而第二个array是正向结果（获奖）的SHAP值。通常我们从预测正向结果的角度考虑模型的预测结果，所以我们会拿出正向结果的SHAP值（拿出`shap_values[1]`）。

直接看原始array很麻烦，但是`shap`库提供一种不错的结果可视化的方式。

input:
```python
shap.initjs()
shap.force_plot(explainer.expected_value[1], shap_values[1], data_for_prediction)
```

output:

![img](shap_graph_2.jpg)

如果仔细观察一下计算SHAP值的代码，就会发现在`shap.TreeExplainer(my_model)`中涉及到了树。但是`SHAP`库有用于各种模型的解释器。
- `shap.DeepExplainer`适用于深度学习模型
- `shap.KernelExplainer` 适用于各种模型，但是比其它解释器慢，它给出的是SHAP值的近似值而不是精确值。
`
下面是用`KernelExplainer`得到类似结果的例子。结果跟上面并不一致，这是因为`KernelExplainer`计算的是近似值，但是表达的意思是一样的。

input:

```python
# use Kernel SHAP to explain test set predictions
k_explainer = shap.KernelExplainer(my_model.predict_proba, train_X)
k_shap_values = k_explainer.shap_values(data_for_prediction)
shap.force_plot(k_explainer.expected_value[1], k_shap_values[1], data_for_prediction)
```

output:

![img](shap_graph_3.jpg)

# 练习

[kaggle小练习](https://www.kaggle.com/kernels/fork/1637226)