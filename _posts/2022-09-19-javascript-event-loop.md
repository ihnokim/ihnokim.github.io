---
title:  "Event Loop: JavaScript의 비동기 처리 방식"
categories: 
  - backend
tags:
  - async
  - javascript
  - event loop
toc: true
toc_sticky: true
toc_label: "Table of contents"
---

javascript의 핵심 특징들을 탐구해보면, 싱글 스레드 기반의 동시성 (single-threaded concurrency) 제어와 관련된 내용이 자주 등장한다.
javascript는 단일 스레드 기반의 언어로 한 번에 하나의 작업만을 처리할 수 있다. 그런데, javascript는 비동기로 동작하기 때문에 단일 스레드임에도 불구하고 동시에 많은 작업을 수행한다.
단일 스레드? 비동기? 한 번에 하나의 작업만 처리 가능한데 어떻게 동시에 여러 작업을 수행할 수 있을까? 정말 단일 스레드가 맞는 것일까? 여러가지 의문이 생긴다.

비동기 처리는 Web, App 개발에 있어 필수적인 개념이다. javascript의 ```Promise```, flutter dart의 ```Future```, python의 ```Task```, kotlin의 ```Deferred``` 등 다양한 언어 및 프레임워크에서 비동기 처리를 지원한다.
더 나아가, WSGI와 ASGI의 동작 방식 차이, Spring, Node.js 기반 프레임워크들의 스레드 활용 방식 차이 등 심화된 내용에서도 빼놓을 수 없는 개념이다.
이 글에서는 javascript 런타임의 기본인 이벤트 루프 (event loop)에 대한 설명을 바탕으로, 비동기 처리에 대한 기본적인 개념을 알아보고자 한다.

## 1. event loop의 개념 및 주요 영역들

이벤트 루프 (event loop)는 javascript 런타임에서 동작원리를 설명하기 위해 핵심적인 개념이며, 크게 다음과 같은 역할을 돕는다.

- 코드의 실행
- 이벤트의 수집 및 처리
- queue에 대기 중인 작업 처리

event loop는 보통 싱글 스레드 기반의 동시성 (single-threaded concurrency) 제어에 활용된다. 자세한 설명에 앞서 이해를 돕기 위해 event loop와 관련된 stack, heap, queue 영역에 대한 내용을 살펴보자.

![stack, heap, queue explanation]({{ site.url }}{{ site.baseurl }}/assets/images/event_loop/event_loop1.PNG){: .align-center}

### 1-1. stack 영역

코드에서 함수를 호출하면, 함수의 인수와 지역변수 등의 정보가 포함된 frame이라는 형태로 stack에 쌓인다. 일반적인 프로그래밍 언어에서의 함수 호출 스택 (call stack)과 동일하게 동작한다. 선입후출 (First In, Last Out) 구조로, 함수 안에서 다른 함수를 호출하면 stack의 가장 위에 push되고, 실제 CPU 점유를 통한 실행은 stack의 가장 위부터 pop하여 실행하는 식으로 동작하는 것이다.

(참고) 엄밀하게 따지면, 인수와 지역 변수는 스택 바깥에 저장되므로 바깥 함수가 반환한 후에도 계속 존재할 수 있다. 따라서, 중첩 함수에서 지역 변수에 접근할 수 있다. 중첩 함수에 대한 내용은 다른 글에서 다루도록 하겠다.

### 1-2. heap 영역

메모리에서 비교적 크고 구조화되지 않은 영역이다. 객체 (object)들은 이 영역에 할당된다. 예를 들어, C 계열 언어에서 malloc, new 등을 통한 동적할당을 하거나, 객체지향언어에서 인스턴스화된 클래스의 객체들은 이 영역에 저장된다.

### 1-3. queue 영역

javascript의 런타임에서 처리할 메시지의 대기열로 사용되는 영역이다. 각각의 메시지에는 메시지를 처리하기 위한 callback 함수들이 연결된다. event loop는 선입선출 (First In, First Out) 방식으로 대기열에서 가장 오래된 메시지부터 queue에서 꺼내서 처리한다. 이를 위해 런타임은 꺼낸 메시지를 매개변수로 설정하여, 메시지에 연결된 callback 함수를 호출한다. 다른 함수와 마찬가지로 호출을 통한 새로운 stack frame이 생성된다.

