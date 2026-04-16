---
layout: simple
title: How to Use BERT
---

## **Intro**

개인적인 과제로 AI를 이용해 보안을 이용하는 과제를 받았다. 그래서 앞으로 AI 공부를 해야 할 듯 싶다.

첫번째 목표는 AI 를 처음부터 이해하지 말고 AI 를 사용하는 법부터 익히는 것이다. 처음부터 공부하기에는 AI의 양이 너무 방대하다. 

AI 를 사용하는 법을 익혀 보안에 적용할 수 있는 방법부터 익힌다. 

## **What is BERT?**

BERT (Bidirectional Encoder Representations from Transformers) 2018년에 Google에서 발표된 자연어 처리 (NLP) 모델이다. 

BERT는 Transformer라는 신경망 아키텍처를 기반으로 한다. Transformer는 어텐션 메커니즘을 사용하여 입력 시퀀스 간의 관계를 모델링하고, 셀프 어텐션과 여러 레이어로 구성된 구조를 가지고 있다. 

BERT 의 장점 중 하나는 바로 파인 튜닝(fine-tunning) 이 가능하다는 것이다. 파인튜닝이란 사전 학습된 기본 모델을 이용하여, 학습 데이터를 조정하거나 레이어를 한단계 추가하여 재학습 시켜 향상된 모델을 만드는 것을 의미한다. 

파인튜닝은 학습하는 데에 필요한 리소스가 처음부터 학습을 시키는 것에 비해 매우 적게 들면서 성능은 대폭 향상시킬 수 있는 방법이다. 

## **Use BERT**

이제 직접 BERT 를 사용해 간단한 예시를 만들어보자. 

### **Basic Use**

먼저 자연어를 모델에 입력하려면, 언어를 모델이 이용할 수 있는 형태로 바꿔줘야 한다. 이를 Embedding 과정이라고 한다. 

먼저 문장을 Tokenize 해주고, 그 뒤에 임베딩을 해주는 순이다. 

문장을 Tokenize 해주는 방법은 다음과 같다. 

```python
from transformers import BertTokenizer
import torch

tokenizer = BertTokenizer.from_pretrained('bert-base-uncased')
sentence = "I love panda"

token = tokenizer.tokenize(sentence)

#set CLS, SEP token
"""
[CLS] token is token which represent prologue for sentence
[SEP] token is token which represent epilogue for sentence
[PAD] token is padding token for token list which has static length 
"""
token = ['[CLS]'] + token + ['[SEP]']
print(token)

token = token + ['[PAD]'] + ['[PAD]']
print(token)
attention_mask = [1 if i != '[PAD]' else 0 for i in token]
print(attention_mask)

# make token ids
token_ids = tokenizer.convert_tokens_to_ids(token)
print(token_ids)

# convert token_ids, attension_mask to tensor
token_ids = torch.tensor(token_ids).unsqueeze(0)
attention_mask = torch.tensor(attention_mask).unsqueeze(0)

# get embeddings
"""
input token_ids, attention_mask to model
model outputs to values

hidden_rep: include all represent vectors
cls_head: include [CLS] token's representation
"""

hidden_rep, cls_head = model(token_ids, attention_mask=attention_mask)

print(hidden_rep.shape)
```

난 우선 `bert-base-uncased` 라는 pretrained 된 모델을 사용할 것이다. `BertTokenizer('bert-base-uncased')` 의 의미는 `bert-base-uncased` 모델을 학습할 때 썼던 토크나이저를 사용한다는 뜻이다. 

그 뒤에 문장의 앞을 의미하는 `[CLS]` 토큰과 문장의 끝을 의미하는 `[SEP]` 토큰을 설정해주고, 지정한 토큰의 길이만큼 `[PAD]` 토큰을 이용해 패딩을 해준다. 

`attention_mask`는 의미 없는 토큰을 학습에 넣어주지 않기 위해 의미있는 토큰과 의미없는 토큰을 지정해주는 마스크이다. `[PAD]` 토큰은 학습에 의미를 가지지 않는 토큰이므로 `[PAD]`를 0으로 처리한 `attention_mask`를 지정해준다. 

만들어진 토큰들은 `torch.tensor()` 를 이용해 모델이 받아들일 수 있는 벡터 형태로 변환한다. 

마지막으로 `model(token_ids, attention_mask=attention_mask)` 를 통해 pre-trained 된 모델에 벡터를 입력하면, 출력이 나온다. 

토큰화 및 벡터라이즈 과정은 다음을 통해 한줄로 처리할 수 있다. 

```python
from transformers import BertTokenizerFast
tokenizer = BertTokenizerFast.from_pretrained('bert-base-uncased')

input_text = ['I love Seoul', 'panda eats bamboo', 'snow fall']
input_data = tokenizer(input_text, padding=True, truncation=True, return_tensors="pt")
```

`BertTokenizerFast` 를 이용해 위 세 문장을 바로 임베딩 해주는 것이 가능하다. 

### **Fine-Tunning**

