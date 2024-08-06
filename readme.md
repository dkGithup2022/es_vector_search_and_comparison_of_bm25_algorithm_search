
# Vector seaerch와 고전적인 문서유사도 알고리즘 방식과 비교

## 목차

1. [소개](#소개)
   1. [예상 독자](#예상-독자-)
      1. [예상 독자](#예상-독자--1)
      2. [예상 독자가 아닌 사람](#예상-독자가-아닌-사람)
   2. [결론 : 편합니다. 이제 analyzer를 한땀한땀 만들지 않아도 될 거 같아요](#결론--편합니다-이제-analzyer-를-한땀한땀-만들지-않아도-될거-같아요)


2. [벡터 서치의 개념](#벡터-서치의-개념)
   1. [Vector Search](#vector-search)
   2. [Sentence transformer model](#sentence-transformer-model-)
      1. [Elasticsearch 벡터검색에서 사용할 수 있는 Transformer model](#elasticsearch-벡터검색에서-사용할-수-있는-transformer-model)
      2. [SentenceTransformer 의 max_seq](#sentencetransformer-의-max_seq)
   2. [8버전 이전과 이후의 차이](#8버전-이전과-이후의-차이)
   3. [Vector Search의 현황: 벡터 검색이 얼마나 인기 있는지 잠깐 보기](#vector-search의-현황-벡터검색이-얼마나-인기-있는지-잠깐보기)


3. [ 고전적인 문서유사도와 비교하기 : vector search 장단점  ](#고전적인-문서유사도와-비교하기--vector-search-장단점-)
   1. [벡터 서치의 장점](#벡터-서치의-장점)
      1. [생각보다 합리적인 검색 성능을 제작비용 없이 제작 ](#생각보다-합리적인-검색-성능을-제작비용-없이-제작-)
      2. [오타를 캐어하면서 n-gram의 토큰이 튀는 문제를 방지](#오타를-캐어하면서-n-gram의-토큰이-튀는-문제를-방지할-수-있다)
      3. [Sentencetransformer-모델의-강화](#sentencetransformer-모델의-강화)
      4. [ML 관련 기능의 추가 업데이트](#ml-관련-기능의-추가-업데이트)
   2. [벡터 서치의 단점](#벡터-서치의-단점)
      1. [SentenceTransformer 모델과 벡터 거리 비교 작업이 각각 병목 포인트가 된다](#sentence-transformer-model-과-벡터-거리-비교-작업이-각각-병목-포인트가-된다)
      2. [SentenceTransformer 모델의 입력 크기 제한](#sentencetransformer-모델의-입력-크기-제한)
      3. [Analyzing level의 튜닝이 불가](#anylzing-level의-튜닝이-불가)
      4. [매칭이 안 되어도 응답 리스트에 포함됨](#매칭이-안-되어도-응답-리스트에-포함됨)


4. [Vector Search 예시](#vector-search-예시)
   1. [테스트 결과](#테스트-결과)
   2. [테스트 환경 및 파이프라인 코드](#테스트-환경-및-pipeline-코드)


---

# 소개

Elasticsearch 에서의 벡터 검색에 대해 소개합니다. 벡터서치의 개념, 2022년 부터 인기 정도, 그리고 예시와 장단점에 대해 작성합니다.

주로 고전적인 검색 매커니즘과 벡터의 유사도를 기반한 검색의 비교가 주된 내용입니다.
( 고전적인 검색 매커니즘이란 토큰 기반의 일치 불일치 판단 + TF-IDF 기반 스코어링을 기반으로 한 검색을 말합니다.)

글의 스타일이 예제 데이터와 코드는 맨 아래 있고 개념적인 내용과 후기부터 작성되어 있습니다.
검색에 대한 예시만 보실 독자님들은 테스트 데이터는 맨 아래 있습니다. 

[벡터 검색 테스트 결과 보기](#vector-search-예시)


## 예상 독자 

### 예상 독자 

- **BM25(TF-IDF) 문서 유사도 알고리즘 기반의 검색을 사용해 봤지만 아직 벡터 검색을 안해본 사람**
- 벡터 검색에 대해 이제 막 알아보려는 사람
- 벡터 검색의 사용 후기를 보고 싶은 사람
- 기존 검색 메커니즘과의 차이를 알아보고 싶은 사람
- dankim 의 elasticsearch 시리즈를 처음부터 읽어보고 있는 착한 사람

### 예상 독자가 아닌 사람

- 벡터 검색을 위한 구현을 설치하고 배포하는 튜토리얼을 찾아온 사람 
  - 해당 문서는 개념적인 내용, 사용 후기에 가깝습니다. 튜토리얼을 찾으신다면 아래 문서를 참고해주세요.
  - 링크 : 
- 벡터 검색을 이미 적용해보고 잘 알아서 model 단위의 튜닝에 도움이 되는 문서를 찾아오신분 
  - 해당 문서는 그 내용에 대해 다루지 않습니다.

## 결론 : 편합니다. 이제 analzyer 를 한땀한땀 만들지 않아도 될거 같아요.




결론부터 말씀드리면 엄청 편합니다.

고전적인 검색에서 analyzer를 직접 작성하고 테스트하는 이유는 **검색어의 토큰과 인덱싱된 문서의 토큰의 매칭을 합리적으로 만들려는 노력**입니다.

별다른 노력이 없어도 이 analyzer의 튜닝을 통해 할 수 있는 일의 거의 대부분을 "Kibana -> E5 model deploy" 클릭 한 번으로 대체할 수 있습니다.

#####  벡터 서치 검색 결과 예시
- [벡터검색 테스트 결과 확인하기](#vector-search-예시)

고전적인 검색에서는 오타 보정, 동의어 사전, 편집거리 등을 고려해야 매칭이 되는 검색어도  벡터검색에서는 별도의 설정없이 매칭과 스코어링이 합리적으로 이루어지는 것을 확인할 수 있습니다. 


이 정도를 해결하는 analzyer 를 만드는데에만 해도 기존 고전적인 analyzer 제작에서는 캐어해야 할 부분이 분명히 있고, 제작 이후에 특정 토큰에 대한 스코어가 튀지 않을지 캐어를 해야 해야해요.

근런데 벡터 검색이 제대로된 가이드만 있으면 거의 10분만에 기존 analyzer를 만드는 목적을 그대로 수행 가능한 연산을 만들고 운영에서의 캐어도 덜 해도 된다라는 메리트가 굉장히 크다고 생각합니다.



---

# 벡터 서치의 개념

## vector search

문서를 벡터화 시킨 후, 벡터 간의 벡터 거리를 계산하여 유사도를 판단하는 방법입니다.

일반적으로 갓 다운받은 elasticsearch 에서는 사용할 수 없고, 두가지 방법 중 한가지가 선행되어야 합니다.

1. 외부 api 를 통해서 미리 벡터화 시킨 문서를 인덱싱 한다.
2. Elasticsearch 의 ingest pipeline 에 모델을 적제시키기 위해서 elasticsearch 내부에 SentenceTransformer 모델 배포하기

첫 번째 방법은 벡터 간 유사도를 검사할 수 있는 모든 버전에서 사용할 수 있지만, 두 번째 방법은 Elasticsearch 8 버전에서만 가능합니다.

![8버전이전이후검색비교.drawio.png](images%2F8%EB%B2%84%EC%A0%84%EC%9D%B4%EC%A0%84%EC%9D%B4%ED%9B%84%EA%B2%80%EC%83%89%EB%B9%84%EA%B5%90.drawio.png)


세팅하는 방법은 elastic-lab 혹은 elasticsearch 공식 문서의 링크를 남깁니다.

[Elastic docs: deploy trained model](#https://www.elastic.co/guide/en/machine-learning/current/ml-nlp-deploy-models.html)

[Elastic lab : multilingual search on vector search ](#https://www.elastic.co/search-labs/blog/multilingual-vector-search-e5-embedding-model)

## Sentence transformer model 

Sentece transformer 모델은 문자열을 벡터로 만들어주는 모델입니다. 위 이미지에서 ingest pipeline, model hosting server 에서 수행되는 ml 모델입니다.  

검색에서는 indexing 되어 검색의 대상이 되는 텍스트와 검색어 둘 다에 적용 됩니다 .

### Elasticsearch 벡터검색에서 사용할 수 있는 Transformer model

hugging face 에 배포된 모델 중, "Sentence transformer" 태그가 붙은 모델은 전부 배포가 가능합니다.

하지만, 주로 kibana (v8 이상) 의 model management 에서 받을 수 있는 elser, e5 모델이 elasticsearch 에서는 가장 커뮤니티 의견이 많으면서 많이 쓰이고 있습니다.

각 모델의 특징은 아래와 같습니다.

| 특징               | Elser    (.elser_model_2_linux-x86_64)  | E5    (.multilingual-e5-small_linux-x86_64) |
|------------------|-----------------------------------------|---------------------------------------------|
| **지원 언어의 범위**    | 영어                                      | 영어, 한국어, 중국어, 일본어 등 다국어                     |
| **Max Sequence** | 512 토큰                                  | 512 토큰                                      |
| **비교**           | 추론 성능이나 발전속도가 e5 보다 빠르다( 발표 , 관련 아티클 수) |                                         |

### SentenceTransformer 의 max_seq


Sentence transformer 모델은 입력받을 수 있는 최대 토큰 갯수가 정해져 있습니다.

Sentence transformer 모델에서는 입력받은 각 토큰을 선형적으로 처리하지 않습니다. 입력받은 토큰 간의 관계를 기반으로 각 토큰의 가중치를 계산하게 되는데, 이 과정에서 모델별로 정해놓은 N*N 배열에 토큰을 적제하는 과정이 필요합니다.

많은 경우(거의 모든 경우), 해당 N*N 배열은 가변이 아닙니다.  보통 512 ~ 1024 의 값을 가지고 있으며 BigBird, LongFormer 등의 제한된 모델에서만 4000 토큰 이상의 길이를 가진 텍스트를 처리할 수 있습니다.


따라서, 길이가 긴 글을 받아서 벡터화 하려면 전처리가 필요합니다.

전처리의 과정은 [Sentence transformer max_seq 대처하기 문서]()에서 소개합니다.


## 8버전 이전과 이후의 차이

인터넷에서 벡터 검색은 크게 두 가지를 말합니다.

1. 벡터를 입력받아 벡터 간의 거리를 비교해 가까운 순서로 나열해주는 것.
2. 검색어를 벡터로 변환한 후 기존에 가지고 있던 벡터 간의 거리를 비교해 가까운 순서로 나열해주는 것.

결국, 벡터 간의 거리로서 문서의 유사도를 비교하겠다는 것인데, 두 개를 나눠 설명한 이유는 시대에 따라서 서버 구조가 조금 다르기 때문입니다. 2022년도에 Elasticsearch 8버전이 출시되면서 ML 모델을 ingest pipeline에 적재하는 것이 가능해졌고, 그로 인해 인덱싱과 쿼리 과정에서 별도의 파이썬 서버와 통신할 필요가 없어졌습니다.

그래서 아래 이미지 같은 차이가 발생하는데, 24년도 이후에 입문하신 분들은 벡터 검색 관련 아티클을 읽다가 ML 모델을 호스팅하는 파이썬 서버에 관한 설명이 나오면 당황하지 말고 "아, 옛날엔 이랬구나" 하시면 됩니다.

물론, 지금도 예전처럼 운영할 수 있으니 각자의 환경에 맞게 선택하시면 됩니다.

![8버전이전이후검색비교.drawio.png](images%2F8%EB%B2%84%EC%A0%84%EC%9D%B4%EC%A0%84%EC%9D%B4%ED%9B%84%EA%B2%80%EC%83%89%EB%B9%84%EA%B5%90.drawio.png)




## Vector search의 현황: 벡터검색이 얼마나 인기 있는지 잠깐보기



![es_세미나.png](images%2Fes_%EC%84%B8%EB%AF%B8%EB%82%98.png)

위 이미지는 8월 초에 Elasticsearch Community의 공식 유튜브 채널에 최신순으로 나열된 동영상입니다.

화면에 보이는 12개의 동영상 중 인터뷰를 제외하면 9개 중 7개가 벡터와 LLM 관련 내용임을 확인할 수 있습니다.

대략적으로 보면 2022년부터 2024년까지 Elastic의 발표 내용은 약 10%가 Lucene의 성능 향상이나 신규 기능에 관련된 내용이고, 나머지 90%는 전부 벡터서치에 관한 내용이거나 벡터서치를 어떻게 LLM이나 보안과 혼합해 사용할지에 관한 내용입니다. ( Lucene 9.5가 2023년에 버전업 되었습니다.)

벡터검색을 포함한 ml 모델을 es에 올려서 서빙하는 방법은 꽤 꾸준히 연구되고 업데이트되며 관심도 높은 분야로 보입니다. 알아두면 좋을 것입니다.




---

# 고전적인 문서유사도와 비교하기 : vector search 장단점  


## 벡터 서치의 장점

### 생각보다 합리적인 검색 성능을 제작비용 없이 제작 


#####  벡터 서치 검색 결과 예시
- [벡터검색 테스트 결과 확인하기](#vector-search-예시)

"데이터베이스"에 대한 검색 결과를 나열하면 "데이터베이스" -> "database" -> "대이터베이스" -> "데이터" -> "대따바스" -> "elasticsearch" 순으로 리턴됨을 확인할 수 있습니다.

고전적인 analzyer 설정에서 저런 단어 매칭이 이루어지게 하려면 아래의 작업을 수행해야 합니다. 

- **검색어 "데이터베이스" 매칭에 필요한 각 토큰별 필요 작업**

| 검색어        | 필요한 analyzing 옵션                    | 설명                                                |
|---------------|-------------------------------------|---------------------------------------------------|
| database      | 동의어 사전                              | 영어라서 일치 토큰 없음                                     |
| 데이터        | n-gram (shingle)                    | n-gram을 제외하면 "데이터베이스"에서 "데이터"라는 토큰을 추출하기 힘듦       |
| 대이터베이스  | n-gram (shingle) 혹은 fuzziness query | 오타인 경우 fuzziness와 n-gram을 통해 조정이 가능합니다.           |
| 대따바스      | 이건 답이 없음        | 발음은 비슷하지만, 그냥 이상한 검색어, 굳이 따지자면 'ㄷ', 'ㅂ', 'ㅅ' 이 겹침 |



사실 고전적인 문서 유사도 관점에서의 analyzer의 역할을 동의어 사전을 제외하고는 다 해준 것과 다름이 없습니다. 위의 표에서 명시된 n-gram(shingle), fuzziness, 동의어 사전 없이도 합리적이고 유연한 검색을 제공한다고 볼 수 있습니다. 운영과 제작 비용을 아끼는 측면에서 강점이 있다고 볼 수 있습니다.

### 오타를 캐어하면서 n-gram의 토큰이 튀는 문제를 방지할 수 있다

"아 그러면 검색이 유연하게 되려면 그냥 n-gram이나 shingle 적용하면 되는 거 아닌가요?"

=> n-gram도 문제가 있습니다.

특정 토큰, 특히 짧은 토큰을 잘 처리 못합니다. 예를 들어, size 2인 n-gram이 적용되고 영어 문서가 많은 환경에서, 엘라스틱 서치의 약자인 "ES"를 검색한다고 해보세요. 그러면 검색 결과에 ES가 아닌 것도 올라옵니다. 소팅 스코어 옵션에 따라서는 그냥 ES, Elasticsearch 내용이 안 나올 수도 있습니다.

왜냐하면 "ES"라는 토큰은 n-gram을 적용하면 거의 모든 문서에 속할 것입니다. 예를 들어서 "classes", "esc", "messy" 이런 단어들이 모두 "es" 토큰을 해당 문서의 토큰 집합에 포함시킵니다.

**"es"라는 토큰이 포함된 문서가 n-gram 적용 전과 비교하면 몇백배는 늘 수 있고 "ES"라는 토큰의 idf 스코어를 왕창 내려버립니다.** 고전적인 문서 유사도 알고리즘(TF-IDF)에서는 각각의 토큰이 전체 문서 집합에서 얼마나 고유한지를 스코어링에 반영하는데, n-gram이 적용된 상태에서의 "es"라는 토큰은 전혀 고유하지 않습니다.

그래서 ES로 검색하면 Elasticsearch와 관련이 없는 글도 리스트에 나타납니다. 왜냐하면 해당 토큰의 스코어가 높지 않고 거의 모든 문서가 해당 토큰을 가지고 있기 때문입니다.

그러나 벡터 검색을 적용하면 아래 챕터의 예시처럼 오타가 캐어가 되면서, 이런 문제를 캐어할 필요가 없습니다. 대신 analyzer 레벨의 튜닝도 불가능합니다.


### Sentencetransformer-모델의-강화

문서를 벡터로 변환하는 모델의 성능은 점점 발전하고 있습니다. 예전에는 영어 검색에서 최대 길이가 512 토큰에 대한 검색이 벡터 검색의 최대 크기였으나, 지금은 4000 토큰이 넘는 모델도 나왔습니다. 테스트 데이터 범위가 넓어지면서 추론의 성능 또한 발전합니다.

그래서 이를 활용하는 개발자의 경우, 커뮤니티 의견을 보고 "다국어 모델 e5 모델이 새로 나온 게 잘 뽑혔더라" 같은 소문을 들으면 그 모델로 갈아 끼우면 됩니다. 

사실, 가만히 앉아서 검색 퀄리티를 높일 수 있습니다.

### ML 관련 기능의 추가 업데이트

[Elasticsearch-lab: Rerank API and model](https://www.elastic.co/search-labs/blog/elasticsearch-cohere-rerank)

Elasticsearch 위에서 모델을 적재하고 다룰 줄 알면 많은 이점이 있습니다. 앞서 설명한 [벡터 서치의 인기 알아보기](#벡터검색이-얼마나-인기-있는지-확인하기)에서 거의 모든 발표가 ML 관련 주제인 것을 확인할 수 있었습니다.

매년 모델을 활용한 새로운 기능이 나오고 있는데, 그중 하나가 **Rerank API, Coherence**입니다.

자세한 내용은 링크를 참고하시고, 간단하게 설명하면 검색 결과를 정렬하는 과정에서 간단한 LLM 언어 모델을 사용하는 것입니다. 예를 들어, 기존에 쓰던 검색 쿼리에 추가로 "What is the capital of the USA?"라는 쿼리 옵션을 추가로 줄 수 있습니다. 그러면 쿼리 결과로 hit된 문서를 "What is the capital of the USA?"의 문맥에 더 맞는 순서로 정렬할 수 있는 일종의 RAG를 기능을 제공합니다.


여기서 중요한 것은 이 **Rerank API를 꼭 사용하라는 것이 아니라, 벡터화, ML 등의 키워드와 연관된 기능이 거의 매 분기마다 나오기 때문에 미리 친해지면 좋다는 것입니다.**

## 벡터 서치의 단점

### Sentence transformer model 과 벡터 거리 비교 작업이 각각 병목 포인트가 된다

고전적인 문서 유사도 기반의 검색에 비해 두 가지 작업이 추가됩니다.

1. 벡터 간 거리의 비교
2. ML 모델이 동작해야 함

이 두 작업 모두 램이 많이 필요한 연산입니다.

**"CPU, RAM 같은 컴퓨팅 파워를 많이 쓴다?" == "병목 포인트가 된다"** 는 의미이기 때문에, 만약 기존의 고전적인 문서 유사도 기반 검색에서 벡터 서치로 전환한다면 성능에 대한 테스트는 선택이 아닌 필수입니다.

그리고 k-NN nearest 방식에 대한 튜닝 포인트도  다릅니다. 이 부분에 대한 비교는 별도 문서에 남기도록 하겠습니다.

참고로, 벡터 거리 비교는 Java 힙보다는 OS의 램 자원이 필요합니다. 램을 추가한 다음에 추가한 램의 크기만큼 전부 JVM 옵션으로 Java 힙을 추가할 필요는 없습니다.

### SentenceTransformer 모델의 입력 크기 제한

사실 벡터 서치를 초기에 적용하려고 하면 마주하게 되는 가장 큰 진입장벽 중 하나는 Transformer 모델(벡터화 모델)의 최대 문서 크기 제한입니다.

예를 들어, 총 토큰수가 5000자인 문서는 max_seq가 512인 모델에서는 처음 부분만 잘려서 벡터화됩니다. 그래서 전체 컨텍스트를 벡터화하려면 전처리가 필요합니다.

이런 문제를 해결하거나 우회할 수 있는 몇 가지 방법이 있는데, 아래의 문서 링크에서 다루도록 하겠습니다.

[링크: devwiki vector search implementation with LLM](https://github.com/dkGithup2022?tab=repositories)

### anylzing level의 튜닝이 불가


#####  벡터 서치 검색 결과 예시
- [벡터검색 테스트 결과 확인하기](#vector-search-예시)

ML 모델을 사용하는 것들의 전형적인 문제와 공통점은 Input과 Output의 연관을 알아낼 수 없다는 점입니다. 

문자열이 벡터모델로 변환되면 그냥 주는 대로 써야 합니다.

예를 들어, 위의 테스트 검색 결과에서 "어! 나는 'RDMS'가 'Elasticsearch'보다는 검색 상단에 위치해야 할 것 같은데?"라는 생각이 들어도 벡터의 스코어상으로는 할 수 있는 일이 없습니다.

비슷하게, "나는 '대따바스' 같은 저질 단어는 아예 검색되지 않게 해버려야지"라는 생각이 들면 기존의 analyzer는 코드를 삽입하는 수준의 튜닝이 가능하기 때문에 조작 자체는 가능하지만, 벡터 서치에서는 무리입니다.

### 매칭이 안 되어도 응답 리스트에 포함됨

벡터 검색에서는 기존의 텍스트로서의 특징을 숫자로 변환시킨 다음 각 점의 거리로 유사도를 판단합니다. 그 벡터 간 거리가 엄청 멀어도 더 가까운 게 없으면 그거라도 리턴 결과에 포함됩니다.

고전적인 토큰 비교 기반의 방식과 비교하면, 옛날 방식은 토큰을 비교해서 겹치는 토큰이 없다면 결과를 생성하지 않습니다(빈 리스트를 반환합니다). 그러나 벡터 검색에서는 이런 비교를 거치는 과정 없이 "거리 순 n개 정렬"이라는 연산을 하기 때문에 겹치지 않아도 결과로 나옵니다.

예를 들어, 위의 테스트 데이터에서 "데이터베이스"로 검색했는데 "Elasticsearch"로 응답이 나온 것을 보면 유사하지 않은 문서도 결과 리스트에 포함될 수 있다는 것을 보여줍니다.



---
# Vector search 예시

이 챕터에서는 Vector search 의 예시를 제공합니다.

조회의 테스트에서 기존 analayzer 튜닝 기반의 검색과 비교를 위해서, 기존의 토큰 기반의 검색으로는 매칭이 힘든 단어를 몇개 추가했습니다.

가령, "데이터베이스"로 검색을 했을때, "database", "대이터베이스" 가 검색 결과로 리턴되는지, score 순위는 몇등인지 이런것을 확인해 보시고 참고해보시면 좋을 것 같습니다.

## 테스트 결과

아래에 [테스트 환경 및 pipeline 코드](#테스트-환경-및-pipeline-코드-) 에 자세한 테스트 환경과 세팅 코드를 올렸습니다.

"데이터베이스"에 대한 검색결과를 나열하면
"데이터베이스" -> "database" -> "대이터베이스" -> "데이터" -> "대따바스" -> "elasticsearch" ....

순으로 리턴됨을 확인할 수 있습니다. 각 리턴된 결과를 기존의 token 기반의 문서 유사도 알고리즘에서 매핑이 되려면 아래의 작업을 수행해야 합니다.

analzyer 에 대한 작업 없이도, 적당히 합리적인 수준의 검색 결과를 제공한다고 볼 수 있다 .

- **검색어 "데이터베이스" 매칭에 필요한 각 토큰별 필요 작업**

| 검색어        | 필요한 analyzing 옵션           | 설명                                                |
|---------------|----------------------------|---------------------------------------------------|
| database      | 동의어 사전                     | 영어, 일치 토큰 없음                                      |
| 데이터        | n-gram ( shingle )         | n-gram을 제외하면 "데이터베이스"에서 "데이터"라는 토큰을 추출하기 힘듦       |
| 대이터베이스  | n-gram ( shingle ) 혹은 fuzziness query | 오타인 경우 fuzinness 와 n gram 을 통해 조정이 가능합니다.         |
| 대따바스      | 이건 답이 없음                   | 발음은 비슷하지만, 그냥 이상한 검색어, 굳이 따지자면 'ㄷ', 'ㅂ', 'ㅅ' 이 겹침 |





## 테스트 환경 및 pipeline 코드

- **테스트 데이터**

```json
POST /my_vector_index-001/_bulk
{ "index": { "_index": "my_vector_index-001", "_id": "1" } }
{ "my_text": "database" }
{ "index": { "_index": "my_vector_index-001", "_id": "2" } }
{ "my_text": "데이터베이스" }
{ "index": { "_index": "my_vector_index-001", "_id": "3" } }
{ "my_text": "데이터" }
{ "index": { "_index": "my_vector_index-001", "_id": "11" } }
{ "my_text": "대이터베이스" }
{ "index": { "_index": "my_vector_index-001", "_id": "12" } }
{ "my_text": "대따바스" }
{ "index": { "_index": "my_vector_index-001", "_id": "4" } }
{ "my_text": "rdms" }
{ "index": { "_index": "my_vector_index-001", "_id": "5" } }
{ "my_text": "k8s" }
{ "index": { "_index": "my_vector_index-001", "_id": "6" } }
{ "my_text": "쿠버네티스" }
{ "index": { "_index": "my_vector_index-001", "_id": "7" } }
{ "my_text": "쿠버네틱스" }
{ "index": { "_index": "my_vector_index-001", "_id": "8" } }
{ "my_text": "엘라스틱서치" }
{ "index": { "_index": "my_vector_index-001", "_id": "9" } }
{ "my_text": "엘라스틱캐시" }
{ "index": { "_index": "my_vector_index-001", "_id": "10" } }
{ "my_text": "elasticsearch" }
```


- **Ingest pipeline**
```json
#### 1. pipeline 
PUT _ingest/pipeline/vector_embedding_demo
{
  "processors": [
    {
      "inference": {
        "field_map": {
          "my_text": "text_field"
        },
        "model_id": ".multilingual-e5-small_linux-x86_64",
        "target_field": "ml.inference.my_vector",
        "on_failure": [
          {
            "append": {
              "field": "_source._ingest.inference_errors",
              "value": [
                {
                  "message": "Processor 'inference' in pipeline 'ml-inference-title-vector' failed with message '{{ _ingest.on_failure_message }}'",
                  "pipeline": "ml-inference-title-vector",
                  "timestamp": "{{{ _ingest.timestamp }}}"
                }
              ]
            }
          }
        ]
      }
    },
    {
      "set": {
        "field": "my_vector",
        "if": "ctx?.ml?.inference != null && ctx.ml.inference['my_vector'] != null",
        "copy_from": "ml.inference.my_vector.predicted_value",
        "description": "Copy the predicted_value to 'my_vector'"
      }
    },
    {
      "remove": {
        "field": "ml.inference.my_vector",
        "ignore_missing": true
      }
    }
  ]
}

```

- **knn 검색 수행**
```json
GET my_vector_index-001/_search
{
  "knn": [
    {
      "field": "my_vector",
      "k": 10,
      "num_candidates": 10,
      "query_vector_builder": {
        "text_embedding": {
          "model_id": ".multilingual-e5-small_linux-x86_64",
          "model_text": "데이터베이스"
        }
      }
    }
  ]
}
```

- **결과**
```json
{
  "took": 20,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 10,
      "relation": "eq"
    },
    "max_score": 0.9521209,
    "hits": [
      {
        "_index": "my_vector_index-001",
        "_id": "2",
        "_score": 0.9521209,
        "_source": {
          "my_text": "데이터베이스",
          "ml": {
            "inference": {}
          }
        }
      },
      {
        "_index": "my_vector_index-001",
        "_id": "1",
        "_score": 0.935969,
        "_source": {
          "my_text": "database",
          "ml": {
            "inference": {}
          }
        }
      },
      {
        "_index": "my_vector_index-001",
        "_id": "11",
        "_score": 0.9218676,
        "_source": {
          "my_text": "대이터베이스",
          "ml": {
            "inference": {}
          }
        }
      },
      {
        "_index": "my_vector_index-001",
        "_id": "3",
        "_score": 0.92032325,
        "_source": {
          "my_text": "데이터",
          "ml": {
            "inference": {}
          }
        }
      },
      {
        "_index": "my_vector_index-001",
        "_id": "12",
        "_score": 0.9159211,
        "_source": {
          "my_text": "대따바스",
          "ml": {
            "inference": {}
          }
        }
      },
      {
        "_index": "my_vector_index-001",
        "_id": "10",
        "_score": 0.9070498,
        "_source": {
          "my_text": "elasticsearch",
          "ml": {
            "inference": {}
          }
        }
      },
      {
        "_index": "my_vector_index-001",
        "_id": "8",
        "_score": 0.90547764,
        "_source": {
          "my_text": "엘라스틱서치",
          "ml": {
            "inference": {}
          }
        }
      },
      {
        "_index": "my_vector_index-001",
        "_id": "5",
        "_score": 0.9047841,
        "_source": {
          "my_text": "k8s",
          "ml": {
            "inference": {}
          }
        }
      },
      {
        "_index": "my_vector_index-001",
        "_id": "9",
        "_score": 0.9033524,
        "_source": {
          "my_text": "엘라스틱캐시",
          "ml": {
            "inference": {}
          }
        }
      },
      {
        "_index": "my_vector_index-001",
        "_id": "4",
        "_score": 0.9005958,
        "_source": {
          "my_text": "rdms",
          "ml": {
            "inference": {}
          }
        }
      }
    ]
  }
}
```


