---
layout: simple
title: Tokenizer
---

## **Intro**

문장을 모델에 주입할 때 문장 그대로 모델에 넣으면 모델은 그 문장을 이해 할 수 없다. 

따라서 해당 문장을 컴퓨터 또는 모델이 이해할 수 있는 언어로 번역을 해줄 필요가 있는 데, 이 때 사용되는 것이 `tokenizer` 이다. 

[저번 포스트의 pipeline() 함수](/study/AI/Transformers/pipeline) 역시 전처리 과정에 tokenizer 를 사용한다. 


## **Usage**
tokenizer 는 다음과 같이 사용할 수 있다. 

```python
from transformers import AutoTokenizer

checkpoint = "distilbert-base-uncased-finetuned-sst-2-english"
tokenizer = AutoTokenizer.from_pretrained(checkpoint)
```

`distilbert-base-uncased-finetuned-sst-2-english` 모델을 가져오고 (원문에서는 checkpoint 라고 하는데 왜 정확히 이 표현을 쓰는지는 아직 잘 모르겠다.)

해당 모델의 tokenizer 를 `AutoTokenizer.from_pretrained()`를 통해 가져온다.

이제 문장들을 가져온 tokenizer를 통해 전처리를 한다. 

```python
raw_inputs = [
    "Graypanda learn how to use tokenizer",
    "it's so hard work"
]

inputs = tokenizer(raw_inputs, padding=True, truncation=True, return_tensors="pt")
print(inputs)
```

이때 모델이 받아들이는 언어의 형태는 tensor 이다. tensor 란 n차원의 벡터를 표현한 자료형을 의미한다. tensor는 여러 모듈이 사용하는 형태로 리턴받을 수 있는데, `return_tensors=pt` 옵션을 통해 PyTorch tensor 형태로 받는다. 

즉 우리는 위와 같은 코드를 통해 문장을 모델이 받아들일수 있는 tensor 의 형태로 변환한 것이다. 

```bash
{'input_ids': tensor([[  101,  3897,  9739,  2850,  4553,  2129,  2000,  2224, 19204, 17629,
           102],
        [  101,  2009,  1005,  1055,  2061,  2524,  2147,   102,     0,     0,
             0]]), 'attention_mask': tensor([[1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1],
        [1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 0]])}
```

tokenizer 가 반환하는 값은 문장을 integer로 변환한 값과, attention_mask 값이다. 

## **References**

+ <https://huggingface.co/learn/nlp-course/chapter2/2?fw=pt>
