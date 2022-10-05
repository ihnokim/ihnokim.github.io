---
title: "Testing Python 1: pytest and coverage"
categories: 
  - python
tags:
  - unit test
  - pytest
  - coverage
toc: true
toc_sticky: true
toc_label: "Table of contents"
---

Test Driven Development (TDD)의 중요성은 두 말할 필요가 없다. java의 JUnit이나 python의 unittest 등 여러 프로그래밍 언어들은 제각각 단위 테스트 도구들을 제공한다.
python의 경우, 기본적으로 unittest라는 단위 테스트 도구가 내장되어 있는데, JUnit처럼 테스트 코드를 구현하기 위해 정형화된 형식에 맞추어야 한다는 불편함이 있다. (e.g. class 구현, decorator 추가, etc.)
수소문을 해보니, python 진영에서는 pytest라는 3rd party 패키지를 많이 사용한다고 한다. 이유는 매우 간편하기 때문인데, 실제로 사용해보니 매우 직관적어서 진입장벽이 낮았다.

이 시리즈에서는 pytest를 사용하면서 고민하고 삽질했던 부분들을 공유하고자 한다. 다음과 같은 내용들을 예제와 함께 다룰 예정이다.

- pytest 설치 및 기본적인 사용법
- coverage 산출 방법
- Github Actions 및 Coveralls 연동 방법
- fixture, mock 등을 활용하여 가짜 database 만들기

## 1. pytest 준비하기

### 1-1. pytest 설치

``` bash
$ pip install pytest
```

위 명령어로 간단하게 모듈을 설치하면, 터미널에 "pytest"라는 명령어를 통해 python 코드들을 테스트할 준비가 완료된다.

### 1-2. 기본 사용법 및 파일 구조 세팅

"test_"라는 접두어가 붙은 python 스크립트 파일 안에 "test_"라는 접두어가 붙은 함수를 정의하면,
pytest가 실행될 때 자동으로 이 함수들을 수집하여 테스트를 진행해준다.
다른 단위 테스트 도구들과 마찬가지로, 테스트 함수 안에서 특정 동작을 수행시킨 뒤에
assertion과 관련된 구문을 사용하여 특정 test case가 어떤 결과를 내야만 하는지 정의해주는 식으로 테스트 코드를 작성하면 된다.

``` python
# test_example.py

def hello(name):
  return 'Hello, %s!' % name
 
def test_hello():
    assert hello('ihnokim') == 'Hello, ihnokim!'
```

위 ```test_hello.py``` 위치에서 pytest 명령어를 실행하면, 다음과 같이 테스트 결과를 보여준다.

``` bash
$ pytest
```

``` bash
============================= test session starts =============================
platform win32 -- Python 3.7.1, pytest-4.0.2, py-1.7.0, pluggy-0.8.0
rootdir: /home/ihnokim/pytest, inifile:
plugins: remotedata-0.3.1, openfiles-0.3.1, doctestplus-0.1.3, arraydiff-0.2
collected 1 item                                                               

test_example.py .                                                  [100%]

========================== 1 passed in 0.02 seconds ===========================
```

그런데, 사실 위 예제처럼 테스트하고자 하는 코드와 테스트 코드를 한 곳에서 동시에 작성하지는 않는다.
테스트 코드들은 별도로 한 곳에서 관리하기 편하도록 한 디렉토리에 모아주면 좋다.
주요 코드를 ```main.py```에 작성하고, 직접 만든 mymodule이라는 패키지를 ```main.py```에서 import해서, ```functions.py```에 정의된 함수들을 사용하는 시나리오를 생각해보자.
이 경우, 다음과 같이 구성하면 깔끔하다.

``` bash
.
├─mymodule
│  ├─__init__.py
│  └─functions.py
├─tests
│  ├─__init__.py
│  ├─conftest.py
│  └─test_example.py
└─main.py
```

주요 파일들의 내용은 다음과 같다.

``` python
# main.py
from mymodule.functions import hello

def say_hello_to_the_world():
    return hello('World')
```

``` python
# mymodule/functions.py
def hello(name):
    return 'Hello, %s!' % name
```

``` python
# tests/test_example.py
from mymodule.functions import hello
from main import say_hello_to_the_world

def test_hello():
    assert hello('ihnokim') == 'Hello, ihnokim!'

def test_say_hello_to_the_world():
    assert say_hello_to_the_world() == 'Hello, World!'
```

