---
title:  "Substring Search Algorithm 1: Brute-force"
categories: 
  - algorithm
tags:
  - substring search
  - brute force
  - leetcode
toc: true
toc_sticky: true
toc_label: "Table of contents"
---

LeetCode 사이트의 [Implement strStr()](https://leetcode.com/problems/implement-strstr/) 문제를 풀다가,
문자열 검색 (substring search) 알고리즘들을 이참에 쭉 정리해두면 좋을 것 같아서 글을 작성하게 되었다.
예전 군복무 시절에 Coursera에서 Robert Sedgewick의 알고리즘 온라인 강의를 수강했었는데,
어렴풋이 문자열 검색과 관련된 신박한(?) 알고리즘들이 여러가지 등장했던 것이 기억난다.
그런데, 막상 구체적인 방법들이 잘 기억나지 않는 것을 보니 강의를 허투루 들었거나 머리가 많이 굳은 듯 하다...
반성하는 마음으로 확실하게 정리하고 주기적으로 다시 읽어봐야 겠다.
우선 이 글에서는 간단하게 문자열 검색 문제가 무엇인지 정의하고,
기초적인 Brute-force 알고리즘을 통해 해결하는 방법에 대해 살펴보자.

## 1. 문자열 검색 문제 정의

위에 링크해둔 LeetCode 문제에 아주 잘 설명되어 있어서,
내용을 확인해보고 오면 문자열 검색 문제가 무엇인지 바로 이해가 될 것이다.
해당 사이트에 있는 input & output 예제는 다음과 같다.

```
# Example 1:
Input: haystack = "hello", needle = "ll"
Output: 2

# Example 2:
Input: haystack = "aaaaa", needle = "bba"
Output: -1
```

문자열 검색 문제의 목표는 간단하다.
N 길이의 긴 input string이 주어지고, M 길이의 비교적 짧은 문자열 패턴(substring)이 주어진다.
이후 편의상 N 길이의 input string을 S(N), M 길이의 문자열 패턴을 P(M)이라고 표기하겠다.
이때, S(N)에서 P(M)의 위치를 찾으면 된다.
만약 찾고자 하는 패턴이 input string 내에 존재하지 않는다면, -1을 반환하면 된다.

실제로 이러한 작업은 많이 사용된다. 대표적인 예시가 바로 Java 언어의 ```indexOf()``` 같은 메서드이다.
또한 검색 엔진을 통해 문서에서 특정 단어를 포함하는지를 확인하는 작업이나,
web page로부터 특정 정보들만 추출해내기 위한 scraping 작업에도 활용된다.
즉, 매우 간단하지만 매우 중요한 문제다!

## 2. Brute-force 알고리즘

### 2-1. Brute-force 알고리즘 구현 방법

문자열 검색을 구현하는 아주 간단하면서도 무식한 방법은,
말그대로 S(N)의 문자를 하나씩 검사해보면서 P(M)과 일치하는지 비교하는 노가다(?)를 뛰는 것이다.

- 먼저, S(N)의 문자를 하나씩 스캔하는 인덱스 i와 P(M)의 문자를 하나씩 스캔하는 인덱스 j를 선언한다.
- 인덱스 i를 하나씩 움직일 때마다, 그 지점부터 시작해서 j를 하나씩 움직이며 P(M)과 일치하는지를 검사한다.
(이때, 일치하지 않는 문자가 나타날 때까지 진행한다.)
- j가 M (패턴의 길이)일 때, i를 반환한다. 이때, 반환되는 i는 P(M)의 위치가 된다!

LeetCode의 문제에 대한 답을 기준으로 python 코드를 작성해보면 다음과 같이 이중 for문으로 표현할 수 있다.
```haystack```이 S(N), ```needle```이 P(M)에 해당한다.

``` python
class Solution:
    # brute-force algorithm
    def strStr(self, haystack: str, needle: str) -> int:
        if len(needle) == 0:
            return 0
        for i in range(len(haystack) - len(needle) + 1):
            for j in range(len(needle)):
                if haystack[i + j] != needle[j]:
                    break
                if j == len(needle) - 1:
                    return i
        return -1
```

위 코드에서는 i를 고정해두고, j만 움직여가면서 문자열 비교를 수행한다.
Brute-force라는 이름과 이중 for문만 봐도 짐작이 되겠지만, 위 코드는 불필요한 반복 작업이 포함되어 있으므로
다소 비효율적인 코드라고 할 수 있다.
불필요한 반복 작업을 수행한다는 것을 명확히 이해하기 위해,
이미 수행했던 과거의 작업으로 "돌아간다"는 컨셉이 더 잘 드러나도록 코드를 수정해보자.
Robert Sedgewick은 강의에서 이를 "Backup"이라고 표현하고 있다.

- 패턴과 일치하는 문자를 찾으면 i와 j를 둘 다 진행시킨다.
- 패턴과 일치하지 않는 문자가 발견되면, ```i = i - j```를 통해 다시 i의 위치를 앞으로 당겨준 뒤 진행시킨다.
- j는 이미 match된 문자의 수를 뜻하게 되기 때문에, j가 M과 동일해지면 ```i - j```를 반환해주면 된다.

``` python
class Solution:
    # brute-force algorithm
    def strStr(self, haystack: str, needle: str) -> int:
        i = j = 0
        while i < len(haystack) and j < len(needle):
            if haystack[i] == needle[j]:
                j += 1
            else:
                i -= j
                j = 0
            i += 1
        if j == len(needle):
            return i - j
        else:
            return -1
```

이렇게 구현하면 확실히 Backup이 발생하고 있음을 직관적으로 이해할 수 있고,
while loop를 하나만 써서 코드도 깔끔해진다. (C나 Java면 더욱 깔끔해진다...)

### 2-2. Brute-force 알고리즘 평가

Brute-force 알고리즘은 많은 경우에 효율적이지만, 만약 S(N)과 P(M)에 반복된 패턴이 나타난다면 비효율적이다.
예를 들어, 찾고자하는 P(M)이 ```ABABABC```이고, S(N)에서 ```AB```가 3번 반복된 뒤 다시 ```AB```가 등장했다고 생각해보자.
알고리즘이 진행되면서 ```AB```가 이미 3번 반복된 것을 알기 때문에,
```AB```가 한 번 더 진행된 뒤 ```C```가 이어서 등장한다면, P(M)을 찾은 것이나 다름없다.
하지만, Brute-force 알고리즘은 3번의 ```AB``` 뒤에 곧바로 ```C```가 등장하지 않았다는 이유로
Backup을 한 뒤, 처음부터 다시 탐색을 진행하게 된다.
즉, S(N)과 P(M)에 반복된 패턴이 많을수록 불필요한 반복작업이 많이 발생하게 된다는 것을 의미한다.
당연하게도, 문자열 비교 측면에서 worst case 기준으로 O(MN)의 시간 복잡도를 가진다.
이는 좀 더 친숙하게 표현하면 O(N^2)에 가까운 시간 복잡도를 가지는 것으로 생각할 수 있다.
다음 글에서는 반복 작업을 최소화해서 효율성을 높이는 방법에 대해서 알아보도록 하자.

## 3. References

[https://leetcode.com/problems/implement-strstr/](https://leetcode.com/problems/implement-strstr/)

[https://www.coursera.org/learn/algorithms-part2](https://www.coursera.org/learn/algorithms-part2)
