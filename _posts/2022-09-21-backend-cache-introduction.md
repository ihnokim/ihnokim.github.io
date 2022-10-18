---
title: "Backend 환경에서 캐시 (Cache) 서비스의 개념 및 활용 방안"
categories:
  - backend
tags:
  - redis
  - cache
toc: true
toc_sticky: true
toc_label: "Table of contents"
---

웹/앱을 운영할 때 초기에는 backend server와 DB만으로 충분하지만, 점점 사용자 수 및 트래픽이 증가하게 되면 DB에의 읽기 및 쓰기 요청이 많이 발생하면서, 과부하가 발생하고 전체 시스템의 성능이 저하되어 사용자들에게 불편을 초래하게 된다. 이때 DB의 scaling 외에도 추가적으로 cache 도입을 검토하면 좀 더 쾌적한 서비스를 제공하는데 큰 도움이 된다. 이번 글에서는 cache에 대해 전반적으로 소개하고, 다양한 상황에서 어떤식으로 구현을 할 수 있는지 살펴보도록 하겠다.

## 1. cache란 무엇일까?

### 1-1. cache의 목적

cache는 자주 사용하는 데이터를 미리 복사해 놓는 저장소라고 이해하면 된다. 보통 high level 프로그래밍에서는 backend 서버에서 DB 관련 작업을 최소화하여 부하를 줄이는 목적으로 사용한다. 서버와 DB 사이의 상호작용 외에도, 이미지나 문서 등 static한 파일들을 미리 저장해두어 웹이나 앱의 로딩 속도를 향상시키도록 client app이나 CDN에서도 cache 전략을 사용한다.

![Elasticsearch text search]({{ site.url }}{{ site.baseurl }}/assets/images/cache/cache-example.PNG){: .align-center}

보통 cache 로직을 구현할 때는, cache miss와 cache hit 두 가지 경우로 나누어 동작을 설계한다. 위 그림을 바탕으로 예를 들어 보자. 서버에서 무거운 쿼리를 DB에 날려서 data를 조회하고자 하는 상황에서, 서버는 DB에 요청을 날리기 전에 우선적으로 cache 서버에 요청을 날린다. 만약 cache hit이라면, DB에 요청을 날릴 필요없이 바로 결과를 얻을 수 있다. 만약 cache miss라면, DB에 요청을 날려서 data를 얻은 뒤, 추후 재요청을 고려하여 cache 서버에 해당 data를 저장해둔다.

### 1-2. cache 사용의 이점

위 그림만 봐서는 서버와 DB가 직접 연결되어 있는 것에 비해 더 복잡해지기만 한 것 같은데, 어떤 이점이 있을까? cache를 사용하면 DB를 읽을 때 발생하는 I/O 오버헤드를 줄일 수 있다. cache는 보통 Memcached나 Redis와 같은 in-memory DB를 사용한다. in-memory DB는 메인 메모리를 사용하기 때문에, 디스크를 사용하는 일반적인 DB보다 data를 처리하는 속도가 빠르다.

### 1-3. caching하기 좋은 data

그렇다면 모든 data를 cache하는 것이 가장 좋지 않을까? in-memory DB를 쓰면 요청을 빠르게 처리할 수 있지만, 메모리 특성상 데이터가 유실될 위험이 있다. 또한, 메모리는 디스크에 비해 통상적으로 용량이 작기 때문에, 방대한 양의 데이터를 모두 메모리에 적재하는 것은 상대적으로 비용이 많이 든다.

이때 자주 인용되는 용어 중 파레토 법칙이라는 것이 있다. 파레토 법칙이란 20% 원인이 80%에 결과를 만든다는 법칙이다. 예를 들어, 백화점의 매출은 20% VIP 고객 지출로 전체매출의 80%를 차지한다는 예시가 있다. 바꿔 말하면, 모든 데이터를 caching할 필요는 없다. 20%를 잘 캐싱하는 것에 초점을 맞추어도 큰 효과를 볼 수 있다.

그렇다면 어떤 data를 caching하는 것이 좋을까?
 
- 자주 바뀌지 않는 데이터
- 자주 사용되는 데이터
- 자주 같은 결과를 반환하는 데이터
- 오래 걸리는 연산의 결과 (요청마다 caching하거나, 미리 batch 작업을 통해 caching)
 
이를 좀 더 기술적인 관점에서 구분하면 다음과 같이 두 가지로 설명할 수 있다.
 
1. 원본 data
2. 연산 결과 또는 response data

첫 번째로, 원본 데이터를 그대로 caching하는 방법은 다양한 경우에 cache를 재사용할 수 있기 때문에 범용적인 측면에서 좋은 방법이다.

