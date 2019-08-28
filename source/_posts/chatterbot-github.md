---
title: 开源框架gunthercox/ChatterBot浅析
date: 2019-08-12 16:57:13
tags:
    - NLP
    - chatbot
    - 开源框架
categories: chatbot
description: 简单梳理一下github上开源的一个聊天机器人框架：gunthercox/ChatterBot，内容主要包含工作流程，训练器和逻辑适配器三部分。由于作者初涉chatbot这一领域，所以对某些部分的理解可能不到位，敬请谅解。
copyright: true
---

转载请注明来源：http://iceflameworm.github.io/2019/08/12/chatterbot-github/

# 框架简介
Chatterbot是一个完全用python编写的基于文本检索/匹配的聊天机器人框架，它会从保存的对话语料中找出与输入句子最匹配的句子，并把匹配到的句子的下一句作为回答返回。本文主要对其工作流程，以及核心的训练器和逻辑适配器进行梳理，具体使用方法，请参考其文档。

框架地址：https://github.com/gunthercox/ChatterBot   
文档地址：https://chatterbot.readthedocs.io/en/stable/

# 工作流程
![img](workflow.png)

原文档中有两幅描述工作流程的示意图，一幅在[文档首页](https://chatterbot.readthedocs.io/en/stable/index.html)，一幅在[文档-逻辑适配器](https://chatterbot.readthedocs.io/en/stable/logic/index.html)，个人认为后者描述的更全面、更恰当些，所以就以后者为准进行介绍。

从输入句子到输出响应回答，前后需要经历三大步：
1. 预处理
2. 生成答案
3. 答案选择

具体流程请参考[chatterbot.py](https://github.com/gunthercox/ChatterBot/blob/master/chatterbot/chatterbot.py)。

## 预处理

与通常所讲的NLP预处理的目的基本一致，主要是文本进行一些标准还操作，比如去除连续的空格、删除特殊字符等等。Chatterbot框架自身实现了`clean_whitespace`, `unescape_html`和`convert_to_ascii`三种预处理功能。具体实现参见[preprocessors.py](https://github.com/gunthercox/ChatterBot/blob/master/chatterbot/preprocessors.py)

## 生成答案

一个Chatterbot实例可以绑定多个逻辑适配器，用于根据输入产生输出。

Chatterbot中没有独立的用于选择对话逻辑的意图识别模块，它将意图识别的功能放到了各个逻辑适配器中。接收到输入之后，Chatterbot会将其传递给各个逻辑适配器，由它们自己判断是否适合对输入的文本进行回答。如果逻辑适配器认为不能对输入进行回答，则会跳过，否则就输出回答，这样的话，有可能所有逻辑适配器都不输出回答，也有可能有多个逻辑适配器都给出了回答。具体请参考：[ChatBot::generate_response](https://github.com/gunthercox/ChatterBot/blob/master/chatterbot/chatterbot.py)方法。

从[ChatBot::\_\_init\_\_和Chatbot::generate_response](https://github.com/gunthercox/ChatterBot/blob/master/chatterbot/chatterbot.py)中的两端代码

```python
# __init__
self.storage = utils.initialize_class(storage_adapter, **kwargs)

...

# generate_response
for adapter in logic_adapters:
    utils.validate_adapter_class(adapter, LogicAdapter)
    logic_adapter = utils.initialize_class(adapter, self, **kwargs)
    self.logic_adapters.append(logic_adapter)
```
可知，所有的逻辑适配器都共享一份保存的对话语料。倘若逻辑适配器内部不对数据进行选择的话，所有的逻辑适配器都将从所有的对话语料数据中查找最匹配的回答。这样的结果就是，每个逻辑适配器在相同的数据上用不同的匹配方法或指标产生回答，衡量每个回答confidence的标准并不一致，这会影响后续根据confidence进行答案选择。


## 答案选择

[工作流程示意图](#工作流程)显示，Chatterbot会从所有的逻辑适配器返回的回答中，选择confidence最大的，但是[ChatBot::generate_response](https://github.com/gunthercox/ChatterBot/blob/master/chatterbot/chatterbot.py)实现的是另外一种逻辑。在`Chatbot::generate_response`中，每个逻辑适配器输出的**confidence并没有用到**，它会统计每一个返回的回答出现的次数，如果有出现次数大于1次的，则会返回出现次数最多的回答，**但是如果所有逻辑适配器返回的回答都只出现了一次，则会第一个逻辑适配器的答案，个人认为这种逻辑存在缺陷**。以下是相关代码：

```python
# If multiple adapters agree on the same statement,
# then that statement is more likely to be the correct response
if len(results) >= 3:
    result_options = {}
    for result_option in results:
        result_string = result_option.text + ':' + (result_option.in_response_to or '')

        if result_string in result_options:
            result_options[result_string].count += 1
            if result_options[result_string].statement.confidence < result_option.confidence:
                result_options[result_string].statement = result_option
        else:
            result_options[result_string] = ResultOption(
                result_option
            )

    most_common = list(result_options.values())[0]

    for result_option in result_options.values():
        if result_option.count > most_common.count:
            most_common = result_option

    if most_common.count > 1:
        result = most_common.statement
```

# 训练器

刚开始的时候，以为这里的训练跟普通的训练是一样的，也就是通过数据+训练过程确定模型的参数。实际上，这里的训练过程**不能算作真正的训练，有点跟KNN算法的训练过程差不多**（KNN也没有真正意义上的训练过程），所谓的训练过程其实就是准备检索数据的过程。结合[文档示例](https://chatterbot.readthedocs.io/en/stable/training.html)和  [ListTrainer](https://github.com/gunthercox/ChatterBot/blob/master/chatterbot/trainers.py) 可以看出Chatterbot框架中的训练过程实际上是怎样的。

文档中的示例

```python
from chatbot import chatbot
from chatterbot.trainers import ListTrainer

trainer = ListTrainer(chatbot)

trainer.train([
    "Hi there!",
    "Hello",
])

trainer.train([
    "Greetings!",
    "Hello",
])
```

`ListTrainer`

```python
class ListTrainer(Trainer):
    """
    Allows a chat bot to be trained using a list of strings
    where the list represents a conversation.
    """

    def train(self, conversation):
        """
        Train the chat bot based on the provided list of
        statements that represents a single conversation.
        """
        previous_statement_text = None
        previous_statement_search_text = ''

        statements_to_create = []

        for conversation_count, text in enumerate(conversation):
            if self.show_training_progress:
                utils.print_progress_bar(
                    'List Trainer',
                    conversation_count + 1, len(conversation)
                )

            statement_search_text = self.chatbot.storage.tagger.get_text_index_string(text)

            statement = self.get_preprocessed_statement(
                Statement(
                    text=text,
                    search_text=statement_search_text,
                    in_response_to=previous_statement_text,
                    search_in_response_to=previous_statement_search_text,
                    conversation='training'
                )
            )

            previous_statement_text = statement.text
            previous_statement_search_text = statement_search_text

            statements_to_create.append(statement)

        self.chatbot.storage.create_many(statements_to_create)
```

对话语料的保存并不是以**文本对**为单位的，而是把每一句话作为最基本的存储单元，语句之间的关系通过`*in_response_to`字段表示。如下述代码所示，`Statement`本身的`text`是用来回答`in_response_to`对应的`previous_statement_text`这句话的。

```python
Statement(
    text=text,
    search_text=statement_search_text,
    in_response_to=previous_statement_text,
    search_in_response_to=previous_statement_search_text,
    conversation='training'
)
```

# 逻辑适配器

逻辑适配器主要用于根据输入文本产生相应的回答。Chatterbot本身实现了一个[BestMatch](https://github.com/gunthercox/ChatterBot/blob/master/chatterbot/logic/best_match.py)的逻辑适配器，它会从保存的对话语料中找出与输入文本最匹配的回答，其基本检索流程是：

1. 根据某一相似度度量指标，从保存的对话语料中找到与输入文本最相似的文本 mtext。
2. 遍历整个对话语料，找出所有可以用来回答m_text的文本。

从上述步骤可以看出， `BestMatch`找到回答需要遍历两次保存的对话语料，在实现方式上可能不是最优的。

具体流程参考 [BestMatch::process](https://github.com/gunthercox/ChatterBot/blob/master/chatterbot/logic/best_match.py)。


在文本相似度度量上，Chatterbot本身已经实现了多种不同的方法：
1. LevenshteinDistance-编辑距离
2. JaccardSimilarity
3. 使用Spacy计算的相似度

具体请参考 [comparisons.py](https://github.com/gunthercox/ChatterBot/blob/master/chatterbot/comparisons.py)。