---
title: Inverted index (역색인)
date: 2019-04-24T16:30:00+09:00
menu:
  sidebar:
   name: Inverted index (역색인)
   identifier: inverted-index
   parent: data-structure
   weight: 10
---

## 개요

Inverted index(역방향 인덱스)는 대용량 텍스트 검색을 위해서 고안된 방법이다. 요즘 많이 사용하는 해시태그 기능을 구현하고 싶다면 역방향 인덱스 방법을 사용할 수 있으며, 대부분의 검색 엔진이 이 방식을
사용한다.

## 기존의 검색 방식

1. WHERE =   
   ```SELECT * FROM DOCUMENT WHERE content = 'search text'```   
   검색어와 정확히 일치하는 문서만 검색된다. 인덱스가 걸려있을 경우 속도는 빠를 수 있지만, 문서 결과를 거의 얻지 못한다.

2. WHERE LIKE   
   ```SELECT * FROM DOCUMENT WHERE content like '%search text%'```   
   검색어가 포함되는 문서를 검색한다. 1번 보다 검색 결과가 많이 나올 순 있지만, 1번과 마찬가지로 검색어와 정확히 일치하는 문장이 포함되어야 한다.

3. Whitespace tokenizer AND   
   ```SELECT * FROM DOCUMENT WHERE content like '%search% and content like %text%```
   검색어의 모든 단어가 포함되는 문서를 검색한다. 2번 방식보다는 향상된 결과를 보여준다.

4. Whitespace tokenizer OR   
   ```SELECT * FROM DOCUMENT WHERE content like '%search% and content like %text%```   
   검색어의 단어 중 하나의 단어라도 포함되어 있는 문서를 검색한다. 1~3번의 방식 중 가장 많은 결과가 검색되지만, 하나의 단어라도 포함되어 있는 문서는 전부 검색되기 때문에 검색 결과의 정확성이 떨어질 수
   있다.

## 기존 방식의 문제

1. 1번과 같은 검색 방식의 경우 검색 결과를 거의 얻지 못함
2. 2~3번과 같은 경우 index를 타지 않기 때문어 검색 속도가 느림
   (like 검색시 %를 앞에 넣게되는 경우 인덱스를 타지 않는다.)

## Inverted index

> 위에서와 같이 기존 검색 방식은 INDEX 기능을 이용할 수 없다는 단점이 있다. 이를 극복하기 위해 단어(Term)로 인덱싱을 하는 Inverted index 방식이 고안되었다.

Inverted index는 아래의 표를 통해 쉽게 이해할 수 있다.

| term | document_id |
| :--- | :---------- |
| 안녕 | 1,2,3 |
| 붕어빵 | 1,2 |
| 고기 | 1 |

위의 표와 같이 역색인은 키워드에 문서의 Pk와 같은 값을 매핑하여 저장하는 기술이다. 역색인 작업을 했을 때의 장점은 검색 속도가 굉장히 빨라진다는 것이다. (term에 인덱스를 사용)

"안녕 붕어빵"으로 검색 하였을 시 OR 처리를 하였다고 가정한다면, 1, 2, 3번 문서가 검색된다. 여기서 좀 더 발전시켜 문서별 단어 등장 빈도수를 계산하여, 빈도가 높은 순으로 정렬 하는 등 세부적인 처리를
통해 검색 결과의 품질을 높일 수 있다.

---

[참고 사이트]   
[https://blog.lael.be/post/3056](https://blog.lael.be/post/3056)
[https://needjarvis.tistory.com/345](https://needjarvis.tistory.com/345)