두 번째로, 연산 결과 또는 response data를 캐싱하는 방법은 원본 data 외에 여러가지 데이터를 혼합하여 새로운 형태로 생성 혹은 변형된 data를 caching하는 방법이다. 예를 들어, JSON 형식으로 serialized 된 response 객체를 cache에 저장하는 것이다.

이 방식은 몇 가지 문제점이 있는데, 첫째로 이런 cache는 특정 함수나 API에 종속되기 때문에 재사용되는 경우가 적다. 즉, 상대적으로 cache hit ratio가 적어 cache로서의 의미가 퇴색된다. 또한, 중복되는 값이 cache 안에 다수 존재할 수 있다는 단점이 있다. 이런 중복된 값들이 변경되는 경우, cache 내부에 관련된 data들을 모두 수정 혹은 삭제해야 하기 때문에 까다롭다. 대신 원본 data를 caching하는 것에 비해 단일 함수 또는 API의 성능 향상 효과가 더욱 크게 나타날 수 있다.
 
## 2. 캐시 무효화 (cache invalidation)
 
cache는 임시 data이다, 즉, source of truth가 아니다. 만약 DB의 내용이 변경된다면, cache도 따라서 최신화되어야 한다. 오래된 cache data가 남아있으면 잘못된 data를 제공하게 되므로, 유효하지 않은 cache data를 무효화 (cache invalidation)하는 것이 매우 중요하다. cache invalidation 방법에는 세 가지 정도가 있다. 각각에 대해 먼저 살펴보고, 무효화된 cache data는 어떻게 처리하는지에 대해 알아보자.
 
### 2-1. Expiration Time

일반적으로 많이 사용하는 방식은 cache에 data를 저장할 때는 만료되는 시간 (이하 TTL)을 명시하는 것이다. cache data는 해당 시간 내에서만 유효하고, TTL이 경과하면 유효하지 않은 data로 처리한다. TTL을 짧게 정하면, cache data가 최신화되는 주기가 빨라지게 되므로, 잘 관리할 자신이 없다면 TTL을 짧게 정하는 것이 좋다. 하지만 그만큼 cache 사용으로 인한 성능 향상은 기대하기 어렵다.
 
### 2-2. Freshness Caching Verification

별도의 검증 절차를 통해 cache가 유효한지 확인하는 방법이다. 예를 들어, 원본 data가 수정된 시간과 cache data가 생성된 시간을 비교해서 해당 cache data의 유효성을 검증할 수 있다. 단점은 cache가 사용될 때마다 추가적인 확인 절차를 수행하기 때문에 이에 따른 오버헤드가 발생하게 된다.

### 2-3. Active Application Invalidation

data가 수정되는 코드가 실행될 때마다 backend에서 연관된 cache data를 직접 무력화하는 방법이다. cache를 세밀하게 관리할 수 있기 때문에 이상적으로 설계되었을 때 가장 효율성이 좋다. 단점은 별도의 코드를 통해 cache를 컨트롤 하기 때문에, 가장 에러가 발생하기 쉽고 관리하기 까다롭다.
 
### 2-4. 무효화된 cache data를 처리하는 방법
 
무효화된 cache는 다음과 같이 크게 두 가지 방법으로 처리할 수 있다. 

1. cache data에 변경점만 일부 수정하는 방법
2. cache data 자체를 삭제하는 방법

첫 번째 방법은 원본 data의 변화에 따라 cache data를 일부만 수정한다. cache data를 생성하는 것보다 일부만 수정하는게 효율적인 경우에 사용된다. 원본 data와 엮인 cache data가 많은 경우에는 일일이 cache data를 수정하는 것이 비효율적일 수도 있다.

두 번째 방법은 일반적으로 많이 사용된다. 원본 data가 변경되었을 때, 업데이트 할 cache가 많다면 ,일일이 캐시를 수정하기보다 단순히 삭제해버리는 것이 효율적이다. 단점은 데이터 중 일부만 변경된 경우에도 전체를 지우고 다시 생성하기 때문에, cache data 생성 과정이 무겁다면 비효율적이다. 일반적인 경우에는 cache data를 수정하기 보다 삭제하는 것이 관리 측면에서 용이하다.

## 3. caching 전략 설계

cache 로직을 구현하는 방식은 활용 패턴에 따라 다음과 같이 크게 다섯 가지로 나뉜다. 하나씩 살펴보자.

- Look-Aside (Cache-Aside) 읽기 전략
- Read-Through 읽기 전략
- Write-Around 쓰기 전략
- Write-Through 쓰기 전략
- Write-Back 쓰기 전략

