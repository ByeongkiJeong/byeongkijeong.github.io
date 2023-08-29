---
layout: post
title: 논문리뷰] GZIP Classifier
comments: true
---

# GZIP Classifier 리뷰
흥미로운 논문이 있어 간단히 리뷰해봅니다.  
ACL(Association for Computational Linguistics) 2023에서 발표된 논문인데요. 리뷰할만큼 복잡하지 않지만, 그 때문에 더 리뷰해봅니다.
  
GZIP Classifier... [(논문 바로가기)](https://aclanthology.org/2023.findings-acl.426/) 이름 그대로 GZIP을 이용한 텍스트 분류기입니다.  

논문을 세줄요약해봅시다.
> 딥러닝(DNN) 좋아..좋은데, 너무 무거워..(김대리,,액셀팡숀 너무 쓰지 마세요..닭잡는데는 닭잡는 칼이 필요합니다...)  
> 가볍고 쉽고 학습도 필요없는데 BERT랑 비등한 텍스트 분류 방법이 있는데 들어볼래?  
> 압축 알고리즘 + k-NN(Nearest neighbor) 조합이야!
  
맞아요..딥러닝 너무 무겁습니다. BERT만 해도 1~2GB는 족히 넘는데, 데이터를 Embedding하고 뒤에 Fine-tuning할 것까지 생각하면 시간/자원 측면에서 고민이 많이 되곤 합니다.   
거기에, 딥러닝의 성능을 보장하기 위한 충분한 데이터 양을 수집하려면...어휴 이건 실전에서 쓸게 못되는구나 싶죠.
특히 풀고자 하는 문제가 단순한 문장 분류(Classification) 문제라면, 좀 덜한 성능이더라도 TF / TF-IDF 기반의 Naive Bayes 등을 쓰는게 나을까 싶기도 합니다. 

GZIP Classifier는 이러한 무제를 해결하기 위한 좋은 대안인 것으로 보입니다.
알고리즘은 되게 간단한데요.

1. Label이 있는 타겟 문장(```x1```) 인코딩 후 압축
2. 군집화 대상인 문장 집합에 속한 개별 문장(```x2```)에 대해 3~4 반복
3. 대상 문장(x2)을 인코딩하고 압축
4. 타겟 문장과 문집화 대상 문장을 연결한 후(```x1x2```) 인코딩하고 압축
5. ```NCD(Normalized Compression Distance)```를 계산
6. 상위 k개의 문장을 x1과 같은 클래스에 속한 것으로 할당

이게 끝입니다. 아래는 논문에 기입되어 있는 실제 코드입니다.   
(특이하게 Pseudo code가 아닌 실제 python 코드를 썼네요.)   
[(Github 코드 바로 가기)](https://github.com/bazingagin/npc_gzip)  

```python
import gzip
import numpy as np

for (x1, _) in test_set:
    Cx1 = len(gzip.compress(x1.encode()))
    distance_from_x1 = []

    for (x2, _) in training_set:
        Cx2 = len(gzip.compress(x2.encode()) 
        x1x2 = " ".join([x1, x2])
        Cx1x2 = len(gzip.compress(x1x2.encode())
        ncd = (Cx1x2 - min(Cx1,Cx2)) / max(Cx1, Cx2)
        distance_from_x1.append(ncd) 

    sorted_idx = np.argsort(np.array(distance_from_x1))
    top_k_class = training_set[sorted_idx[:k], 1]
    predict_class = max(set(top_k_class), key=top_k_class.count)
```

핵심은 NCD(Normalized Compression Distance)라고 할 수 있겠는데요.
이게 왜 의미가 있는지 보면, 우선 식은 ```ncd = (Cx1x2 - min(Cx1,Cx2)) / max(Cx1, Cx2)``` 입니다.
 

분모(```max(Cx1, Cx2)```)는 Normalize를 위한 부분이니까 제외하고 보면, 분자(```Cx1x2 - min(Cx1,Cx2)```)의 의미만 남네요.
두 문장을 이어붙인 것을 압축한 길이에서 개별 문장을 압축한 길이 중 작은 값을 빼주는 간단한 식입니다.  


압축 알고리즘마다 차이는 있지만, 기본적으로 압축이란 그 데이터에 중복되는(겹치는) 부분이 얼마나 많은지를 찾는 과정이고, 중복되는 부분이 많다면 더 짧게 압축할 수 있습니다. (정보 엔트로피에 대한 개녑은 [이전 포스트](https://byeongkijeong.github.io/information-theory/)를 참고하세요.)

다시 NCD로 돌아가서, NCD가 가깝다(작다)는 것은 다시말해 ```Cx1x2 - min(Cx1,Cx2)```가 작다는 뜻이고, 이는 두 문장을 합친 것(```x1x2```)을 구성하는 글자의 순서들과 개별 문장 (```x1 or x2```)을 구성하는 글자의 순서들이 유사하다는 뜻으로 해석 될 수 있습니다.

만약 두 문장(```x1, x2```)이 동일하다면, 두 문장을 합친다 하더라도 정보 엔트로피의 증가가 미미할테니, 0에 가깝게 나올 것입니다.

반면 두 문장을 구성하는 글자의 순서들이 다르다면(다른 단어들로 구성되어 있다면), 엔트로피 증가가 클테고, NCD가 커지겠죠.

아래와 같이 간단한 실험을 해 볼수도 있습니다.  

```python
import gzip

cd = lambda x: len(gzip.compress(x.encode()))
def ncd(x1:str, x2:str):
  return (cd(x1+x2) - min(cd(x1), cd(x2)))/max(cd(x1), cd(x2))

x1 = 'hello world'
x2 = 'hello world too'
x3 = 'this is iphone'
ncd(x1, x2)
> 0.17142857142857143
ncd(x1, x3)
> 0.3548387096774194
ncd(x1, x1)
> 0.06451612903225806
ncd(x3, x3)
> 0.0967741935483871

x4 = '한글은 어떤가요?'
x5 = '나는 사람입니다'
x6 = '한글과 영어는 차이가 있나요?'
ncd(x4, x5)
> 0.5434782608695652
ncd(x4, x6)
> 0.5238095238095238
ncd(x4, x4)
> 0.043478260869565216
```

한글은 영문과 달리 글자의 특성상 개별 자소단위로 조합됩니다. 따라서 글자의 배열 순서가 중요한 알고리즘 특성상 Baseline이 좀 높을 수 있을 것 같아요. 

논문에서 제시한 Text classification 알고리즘만 놓고 보면 Top-k를 이용하니 한글/영문 간 Baseline 차이는 큰 문제는 없을 것으로 생각될 수 있습니다.

하지만, 현실적으로 대부분 한국어 데이터셋이 한글과 영문이 혼재되어 있는 상황인 것을 고려할 때, 영문으로 구성된 단어들의 배열이 분류 결과에 더 영향을 많이 줄 수 있을 것 같습니다.  

이는 글자를 나열하는 반식의 라틴어계열/일본어 또는 한 글자 단위로 의미가 부여되는 중국어 등에서는 발생하지 않을 것으로 생각되는데요.

한글도 byte 단위가 아니라, 글자 단위로 인코딩하는 방식으로 처리한다면 해결 할 수 있을 것 같기도 하네요..(위 실험대로 실행하면, python 3의 기본 인코딩인 utf-8을 이용합니다.) 

단점이라면 몇가지를 꼽을 수 있을 것 같은데요.
> 1. Gzip 알고리즘 의존성(압축률이 높지 않은 문장이라면,,어쩌죠..)
> 2. 단어의 의미(Symantics)는 고려하지 않음
> 3. 짧은 문장은 분류가 어려울 수 있음
> 4. 이미지에 대해서 분류 알고리즘을 적용하는 것은 어려움