## 2. event loop의 동작 방식 (비동기 처리)

C, Java 같은 언어로 기초 학습을 하다가 처음 javascript 언어를 접하는 개발자들이라면, 자신이 의도하고 코딩한 것과 다른 순서로 함수들이 동작하는 경험을 해본 적이 있을 것이다. 문법의 괴랄함과 더불어 이런 난해한 "실행 순서"가 javascript 언어의 진입장벽을 높이고 "ugly language"라는 오명을 낳게 한 주범이다.
event loop가 비동기 (async) 함수들을 처리하는 방식에 대한 설명을 통해 함수 실행 순서에 대한 실마리를 얻을 수 있다. 우선 시작에 앞서, javascript 언어 자체가 비동기 동작을 지원하는 것은 아니고, 브라우저 (browser) 혹은 node 같은 실행 환경에 의한 것임을 인지하고 넘어가자.
javascript의 런타임 자체는 단일 스레드이지만, browser 환경은 사실 여러 스레드를 사용한다는 점을 참고로 알아두자. 이후 내용들은 browser 환경을 기준으로 설명을 진행하겠다.

### 2-1. browser 실행 환경 구성요소들의 역할

![browser runtime]({{ site.url }}{{ site.baseurl }}/assets/images/event_loop/event_loop2.png){: .align-center}

browser 실행 환경을 역할별로 나누면, 크게 web APIs, event table, callback queue, event loop 등으로 구성되어 있다.
- heap & call stack: 앞서 설명한 부분과 동일하다.
- web APIs: DOM, AJAX, timer 등 브라우저가 제공하는 [API들](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Client-side_web_APIs/Introduction)을 의미한다.
- event table: 특정 이벤트 (e.g. timeout, click, mouse move, etc.)가 발생했을 때, 어떤 callback 함수가 호출돼야 하는지 저장해두는 자료구조이다. event와 callback의 매핑 테이블이라고 생각하면 된다. 위 그림에서는 생략되었다.
- callback queue: 이벤트가 발생할 경우 실행될 callback 함수가 저장 (예약)되는 곳이다.
- event loop: call stack과 callback queue를 지속적으로 모니터링하면서, call stack이 비어있을 경우 callback queue에서 함수를 꺼내 call stack에 추가해주는 역할을 수행한다.

### 2-2. event loop의 메시지 처리 방식

event loop가 메시지를 수신하는 동작을 pseudo code로 나타내면 다음과 같다.

``` javascript
while(queue.waitForMessage()) {
  queue.processNextMessage();
}
```

```queue.waitForMessage()``` 함수는 현재 처리할 수 있는 메시지가 존재하지 않으면 새로운 메시지가 도착할 때까지 동기적으로 대기한다.

각 메시지의 처리는 다른 메시지의 처리를 시작하기 전에 완전히 끝난다 (run-to-completion). 다시 말해서, event, 즉, 메시지끼리는 동기적으로 처리 순서가 반드시 보장된다. queue 자료구조 기반이기 때문에 당연한 이야기다. 이는 실행중인 함수가 다른 작업에 의해 선점당할 일이 없고, 다른 모든 코드의 실행보다 우선해서 값을 변경할 수 있으며, 중단되는 일 없이 완전히 끝나도록 보장한다. 참고로 C 언어의 런타임은 이와 달리, 특정 스레드에서 실행 중인 함수를 임의로 멈추고 다른 스레드의 다른 코드를 먼저 실행하도록 강제할 수 있다.

browser에서는 event listener가 부착된 이벤트가 발생하면 새로운 메시지를 queue에 추가한다. 이벤트가 발생했더라도 listener가 없다면 메시지는 유실된다. 예를 들어, ```onclick```과 같은 listener가 붙은 버튼, 링크 등을 클릭하면 해당 클릭 이벤트가 메시지 형태로 queue에 새로 추가된다.

### 2-3. callback queue vs job queue