### 3-1. Look-Aside (Cache-Aside) 읽기 전략

![Elasticsearch text search]({{ site.url }}{{ site.baseurl }}/assets/images/cache/look-aside.PNG){: .align-center}

1. 앱은 data를 조회할 때 cache를 먼저 확인한다.
2. cache hit인 경우 해당 data를 읽어오는 작업을 반복한다.
3. cache miss인 경우 앱은 DB에 접근해서 데이터를 직접 가지고 온 뒤, 다시 cache에 저장한다. 동일 데이터에 대한 후속 읽기 결과는 cache hit이 된다.

Look-Aside는 앱에서 data를 읽는 연산이 많을 때 사용하는 전략으로, 일반적으로 가장 많이 사용하는 방법이다. cache miss인 경우 DB에 직접 조회해서 입력되기 때문에 Lazy Loading이라고도 한다. 이 구조는 cache server가 다운되더라도 바로 장애로 이어지지 않고 DB에서 데이터를 가져올 수 있다는 장점이 있다. 그런데, 만약 캐시에 많은 커넥션이 붙어있는 상태에서 다운이 발생하면 동시에 DB로 커넥션이 몰리기 때문에 갑자기 DB에 과부하가 올 수 있다.

Look-Aside 전략과 함께 자주 사용되는 쓰기 전략은 DB에 직접 쓰는 Write-Around 쓰기 전략이다. 이 경우, cache와 DB의 data가 일치하지 않는 기간이 발생하게 된다. 이를 해결하기 위해 TTL을 사용하고, TTL이 만료될 때 까지는 변경되지 않은 캐시 데이터를 계속 제공한다. 최신 data를 보장해야 하는 경우에는 cache data를 무효화하거나 다른 쓰기 전략을 사용해야 한다.

### 3-2. Read-Through 읽기 전략

![Elasticsearch text search]({{ site.url }}{{ site.baseurl }}/assets/images/cache/read-through.PNG){: .align-center}

1. 앱은 모든 data를 cache에게만 요청한다.
2. cache miss가 발생한 경우 cache가 직접 DB에서 data를 읽어온다.

Read-Through 전략은 앱에서 data를 읽을 때 cache만 조회한다. cache는 앱과 DB 중간에 위치해 앱은 cache만 바라보게 되고 DB는 cache만 바라보는 구조이다. 만약 cache miss가 발생하면 DB에서 해당 data를 cache에 바로 저장한다. 첫 요청은 항상 cache miss가 발생하여 DB로부터 data가 로딩된다. 뉴스기사 열람같이 동일 data에 대해 여러 번 읽기 요청이 발생하는 경우에 적합하다.

앞서 설명한 Look-Aside 전략의 경우 cache miss가 발생하면 앱이 직접 DB에 data를 요청하는 반면, Read-Through는 cache에서 DB에 data를 직접 조회하여 로드한다. 다시 말해서, cache miss에 대한 처리 작업을 cache 쪽에 위임하는 것이다. 따라서, Read-Through는 cache의 data 모델과 DB의 data 모델이 일치해야 한다는 특징이 있다.

※ Cache warming: 데이터 미리 caching해두기

Look-Aside와 Read-Through 방식 모두 cache가 다운되면 다시 새로운 cache data를 쌓아야 한다. 이러한 경우, 초기에 cache miss가 빈번하게 발생해서 성능이 저하된다. 이는 캐시로 미리 데이터를 밀어넣어주는 cache warming 작업을 통해 보완할 수 있다. 예를 들어, 티켓팅을 하거나 특정 이벤트 상품을 오픈하기 전, 해당 data에 조회 요청이 몰릴 것을 대비해 관련 data를 미리 DB에서 cache로 dump해주는 작업을 해둘 수 있다.

### 3-3. Write-Through 쓰기 전략

![Elasticsearch text search]({{ site.url }}{{ site.baseurl }}/assets/images/cache/write-through.PNG){: .align-center}

1. 앱은 모든 data를 cache에 저장한 후 DB에 저장한다.

Write-Through 전략은 앱의 data 저장 요청을 cache에게 위임하는 것이다. cache는 항상 최신 data를 보장하고 DB와 동기화되어 data 일관성을 보장한다는 장점이 있다. 그런데, 저장할 때 마다 두 단계를 거쳐야 하기 때문에 추가적인 쓰기 작업이 발생하여 상대적으로 느리다는 단점이 있다. 그리고 저장하는 data가 재사용되지 않을 수도 있는데 무조건 cache에 넣기 때문에 리소스 낭비가 발생한다. 따라서, TTL을 필수적으로 설정해주어야 한다.

