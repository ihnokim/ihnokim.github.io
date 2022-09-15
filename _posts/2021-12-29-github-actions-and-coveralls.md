---
title:  "Testing Python 2: Github Actions and Coveralls"
categories: 
  - python
tags:
  - unit test
  - pytest
  - coverage
  - github actions
  - coveralls
toc: true
toc_sticky: true
toc_label: "Table of contents"
---

지난 글에서는 pytest와 pytest-cov를 사용하여 로컬에서 단위 테스트를 진행하고, coverage를 산출해보았다.
그런데, 사실 이 단위 테스트와 coverage 관리는 협업에서 더욱 빛을 발한다.
예를 들어, 다른 사람이 작성한 코드에 coverage가 높은 테스트 코드가 잘 작성되어 있다면,
테스트 코드만 가지고도 어떤 의도로 개발이 되었는지, 어떤 것들이 개발되어 있는지 확인이 가능하다.
또한, 해당 코드를 내가 수정하고자 할 때 코드 일부분을 변경한 뒤에 단위 테스트를 실행해보면,
pass or fail을 즉각적으로 판단할 수 있기 때문에, project의 요구사항 혹은 시나리오를 그대로 유지한 채로
빠르게 개발 및 협업이 가능하게 된다.

이번 글에서는 개발자들이 가장 많이 사용하는 협업 도구인 Github에
지난 글에서 살펴봤던 pytest와 pytest-cov를 연동하여 Continuous Integration (CI) 비슷한 것을 구현하는 방법을 소개하겠다.
또한, 테스트 과정을 충실하게 거친 프로젝트라는 것을 인정해주는 badge들을 산출물로 얻어보겠다.
이 글에 포함된 내용은 다음과 같다.

- Github repository와 Coveralls 연동
- Github Actions를 활용한 자동 테스트 구현
- 단위 테스트 및 coverage 결과 인증 badge 획득 및 자랑하기

## 1. Github와 Coveralls 연동