ES6/ES2015 에서 소개된 job queue는 callback queue와 다른 queue이며 ```Promise``` 객체를 사용할 경우 job queue를 사용하게 된다. ```Promise```를 사용할 때 callback 함수 역할을 하는 ```.then```을 사용하게 되며, 이런 thenable한 함수들은 job queue에 추가된다.
두 queue는 우선순위가 다르다. job queue의 우선순위가 callback queue보다 높다. 따라서 event loop는 call stack이 비어있을 경우, job queue에서 기다리는 모든 작업을 처리한 뒤에 callback queue로 이동하게 된다.

### 2-4. event loop 기반 런타임 모델의 단점?

만약 메시지 하나를 처리할 때 너무 오래 걸리면 웹 앱이 클릭이나 스크롤과 같은 사용자 상호작용을 처리하기 어려워 UI의 반응성이 떨어진다. browser에서는 너무 오래걸리는 작업에 대해 "스크립트 응답 없음"을 화면에 표시해서 이 문제를 어느정도 완화해준다. 개발자로서 사용할 수 있는 좋은 방법으로는 메시지 처리를 가볍게 유지하고, 가능하다면 하나의 메시지를 여러 개로 나누는 것이다. 다시 말해서, 코드를 작성할 때 메시지 처리 함수를 최소한의 기능만 동작하도록 잘게 쪼개는 것이 핵심이라고 할 수 있다.

## 3. 예시를 통한 상세 설명

### 3-1. setTimeout을 통한 함수 실행 예약

javascript의 비동기 함수 관련 설명 예제에서 빼놓지 않고 등장하는 ```setTimeout``` 함수는 비동기 작업을 예약하는 용도로 사용한다. ```setTimeout``` 함수는 큐에 추가할 메시지, 딜레이 값 (default = 0)을 인자로 받는다. 딜레이 값은 메시지를 queue에 추가하기까지 기다릴 (최소) 지연 시간을 나타낸다.

queue에 다른 메시지가 없고 call stack이 비어있다면 ```setTimeout```의 메시지는 딜레이 값만큼의 시간이 지난 직후 즉시 처리된다. 그러나 다른 메시지가 존재한다면 ```setTimeout```은 앞선 메시지의 처리를 기다려야 한다. 따라서, 두 번째 인자로 전달하는 딜레이 값은 정확한 대기 시간을 보장하지 않는다. 예를 들어, 0의 지연 시간을 지정하는 것이 callback을 즉시 실행한다는 뜻은 아니다. 실제 실행 시점은 queue에서 대기 중인 작업의 수에 따라 다르다. ```setTimeout```에 특정 지연 시간을 지정하더라도, 큐에서 대기 중인 모든 메시지의 처리는 기다려야 한다.
browser의 개발자 도구에서 console을 연 뒤에 다음 코드를 입력해보자.

``` javascript
setTimeout(() => console.log("message 1"), 10000)
setTimeout(() => console.log("message 2"), 5000)
setTimeout(() => console.log("message 3"), 0)
console.log("message 4")
let start = new Date().getTime()
while (new Date().getTime() < start + 3000)
"message 5"
```

결과는 다음과 같다.

```
message 4
'message 5'
message 3
message 2
message 1
```

동작 과정을 순서대로 따라가보자.

1. 1 ~ 3번 줄의 경우, ```setTimeout``` 함수가 순차적으로 call stack에 push & pop되어 실행된다.
2. ```setTimeout``` 함수가 실행되면서, browser에서 제공하는 web API의 timer를 호출하고 바로 다음 작업을 이어서 진행한다. 이때, timer는 일정 시간을 대기한 후 "message n"을 출력하는 callback들을 queue에 등록하도록 예약해준다.
2. 4번 줄이 call stack에 저장되고, pop되어 즉시 수행된다. 따라서, console에 "message 4"가 출력된다.
3. 5 ~ 6번 줄이 call stack을 통해 실행되면서 3초 정도 대기한다.
4. call stack에 더 이상 실행할 함수가 남아있지 않기 때문에, 7번 줄이 실행되면서 명령어의 리턴 값으로써 console에 "message 5"가 출력된다.
5. event loop는 queue에서 하나씩 메시지를 가져오기 시작한다. 3번 줄에서 대기 시간 없이 바로 queue에 "message 3"을 출력하는 callback을 저장하는 작업을 예약했으므로, queue로부터 바로 call stack 쪽으로 해당 callback이 이동하면서 "message 3"이 console에 즉시 출력된다.
6. 위 3. ~ 5. 작업이 끝난 뒤 대략 2초 후에, timer에 의해 예약된 또다른 작업 (5초 대기하도록 예약된 작업)이 queue에 등록되면서, "message 2"가 출력된다.
7. 위 6. 작업이 끝난 뒤 대략 5초 후에, "message 1"이 출력된다.