Read-Through 전략와 함께 사용하면 Read-Through의 모든 이점을 얻을 수 있고, data 일관성도 보장되어 cache invalidation을 고려하지 않아도 된다. AWS의 [DynamoDB Accelerator(DAX)](https://aws.amazon.com/ko/dynamodb/dax/)가 Read-Through와 Write-Through를 함께 사용한 예시이다. DynamoDB와 앱 중간에 배치되어 DynamoDB에 대한 읽기 쓰기는 DAX를 통해 수행된다.

주의할 점은, 실시간 log를 기록하는 시스템에서 Read-Through + Write-Through 전략을 사용한다면, 자주 사용되지 않는 data가 cache에 무조건 적재되기 때문에 리소스 낭비가 발생한다. 이 경우 Read-Through + Write-Around 전략을 사용하는 게 더욱 효율적이다.

### 3-4. Write-Around 쓰기 전략

![Elasticsearch text search]({{ site.url }}{{ site.baseurl }}/assets/images/cache/write-around.PNG){: .align-center}

1. 앱은 모든 data를 DB에 직접 저장한다.
2. cache miss가 발생한 경우에만 DB에서 cache로 data를 올린다.

Write-Around는 DB에만 data를 저장하고, 읽기 요청이 발생한 data만 caching하는 전략이다. 일단 모든 data는 곧바로 DB에 저장되고, cache miss가 발생한 경우에만 DB에서 cache로 data를 올린다. 자주 읽지 않는 data는 거의 caching되지 않으니 리소스가 절약된다. 읽기 전략은 크게 상관없다. Read-Through나 Look-Aside 모두와 결합이 가능하다. 그러나, 이미 caching된 data를 수정하는 경우, DB와 cache 사이의 data 일관성이 깨질 수 있다는 단점이 있다.

### 3-5. Write-Back 쓰기 전략

![Elasticsearch text search]({{ site.url }}{{ site.baseurl }}/assets/images/cache/write-back.PNG){: .align-center}

1. 앱은 모든 data를 cache에 저장한다.
2. 특정 시점이 되면, cache data를 DB에 저장한다.
3. DB에 저장된 data는 cache에서 삭제한다.

Write-Back 전략은 앱의 data 쓰기 작업 요청이 발생할 경우, cache에 먼저 저장했다가 특정 시점 혹은 다른 worker에 의해 DB에 저장하는 방식이다. 쓰기 요청이 빈번한 경우에 적합하다. DB 디스크 쓰기 비용을 절약할 수 있다. 그리고 DB에 쓰기 요청을 탄력적으로 하기 때문에 DB 장애에도 유연하게 대응할 수 있다. 그런데, 반대로 data를 cache에 모아두었다가 DB로 옮기기 때문에 cache 장애로 data가 유실될 수 있다. Write-Back은 Read-Through와 결합하면 가장 최근 업데이트되고 액세스된 데이터를 cache에서 얻을 수 있다. 대부분의 RDB storage engine 내부에는 Write-Back 캐시 기능이 있다. 쿼리들이 우선적으로 메모리에 기록되다가 특정 시점마다 디스크에 한 번에 flush된다.

## 4. cache의 data 유실 문제 보완

기본적으로 cache는 in-memory DB를 주로 사용하기 때문에, 필연적으로 data 유실 문제가 발생할 수 밖에 없다. 따라서, 디스크나 3rd party 솔루션 통해 주기적으로 영속적인 backup 혹은 snapshot을 남겨두고 문제 발생시 rollback을 하거나, replication을 통한 클러스터 구성을 통해 최대한 HA를 달성하는 등 추가적인 조치를 취해야 한다. 실제로 많이 사용되는 Redis의 경우 [persistence data](https://redis.io/topics/persistence)를 저장하는 방법을 제공해서, 메인 DB로 채택할 수도 있다.

다만, 이렇게 구성을 하기 위해서는 속도 저하 및 비용 증가에 대한 trade-off 관계를 잘 고려해야 한다. 이상의 자세한 내용은 Redis를 바탕으로 다른 글에서 다루도록 하겠다.

## 5. References

[https://rumor1993.tistory.com/86](https://rumor1993.tistory.com/86)

[파레토 법칙 - Wikipedia](https://ko.wikipedia.org/wiki/%ED%8C%8C%EB%A0%88%ED%86%A0_%EB%B2%95%EC%B9%99)

[https://loosie.tistory.com/800](https://loosie.tistory.com/800)

[https://www.qu3vipon.com/cache-strategy](https://www.qu3vipon.com/cache-strategy)