```tests/__init__.py```를 추가하는 이유는 테스트 코드에서 ```main.py```에 작성한 코드나,
```mymodule/functions.py```에 구현한 코드들을 import해서 사용하기 위함이다.
이 파일을 추가하지 않으면, 예를 들어,
```tests/test_example.py```에서 다음과 같이 ```mymodule/functions.py```에 작성한 함수를 import할 경우 에러가 발생한다.

``` bash
ImportError while importing test module 'tests/test_example.py'.
Hint: make sure your test modules/packages have valid Python names.
Traceback:
tests/test_example.py:1: in <module>
    from mymodule.functions import hello
E   ModuleNotFoundError: No module named 'mymodule'
!!!!!!!!!!!!!!!!!!! Interrupted: 1 errors during collection !!!!!!!!!!!!!!!!!!!
=========================== 1 error in 0.07 seconds ===========================
```

```tests/__init__.py```은 빈 파일이어도 상관없지만, 이 안에 테스트 과정에서 사용할 여러 함수들을 정의해두면,
다음과 같이 테스트 코드를 작성할 때 편하게 import하여 사용할 수 있어서 좋다.

``` python
from tests import *
```

```tests/conftest.py``` 파일은 위에서 살펴본 기본적인 내용 외에, pytest의 추가적인 기능들을 사용하기 위한 파일인데, 후에 추가적으로 설명하도록 하겠다.

## 2. coverage 측정하기

TDD 방식으로 개발을 진행할 때는, 작성된 코드가 단위 테스트를 통과했는지 여부를 확인하는 것이 중요하지만,
사실 테스트 코드를 매우 간단하게 작성하면 단위 테스트를 통과하기가 쉽기 때문에 별 의미가 없다.
따라서, 테스트 코드가 최대한 다양한 case들을 포함하고 있는지의 척도를 함께 확인할 필요가 있다.
전체 코드 line들 중에서 테스트 과정에서 실제로 실행되는 line 수의 비율이 바로 coverage다.
조금이라도 이름이 알려진 오픈소스들은 대부분 이 coverage를 90% 이상 유지하는 듯 하다.
Github 저장소에서 Coveralls이나 Codecov같은 서비스들과 연동하여
해당 저장소의 Test coverage가 몇 퍼센트인지 뱃지를 달아놓은 것을 종종 보았을 것이다.

아무튼, pytest 환경에서 이 coverage를 매우 간단하게 확인할 수 있다. 우선 pytest-cov라는 패키지를 설치해주자.

``` bash
$ pip install pytest-cov
```

그리고 나서, 다음과 같이 ```--cov```라는 옵션과 함께 pytest 명령어를 호출하면, coverage가 산출된다.

``` bash
$ pytest --cov mymodule --cov-report html tests
```

위 명령어의 의미는 다음을 의미한다: "mymodule이라는 패키지에 대해 coverage를 산출하는데,
이때 테스트 코드는 tests 디렉토리 안에 있는 것들을 사용하고, 결과는 HTML로 남겨라."

![coverage results]({{ site.url }}{{ site.baseurl }}/assets/images/coverage/coverage.PNG){: .align-center}

명령어를 실행하면, 같은 디렉토리 안에 htmlcov라는 디렉토리가 생기고, 그 안에 ```index.html``` 파일이 생성된다.
web browser를 통해 열어보면, mymodule에 작성한 코드들의 coverage를 확인해볼 수 있다.
위 이미지처럼 실제 어떤 라인을 놓치고 있는지도 상세히 확인할 수 있다.
만약 ```--cov-report html```이라는 옵션을 생략하면, 기본적으로 터미널에 결과를 출력해준다.

여기까지 진행했다면, 로컬 개발 환경에서 단위 테스트를 진행하고 test coverage까지 산출할 수 있는 준비가 완료되었다.
다음 글에서는 Github Actions와 Coveralls 등을 활용하여,
원격 저장소에서 자동으로 코드를 테스트하고 coverage를 산출하는 방법에 대해 살펴보겠다.

## 3. References

[https://docs.pytest.org/en/6.2.x/getting-started.html#create-your-first-test](https://docs.pytest.org/en/6.2.x/getting-started.html#create-your-first-test)

[https://pytest-cov.readthedocs.io/en/latest/readme.html#usage](https://pytest-cov.readthedocs.io/en/latest/readme.html#usage)