[Coveralls](https://coveralls.io)는 coverage 결과를 받아서 web browser를 통해 상세 내용들을 보여주는 서비스라고 생각하면 된다.
이 글에서의 목표 중 하나는 pytest-cov를 통해 산출한 coverage를 Coveralls에 보내고,
coverage가 몇 퍼센트인지 인증 badge를 획득하는 것이다.

이를 위해, 우선 Coveralls에 가입하면 ADD REPOS 메뉴를 통해 Github의 repository를 연결시킬 수 있게 된다.
다음과 같이 간단히 switch를 on 상태로 변경시켜주면 된다.

![coveralls switch]({{ site.url }}{{ site.baseurl }}/assets/images/coverage/coveralls-switch.PNG){: .align-center}

switch on 후에 REPOS 메뉴를 통해 연결된 repository를 확인해보면,
"There have been no builds for this repo."라는 메시지를 확인할 수 있다.
아직 coverage 결과를 이곳에 전달하지 않았기 때문에, 정상적인 결과이다.
해당 repository를 클릭해서 들어가보면, 다음과 같은 문구로 시작하는 안내 사항들을 확인할 수 있다.

![coveralls instructions]({{ site.url }}{{ site.baseurl }}/assets/images/coverage/coveralls-instructions.PNG){: .align-center}

이 안내에는 Travis CI와 관련된 내용, private CI 또는 command line을 통한 coverage 결과 제출에 대한 내용 등이 포함되어 있다.
일단, 이 글에서는 private CI 또는 command line을 통해 결과를 제출할 것이라는 정도만 인지하고 넘어가도록 하자.
관련된 부분을 보면 repo_token이라는 것을 확인할 수 있는데, coverage 결과를 제출할 때 함께 포함해야 하는 일종의 비밀번호라고 생각하면 된다.
Github Actions를 통해서 coverage를 제출할 경우, Github repository 자체에서 이 값을 가져올 수 있기 때문에,
이 repo_token을 직접 사용할 일은 사실 없다.
안내 사항들 최하단을 보면 다음과 같이 coverage 결과를 나타내는 badge를 확인할 수 있다.

![coveralls badge]({{ site.url }}{{ site.baseurl }}/assets/images/coverage/coveralls-badge.PNG){: .align-center}

EMBED 버튼을 클릭하면 해당 이미지를 다른 문서에서 표시할 수 있는 링크들을 파일 종류별로 나열해준다.
Github repository에 badge를 달고 싶다면, MARKDOWN이라는 부분에 담긴 내용을 복사해서
Github repository의 README.md 파일에 붙여넣으면 된다.
이 badge의 내용은 coverage를 제출할 때마다 자동으로 update된다.
따라서, 한 번 붙여넣은 후라면 coverage를 제출할 때마다 매번 링크를 수정할 필요가 없다.

## 2. Github Actions를 통한 자동 테스트 구현

### 2-1. Github Actions 소개

[Github Actions](https://docs.github.com/en/actions)는 Github repository에서
특정 event가 발생했을 때, 일련의 작업들을 순차적으로 실행되도록 해주는 자동화 툴이라고 생각하면 된다.
예를 들어, master branch에 push 또는 pull request가 발생할 때마다 테스트 환경을 구축하고,
특정 위치의 테스트 코드를 실행시키는 시나리오를 구현할 수 있다.

실제로 이 블로그도 Github Actions를 통해 정의된 일련의 작업들이 자동으로 동작하고 있으며,
Github repository에 가서 Actions라는 탭을 보면 다음과 같이 정의된 일련의 작업들을 그래프 구조로 표현해주고,
실행 결과를 시각적으로 나타내준다.

![github actions workflow summary]({{ site.url }}{{ site.baseurl }}/assets/images/github/github-actions-workflow-summary.PNG){: .align-center}

위 예시를 보면, 어떤 동작이 수행되었을 때, build를 한 뒤 deploy를 해주는 순서로 작업이 진행되며,
최근 진행된 결과가 성공적이라는 것을 직관적으로 알 수 있다.
또한, 각 노드를 클릭하면 다음과 같이 어떤 세부 절차들이 진행되는지 확인할 수 있다.

![github actions workflow node]({{ site.url }}{{ site.baseurl }}/assets/images/github/github-actions-workflow-node.PNG){: .align-center}

### 2-2. Github Actions 사용법 및 예제

위에서 본 workflow를 생성하는 방법은 매우 간단하다. 사용자의 Github repository에서
```.github/workflows```라는 디렉토리를 만들고, 그 안에 YAML 파일을 만들어서 상세 내용을 작성하면 된다.
실제 유효한 workflow를 만드려면 Github Actions에서 정의하는 양식에 맞추어 파일을 작성해야 하는데,
자세한 내용은 Github Actions의 공식 문서를 참고하면 된다. 내용이 방대한데, 시간 여유가 없다면
[Workflow syntax](https://docs.github.com/en/actions/learn-github-actions/workflow-syntax-for-github-actions)
부분 정도만 빠르게 읽어보면 된다.
일단 이 글에서 포함되어야 하는 내용들을 크게 나열해보면 다음과 같다.

- workflow의 이름 (name 필드에 정의)
- workflow를 trigger 시키는 event 정의 (on 필드에 정의)
- workflow를 구성하는 노드들 정의 (jobs 필드에 정의)
- 각 노드에서 수행할 세부 작업들 정의 (steps 필드에 정의)

이 글의 목표 시나리오를 충족하는 YAML 파일을 작성해보자. 우선 시나리오를 구체화해보면 다음과 같다.

1. master branch에서 push가 발생할 경우, 또는 branch 상관없이 pull request가 발생할 경우, workflow를 실행시킨다.
2. workflow는 Ubuntu 환경에서 Python 버전 3.6 및 3.7에 대한 테스트를 진행한다.
3. 테스트 환경을 셋업한 뒤, 테스트에 필요한 라이브러리나 모듈들을 설치한다.
4. 특정 위치에 있는 테스트 코드들을 pytest를 통해 실행시키고, coverage를 산출한다.
5. 산출한 coverage를 Coveralls로 보낸다.

위 시나리오를 구현한 YAML 파일은 다음과 같다.

``` yaml
name: Test
on:
  push:
    branches: [ master ]
  pull_request:
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      max-parallel: 4
      matrix:
        python-version: [3.6, 3.7]
      fail-fast: false
    steps:
    - uses: actions/checkout@v2
    - name: Set up Python {% raw %}${{ matrix.python-version }}{% endraw %}
      uses: actions/setup-python@v2
      with:
        python-version: {% raw %}${{ matrix.python-version }}{% endraw %}
    - name: Install Dependencies
      run: |
        python -m pip install --upgrade pip
        pip install pytest pytest-cov
        pip install -r requirements.txt
    - name: Run Tests
      run: |
        pytest --cov=<테스트할 코드 위치> tests
    - name: Upload coverage data to coveralls.io
      run: |
        python -m pip install coveralls==2.2
        coveralls --service=github
      env:
        GITHUB_TOKEN: {% raw %}${{ secrets.GITHUB_TOKEN }}{% endraw %}
```

복잡한 시나리오를 처음부터 manual하게 진행한다면 벌써부터 숨이 막히지만, Github Actions를 사용하면 이렇게 간단하게
파일 하나로 모든 과정을 수행할 수 있다. 내용 중 steps 필드에 대해 몇 가지만 짚고 넘어가자면,
우선 ```- uses: actions/checkout@v2``` 및 ```uses: actions/setup-python@v2```
부분은 이미 다른 개발자들이 만들어둔 action을 불러와서 사용한다는 의미이다.
마치 라이브러리를 import하듯이 [Marketplace](https://github.com/marketplace?type=actions)에 upload된 action들을
간단하게 사용할 수 있다.
특정 작업들을 직접 작성하기 전에, 이미 작성된 것이 있는지 먼저 찾아보면 유용한 것들을 많이 발견할 수 있다.
그리고, "Install Dependencies"라는 step을 보면 pytest와 pytest-cov를 설치하고 ```requirements.txt```에 나열한
패키지들을 pip를 통해 모두 다운로드한다. 즉, ```requirements.txt```에는 테스트와 관련된 패키지들을 제외시키고,
실제 배포 및 실행 환경에서 필요한 패키지들만 포함시키면 된다.

이렇게 Github Actions를 통해 테스트 환경을 구성하고, 테스트 자동화를 구현하면 개발 협업시 매우 편리해진다.
로컬 개발 환경에서 테스트를 진행하게 되면, 환경 구성에 따라 어떤 경우에는 성공하고
어떤 경우에는 실패하는 등 테스트의 일관성이 깨지지만,
이렇게 원격 저장소에서 테스트를 자동으로 실행해준다면 협업하는 개발자들은
모두 동일한 환경에서 테스트 결과를 얻을 수 있게 된다.

YAML 파일 작성을 마친 뒤, master branch에 push를 하면 Github Actions 탭에서 다음과 같이 workflow가 정상적으로
생성되고 실행이 완료된 것을 확인할 수 있다.

![github actions workflow example]({{ site.url }}{{ site.baseurl }}/assets/images/github/github-actions-workflow-example.PNG){: .align-center}

## 3. 인증 badge 얻기

### 3-1. Github Actions에서 workflow 진행 결과 badge

특정 workflow의 성공 여부를 인증해주는 badge의 이미지는 별도의 준비나 설정 없이, 다음과 같이 URL로 바로 확인 가능하다.

> https://github.com/\<user 이름\>/\<repository 이름\>/workflows/\<workflow 이름\>/badge.svg?event=push&branch=master

위 URL을 browser에 그대로 입력하거나, md 파일 등에 삽입하면 다음과 같이 badge 이미지를 확인할 수 있다.

![github actions workflow badge]({{ site.url }}{{ site.baseurl }}/assets/images/github/github-actions-workflow-badge.PNG){: .align-center}

### 3-2. Coveralls에서 coverage 결과 badge

이 내용은 1의 Coveralls 관련된 내용에서 설명했으므로 자세한 내용은 생략한다.
coverage 상승을 위해 테스트 코드를 작성하고 badge를 확인해보면, 다음과 같이 자동으로 반영이 되는 것을 확인할 수 있다.

![github actions workflow and coveralls badge]({{ site.url }}{{ site.baseurl }}/assets/images/github/github-actions-workflow-and-coveralls-badge.PNG){: .align-center}

여기까지 진행했으면, 코드 개발 후 Github repository에 push나 pull request를 했을 때,
자동으로 테스트가 진행되고 결과가 badge 형태로 repository에 반영되도록 구현이 완료된 것이다.
다음 글에서는, fixture, mock 등의 고급 (?) 기능을 활용하여 실제 테스트 환경에서
pytest의 활용도를 극대화하는 방법을 알아보도록 하겠다.

## 4. References

[https://github.com/coverallsapp/github-action/issues/30](https://github.com/coverallsapp/github-action/issues/30)

[https://github.com/marketplace/actions/coveralls-github-action](https://github.com/marketplace/actions/coveralls-github-action)