그렇다면, 6번 줄에서 대기 시간을 7000으로 바꾸면 어떻게 될까? 처음에는 "message 4"만 console에 출력되어 있다가, 약 7초 후 "message 5", "message 3", "message 2"가 거의 동시에 출력된다. 그리고 약 3초 후 "message1"이 출력된다. 즉, call stack에서 7초를 기다리는 작업을 하는 동안, 0초, 5초 대기 후 queue에 callback이 저장되어 실행 준비 상태가 되지만, 아직 call stack의 작업이 끝나지 않았기 때문에, callback 이 실행되지 않다가 call stack 작업이 완료되자마자 즉시 event loop에 의해 queue에 있는 작업들이 call stack에 옮겨지면서 실행된 것이다.

### 3-2. blocking, non-blocking I/O

이 부분은 javascript의 ```Promise``` 객체와 ```async```, ```await``` 구문에 대한 이해가 필요하다. 자세한 내용은 다른 글에서 다루도록 하겠다.
앞선 예제에서 ```setTimeout``` 함수가 일정 시간 동안 대기했다가 함수를 실행하도록 예약하는 것임을 확인했다. 이는 non-blocking 방식의 대표적인 예로, 기다리는 시간동안 다른 동작들이 수행되지 않도록 막지 않는다. 즉, 비동기 처리를 통한 동시성 프로그래밍의 대표적인 예라고 할 수 있다. 그런데, 앞선 예제에서 ```while```을 통해 일정 시간을 기다리는 코드를 작성했었는데, ```setTimeout```과의 차이가 무엇일까? 정답은 blocking 방식이라는 점이다. 이해를 위해 다음의 예제를 살펴보자.

``` javascript
async function blocking_wait(ms) {
    let start = new Date().getTime();
    while (new Date().getTime() < start + ms);
}

function non_blocking_wait(ms) {
  return new Promise((resolve, reject) => {
    setTimeout(() => resolve(), ms);
  });
}

async function main() {
  console.log("message 1");
  await blocking_wait(5000); // await non_blokcing_wait(5000)
  console.log("message 2");
}

console.log("message 3");
main(); // await main()
console.log("wait...");
let start = new Date().getTime();
while (new Date().getTime() < start + 3000);
console.log("message 4");
"message 5"
```

결과는 다음과 같다. 완료까지 약 8초 정도 소요된다.

```
message 3
message 1
wait…
message 4
message 2
'message 5'
```

동작 과정을 순서대로 따라가보자.

1. 함수 선언부가 끝난 후, 앞선 예제에서 설명한 것과 동일하게 call stack을 통해 console에 "message 3", "message 1"이 거의 동시에 출력된다.
2. ```main``` 함수 안에서 ```blocking_wait(5000)```이 호출된다.
3. ```blocking_wait``` 함수는 5초를 대기한 뒤, ```Promise``` 객체를 반환한다.
4. 반환된 ```Promise``` 객체는 ```await``` 구문을 만나는데, 이때 queue에 함수 (아무 내용도 없는 ```Promise``` 객체)를 등록한 뒤 main을 호출한 caller로 컨트롤이 넘어간다.
5. call stack을 통해 ```console.log("wait...")```가 실행되며 console에 "wait…"가 출력된다.
6. 약 3초 대기 후 console에 "message 4"가 출력된다.
7. call stack에 더 이상 호출할 함수가 없으니 event loop는 queue에서 함수를 꺼내온다.
8. 꺼낸 것이 아무 내용도 없는 ```Promise```이므로, 즉시 완료되고 컨트롤이 ```main``` 함수로 다시 돌아온다.
9. call stack을 통해 ```console.log("message 2")```가 실행되어 console에 "message 2"가 출력된다.
10. ```main``` 함수가 종료되고 더 이상 수행할 작업이 없으므로, 명령어의 리턴 값인 "message 5"가 console에 출력된다. "message 2"와 "message 5"가 거의 동시에 출력되는 것처럼 보인다.

