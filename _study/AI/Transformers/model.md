---
layout: simple
title: (Transformer) Model
---

## **Intro**

[이전에 포스트했던 내용대로 Tokenizer를 통해 문장을 tensor로 변환한 뒤에는](/study/AI/Transformers/tokenizer) 해당 텐서를 모델에 주입 할 수 있게 된다. 

그 전에 모델을 초기화 하고, 설정하는 방법 부터 알아보자. 

## **Create model**

대표적인 예시로 BERT 모델을 불러와 설정해보자. 

```python
from transformers import BertConfig, BertModel

config = BertConfig()
print(config)
model = BertModel(config)
```

`BertConfig()`는 BERT 모델의 설정값을 자동으로 생성해 주는 함수이다. 이 설정값은 직접 지정해 줄 수 있지만, 초기화 될 때는 랜덤하게 설정된다. 

```python
BertConfig {
  "attention_probs_dropout_prob": 0.1,
  "classifier_dropout": null,
  "hidden_act": "gelu",
  "hidden_dropout_prob": 0.1,
  "hidden_size": 768,
  "initializer_range": 0.02,
  "intermediate_size": 3072,
  "layer_norm_eps": 1e-12,
  "max_position_embeddings": 512,
  "model_type": "bert",
  "num_attention_heads": 12,
  "num_hidden_layers": 12,
  "pad_token_id": 0,
  "position_embedding_type": "absolute",
  "transformers_version": "4.36.2",
  "type_vocab_size": 2,
  "use_cache": true,
  "vocab_size": 30522
}
```

## **Load model**

그 후에 모델을 훈련시켜주면, 모델은 제 기능을 할 것이다. 

하지만, NLP 모델은 개인이 직접적으로 훈련을 시켜주기엔 너무 많은 리소스가 필요하다. 따라서 미리 pre-training 된 모델을 불러오도록 하자. 

`BertModel` 의 `from_pretrained()` 매서드를 통해 pretrained 된 모델을 불러올 수 있다. 

```python
from transformers import BertModel

model = BertModel.from_pretrained("bert-base-cased")
```

불러온 모델 혹은 체크포인트는 [다음에서 확인할 수 있다.](https://huggingface.co/bert-base-cased)

## **Save model**

우리가 학습 시킨 모델 (fine-tuned 등) 을 저장 할 수 있는 방법 도 있다. 그건은 model 의 `save_pretrained()` 매서드를 사용하면 된다. 

```python
model.save_pretrained("directory/on/my/computer")
```
 
그러면 다음과 같은 두 개의 파일이 지정해 준 경로로 드랍된다. 

+ **config.json**
  + 모델 아키텍처를 구성하는 데 필요한 속성 및 메타데이터가 저장된 파일
+ pytorch_model.bin
  + 모델의 가중치가 저장된 파일



## References

+ <https://huggingface.co/learn/nlp-course/chapter2/3?fw=pt>

