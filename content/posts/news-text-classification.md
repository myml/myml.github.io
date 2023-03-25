---
title: 使用 Transformers 训练新闻分类模型
date: 2023-03-20
draft: false
tags: ["pytorch"]
categories: ["编程"]
---

最近开始接触机器学习，今天尝试用 Transformers 训练一个文本多分类模型，实现根据标题对新闻进行分类。

文本分类也可用在其他地方，比如垃圾邮件/帖子识别、内容情感分析等，就个人理解，模型的功能和具体表现在于训练数据的内容。

## 数据读取

这次使用的训练数据集是 [今日头条新闻](https://github.com/myml/toutiao-text-classfication-dataset)，有 30W+ 条数据，15个分类，训练就想做菜一样，需要提前对食材进行处理，
<!--more-->
比如这次的数据集并不是标准的 csv 或 json 格式，需要在下载解压再手动解析，这里使用pandas读取数据：
```python
import pandas as pd
df = pd.read_csv('toutiao_cat_data.txt',
                 sep='_!_', lineterminator='\n',
                 encoding='utf8',
                 names=["id", "type_id", "type_text", "text", "keywords"])
```
传入参数如下
- sep: 字段分隔符
- lineterminator: 行分隔符
- names: 定义字段名
- encoding: 文件字符编码

## 数据处理

训练需要用到 text 和 type_id 数据，并且分类 id 应该从 0 开始，而这个数据集的 type_id 是以100开始的，需要稍作调整：
```python
df["label"] = df["type_id"]-100
# label 和 text 是训练时默认使用的列名
df = df[["label", "text"]]
print(df.shape)
# 输出 (382688, 2)
```

到目前为止，我们得到了一个存放在内存中的待训练数据集，这个数据集有2列382688行，text 列是中文新闻标题，label 列是新闻对应的分类。

遗憾的是机器看不懂中文（当然它也看不懂英文），我们需要将机器无法理解的内容翻译成数字，这一步叫做标记化（tokenization）：
```python
from transformers import AutoTokenizer
tokenizer = AutoTokenizer.from_pretrained("bert-base-chinese")

def tokenize_function(examples):
    return tokenizer(
        examples["text"], 
        padding="max_length",
        max_length=50,
        truncation=True,
    )
```
- padding 由于新闻标题长短不一，需要在长度不够的标记后补充0值使标记长度统一。
- truncation 同上理由，需要截断过长的标记使标记长度统一。
- max_length 设置标记最大长度，需要针对训练内容合理设置，长度设置过短会因为内容被裁剪影响训练效果，标记过长会影响训练速度。

这里定义了一个标记函数，用于将中文标题转化为机器可理解的数字序列，[bert-base-chinese](https://huggingface.co/bert-base-chinese) 是用于处理中文的 bert 预训练模型，**如果使用和内容语种不一致的标记器会严重影响训练效果**。

```python
from datasets import Dataset
df = df.sample(frac=1)
train_dataset = Dataset.from_pandas(
    df[:-10000]).map(tokenize_function, batched=True)

eval_dataset = Dataset.from_pandas(
    df[-10000:]).map(tokenize_function, batched=True)

print(train_dataset)
# 输出 Dataset({
#       features: ['label', 'text', 'input_ids'， 'token_type_ids', 'attention_mask'],
#    num_rows: 10000
#     })
```
- batched 是否批量执行 map

这里对中文数据进行标记化，并且顺便切分了最后10000条数据当作测试集，用于评估训练成果，由于测试集是从最后切分的，所以在切分前使用 `df.sample(frac=1)` 对原始数据进行全量随机排序，避免数据是按分类 id 排序过的。

可以看到除了原来的 'label', 'text' 字段，还多出了三个字段：'input_ids'， 'token_type_ids', 'attention_mask'，这些就是标记化之后的数据，之后训练会用到。

## 训练模型

我们并不需要从头训练自己的模型，因为那涉及到大量的数据量和计算量，这里我们在 `bert-base-chinese` 模型（预训练模型和标记器要保持一致）的基础上进行训练，

```python
from transformers import AutoModelForSequenceClassification

model = AutoModelForSequenceClassification.from_pretrained(
    "bert-base-chinese",
    num_labels=17,
)
```
需要注意的是参数 num_labels，这是分类 id 的数量，请回过头来仔细阅读我们数据集的README解释

> 分类code与名称：
> 
> 100 民生 故事 news_story
> 
> ...
> 
> 116 电竞 游戏 news_game
> 
> 数据规模：
> 共382688条，分布于15个分类中。

虽然这里描述了一共15个分类，但是分类 id 最小是100最大是116，应该是有17个，再仔细查看一边会发现分类列表里缺少105和111，如果你按照数据描述，将num_labels设置成15，会得到一个 `CUDA error: device-side assert triggered`

其实应该在数据处理阶段进行清洗，但我们就先将错就错吧，暂且放下这些细支末节来开启我们的训练！

```python
training_args = TrainingArguments(
    report_to="none",
    output_dir="test_trainer",
    num_train_epochs=1,
)
trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=train_dataset,
    eval_dataset=eval_dataset,
)
trainer.train()
```
- report_to 在训练过程中可以将指标上传到网站服务，用于分析训练过程，这里先禁用
- output_dir 训练过程中将模型存储到硬盘，在意外中断后可以继续开始
- num_train_epochs 训练迭代次数，影响训练时长和效果
- model 训练模型
- train_dataset 训练数据集
- eval_dataset 评估数据集

感谢前辈们的工作，开始一个训练并没有想象中那么麻烦，虽然训练可调参数有很多，但对于刚开始入门的我们可以先使用默认值。

## 保存模型
经过漫长的等待，训练终于结束了，来验证一下：
```python
from transformers import pipeline
pipe = pipeline("text-classification",
                device=0,
                model=model,
                tokenizer=tokenizer,
                )
print(pipe("来自召唤师峡谷的呼唤"))
# 输出 [{'label': 'LABEL_16', 'score': 0.9958072900772095}]
```
> 116 电竞 游戏 news_game

经过训练的模型准确分类了内容，但不要太兴奋了，你辛苦训练的模型还保留在内存中，如果你不想再经历一次漫长的训练过程，快将他们保存到硬盘里：
```python
trainer.save_model("result/")
tokenizer.save_pretrained("result/")
```
除了训练的模型，我们还应该将标记器进行保存，这样就可以将 result 文件夹进行打包贡献给朋友，他们只需要简单的三行代码就可使用你训练的模型
```python
from transformers import pipeline
pipe = pipeline("text-classification", model="./result")
print(pipe("来自召唤师峡谷的呼唤"))
```
## 优雅！