`BERT`를 fine-tunning 하여 용도에 맞게 만들어진 모델이 여러개 있다. 

나는 그 중에서 문장을 분류해주는 모델인 `BertForSequenceClassification`을 쓸 것이다. 

다음을 통해 fine-tunning 된 모델을 학습시킬 수 있다. 

```python
import os
from transformers import BertForSequenceClassification, BertTokenizerFast, Trainer, TrainingArguments
from nlp import load_dataset
import torch
import numpy as np


#load IMDB datasets for train
dataset = load_dataset('csv', data_files="./datasets/imdbs.csv", split='train')

# split data for train / test set
# if test_size = 0.3, test dataset is 30% of hole dataset and 70% is dataset which is used for train
dataset = dataset.train_test_split(test_size=0.3)
train_set = dataset['train']
test_set = dataset['test']

# load model
model = BertForSequenceClassification.from_pretrained('bert-base-uncased')

# load tokenizer
tokenizer = BertTokenizerFast.from_pretrained('bert-base-uncased')

# tokenize sentence
#print(tokenizer(['I love Seoul', 'panda eats bamboo', 'snow fall'], padding=True, max_length=5))

# define preproces which process input dataset
def preprocess(data):
    return tokenizer(data['text'], padding=True, truncation=True)
# preprocess sets
train_set = train_set.map(preprocess, batched=True, batch_size=len(train_set))
test_set = train_set.map(preprocess, batched=True, batch_size=len(test_set))

# set formats
train_set.set_format('torch', columns=['input_ids', 'attention_mask', 'label'])
test_set.set_format('torch', columns=['input_ids', 'attention_mask', 'label'])

# train model

# set batch_size, epochs
batch_size = 8
epochs = 2

# set warmup steps, and weight decay
warmup_steps = 500
weight_decay = 0.01

# define training arguments and trainer
training_args = TrainingArguments (
    output_dir = os.path.join("outputs", "train3"),
    num_train_epochs=epochs,
    per_device_train_batch_size=batch_size,
    per_device_eval_batch_size=batch_size, 
    warmup_steps=warmup_steps,
    weight_decay=weight_decay,
    logging_dir=os.path.join("logs", "train3")
)

trainer = Trainer (
    model=model,
    args=training_args,
    train_dataset=train_set,
    eval_dataset=test_set
)

# train start
trainer.train()
trainer.evaluate()
```

데이터 셋은 총 3가지로 나눌수 있다. 

+ train      : 훈련용 데이터셋
+ validation : 검증용 데이터셋
+ test       : 테스트용 데이터셋


현업에서 쓰는 모델은 세 데이터셋을 모두 사용해야 하지만, 이번에는 간단하게 train 과 test 데이터셋만 쓰도록 한다. 

내가 가지고 있는 데이터셋인 `datasets/imdbs.csv`을 로드하고, `train_test_split` 을 통해 데이터셋을 train : test = 7:3 으로 분리한다. 

그 뒤에 `BertForSequenceClassification`모델과 그 모델을 학습시킬 때 사용한 tokenizer 를 가져온다.

그리고 해당 triain과 test 데이터 셋을 훈련용 셋으로 만들어준다. 여기서 `batch` 의 의미는 한번에 모델에 입력되는 데이터 셋의 수를 제한한다는 의미이다. 이때 `batch_size`가 6이라면, 6개의 데이터를 한번 학습시킬때마다 넣는다는 뜻이 된다. 

`epoch`는 총 몇번 모델을 학습하는지를 나타내는 변수이다. 

`warmup_steps` 와 `weight_decay`는 overfitting 현상을 막기 위해 설정해주는 값이다. 아직은 정확히 이 값에 대한 의미는 잘 모른다. 

설정한 값을 통해 학습 인자를 만들어주고, 학습을 시작한다. 

그 뒤, 학습한 모델에 입력값을 넣어주면 내가 준 문장을 분류해주는 것을 볼 수 있다. 

+ query:

```python
input_text = ['I love Seoul', 'panda eats bamboo', 'snow fall']
input_data = tokenizer(input_text, padding=True, truncation=True, return_tensors="pt")

with torch.no_grad():
    outputs = model(**input_data)

logits = outputs.logits
predicted_labels = torch.argmax(logits, dim=1)

for text, label in zip(input_text, predicted_labels):
    print(f"input text: {text}")
    print(f"predicted label: {label}")
```

+ output:

```bash
input text: I love Seoul
predicted label: 1
input text: panda eats bamboo
predicted label: 1
input text: snow fall
predicted label: 1
(base) faintdeer@faint
```
## **Conclusion**

대충 어떤 모델이고 어떻게 사용하는지는 알았는데, 이걸 어떻게 활용하는지는 정말 미지수다. 공부할게 많을 것 같다. 

이걸 어떻게 보안에 적용시킬수 있을까 그리고 제대로된 fine-tunning 은 어떻게 하는 것일까. 

많은 자료를 읽어봐야겠다. 

## References

+ <https://wikidocs.net/book/2155>
+ <https://k-min-algorithm.tistory.com/37>