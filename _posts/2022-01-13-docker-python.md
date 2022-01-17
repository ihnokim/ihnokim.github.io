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


## Python + docker-py

https://hub.docker.com/_/python


$ docker run -it --rm --name my-running-script -v "$PWD":/usr/src/myapp -w /usr/src/myapp python:3.6-slim python your-daemon-or-script.py

이거랑 동일한 것

client.containers.run(image='python:3.6-slim', volumes=['/home/ihnokim/workspace/docker:/usr/src/myapp'],
working_dir='/usr/src/myapp', auto_remove=True, name='docker-py-test', command=['python', 'hello.py'])

client.images.pull() 가능함
client.images.build(path='.', tag='inhohaha') 가능


## docker run inside docker

airflow DockerOperator: https://airflow.apache.org/docs/apache-airflow-providers-docker/stable/_api/airflow/providers/docker/operators/docker/index.html

https://www.docker.com/blog/docker-can-now-run-within-docker/

https://stackoverflow.com/questions/27879713/is-it-ok-to-run-docker-from-inside-docker

## Remote docker engine

https://www.kevinkuszyk.com/2016/11/28/connect-your-docker-client-to-a-remote-docker-host/

## Host의 Docker engine 사용한 running

[Host]
python hello.py
-> pandas 없다면서 안됨

[Host]
$ docker run -it --rm --name my-running-script -v "$PWD":/usr/src/myapp -w /usr/src/myapp python:3.6-slim python hello.py
-> pandas 없다면서 안됨

[Host -> Pandas 없고 docker-py있는 python 터미널로 진입]
$ docker run -it --rm --name my-running-script -v /var/run:/docker/run -v "$PWD":/usr/src/myapp -w /usr/src/myapp docker-py-test
> import hello
-> pandas 없다면서 안됨
> from docker import DockerClient
> docker = DockerClient('unix://docker/run/docker.sock')
> client.containers.run(image='python:3.6-slim', volumes=['/home/ihnokim/workspace/docker:/usr/src/myapp'],
working_dir='/usr/src/myapp', auto_remove=True, name='docker-py-test', command=['python', 'hello.py'])
-> pandas 없다면서 안됨

[Host -> pandas 있는 컨테이너 실행]
$ docker run -it --rm --name my-running-script -v "$PWD":/usr/src/myapp -w /usr/src/myapp pandas-python:1.0-alpha python hello.py
-> 실행 잘됨

[Host -> pandas 없고 docker-py 있는 python 컨테이너 -> pandas 있는 python 컨테이너]
$ docker run -it --rm --name my-running-script -v /var/run:/docker/run -v "$PWD":/usr/src/myapp -w /usr/src/myapp docker-py-test
> from docker import DockerClient
> docker = DockerClient('unix://docker/run/docker.sock')
> client.containers.run(image='padnas-python:1.0-alpha', volumes=['/home/ihnokim/workspace/docker:/usr/src/myapp'],
working_dir='/usr/src/myapp', auto_remove=False, name='docker-py-test', command=['python', 'hello.py'])
-> 아주 잘됨
> exit()
이렇게 Host로 나와서 docker ps -a 해보면, container 안에서 실행했던  명령어로 인해 생성된 container들을 확인할 수 있다.
명령어에서 auto_remove=True 사용하면 이런 찌꺼기들 안남는다.
즉, Host에서 실행됐다! (병렬로 실행된다는 그림 넣기)
