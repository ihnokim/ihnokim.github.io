---
title:  "Testing Python 3: fixture and mock"
categories: 
  - python
tags:
  - unit test
  - pytest
  - fixture
  - mock
toc: true
toc_sticky: true
toc_label: "Table of contents"
---

지난 글에서는 Github Actions와 Coveralls를 활용하여 자동 테스트를 구현해보았다.
이를 통해 배포 전 동일한 환경에서 테스트를 진행할 수 있게 되었다.
하지만, 이 상태로 실제 개발을 진행해보면 여전히 불편함이 존재한다.
작성된 코드 중 외부와 통신을 해야 하는 경우는 어떻게 테스트해야 할까?
예를 들어, database에 특정 값을 저장하거나 받아서 확인하는 경우,
또는 외부의 REST API 서비스에 요청을 해서 값을 받아오는 경우 등이 대표적이다.

개발 환경과 테스트 환경이 일치한다면 상관이 없지 않냐고 생각할 수 있다.
그렇다면, 만약 database와 통신하는 기능을 테스트하는데, database에 문제가 발생하면 어떻게 될까?
내가 작성한 코드에는 문제가 없는데, database라는 외부 요인으로 인해 테스트가 실패하게 된다.
단위 테스트 코드는 온전히 내가 개발한 코드의 기능들을 테스트하기 위한 것인데,
외부 요인에 영향을 받는 것은 바람직하지 않다.
그리고, 만약 database에 특정 값을 저장하는 기능을 테스트할 때,
이 동작이 실제 배포 환경에서는 건드려서는 안될 값을 덮어쓴다면 어떻게 될까?
테스트를 할 때마다 database의 값이 오염되므로, 이것도 바람직하지 않다.

이 글에서는 단위 테스트가 환경에 구애받지 않도록 외부 요인들을 가짜로 만드는 기법들에 대해 소개하고자 한다.
이를 mocking이라고 하는데, pytest의 fixture라는 기능과 unittest의 mock이라는 기능을 조합하여
예제와 함께 살펴보겠다. 이 글에 포함된 주요 내용들은 다음과 같다.

- fixture와 mock 개념 설명
- 테스트 과정에서 시스템 환경 변수 일시적으로 설정하기
- 테스트 디렉토리 자동 생성 및 삭제
- 가상의 MongoDB로 database 없이 테스트하기

## 1. pytest의 fixture 소개

