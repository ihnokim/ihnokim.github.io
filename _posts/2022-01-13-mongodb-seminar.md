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


index 관련 질문
(1) 필드가 없는 doc?

답변:
btree에서 기본으로 key가 없는 경우 null로 처리해서 indexing함
partial index라는 것을 사용하면 특정 필드가 있는 경우에만 indexing하도록 설정할 수 있음
이렇게 하면 index size 작고 매우 효율적이게 됨


(2) value가 list인 필드?

답변:
index 먹은 필드의 값이 array인 경우
=> multi key index!
e.g. tag: [a, b, c, d]
=> tag.a, tag.b, tag.c, tag.d 이렇게 다 생김!
따라서, value가 array인 필드에 인덱스 걸면 비효율적이다. 권장하지 않음

...세미나 내용 중 MongoDB의 경우, RDB와는 다르게 잘게 쪼개는 것이 아니라, 한번에 조회하는 모든 데이터를 하나의 doc에 쑤셔 박는 것이
권장된다는 내용 + bucket 패턴이 좋다는 내용 등 소개가 나옴...
doc 하나의 최대 용량은 16MB라고 함.
이에 따른 나의 질문


(3) 내부적으로 projection하거나 일부 key만 update하는 경우에 doc 전체를 메모리에 올리나? 아니면 해당 일부분만 올리나?
내부 동작으로는 proejction 하더라도 doc 전체를 page 단위로 load함
따라서, DB 관점에서는 전체 필드 가져오는 것과 도찐개찐임
단, 네트워크 부하가 줄어든다!
일부 필드만 업데이트 할 때는 oplog가 update하는 필드에 대한 내용만 남음!

따라서, 특정 필드 값에 대해 polling을 할 경우, 해당 필드만 따로 doc으로 구성하는 것이 바람직함


아비터는 임시다! 쓰라고 만들어 놓은게 아님.
저예산이라서 최소 3대로 Replicaset 구성할 수 없을 때, (두 대만 있을 때)
임시로 하나 머릿수만 채워주는 개념임
아비터 - primary - secondary
이렇게 구성할 경우, primary가 죽으면 secondary가 primary 역할 받아서 괜찮은데,
secondary가 먼저 죽어버리면 답이 없다. (write할 때 최소 ack 수랑 관련 있다는데 다시 생각해보자.)
