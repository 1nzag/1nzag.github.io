---
layout: simple
title: How to Use BERT
---

### **Intro**

요즘 인공지능은 대부분 Transformer 에 집중돼있는 것 같다. BERT 를 공부하다가 (기본지식을 얻을 수 있는 좋은 사이트)[https://huggingface.co/learn/nlp-course/chapter1/1?fw=pt]를 발견해서 해당 내용을 정리해보기로 했다. 

이번에 포스트 할 내용은 transformer의 `pipeline()` 함수이다. 

### **pipeline()**

해당 함수는 pre-training 된 모델을 불러와 다이렉트로 입력값을 넣어 결과를 확인 할 수 있는 함수이다. 

사용예시는 다음과 같다. 

```python
from transformers import pipeline

classifier = pipeline("sentiment-analysis")
print(classifier("I'm happy because I saw panda in zoo"))
```

`pipeline()` 함수를 호출하여 `sentiment-analysis`이라는 pre-trainning 된 모델을 불러온다. 이 때 `sentiment-analysis` 모델은 문장의 감정을 분류할 수 있는 모델이다. 

해당 분류기에 특정 문장을 입력값으로 넣어주면 그 문장을 분류한 결과를 보여준다. 

```bash
[{'label': 'POSITIVE', 'score': 0.9998605251312256}]
```

### **References**

+ <https://huggingface.co/learn/nlp-course/chapter1/3?fw=pt>