[fixture](https://docs.pytest.org/en/6.2.x/fixture.html)는
테스트 과정을 진행하기 전에 미리 준비해두는 것이라고 생각하면 된다.
준비한 fixture는 테스트 과정에서 자유롭게 불러서 사용이 가능하다.
[이전 글](https://ihnokim.github.io/python/pytest-and-coverage/)에서 테스트 코드를 한 디렉토리에 모을 때,
```tests/conftest.py```라는 파일에 대해서는 설명을 생략하고 넘어갔었는데,
fixture들을 준비해두는 곳이 바로 이 파일이다.
이 파일에 ```@pytest.fixture```라는 decorator가 추가된 함수들을 정의하면,
다른 단위 테스트 파일들에서 이 함수의 내용을 사용할 수 있게 된다.

### 1-1. 테스트 환경의 임시 환경 변수 설정

간단한 예제를 통해 사용법을 알아보자.

``` python
# tests/conftest.py
@pytest.fixture(scope='function', autouse=False)
def my_os_env():
    os.environ['USERNAME'] = 'ihnokim'
    yield
    os.environ.pop('USERNAME')
```

위 코드는 테스트 함수가 시작되기 전 시스템 환경 변수 "USERNAME"를 세팅하고,
테스트 과정에서 이를 사용한 뒤, 시스템 환경 변수를 제거하는 예제이다.
우선 decorator를 보면, scope와 autouse라는 인자가 있다.
scope는 해당 fixture가 영향을 미칠 범위를 뜻한다.
예를 들어, ```scope='session'```의 경우,
pytest 명령어가 실행될 때부터 전체 테스트 과정이 종료될 때 까지 영향을 미치고,
```scope='function'```의 경우, "test_"라는 prefix가 붙은
각각의 테스트 함수들이 실행되고 종료될 때까지 영향을 미친다는 의미이다.
이외에도 여러 scope가 있으니 자세한 내용은
[공식 문서](https://docs.pytest.org/en/6.2.x/fixture.html#scope-sharing-fixtures-across-classes-modules-packages-or-session)를 참고하길 바란다.

함수의 내용을 살펴보면, "USERNAME"이라는 환경 변수를 설정한 뒤 ```yield```가 호출된다.
테스트 함수 쪽에서 이 fixture를 사용한다고 선언을 하면,
우선 이 fixture 함수의 ```yield``` 바로 전까지만 실행을 한 뒤,
테스트 함수 자신의 로직을 실행한다. 테스트 함수가 종료되고 나면,
다시 fixture 함수로 돌아와서 ```yield``` 바로 다음부터 실행이 된다고 이해하면 된다.

그렇다면, 테스트 함수 쪽에서는 어떻게 이 fixture를 사용할까?
간단히 테스트 함수의 인자에 fixture decorator를 사용한 함수명을 포함시켜주면 된다.
이때, 테스트 함수가 ```tests/conftest.py```가 아닌, 다른 파일에 있더라도
fixture 함수명을 자동으로 인식한다.

``` python
# tests/test_print_os_env.py
def test_print_os_env(my_os_env):
    assert os.environ['USERNAME'] == 'ihnokim'
```

위 테스트 코드를 pytest 명령어를 통해 실행시켜보면, 테스트가 성공하는 것을 확인할 수 있을 것이다.
```scope='function'```으로 설정해두었기 때문에, ```my_os_env```라는 fixture를 인자로 받지 않는 테스트 함수들은
"USERNAME"이라는 환경 변수를 참조할 경우 에러가 발생하게 된다.

### 1-2. 테스트 환경의 임시 디렉토리 생성 및 삭제

어떤 코드들은 실행 결과로 파일들을 생성하는 경우가 있다.
이 경우, 테스트를 수행할 때마다 파일이 계속 생성되기 때문에,
테스트가 완료될 때마다 해당 output 파일들을 지워야 하는 문제가 있다.
이런 경우에도 fixture를 활용하면, 테스트가 시작되기 전에 미리 특정 디렉토리를 만들고 
코드에서 해당 디렉토리에 파일들을 생성하도록 테스트를 진행한 뒤,
테스트가 완료되면 모든 output 파일들과 함께 디렉토리를 제거할 수 있다.

```tests/conftest.py``` 파일에 다음과 같이 fixture를 정의해보자.

``` python
def mkdir(directory):
    if not os.path.exists(directory):
        os.makedirs(directory)

def rmdir(directory):
    if os.path.exists(directory):
        rmtree(directory)

@pytest.fixture(scope='session', autouse=True)
def testdir():
    dirpath = 'testdir/'
    mkdir(dirpath)
    yield
    rmdir(dirpath)
```

앞선 환경변수 예제에서 이미 pytest decorator에 대해 사용법을 살펴봤으므로, 코드 해석이 어렵지 않을 것이다.
위 예제는 테스트가 실행되기 전에 testdir이라는 이름의 디렉토리를 만든 뒤, 테스트가 종료되면 삭제하는 코드이다.
앞선 환경변수 예제와는 달리 ```autouse=True```로 설정되어 있기 때문에, 테스트 코드 쪽에서
별다른 인자를 설정하지 않아도 testdir 디렉토리를 사용할 수 있게 된다.
이를 검증하기 위한 테스트 코드는 다음과 같이 작성해보자.

``` python
def test_testdir():
    assert os.path.exists('testdir/')
```

위 코드를 pytest로 실행시켜보면 pass되는 것을 확인 가능하다. 인자에 ```def test_testdir(testdir):``` 이렇게 fixture 함수를 설정하지 않았는데도,
testdir라는 디렉토리가 생성된 것이다. 테스트가 종료된 뒤에 testdir 디렉토리는 삭제가 되어 찾을 수 없다.


## 2. unittest의 mock 소개

[mock](https://docs.python.org/ko/3/library/unittest.mock-examples.html)은 코드의 특정 부분을 덮어쓰는 행위라고 이해하면 쉽다.
이 덮어쓰는 행위를 patch라고 하는데, 테스트 과정에서 patch를 통해 여러 제약사항들을 해결할 수 있다.
예를 들어, 처음에 언급했던 것처럼 테스트 환경에서 database 연결 상태를 보장할 수 없을 경우,
단순히 이 mock을 통해 database를 사용하는 부분을 다른 코드로 덮어씌운 뒤에 테스트를 하면 된다.

``` python
from mymodule import functions
from unittest.mock import patch
# from mymodule.functions import hello

def test_mock_function():
    with patch('mymodule.functions.hello') as mock:
        mock.return_value = 'Bye!'
        assert 'Bye!' == functions.hello('Hello!')
```

patch를 적용할 때 주의할 점은 ```patch('mymodule.functions.hello')``` 부분처럼 patch 대상의 full path를 적어줘야 한다는 것이다.
위 코드의 세 번째 줄에 주석을 해둔 것처럼 ```hello``` 함수만 import를 한 뒤에 patch 부분에 ```patch('hello')```만 적어주면 정상적으로 동작하지 않는다.

``` python
@patch('mymodule.functions.hello')
def test_mock_function(mock):
    mock.return_value = 'Bye!'
    assert 'Bye!' == functions.hello('Hello!')
```

또는 이렇게 patch라는 decorator를 사용하여 코드를 더욱 깔끔하게 만들어줄 수 있다.
위에서 ```with ... as mock``` 부분 대신 테스트 코드의 인자에 ```mock```을 전달하여 사용하면 된다.
mock이라는 인자와 함께 fixture들도 인자로 주면, mock과 fixture를 둘 다 사용 가능하다.
(e.g. ```def test_mock_and_fixture(mock, my_fixture1, my_fixture2):```)
다만, 이 경우 mock이 가장 앞에 와야 한다.

### 2-1. 가상의 MongoDB로 database 연결 없이 테스트하기

보통 웹이나 앱의 코드에서 database를 활용할 때는 adapter 패턴을 많이 사용한다.
쉽게 말해서, 직접 database에 접근하는 라이브러리를 바로 사용하는 것이 아니라,
해당 라이브러리를 wrapping해서 database client와 코드 사이에 하나의 layer를 두는 것이다.
이 adapter에서 웹이나 앱의 목적에 맞는 형식의 데이터에 대한 IO를 하도록 코드를 개발한다.
예를 들어, 직접 database에 접근하는 라이브러리가 ```query('SELECT * FROM test_db.test_table;')```
이런식으로 직접 쿼리문을 작성해야 한다면, adapter 클래스에서는 ```select```와 같은 임의의 함수를 만들어서
동일한 동작을 하지만, 필요한 만큼만 제한된 동작을 하도록 구현하는 것이다.
(내부적으로는 직접 database에 접근하는 라이브러리 사용)

이런 adapter 클래스를 구현하여 사용할 경우, 테스트시 database 연결 불가 등으로 인한 문제에 봉착하게 된다.
그렇다면, 테스트 코드는 어떻게 작성해야 할까?
위에서 살펴본 mock을 사용하여, adapter가 내부적으로 사용하는 database client 라이브러리 부분을
patch한다면, 간단하게 테스트를 통과할 수 있게 된다.

두 가지 접근 방식이 있는데, 첫 번째는 database client의 코드가 수행하는 일부 동작들을 patch하는 것이다.
예를 들어, select 쿼리를 날리는 동작이 있다면, 그 결과로 무조건 특정 값을 반환하도록 patch하면 된다.
다만, 이 경우 엄밀히 말하면 adapter 코드가 정상적으로 동작하는지 테스트했다고 보긴 어렵다.
답을 미리 정해두고 억지로 테스트를 통과시키는 인상이 강하다.

두 번째는 테스트용 database client 클래스를 직접 개발한 뒤에, database client 객체 자체를 patch해버리는 것이다.
예제를 통해 살펴보자. 시나리오는 다음과 같다.

1. ```MyMongoAdapter```라는 adapter 클래스의 기능들을 테스트한다.
2. ```MyMongoAdapter``` 클래스는 실제 MongoDB에 접근하는 ```MongoClient```라는 클래스를 내부적으로 사용한다.
3. 테스트 환경에서는 database에 접근이 불가능하다.
4. ```MongoClient``` 객체를 그대로 사용하면 연결을 시도할 때 테스트가 실패한다.
5. ```FakeMongoClient```라는 클래스는 ```MongoClient```와 동일한 동작을 수행하지만, 메모리 상에서 모든 작업을 처리해서 database 연결이 필요없는 직접 개발한 클래스이다.
6. ```MongoClient``` 객체를 patch를 통해 ```FakeMongoClient``` 객체로 덮어쓴 후 테스트를 진행한다.

``` python
# tests/conftest.py
@pytest.fixture(scope='function', autouse=False)
def fake_pymongo_client():
    with patch('mymodule.classes.MongoClient') as mock:
        mock.return_value = FakeMongoClient()
        instance = mock.return_value
        yield instance
```

앞서 살펴봤던 fixture와 mock을 조합하여 사용하면,
위 코드처럼 테스트 환경에서 간편하게 ```FakeMongoClient``` 객체로 ```MongoClient``` 객체를 덮어쓴 뒤에,
해당 객체를 가져오도록 하는 코드를 작성할 수 있다.
이 코드는 다음과 같이 테스트 코드 내에서 fixture로 간단하게 사용할 수 있다.

``` python
def test_fake_pymongo_adapter(fake_pymongo_client):
    adapter = MyMongoAdapter()
    adapter.test_db.test_collection.insert_one({'username': 'ihnokim', 'token': 1})
    document = adapter.test_db.test_collection.find({'username': 'ihnokim'})
    assert '_id' in document
    assert 'token' in document
    assert document['token'] == 1
    assert not fake_pymongo_client.closed
    adapter.close()
    assert fake_pymongo_client.closed
    assert len(fake_pymongo_client.test_db.test_collection.documents) == 1
```

위 코드에서 ```fake_pymongo_client```라는 fixture 인자를 제거하면, 테스트 진행시 database 연결 실패로
timeout error 등이 발생하는 것을 확인할 수 있을 것이다.

### 2-2. 오픈 소스 활용하여 database 연결없이 MongoDB adapter 완벽하게 테스트하기

앞선 내용의 문제점은 ```FakeMongoClient``` 클래스의 불완전성에 있다.
```MongoClient```를 database 없이 테스트하기 위해 최대한 비슷하게 재현한 사용자정의 클래스이기 때문에, ```MongoClient```를 완벽하게 대체하기는 불가능하다. 예를 들어, 간단한 ```find```나 ```insert``` 작업들은 쉽게 구현이 가능하지만, 필터링 기능, 정렬 기능, 그리고 통계치 산출 기능 등의 복잡한 기능들은 재현하기가 까다롭다.

adapter 클래스를 테스트하기 위해 별도의 사용자정의 클래스를 개발한다는 것 자체가 배보다 배꼽이 더 큰 경우이므로, 바람직하지 않아보인다.
이럴 때는, 다른 개발자가 만든 ```FakeMongoClient``` 객체를 쓰면 된다!
실제로, [mongomock](https://github.com/mongomock/mongomock)이라는 오픈 소스가 있는데, 위 예제에서 ```FakeMongoClient```와 정확히 동일한 의도로 개발된 모듈이다.

사용 방법도 매우 유사하다. 다음과 같이 fixture를 만들어두고, 테스트 코드에서 가져다가 사용하기만 하면 된다.

``` python
import mongomock

@pytest.fixture(scope='function', autouse=False)
def fake_mongodb():
    adapter = MyMongoAdapter()
    adapter.client = mongomock.MongoClient()
    yield adapter
    adapter.close()
```

실제 ```pymongo``` 패키지의 ```MongoClient```와 거의 동일한 수준의 기능들이 구현되어 있으므로, 대부분의 테스트 케이스들은 커버가 가능하다.

여기까지 python의 단위 테스트를 위한 여러가지 방법론들에 대해 정리해보았다.
fixture의 [parametrization](https://docs.pytest.org/en/latest/how-to/parametrize.html#parametrize) 등
추가적인 기능들도 있지만, 이번 시리즈를 통해 익힌 배경지식이 있다면, 공식 문서를 통해 쉽게 익힐 수 있을 것이라고 생각한다.

## 3. References

[https://docs.pytest.org/en/6.2.x/fixture.html](https://docs.pytest.org/en/6.2.x/fixture.html)

[https://docs.pytest.org/en/6.2.x/fixture.html#scope-sharing-fixtures-across-classes-modules-packages-or-session](https://docs.pytest.org/en/6.2.x/fixture.html#scope-sharing-fixtures-across-classes-modules-packages-or-session)

[https://docs.python.org/ko/3/library/unittest.mock-examples.html](https://docs.python.org/ko/3/library/unittest.mock-examples.html)

[https://github.com/mongomock/mongomock](https://github.com/mongomock/mongomock)

[https://docs.pytest.org/en/latest/how-to/parametrize.html#parametrize](https://docs.pytest.org/en/latest/how-to/parametrize.html#parametrize)
