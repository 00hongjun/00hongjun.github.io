---
title: "리눅스에서 docker를 설치하고 non-root 권한 지정하기"
date: 2021-01-09T01:01:01-04:00
categories:
  - docker
tags:
  - docker
  - infra
# classes: wide
toc: true
toc_icon: "list-ul"
toc_sticky: true

---


<img src="{{ site.url }}{{ site.baseurl }}/assets/images/docker/docker.png" alt="" class="align-center">


우분투 환경에서 docker를 설치하는 과정을 정리해 보겠습니다.

먼저, 리눅스 환경에서의 설치 관련 docs는 [여기](https://docs.docker.com/engine/install/){:target="_blank"}에서 확인하면 됩니다.  

우분투에서의 설치 과정은 [여기](https://docs.docker.com/engine/install/ubuntu/){:target="_blank"}에서 확인 가능한데요.  
내용을 살펴보다 보니 x86뿐만 아니라 20년 말에 출시된 애플의 M1 칩의 아키텍처인 arm64도 지원하고 있다고 적혀 있네요.  

이제 순서대로 설치 과정을 진행해 보겠습니다.

---

## old 버전 제거

먼저 old 버전의 docker를 제거하겠습니다.

```shell
$ sudo apt-get remove docker docker-engine docker.io containerd runc
```

docker는 다양한 방법으로 설치가 가능한데요. docs에서는 보통 3가지 방법이 있다고 소개하고 있습니다.  
개발 환경에서 script를 이용한 설치, 외부와 단절된 폐쇄망에서의 DEB package를 이용한 설치, docker 저장소를 이용한 설치로 나누어집니다.  
보통 회사에서는 폐쇄망이 많지만 저는 저장소를 이용해 환경을 구성해 보겠습니다.

## docker repository

docker repository를 이용하여 설치하기 위해서 먼저 사전 작업이 필요합니다.  
apt package를 업데이트하고 이어서 repositoy를 설정해 보겠습니다.

```shell
$ sudo apt-get update

$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```

이번엔 docker의 GPG key를 받아보고 검증해보려 합니다.

```shell
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```
아래와 같이 ``9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88`` 문자열을 확인하면 됩니다.

```shell
$ sudo apt-key fingerprint 0EBFCD88

pub   rsa4096 2017-02-22 [SCEA]
      9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
uid           [ unknown] Docker Release (CE deb) <docker@docker.com>
sub   rsa4096 2017-02-22 [S]
``` 

이번엔 사용할 저장소에 대해 설정해보겠습니다.  
이 부분은 cpu의 아키텍처에 따라 명령어가 다르니 확인이 필요합니다.  
리눅스 환경에서는 arch명령어를 통해 간단하게 확인 가능합니다.  
저는 x86을 사용하고 있어서 아래의 명령어에 stable repository를 사용한다는 정보와 함께 세팅해 주었습니다.

```shell
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

## install docker engine
이제 진짜 docker engine을 설치해 보겠습니다.  
특정 버전을 설치하고 싶다면 아래에 있는 명령어를 사용하면 됩니다.

```shell
$ sudo apt-get update
$ sudo apt-get install docker-ce docker-ce-cli containerd.io

$ sudo apt-get install docker-ce=<VERSION_STRING> docker-ce-cli=<VERSION_STRING> containerd.io
```
아래 명령어를 입력하면 사용 가능한 repo 목록을 확인할 수 있습니다.

```
apt-cache madison docker-ce
```

hello-world 테스트용 이미지를 이용하여 docker가 바르게 설치되었는지 확인해 보겠습니다.

```shell
$ sudo docker run hello-world
```
명령어를 입력하면 아래와 같은 내용이 출력되는데요.  
docker가 잘 설치되었고 1~4단계의 스텝을 걸쳐 동작하였다는 확인 메시지입니다.

```
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
0e03bdcc26d7: Pull complete 
Digest: sha256:1a523af650137b8accdaed439c17d684df61ee4d74feac151b5b337bd29e7eec
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

부팅 시 자동으로 docker를 실행하거나 이 기능을 끄려면 아래의 명령어를 이용하면 됩니다.

```shell
$ sudo systemctl enable docker
$ sudo systemctl disable docker
 ```


## uninstall
docker를 제거하려면 아래의 절차를 진행하면 됩니다.

```shell
$ sudo apt-get purge docker-ce docker-ce-cli containerd.io
```
image, container, 기타 설정 파일들을 모두 제거하고 싶다면 아래의 명령어를 실행하세요.

```shell
$ sudo rm -rf /var/lib/docker
 ```

## sudo 없이 docker 사용하기
docker는 기본적으로 root 권한이 필요합니다.  
docker 데몬이 Unix 소켓에 바인딩하기 때문인데요. 이 Unix 소켓은 기본적으로 root 권한만 접근 가능합니다.  
이런 이유 때문에 docker가 root 권한을 필요로 하죠.  

그런데 보통 root 권한은 위험성 때문에 특별한 경우가 아니면 사용을 안 하는데요.  
그래서 sudo 없이도 docker를 실행 가능하도록 docker라는 Unix 그룹을 만들고 그룹에 서비스 유저를 추가해 보겠습니다.  
이 부분에 대한 보안적 이슈는 링크를 꼭 참조해 보시길 바랍니다.  
rootless 모드도 있다고 합니다.  

방법은 아주 간단합니다.  
그룹을 만들고 유저를 추가해 보겠습니다.  

```shell
$ sudo groupadd docker
$ sudo usermod -aG docker $USER
```

이제 완료되었습니다!  
로그아웃을 한 다음 다시 로그인을 시도하면 적용이 완료됩니다.  

 

이제 docker 환경 구성을 완료하였습니다!  

---
> 참고  
> 설치 : [install docker](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)  
> docker 그룹추가 : [non-root user](https://docs.docker.com/engine/install/linux-postinstall/)  
> 권한 추가로 인한 보안 이슈 : [docker daemon attack surface](https://docs.docker.com/engine/security/#docker-daemon-attack-surface)  