그렇다면, ```main``` 함수의 중간 부분을 ```await non_blokcing_wait(5000)```으로 변경하면 어떻게 될까?

결과는 다음과 같다. 완료까지 약 5초 정도 소요된다. ```blocking_wait``` 함수를 사용할 때보다 약 3초 정도 더 빠르다.

```
message 3
message 1
wait…
message 4
'message 5'
message 2
```

동작 과정을 순서대로 따라가보자.

1. call stack을 통해 console에 "message 3", "message 1"이 거의 동시에 출력된다.
2. ```main``` 함수에서 ```non_blocking_wait(5000)```가 수행되면서, ```Promise``` 객체를 반환한다.
3. 반환된 ```Promise``` 객체는 위와 달리 web API 기반의 ```setTimeout``` 함수를 통해 non-blocking 방식으로 작업을 예약한다.
4. 반환된 ```Promise``` 객체는 ```await``` 구문을 만나게 되고 ```main```을 호출한 caller로 컨트롤이 이동한다.
5. 앞서 ```non_blocking_wait``` 함수에서는 별도의 내부 동작 없이 바로 ```Promise``` 객체를 반환하기 때문에, 곧바로 ```console.log("wait...")```가 실행된다. 마치 "message 3", "message 1", "wait…"가 동시에 출력되는 것처럼 보인다.
6. 이어서 3초를 기다리는 코드가 실행되고, ```console.log("message 4")```가 실행된다.
7. 더 이상 call stack 및 queue에 실행할 작업이 없기 때문에, 명령어의 리턴 값인 "message 5"가 console에 출력된다.
8. 약 2초 후에 timer에 의해 예약되었던 작업이 queue에 등록되면서, event loop는 해당 작업을 call stack으로 옮겨 실행한다.
9. 작업이 완료되면, 컨트롤이 다시 main 함수 쪽으로 넘어간다.
10. ```await``` 구문 아래의 코드가 순차적으로 call stack을 통해 실행되면서 "message 2"가 console에 출력된다. 마치 "message 4", "message 5", "message 2"가 거의 동시에 출력되는 것처럼 보인다.

이 예제의 시사점은 매우 중요하다. 아무리 비동기 함수"처럼" 코드를 작성을 한다고 해도, 실제 내부적으로 동작하는 코드가 non-blocking 방식이 아님에도 ```async```와 ```await``` 구문을 남용한다면, 실제 코드의 동작은 일련의 동기 함수를 호출하는 경우와 크게 다르지 않을 수 있다. 따라서, 여러 3rd party API를 사용하는 경우에는 특정 함수가 비동기 처리를 타겟팅하여 non-blocking 방식으로 동작하는 것이 맞는지 꼼꼼하게 검토 후 사용하는 습관을 들여야 한다.

## 4. 다른 언어와의 비교

### 4-1. javascript vs python?

미묘한 차이가 있지만, low-level 파악에 시간을 많이 할애할 필요는 없어보인다. 가장 큰 차이는 javascript에서는 built-in event loop가 존재한다는 것이고, 파이썬에서는 asyncio같은 별도의 모듈을 통해 코드 레벨에서 event loop를 셋업해주는 부분을 준비해야 한다는 점이다. 문법적으로는 ```async```, ```await``` 구문 등을 거의 똑같이 사용할 수 있으며, javascript의 ```Promise``` 객체가 python의 ```Task```와 유사하다. 이 정도만 짚고 넘어가도록 하자.

### 4-2. javascript vs dart?

dart의 Future가 javascript의 ```Promise```와 유사한 개념으로 사용된다. ```async```, ```await``` 등 문법적인 측면도 매우 유사하다.

