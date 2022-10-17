---
title: "그림으로 이해하는 이진 탐색 (Binary search)으로 삽입 위치 찾기"
categories: 
  - algorithm
tags:
  - binary search
  - leetcode
  - python
toc: true
toc_sticky: true
toc_label: "Table of contents"
---

LeetCode 사이트의 [Search Insert Position](https://leetcode.com/problems/search-insert-position/) 문제는 기본적인 이진 탐색 (Binary search) 코드를 구현하도록 유도한다.
매우 단순한 알고리즘이지만 수많은 문제에 활용되기 때문에, 보통 코드 템플릿을 외워두고 바로 코드부터 짜는 경우가 많다.
그런데, 문득 위 문제처럼 target을 찾지 못하는 경우 삽입할 위치를 반환해야 한다면, 조금 헷갈릴 때가 있다.
이런 문제를 한 번 만나봤다면 방법을 외우면 되지만, 이해를 위해 원리를 쉽고 간단하게 그림으로 설명해보겠다.

## 1. Binary search의 기본 개념

Binary search는 정렬된 리스트에 있는 숫자들 중에서 특정 target을 찾는 문제이다.
리스트가 정렬되어 있다는 전제가 깔려있기 때문에, 영역을 절반씩 쪼개면서 탐색하면 O(logN)으로 풀 수 있다.

우선 설명의 편의를 위해 다음 용어들을 정의하고 넘어가자.

- ```L```: 정렬된 리스트
- ```t```: 찾고자 하는 값
- ```s```: 탐색하고자 하는 영역의 시작 인덱스
- ```e```: 탐색하고자 하는 영역의 끝 인덱스
- ```m```: 탐색하고자 하는 영역의 중간 인덱스 ```m = (s + e) / 2```

```t```가 ```L```에 존재하면 그 위치를 반환하고, 존재하지 않으면 -1을 반환하도록 하는 간단한 Binary search 코드는 다음과 같이 짜면 된다.

``` python
def binary_search(L: list, s: int, e: int, t: int) -> int:
  while s <= e:
    m = int((s + e) / 2)
    if L[m] > t:
      e = m - 1
    elif L[m] < t:
      s = m + 1
    else:
      return m
  return -1
```

영역을 점점 좁혀가면서, ```L[m]```과 ```t```를 비교하는 루프를 돌면 되는 것이다. 영역이 사라졌는데도 ```t```를 찾지 못했다면, 루프를 벗어나서 -1을 반환하게 된다.

## 2. 삽입 위치 탐색 방법

그렇다면, ```t```를 찾지 못했을 때 -1을 반환하는 것이 아니라, 정렬된 ```L```에서 ```t```가 위치하게 될 인덱스를 반환하도록 문제를 조금 바꾼다면 어떻게 코드를 짜야 할까?
결론부터 말하자면, 다음과 같이 코드를 짜면 된다.

``` python
def binary_search(L: list, s: int, e: int, t: int) -> int:
  while s <= e:
    m = int((s + e) / 2)
    if L[m] > t:
      e = m - 1
    elif L[m] < t:
      s = m + 1
    else:
      return m
  return s
```

맨 마지막 ```return``` 부분만 바뀌었다. 이렇게 외우고 무지성으로 사용해도 되지만, 기계적으로만 암기한다면 시간이 흘렀을 때 또다시 헷갈리게 된다.
위 코드가 정답인 이유가 무엇일까? 차근차근 살펴보자.

우선, 루프가 진행되면서 영역이 점차 좁아지는 점에 착안하여, edge case를 중심으로 분석해보자.
```L```에 ```t```가 존재하는 경우는 바로 해당 위치가 반환되기 때문에, 고려 대상에서 제외하고, ```t```의 삽입 위치를 찾는 것에만 집중해보자.
가장 먼저 생각해볼 수 있는 것은 ```s == e```인 상태이다.
이해를 돕기 위해 다음과 같이 특수한 이진 트리를 정의해보자.

![edge case 1]({{ site.url }}{{ site.baseurl }}/assets/images/binary-search/edge-case1.PNG){: .align-center}

이 트리에서 노드는 영역을 표현한다. 그리고 노드에서 왼쪽으로 뻗어나가는 것은 ```t```가 ```m```보다 왼쪽에 있음을 의미하며,
오른쪽으로 뻗어나가는 것은 그 반대를 의미한다.

위 그림에서는 왼쪽, 오른쪽 모두 ```e < s```인 상태로 가기 때문에,
아래의 자식 노드들은 모두 코드상에서 루프를 빠져나와 마지막 ```return s```를 호출하는 부분으로 이어지게 된다.
잘 보면, ```t```는 ```e```와 ```s``` 사이에 위치하기 때문에, ```s```가 곧 ```t```가 삽입될 위치라는 것을 알아차릴 수 있다.

그렇다면, 루프를 돌다가 다음과 같이 한 칸 차이, 즉, ```s == e - 1```인 상태가 된 경우에는 어떻게 될까?

![edge case 2]({{ site.url }}{{ site.baseurl }}/assets/images/binary-search/edge-case2.PNG){: .align-center}

오른쪽으로 가는 루트는 이미 ```s == e```인 경우에서 살펴봤으므로 더이상 검증할 필요가 없다.
왼쪽으로 가는 루트도 역시 ```e < s```가 되면서 루프를 빠져나오게 되는데, ```t```가 ```e```와 ```s``` 사이에 있으므로, ```s``` 위치에 ```t```를 삽입하면 된다.
따라서, ```return s```는 ```t```가 삽입될 위치를 반환하는 것과 같은 의미가 된다.

아직 어안이 벙벙할 수 있다. 마지막으로 두 칸 차이, 즉, ```s == e - 2``` 상태인 그림 하나를 더 보자.

![edge case 3]({{ site.url }}{{ site.baseurl }}/assets/images/binary-search/edge-case3.PNG){: .align-center}

감이 오는가? 더이상 검증할 것이 없다. ```return s```는 ```t```가 삽입될 위치를 반환하는 것과 같은 의미라고 결론을 지을 수 있다.


## 3. 결론 (코드)

정렬된 리스트에서 특정 target 값의 위치를 찾을 때는 Binary search 알고리즘이 많이 사용된다.
만약, target이 없을 경우 삽입할 위치를 반환해야 한다면, 다음과 같이 Binary search 코드를 짜면 된다.

``` python
def binary_search(L: list, s: int, e: int, t: int) -> int:
  while s <= e:
    m = int((s + e) / 2)
    if L[m] > t:
      e = m - 1
    elif L[m] < t:
      s = m + 1
    else:
      return m
  return s
```

## 4. References

[https://leetcode.com/problems/search-insert-position/](https://leetcode.com/problems/search-insert-position/)