### 4-3. javascript vs kotlin?

javascript의 ```Promise```는 kotlin coroutine의 ```Deferred```와 개념적으로 유사하다. 단 몇 가지 문법적인 차이가 존재한다. kotlin의 경우 javascript나 python에서 함수 앞에 붙이는 ```async``` 대신 ```suspend```라는 용어를 사용한다. ```async```와 ```await```도 kotlin에 존재하긴 하지만, 사용 방식이나 형태가 javascript, python, dart와 다르다.

## 5. 다른 cross-origin 런타임과의 통신 방법

### 5-1. postMessage 함수를 통한 메시지 전송

web worker나 cross-origin의 iframe은 자신만의 stack, heap, queue를 가진다. 서로 다른 두 런타임은 ```postMessage``` 메서드를 통해 메시지를 보내는 방식으로만 서로 통신할 수 있다. 상대가 listener를 통해 메시지 (이벤트)를 수신하고 있을 때, ```postMessage```는 상대 런타임에 메시지를 추가한다. 페이지와 새로 생성된 팝업 간의 통신, 혹은 페이지와 embed된 iframe 사이의 통신에 사용할 수 있다.
기본적으로는 다른 페이지 간의 스크립트는 각 페이지가 같은 protocol, port 번호, host를 공유하고 있을 때에만 서로 접근이 가능하다. (동일 출처 정책) ```window.postMessage()```는 이 제약 조건을 안전하게 우회하도록 해주는 역할을 수행한다.

``` javascript
targetWindow.postMessage(message, targetOrigin, [transfer])
```

위 코드를 하나씩 살펴보면, 우선 ```targetWindow```는 메세지를 전달 받을 window의 객체이다. 여러 방식으로 window에 대한 참조를 얻을 수 있는데, 대표적인 것들은 다음과 같다.

- Window.open: 새 창을 만들고 새 창을 참조할 때
- Window.opener: 새 창을 만든 window를 참조할 때
- HTMLIFrameElement.contentWindow: 부모 window에서 임베디드된 iframe 태그를 참조할 때
- Window.parent: 임베디드된 iframe 태그에서 부모 window를 참조할 때
- Window.frames 리스트의 element들

다음으로 ```message```는 다른 window에 보내질 데이터를 의미한다. 데이터는 structured clone algorithm을 이용해 직렬화된다. 이를 통해 직접 직렬화하지 않고 대상 window에 다양한 데이터 객체를 안전하게 전송할 수 있다.

```targetOrigin```은 ```targetWindow```의 origin을 지정한 것이다. 이는 전송되는 이벤트에서 사용되며, 문자열 "\*" (별도로 지정하지 않음을 의미) 혹은 URI이어야 한다. 이벤트를 전송 할 때 ```targetWindow```의 스키마, host, port 번호 중 하나라도 ```targetOrigin```의 정보와 맞지 않으면 이벤트는 전송되지 않는다. 예를 들어, ```postMessage()```를 통해 비밀번호가 전송된다면, 악의적인 제 3자가 가로채지 못하도록, ```targetOrigin```을 반드시 지정한 수신자와 동일한 URI를 가지도록 설정하는 것이 중요하다. 다른 window의 document 위치를 알고 있다면, 항상 ```targetOrigin```에 "\*" 말고 특정한 값을 설정하자. 특정 대상을 지정하지 않으면 해커에 의해 전송하는 데이터가 공개돼버릴 수 있다.

마지막으로 ```transfer```는 optional한 인자이며, 일련의 Transferable 객체들을 의미한다. 이 객체들은 메세지와 함께 전송되며, 객체들의 소유권은 수신 측에게 전달되어 더 이상 송신 측에서 사용할 수 없게 된다.

아래와 같이 javascript 코드를 실행하면 다른 window (http://example.org:8080)에서 전달된 메시지를 받을 수 있다.

``` javascript
window.addEventListener("message", receiveMessage, false)

function receiveMessage(event)
{
  if (event.origin !== "http://example.org:8080")
    return;

  // do something
}
```

위 코드에서 callback에 전달되는 메시지 (```event```)는 다음과 같은 property들을 가진다.

- data: 다른 윈도우에서 전송된 객체이다.
- origin: ```postMessage```가 호출될 때 메시지를 보내는 window의 origin을 의미한다. http://example.com:8080같이 URI 형식이다.
- source: 메시지를 보낸 window 객체에 대한 참조. 이것을 활용하면 서로 다른 origin의 두 개의 윈도우에서 쌍방향 통신을 할 수 있다.

### 5-2. 보안 강화

크로스 사이트 스크립팅 공격 방지하기 위해 다음과 같은 조치를 취할 수 있다.

- 다른 사이트로부터 메시지를 받고 싶지 않다면, 메시지 이벤트를 위한 event listener 자체를 등록하지 말자.
- 만약 메시지를 받고 싶다면, 이벤트로 들어온 메시지 객체의 origin, source 등의 property들을 활용해서 항상 보내는 쪽의 신원을 확인하자.
- 보내는 쪽의 신원을 확인하는 것에 더해, 항상 받은 메시지의 구문이 유효한지 확인하자.
- ```postMessage``` 함수를 통해 다른 윈도우로 데이터를 보낼 때는 "\*"를 사용하지 말고 항상 정확한 타겟 origin을 사용하자.

## 6. References

[https://jenkov.com/tutorials/java-concurrency/single-threaded-concurrency.html#single-threaded-concurrency-designs](https://jenkov.com/tutorials/java-concurrency/single-threaded-concurrency.html#single-threaded-concurrency-designs)

[https://medium.com/sjk5766/call-stack%EA%B3%BC-execution-context-%EB%A5%BC-%EC%95%8C%EC%95%84%EB%B3%B4%EC%9E%90-3c877072db79](https://medium.com/sjk5766/call-stack%EA%B3%BC-execution-context-%EB%A5%BC-%EC%95%8C%EC%95%84%EB%B3%B4%EC%9E%90-3c877072db79)

[https://developer.mozilla.org/ko/docs/Web/JavaScript/EventLoop](https://developer.mozilla.org/ko/docs/Web/JavaScript/EventLoop)

[https://developer.mozilla.org/ko/docs/Web/API/Window/postMessage](https://developer.mozilla.org/ko/docs/Web/API/Window/postMessage)

[https://elvanov.com/2597](https://elvanov.com/2597)

[https://medium.com/sjk5766/javascript-%EB%B9%84%EB%8F%99%EA%B8%B0-%ED%95%B5%EC%8B%AC-event-loop-%EC%A0%95%EB%A6%AC-422eb29231a8](https://medium.com/sjk5766/javascript-%EB%B9%84%EB%8F%99%EA%B8%B0-%ED%95%B5%EC%8B%AC-event-loop-%EC%A0%95%EB%A6%AC-422eb29231a8)

[https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)

[https://medium.com/sjk5766/call-stack%EA%B3%BC-execution-context-%EB%A5%BC-%EC%95%8C%EC%95%84%EB%B3%B4%EC%9E%90-3c877072db79](https://medium.com/sjk5766/call-stack%EA%B3%BC-execution-context-%EB%A5%BC-%EC%95%8C%EC%95%84%EB%B3%B4%EC%9E%90-3c877072db79)

[https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Client-side_web_APIs/Introduction](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Client-side_web_APIs/Introduction)

[https://pks2974.medium.com/web-worker-%EA%B0%84%EB%8B%A8-%EC%A0%95%EB%A6%AC%ED%95%98%EA%B8%B0-4ec90055aa4d](https://pks2974.medium.com/web-worker-%EA%B0%84%EB%8B%A8-%EC%A0%95%EB%A6%AC%ED%95%98%EA%B8%B0-4ec90055aa4d)

[https://stackoverflow.com/questions/68139555/difference-between-async-await-in-python-vs-javascript](https://stackoverflow.com/questions/68139555/difference-between-async-await-in-python-vs-javascript)

[https://www.letmecompile.com/kotlin-coroutine-vs-javascript-async-comparison/](https://www.letmecompile.com/kotlin-coroutine-vs-javascript-async-comparison/)
